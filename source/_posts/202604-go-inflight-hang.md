---
title: 'Go channel을 잘못 다뤄 in-flight 요청이 hang된 버그 수정'
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

추가로, 이런 hang 버그는 단위 테스트로 재현하기 까다로워서 결국 통합 테스트에서 잡혔어요.
pprof의 goroutine 덤프를 보면 대기 중인 고루틴이 어느 select에 멈춰 있는지 바로 보여서 디버깅에 큰 도움이 됐어요.
registerPending과 unregisterPending은 같은 뮤텍스로 보호해서 map 동시 접근 race를 막았어요.
오늘은 비가 와서 그런지 커밋할 맛이 안 나네요. 그래도 메모는 남겨둬요.
awaitPending의 select 분기 순서는 동작에 영향이 없지만, 가독성을 위해 정상 경로를 맨 위에 뒀어요.
time.After는 타이머를 매번 새로 만들기 때문에 호출이 잦으면 time.NewTimer + Stop 패턴이 더 효율적이에요.
주말이라 잠깐 코드를 다시 들여다봤는데, 역시 등록 순서가 핵심이라는 생각이 드네요.
비슷한 패턴을 다른 프로토콜 핸들러에도 적용해뒀어요. 한 곳에서 검증되니 재사용이 편했어요.
context.WithTimeout으로 상위에서 데드라인을 걸면 ctx.Done() 전파만으로도 자연스럽게 정리돼요.
atomic.Uint32 대신 sync.Mutex로도 가능하지만, 단일 값이라 atomic이 더 간결했어요.
오늘 점심은 김치찌개. 코드 리뷰 전에 든든하게 먹어둬요.
notifyPendingResponse는 채널이 이미 닫혔는지 확인 후 보내야 panic을 피할 수 있어요.
inFlightDataSeq를 0으로 초기화하는 시점을 defer에 둔 게 누락 방지에 핵심이었어요.
go test -race로 돌려보니 추가로 잡히는 race는 없었어요. 다행이에요.
request-response 매핑은 결국 '키를 어떻게 잡느냐'의 문제라는 걸 다시 느꼈어요.
DR 환경은 네트워크 지연이 커서 race가 더 잘 드러나요. 테스트 환경으로는 오히려 유용했어요.
select에 default를 넣으면 non-blocking이 되니, 의도와 다르면 조심해야 해요.
오늘은 회의가 많아서 코드는 조금밖에 못 봤어요. 메모만 남겨요.
채널 버퍼 크기를 1로 둘지 0으로 둘지도 고민했는데, 여기선 동기 전달이 맞아서 0으로 했어요.
타임아웃 값은 설정으로 빼서 환경별로 조정 가능하게 했어요.
goroutine 누수는 pprof의 goroutine 수 추이로 모니터링하고 있어요.
에러 메시지에 seq를 같이 남기니 로그만 봐도 어느 요청이 hang됐는지 추적이 쉬웠어요.
정리해두고 보니 세 버그 모두 '타이밍'이 공통 키워드네요.
비슷한 실수를 막으려고 팀 위키에 체크리스트로 옮겨 적었어요.
오늘 날씨 맑음. 창밖 보면서 잠깐 머리 식히고 다시 집중해요.
ctx 전파는 함수 시그니처를 바꿔야 해서 번거롭지만, 길게 보면 꼭 해두는 게 좋아요.
재현 테스트를 하나 추가해서 회귀를 막아뒀어요.
shutdownCh 분기를 빼먹으면 종료 시 hang되니, 이것도 같이 챙겨야 해요.
이 시리즈는 여기서 마무리. 다음엔 다른 주제로 돌아올게요.
---
`eod`