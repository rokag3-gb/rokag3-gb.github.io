---
title: '서버 내부에서 발생한 Race Condition 2가지 수정'
date: 2026-03-25
description: 'TCP 재연결 상황에서 새 커넥션을 잘못 닫는 race와, 동시 전송 시 메시지 프레임이 섞이는 race를 수정한 과정을 정리했습니다.'
categories: [Go, Backend]
tags: [Go, Golang, TCP, RaceCondition, Mutex, Concurrency]
keywords: ['Go race condition', 'TCP mutex', 'goroutine', '동시성', 'sync.Mutex']
index_img: /img/2026-go/go-race-condition.png
---

Go로 서버를 개발하는 과정에서 서버 내부에 TCP 클라이언트 구조체(`rq_client.go`, `rd_client.go`)에서 두 가지 race condition을 발견해서 수정함. 나중에 비슷한 경험을 할 수 있어서 기억하려고 글로 남겨놓음.

## 문제 1: 재연결 후 새 커넥션을 닫는 race

`handleDisconnect()`는 연결이 끊기면 `c.conn`을 닫고 nil로 초기화하는 함수. 재연결이 완료되어 이미 새 커넥션이 할당된 상태에서, 이전 커넥션의 readLoop 고루틴이 뒤늦게 `handleDisconnect()`를 호출하면 새 커넥션까지 닫아버리는 문제가 있었음.

**race가 터지는 타이밍:**

```
[고루틴 A - 구 readLoop]     [고루틴 B - reconnect]
ReadFrame() → 에러 감지
                              connect() 성공 → c.conn = newConn
handleDisconnect() 호출
  c.conn != nil 확인 → true  ← 이미 newConn이 들어있는 상태
  c.conn.Close()             ← newConn을 닫아버림!
  c.conn = nil
```

재연결과 이전 readLoop 고루틴의 종료 타이밍이 겹칠 때 발생함. 네트워크가 불안정해서 재연결이 빠르게 이루어질수록 재현 확률이 높아짐.

```go
// before: 현재 커넥션이 누구든 무조건 닫음
func (c *RQClient) handleDisconnect() {
    c.mu.Lock()
    if c.conn != nil {
        _ = c.conn.Close()
        c.conn = nil
    }
    c.mu.Unlock()
}
```

수정: 고루틴이 읽던 커넥션(`oldConn`)을 파라미터로 받아서, 현재 `c.conn`과 동일할 때만 정리함.

```go
// after: oldConn이 현재 커넥션과 다르면 no-op
func (c *RQClient) handleDisconnect(oldConn net.Conn) {
    c.mu.Lock()
    if c.conn == nil || c.conn != oldConn {
        c.mu.Unlock()
        return
    }
    _ = c.conn.Close()
    c.conn = nil
    atomic.StoreInt32(&c.connected, 0)
    c.reader.Reset()
    c.mu.Unlock()
    // ...
}
```

호출부에서는 readLoop 시작 시점의 커넥션을 캡처해서 넘기면 됨.

```go
func (c *RQClient) readLoop() {
    conn := c.conn  // 이 고루틴이 담당하는 커넥션
    for {
        _, err := c.reader.ReadFrame()
        if err != nil {
            c.handleDisconnect(conn)  // 자기 커넥션만 정리
            continue
        }
        // ...
    }
}
```

## 문제 2: 동시 전송 시 프레임 interleaving

`Send()`에서 뮤텍스를 잡고 커넥션 포인터만 복사한 뒤 락을 풀고 `Write()`를 호출하고 있었음. 두 고루틴이 동시에 `Send()`를 호출하면 Write가 동시에 실행되어 TCP 스트림에 프레임이 섞일 수 있음.

**race가 터지는 타이밍:**

```
[고루틴 A]                   [고루틴 B]
mu.Lock() → conn 복사
mu.Unlock()
                             mu.Lock() → conn 복사
                             mu.Unlock()
conn.Write(frameA 앞부분)
                             conn.Write(frameB 전체)   ← 중간에 끼어듦
conn.Write(frameA 뒷부분)
```

TCP는 스트림 프로토콜이라 수신 측에서 프레임 경계를 직접 파싱함. 프레임이 섞이면 파싱이 깨져 이후 모든 요청이 오동작하게 됨. 동시 요청이 많을수록 재현 확률이 높아짐.

```go
// before: 락 해제 후 Write → 동시 Write 가능
func (c *RQClient) Send(data []byte) error {
    c.mu.Lock()
    conn := c.conn
    c.mu.Unlock()       // 여기서 락 해제
    _, err := conn.Write(data)  // 동시 실행 가능
    // ...
}
```

Write까지 락 보호 구간에 포함시키면 해결됨.

```go
// after: Write까지 뮤텍스로 보호
func (c *RQClient) Send(data []byte) error {
    c.mu.Lock()
    defer c.mu.Unlock()

    if c.conn == nil {
        return errors.New("not connected")
    }
    _, err := c.conn.Write(data)
    if err != nil {
        _ = c.conn.Close()
        c.conn = nil
        atomic.StoreInt32(&c.connected, 0)
        c.reader.Reset()
        return err
    }
    return nil
}
```

## 정리

| 문제 | 원인 | 수정 |
|---|---|---|
| 새 커넥션이 닫힘 | handleDisconnect가 현재 커넥션 무조건 닫음 | oldConn 파라미터로 동일성 확인 |
| 프레임 섞임 | Send에서 Write를 락 밖에서 실행 | defer c.mu.Unlock()으로 Write까지 보호 |

`go test -race`로 race detector를 돌리면 이런 문제를 사전에 잡을 수 있음. 테스트 코드 작성도 함께 병행함.
