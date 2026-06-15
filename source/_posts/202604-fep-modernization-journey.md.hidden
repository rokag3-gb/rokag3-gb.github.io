---
title: 'NEXT BUILDER 첫번째 컨퍼런스 발표를 마치며'
date: 2026-06-15
mermaid: true
description: '사내 첫 테크 컨퍼런스 NEXT BUILDER에서 "FEP Modernization Journey"를 발표하기까지, 신규 프로덕트의 FEP/Gateway를 Go언어로 백지에서 설계한 5개월의 기록. 평균이 아니라 꼬리지연시간(tail latency)을 기준으로 "충분히 빠르다"를 정의·측정하고, 레퍼런스가 없는 자리를 측정으로 메워간 과정을 정리했습니다.'
categories: [Backend, Architecture]
tags: [NEXT BUILDER, Go, Golang, FEP, Gateway, TCP, tail-latency, p99, GC, STW, 초저지연, 저지연, 증권, 매매시스템, ADR, Architecture Decision Record, 아키텍처, Observability, 관측성]
index_img: img/202604-fep/speaker-jw.png
keywords: ['FEP', 'tail latency', 'p99.99', 'Go gateway', 'GC STW', 'low latency trading', '꼬리지연시간', '주문 게이트웨이', 'Go vs Kotlin vs Rust', 'NEXT BUILDER']
---

# NEXT BUILDER 첫번째 컨퍼런스 발표를 마치며

지난 4월 16일 우리 회사에서 처음으로 사내 테크 컨퍼런스 **NEXT BUILDER** 를 개최했어요. 저는 운이 좋게도 *"FEP Modernization Journey"* 라는 제목으로 세션 발표를 할 수 있는 기회를 받게 되었어요. 신규 프로덕트의 External Layer에서 요구되는 성능 기준을 어떻게 측정했는지, 그리고 Go로 현대화한 FEP Gateway의 구조와 설계 고려사항을 공유할 수 있는 자리였어요.

2달 전에 컨퍼런스 발표했던 내용을 이제서야 올리게 되었네요.😅

슬라이드를 만들면서 자연스럽게 지난 다섯 달을 되돌아보게 됐었는데, 간략히 세션 내용 리뷰와 회고를 적어보았어요.

![4/16 NEXT BUILDER에서 발표하고 있는 모습](img/202604-fep/latency-measurement.jpg)

<br>

---

# 1. 왜 *Modern* FEP였나

회사는 신규 사업을 위한 프로덕트를 준비하고 있는데, 금융권에서 보통 이런 시스템은 몇가지의 영역으로 나뉘곤 해요.

- 사용자를 마주하는 **채널계**,
- 계좌·잔고·주문을 관장하는 **계정계**,
- 그리고 거래소·결제기관 등 과 외부 통신하는 **대외계**(External Layer)예요.

제가 맡은 건 이중에서 대외계, 특히 **국내주식 주문·체결**을 책임지는 **FEP**(Front-End Processor)라고 부르는 컴포넌트였어요.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'primaryColor':'#eef2ff','primaryBorderColor':'#6366f1','primaryTextColor':'#1e293b','lineColor':'#94a3b8','fontFamily':'inherit'}}}%%
flowchart LR
  B["Backend"]
  B --> C["FEP"]
  C -->|"byte</br>TCP"| D[("External</br>Brokerage</br>(KRX 등)")]
