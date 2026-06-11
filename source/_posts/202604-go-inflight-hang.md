---
title: 'Go channel request-response 패턴에서 내가 저지른 hang 버그 3가지'
date: 2026-04-20
description: '채널 등록 순서와 seq 키 처리를 잘못 짠 제 코드 때문에 대기 고루틴이 영원히 깨어나지 못하던 hang 버그를 수정한 과정이에요. Go의 문제가 아니라 흔한 구현 실수였어요.'
categories: [Go, Backend]
tags: [Go, Golang, Channel, Concurrency, Hang, Timeout, FEP]
keywords: ['Go channel hang', 'goroutine 대기', 'in-flight', 'DAOK ERCD', 'atomic']
index_img: /img/2026-go/go-hang.png
---

개발 중인 애플리케이션 내부에서 DR 환경 테스트 중 특정 조건에서 HTTP 요청이 30초 타임아웃까지 hang되는 버그가 발견됐어요. 처음엔 "Go channel이 원래 이런가?" 싶었는데, 파보니 Go나 채널 자체의 문제가 전혀 아니었어요. 채널을 다루는 제 코드에 있던 사소한 실수 세 가지가 원인이었어요.

## 버그 1: 응답이 등록보다 먼저 도착하는 race

DATA를 전송하고 DAOK(응답 확인) 채널을 등록하는 순서가 문제였어요.

```go
// before: Send 후에 채널 등록
if err := rq.reqClient.Send(data); err != nil {
    return err
}
// 서버가 매우 빠르게 응답하면 이미 이 시점에 ResponseDAOK가 불림
resp, err := rq.waitForResponse(seq, daokTimeout) // 채널 생성 + 대기
```

`waitForResponse` 내부에서 채널을 만들고 map에 등록하는데, 서버가 Send 직후 DAOK/ERCD를 응답하면 등록 전이라 notify가 유실되고, 결국 `daokTimeout`(최대 30초)까지 대기해요.

수정: **Send 전에 채널을 등록**해요.

```go
// after: Send 전에 채널 등록
ch := rq.registerPending(seq)       // 먼저 등록
atomic.StoreUint32(&rq.inFlightDataSeq, seq)
defer func() {
    atomic.StoreUint32(&rq.inFlightDataSeq, 0)
    rq.unregisterPending(seq)
}()

if err := rq.reqClient.Send(data); err != nil {
    return err
}
resp, err := rq.awaitPending(ctx, ch, daokTimeout) // 이미 등록된 채널 대기
```

## 버그 2: ERCD seq 불일치로 대기자를 못 깨우는 문제

서버에서 `ERCD 13`(DATA 내용 오류) 같은 일부 에러를 DATA와 **다른 seq(혹은 0)** 로 응답해요. 기존 코드는 `notifyPendingResponse(frame.Sequence(), cf)`로 seq를 키로 대기자를 찾는데, seq가 달라서 대기자를 찾지 못하고 넘어가요. 결과적으로 `RequestData()`의 DAOK 대기자가 timeout까지 hang돼요.

수정: ERCD seq로 대기자를 못 찾으면 현재 in-flight DATA seq로 폴백해요.

```go
func (rq *RQProtocol) ResponseERCD(frame *message.Frame, ...) {
    // ...
    // 1차: ERCD 자체 seq로 대기자 탐색
    if !rq.notifyPendingResponse(frame.Header.Sequence(), cf) {
        // 2차 폴백: 현재 송신 중인 DATA seq로 대기자를 깨움
        if dataSeq := atomic.LoadUint32(&rq.inFlightDataSeq); dataSeq != 0 {
            rq.notifyPendingResponse(dataSeq, cf)
        }
    }
}
```

`inFlightDataSeq`는 `atomic.Uint32`로 관리해요. DATA 전송 시작 시 저장하고, 완료(성공/실패) 시 0으로 초기화해요.

## 버그 3: HTTP ctx 미전파로 인한 불필요한 대기

HTTP 요청이 클라이언트 타임아웃으로 이미 끊겼는데, DAOK 대기가 최대 30초 더 진행되던 문제예요. `ctx`를 `RequestData`까지 전파하고, `awaitPending`에서 `ctx.Done()`을 함께 감시해요.

```go
func (rq *RQProtocol) awaitPending(
    ctx context.Context,
    ch chan *message.ControlFrame,
    timeout time.Duration,
) (*message.ControlFrame, error) {
    select {
    case resp, ok := <-ch:
        if !ok {
            return nil, errors.New("connection lost")
        }
        return resp, nil
    case <-time.After(timeout):
        return nil, errors.New("response timeout")
    case <-ctx.Done():
        return nil, ctx.Err()   // HTTP ctx 종료 시 즉시 반환
    case <-rq.shutdownCh:
        return nil, errors.New("shutdown")
    }
}
```

## 정리

| 버그 | 원인 | 수정 |
|---|---|---|
| notify 유실 | Send 후 채널 등록 | Send 전에 registerPending |
| ERCD hang | seq 불일치로 대기자 못 찾음 | inFlightDataSeq 폴백 |
| 불필요한 30초 대기 | ctx 미전파 | awaitPending에 ctx.Done() 추가 |

세 버그 모두 언어 탓이 아니라 채널을 쓰는 쪽에서 흔히 저지를 수 있는 구현 실수였어요. Go channel 기반 request-response 패턴에서 "등록은 항상 전송 전에", "대기 키는 응답 경로와 반드시 일치", "ctx는 끝까지 전파" — 이 세 가지만 점검해도 비슷한 hang은 피할 수 있어요.

오늘의 코딩 메모 끝~

제목을 좀 더 직관적으로 다듬었어요. 내용 자체는 그대로예요.
다시 읽어보니 도입부에서 원인을 미리 요약해두면 좋을 것 같아 메모해둬요.
버그 재현 조건을 한 줄로 요약하면 '응답이 등록보다 빠를 때'예요.
---
`eod`