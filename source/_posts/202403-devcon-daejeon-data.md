---
title: '스타트업에서 데이터를 허투로 버리지 않고 소중히 모시는 방법'
date: 2024-03-23
description: '데브콘 대전 "삼월엔 데이터" 컨퍼런스 발표'
categories: [Story]
tags: [.NET, .NET 8, Ubuntu, Chiseled Ubuntu, 데브콘, 데브콘 대전, Devcon, Devcon Daejeon]
index_img: img/202403-devcon-dj/event-cover.png
---

# 데브콘 대전 : 삼월엔 데이터

![](img/202403-devcon-dj/event-cover.png)

커뮤니티 운영진 한분과의 인연으로 "스타트업에서 데이터를 허투로 버리지 않고 소중히 모시는 방법"이라는 제목으로 주니어 분들 앞에서 제가 해왔던 기술 이야기를 조금이나마 나눌 수 있는 즐거운 시간이었어요.

[데브콘 대전 : 삼월엔 데이터 | Festa! - 페스타](https://festa.io/events/4836)

![대전 최대 아웃풋 is 성심당](img/202403-devcon-dj/4327682590718.jpg)

# Data Ingestion, Data Analysis, Visualization

<!--![](img/202403-devcon-dj/pt-20.png)-->

![](img/202403-devcon-dj/pt-21.png)

![](img/202403-devcon-dj/pt-23.png)

![](img/202403-devcon-dj/pt-25.png)

# Observability 관측성

[관측 가능성 - 위키백과](https://ko.wikipedia.org/wiki/%EA%B4%80%EC%B8%A1_%EA%B0%80%EB%8A%A5%EC%84%B1)

우리가 개발한 서비스의 관측성이 매우 절실해진 상황이에요.

뭐 이것저것 만들었는데, VM도 몇개 있고 PaaS도 몇개 있다보니, 장애포인트를 찾아내는 데에 소요되는 시간이 점차 늘어났어요.

![](img/202403-devcon-dj/pt-27.png)

로그 수집은 한곳에서 하기 위해서, 모든 클라이언트가 바라보는 endpoint인 API Gateway 서버에 fluentd를 설치하기로 했어요.

👉 Fluentd는 데이터 수집 OSS 프로젝트에요. [https://docs.fluentd.org/](https://docs.fluentd.org/)

![](img/202403-devcon-dj/pt-34.png)

fluentd 통해서 모든 로그 데이터를 Azure Blob 으로 전송하도록 했고 Azure Blob에 파일을 읽어서 DBMS에서 비정형(JSON) 형태로 저장하도록 했어요.

# Grafana 시각화

Grafana에서는 DBMS에 쿼리하여 JSON 데이터를 시각화하도록 대시보드를 개발했구요. 이를 통해서 어느 도메인, 어느 서비스, 어느 호스트, 어느 API인지 더욱 빠르게 찾아낼 수 있게 되었고, 구체적으로 언제부터 언제까지 에러(http status code 4** ~ 5**)가 발생했는지에 대해서도 빠르게 파악할 수 있게 되었어요.

![Grafana 대시보드 캡처](img/202403-devcon-dj/pt-40.png)

<!--![컨퍼런스 시작 2시간 전](img/202403-devcon-dj/KakaoTalk-20241002-093250512-(7).jpg)-->

# Conclusion

신규 구축하게 된 SaaS의 여러 호스트 마다의 logging은 서비스 확장 시 관측성 측면에서 매우 중요하며, 그중에서 모든 http log를 수집, 가공, 시각화하여 서비스의 상태를 최대한 빠르게 알아낼 수 있도록 했다는 내용이 핵심이에요.

![성욱님과 사진 한장ㅋ](img/202403-devcon-dj/91785.jpg)

# 발표 자료

발표 자료
[스타트업에서 데이터를 허투로 버리지 않고 소중히 모시는 방법 - 2024.03.20 김정우.pdf](https://drive.google.com/file/d/1crx4cqKEC2C3ZcXd9NB5TmYA1Ei1JA4B/view?usp=sharing)

발표 영상
{% raw %}
<div class="video-embed" style="text-align: center;">
  <iframe width="100%" height="480" src="https://www.youtube.com/embed/2Ezz2saXGfY" title="스타트업에서 데이터를 허투로 버리지 않고 소중히 모시는 방법 - 데브콘 대전 삼월엔 데이터 컨퍼런스" frameborder="0" allow="accelerometer; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
</div>
{% endraw %}

[Youtube 발표 영상 - 스타트업에서 데이터를 허투로 버리지 않고 소중히 모시는 방법 - 데브콘 대전 "삼월엔 데이터" 컨퍼런스](https://youtu.be/2Ezz2saXGfY)

<br>

<!--![성욱님은 Proxy SQL 발표 중](img/202403-devcon-dj/KakaoTalk-20241002-093250512-(13).jpg)-->

![패널 토크 중 잠시 주마등이 스쳐지나감](img/202403-devcon-dj/KakaoTalk-20241002-093250512-(17).jpg)