```

*"회사에 이미 FEP가 있지 않나?"* 싶지만, 네 맞아요. 기존에 영위하고 있는 비즈니스에 맞게 최적화되어 장기적으로 운영되고 있는 FEP가 있어요. 하지만 이번에 신규 사업을 위해서 모든 인프라, 서버, 클라이언트를 새로 만들고 있는 상황이라, FEP쪽도 유산을 물려받는 것이 아니라 **백지에서 새로 설계**하는 것을 선택했고, 주어진 시간은 **9개월**이였어요. (2026.01 ~ 09)

<br>

---

# 2. "초저지연"을 정의, 측정하고 개발스택을 결정한다

현재 개발하고 있는 Modern FEP의 본질은 결국 **JSON과 바이너리 사이의 번역가**예요. 모던 웹서비스의 JSON 요청을 대외기관에서 요구하는 전문 규격에 맞춰 고정길이 byte 전문으로 바꿔 보내고, 돌아온 응답을 다시 풀어내요. 그리고 이 `번역`은 주문이라는 거래의 **핫패스 한가운데**에 있어요. 여기서 생기는 모든 지연이 곧장 거래 품질에 안좋은 영향을 끼칠 수 있다는거죠.

그래서 가장 먼저 한 일은 코딩이 아니라 성능 기준을 정의하는 거였어요. "빠르게" 라는 추상적인 표현 말고 정량적인 기준이 필요하다고 생각했고, 핵심은 평균 지연시간이 아니라 **꼬리 지연시간**(tail latency)이었어요. 99%가 1ms 안에 끝나도 나머지 1%가 수백~수천 ms로 늦어지면, 그 1% 만큼의 사용자에게는 최악의 거래경험을 줄 수 있잖아요. 그래서 판단 기준을 `p99.9`·`p99.99` 같은 tail latency로 정의하고, 거기에 분명한 상한선을 숫자로 정해두고 시작했어요.

다음은 "어디서 잴 것인가"였어요. 요청이 들어온 순간(t0)부터 인코딩·전송·수신·디코딩을 거쳐 응답이 나가기까지(t4)를 측정점으로 잘게 쪼갰어요. 어느 구간이 느린지 보이지 않으면 최적화할 수도 없으니까요.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'primaryColor':'#eef2ff','primaryBorderColor':'#6366f1','primaryTextColor':'#1e293b','lineColor':'#94a3b8','fontFamily':'inherit','actorBkg':'#eef2ff','actorBorder':'#6366f1','actorTextColor':'#1e293b','signalColor':'#64748b','signalTextColor':'#334155','noteBkgColor':'#f1f5f9','noteBorderColor':'#cbd5e1','noteTextColor':'#334155'}}}%%
sequenceDiagram
    participant C as 백엔드
    participant G as FEP Gateway
    participant P as 외부기관
    C->>G: t0 · HTTP 요청 진입
    Note over G: t1 · 인코딩 시작 (Unmarshal: JSON to object)
    G->>P: t2 · 전송 완료
    P-->>G: t3 · 응답 수신 시작
    Note over G: t4 · 디코딩 완료 (Marshal: object to JSON)
    G-->>C: 응답 반환
```

이 기준을 확정한 이후에 이제는 *"무엇으로 만들 것인가"*를 물었고, 그 후보는 **Kotlin(JVM)·Go·Rust** 3가지 였어요.

3가지 언어로 똑같은 작동을하는 Gateway stub을 만들어 동일한 부하로 꼬리지연을 측정했어요. 평균은 셋 다 비슷했지만, **tail latency**에서 차이나 많이 났어요.
JVM 계열은 GC의 **Stop-The-World**(STW) 때문에 p99.99에서 눈에 띄게 높아졌고, Go는 안정적으로 짧게 형성되는 꼬리지연시간을 보여줬어요. GC가 아예 없는 Rust도 후보였지만, 우리 환경에서 가장 예측 가능한 꼬리지연을 보인 건 오히려 Rust보다 Go였어요.

![3가지 버전의 stub으로 구간별 지연시간을 히트맵으로 도식화](img/202604-fep/percentile-99.99-tail-latency.png)

1. Tail Latency 안정성
2. 중장기 아키텍처 진화
3. 조직·운영 리스크의 균형 (Rust 경력 개발자 채용 가능성⬇️, 개발자 온보딩 비용⬆️)

> 그렇게 해서 Go를 택하게 되었어요.

