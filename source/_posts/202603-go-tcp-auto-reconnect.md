---
title: 'Exponential Backoff 방식의 TCP 소켓 자동 재연결 구현'
date: 2026-03-30
description: '서버에서 TCP 소켓이 끊겼을 때 Exponential Backoff로 자동 재연결하는 로직을 구현한 과정을 정리했음.'
categories: [Go, Backend]
tags: [Go, Golang, TCP, Reconnect, ExponentialBackoff, Concurrency]
keywords: ['Go TCP reconnect', 'exponential backoff', '자동 재연결', 'goroutine', 'channel']
index_img: /img/2026-go/tcp-socket-reconnect.png
---

TCP 소켓을 장기간 유지해야 하는 서버를 만들고 있었음. 소켓이 끊겼을 때 수동 재시작 없이 자동으로 재연결되도록 Exponential Backoff 기반 재연결 루프를 구현함.

## 구조

재연결 루프는 별도 고루틴으로 실행하고, 연결 끊김 이벤트를 채널로 수신하는 구조임.

```go
type RQProtocol struct {
    reconnectCh  chan struct{}     // 재연결 트리거 채널
    reconnecting atomic.Bool      // 재연결 진행 중 여부
    config       RQConfig
}

type RQConfig struct {
    ReconnectInitialDelay time.Duration // 첫 재시도 대기 (기본 5s)
    ReconnectMaxDelay     time.Duration // 최대 대기 간격 (기본 60s)
}
```

## StartReconnectLoop

```go
func (rq *RQProtocol) StartReconnectLoop() {
    go func() {
        delay := rq.config.ReconnectInitialDelay
        for {
            select {
            case <-rq.reconnectCh:
                rq.reconnecting.Store(true)
            case <-rq.shutdownCh:
                return
            }

            for {
                rq.log.Info("재연결 시도", zap.Duration("delay", delay))
                time.Sleep(delay)

                if err := rq.connect(); err != nil {
                    // 실패 시 delay를 두 배로, 최대값 초과하지 않게
                    delay = min(delay*2, rq.config.ReconnectMaxDelay)
                    continue
                }

                // 연결 성공 → 핸드셰이크
                if err := rq.PerformHandshake(); err != nil {
                    rq.log.Warn("핸드셰이크 실패, 재시도", zap.Error(err))
                    delay = min(delay*2, rq.config.ReconnectMaxDelay)
                    continue
                }

                // 성공 → delay 초기화
                delay = rq.config.ReconnectInitialDelay
                rq.reconnecting.Store(false)
                break
            }
        }
    }()
}
```

## TriggerReconnect

연결 끊김을 감지한 곳(readLoop, Send 실패 등)에서 호출함. 채널이 non-blocking이라 이미 재연결 중이면 중복 트리거는 무시됨.

```go
func (rq *RQProtocol) TriggerReconnect() {
    select {
    case rq.reconnectCh <- struct{}{}:
    default: // 이미 재연결 대기 중이면 skip
    }
}
```

## 초기 연결 실패 처리

서버 시작 시 상대 서버가 아직 안 떠있을 수 있음. 초기 연결 실패도 재연결 루프에서 처리하도록 했음.

```go
// main.go
rqProtocol.StartReconnectLoop()

if err := rqProtocol.Start(); err != nil {
    log.Warn("초기 TCP 연결 실패, 재연결 루프에서 재시도", zap.Error(err))
    rqProtocol.StartReadLoops()
    rqProtocol.TriggerReconnect()
} else {
    if err := rqProtocol.PerformHandshake(); err != nil {
        rqProtocol.TriggerReconnect()
    }
}
```

## Health API에 재연결 상태 노출

```go
type HealthResponse struct {
    Status          string `json:"status"`
    PbSessionState  string `json:"pb_session_state"`
    PbReconnecting  bool   `json:"pb_reconnecting"`
}
```

`pb_reconnecting: true`가 보이면 재연결 중이라는 의미임. 모니터링에서 이 필드를 감시하면 소켓 단절 시 즉시 파악 가능함.

## 동작 흐름

```
TCP 끊김 감지
    → TriggerReconnect()
    → 5초 대기 후 connect() 시도
    → 실패 시 10초, 20초, 40초... (최대 60초)
    → 성공 시 PerformHandshake()
    → 세션 복구 완료, delay 5초로 초기화
```

재연결 성공 후 pending 중이던 요청들은 timeout으로 빠져나오고, 클라이언트에서 재시도하는 구조임.

오늘의 개발 메모 끝~

delay에 jitter(무작위 오프셋)를 추가하면 여러 인스턴스가 동시에 재연결을 시도하는 thundering herd 문제를 완화할 수 있음.
reconnectCh를 버퍼 없는 채널 + select default로 구성한 이유는 재연결이 이미 진행 중일 때 중복 신호를 드롭하기 위함임.
reconnecting 필드를 sync.Mutex 대신 atomic.Bool로 선택한 이유는 단순 플래그 읽기/쓰기에 뮤텍스 오버헤드가 불필요하기 때문임.
---
`eod`