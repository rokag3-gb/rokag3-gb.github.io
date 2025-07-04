---
title: 'ORM의 특성을 비교해보자'
date: 2022-01-21
description: '.NET에서 사용하는 ORM(Object-Relational Mapping) 도구들의 성능과 특성을 비교 분석합니다. Entity Framework, Dapper, ADO.NET의 장단점과 실제 성능 테스트 결과를 .NET Conf 2022 x Seoul 발표 내용을 통해 알아봅니다.'
categories: [Dev]
tags: [ORM, Object-Relational Mapping, Dapper, Entity Framework, ADO.NET, .NET, 성능비교, 데이터베이스, Microsoft, 개발도구]
index_img: img/dot-net-conf-2022-orm-2022-01/cover.png
keywords: [ORM, Dapper, Entity Framework, ADO.NET, .NET, 성능비교, 데이터베이스]
---

# 온라인 컨퍼런스

2021년 1월에 이어 연속으로 2022년 1월에도 .NET Conf Seoul에서 "ORM의 특성을 비교해보자"라는 제목으로 세션 발표에 참여할 수 있는 기회를 얻게 되었습니다. 온라인 컨퍼런스이다 보니 미리 준비하고 사전에 영상을 녹화해놓는 것이라 크게 부담도 없습니다.

# 발표 내용 요약

널리 사용하고 있는 Entity Framework, Dapper 뿐만 아니라 전통적인 ADO.NET 까지 장단점을 비교해보려 합니다.

![](img/dot-net-conf-2022-orm-2022-01/p00.png)

경량 ORM으로서 Dapper의 간결함과 가벼움을 잠시 설명하고 있습니다.

![](img/dot-net-conf-2022-orm-2022-01/p09.png)

이번 세션을 준비하면서 핵심 장표라고 할 수 있는데요. ADO.NET, Entity Framework, Dapper에 대해서 몇가지 특징에 대해서 설명하고 있습니다.

![](img/dot-net-conf-2022-orm-2022-01/p11.png)

![](img/dot-net-conf-2022-orm-2022-01/p12.png)

동일한 쿼리를 3가지 방식 (ADO.NET, Entity Framework, Dapper)으로 여러번 (max. 3,000번) 작동시켜 dutation을 꺽은선 그래프로 비교해본 것 입니다. 테스트 환경, 쿼리에 따라 다르겠지만 Dapper가 가장 짧은 Duration을 기록하였습니다.

![](img/dot-net-conf-2022-orm-2022-01/p31.png)

# Youtube 발표 영상

<!-- {% video <iframe width="560" height="315" src="https://www.youtube.com/embed/42fRHSOQwZ0" title=".Net Conf 2022 x Seoul 발표 자료 - ORM의 특성을 비교해보자" frameborder="0" allow="accelerometer; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe> %} -->

{% raw %}
<div class="video-embed">
  <iframe width="100%" height="480" src="https://www.youtube.com/embed/42fRHSOQwZ0" title="ORM의 특성을 비교해보자 - .NET Conf 2022 x Seoul 발표" frameborder="0" allow="accelerometer; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
</div>
{% endraw %}

[Youtube - ORM의 특성을 비교해보자 - .NET Conf 2022 x Seoul 발표](https://youtu.be/42fRHSOQwZ0)