<br>

---

# 3. Go언어로 개발한 Modern FEP

언어가 정해진 뒤의 설계 원칙은 단순했어요. Modern FEP는 화려할 필요가 없고, **외부 기관으로 전문을 하나도 빼놓지 않고 정확히 전달**하는 것에 집중해야 한다고 생각했어요. 그래서 API도 자원을 표현하는 RESTful 대신, 전달이라는 본래의 역할에 맞게 단순하게 가져갔어요.

대신, 신경 써서 공을 들인 쪽은 따로 있었어요.

|No.|항목|
|--:|--|
|1|FEP는 부하분산기 뒤에서 Active-Active, Active-Standby 모두 운영 가능하도록 설계|
|2|시스템이 down되어도 재기동 때 다시 전송할 수 있는 recovery 기능 구현|
|3|백엔드와 FEP 사이에 Kafka를 통해서 백엔드는 주문요청 메시지를 발행하고, FEP는 메시지를 구독하는 형태로 구성|
|4|매일 새벽 특정 시간에 모든 자원을 반납하는 등 의 초기화 기동|
|5|Graceful shutdown으로 진행 중인 주문을 흘려보낼 수 있도록 함|
|6|MDC와 Open Telemetry를 채택하여 구조화된 로그와 메트릭을 통하여 관측성(Observability) 높이기|

일부 대외비들이 있어서 전부 다 글로 표현할 수 는 없지만 핵심적으로는 위와 같은 기능들에 많은 공을 들였어요.

<br>

---

# 4. 이제 시작이다

지금 이 Moden FEP는 외부 기관과의 연동을 책임지지만, 그건 **단기적인 모습**이라고 생각해요. 중장기적으로는 외부 의존을 줄이고 **자체 원장** 중심으로 옮겨가면서, FEP가 모든 전문 검증, 주문 멱등성 판단, 이벤트 발행까지 맡는 핵심 컴포넌트로 진화하는 그림을 그리고 있어요.

> 2026년 오늘의 선택이 앞으로 3~5년 뒤 우리 Modern FEP 전체의 모습을 결정할 수 있다고 생각해요.

![우리 헤드 Jay님의 발표 전경](img/202604-fep/craftmanship.png)

---

# 5. 회고 — 발표를 준비하며

발표 슬라이드를 만들면서 가장 많이 떠올린 건 기술이 아니라 지난 다섯 달의 막막함이었어요.

솔직히 저는 **금융 도메인 경험이 많이 부족한 상태**에서 이 FEP를 시작했어요. 그래서 가장 힘들었던 건 어떤 기술적 난제가 아니라, **의존할 레퍼런스가 없다는 것**이었어요. "원래 이렇게 한다"는 정답지가 없으니, 크고 작은 결정 하나하나가 맨땅에서 시작하는 느낌이었죠.

그런데 그 막막함이 오히려 한 가지를 분명하게 만들어줬어요. **레퍼런스가 없는 만큼, 감이 아니라 항상 정량적으로 측정해야 한다**는 거예요. 누가 "이게 맞다"고 말해줄 수 없다면, 직접 재서 데이터로 확인하는 수밖에 없으니까요.

그래서 NEXT BUILDER 무대에서도 "무엇을 만들었다"가 아니라 "**왜 그렇게 판단했는가**"를 이야기하고 싶었어요. 정답이 없는 자리에서 어떻게 기준을 세우고 측정으로 결정해 갔는지, 그게 같은 막막함을 지나는 동료에게 가장 나눌 만한 이야기라고 생각했거든요.

정말 감사하게도, NEXT BUILDER 발표자 투표 중 가장 많은 표를 얻어 1위 상품도 받고 꽃다발도 받아서 너무 기쁜 하루였어요.

![영롱한 꽃다발과 Blue Label 뱀띠 에디션](img/202604-fep/awards.png)

---
`eod`