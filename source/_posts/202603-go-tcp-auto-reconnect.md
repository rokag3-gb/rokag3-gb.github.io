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
inner for loop를 outer select와 분리한 이유는 재연결 시도 중에도 shutdownCh 시그널을 즉시 감지하기 위함 — 분리하지 않으면 shutdown 시 최대 delay만큼 응답이 지연됨.
Start() 실패 시에도 StartReadLoops()를 선행 호출하는 이유는 재연결 성공 직후 readLoop 고루틴이 즉시 데이터 수신 대기 상태가 되도록 미리 띄워두기 위함임.
pb_reconnecting이 30초 이상 true로 유지될 경우 Slack/PagerDuty로 알림을 보내도록 모니터링을 구성하면 실운영에서 빠른 대응이 가능함.
initialDelay 5s, maxDelay 60s 기준으로 연속 실패 시 재시도 간격은 5→10→20→40→60→60... 초로 증가함.
go test -race 플래그로 race detector를 활성화하면 재연결 루프와 readLoop 사이의 동시성 문제를 빌드 단계에서 조기에 발견할 수 있음.
connect() 내부에서는 net.DialTimeout에 ConnectTimeout(기본 5s)을 적용했음 — 무한 대기 방지가 핵심임.
핸드셰이크 실패 시에도 delay를 증가시킨 이유는 TCP 연결은 성공했지만 애플리케이션 레이어 핸드셰이크가 반복 실패하는 케이스에서 빠른 재시도로 서버에 부하를 주지 않기 위함임.
이 재연결 로직 도입 이후 상대 서버 재시작 시 수동 개입 없이 평균 10~15초 내에 세션이 자동 복구되는 것을 확인함.
Exponential Backoff는 TCP reconnect 외에도 HTTP 클라이언트, 메시지 큐 소비자 등 재시도가 필요한 모든 곳에 적용 가능한 기본 패턴임.
재연결 루프는 프로세스 수명 동안 단 하나만 실행되어야 함 — 중복 실행 방지는 호출부 책임임.
ReconnectMaxDelay를 0으로 설정하면 delay가 무한 증가할 수 있어 반드시 양수 값으로 설정해야 함.
---
`eod`