---
title: 'Azure SQL database의 Geo Replica와 Managed Instance의 Failover group을 통한 CQRS 구현'
date: 2023-03-17
description: SQL PASS Korea 세미나 발표
categories: [DB]
tags: [Azure SQL, Azure SQL single database, SQL Managed Instance, Managed Instance, Geo Replica, Failover, Failover Group, FOG, PASS Korea, readonly listener]
index_img: img/sql-pass-korea-seminar-2023-03/KakaoTalk_20230917_205807945_11.jpg
---

# 도입

2023년2월 경 시퀄로 김정선 이사님께 내가 "발표할 일 있음 연락주세요" 말씀 드렸다가 막상 "발표해보시겠어요?" 제안 받으니까 걱정되고 두렵다. 비록 나는 DB 전문가는 아니지만 무장적 해보기로 했다. (어떻게든 되겠지ㅎ)

# 세미나 정보

[SQL PASS Korea 세미나 시즌2 - 신청 페이지](https://onoffmix.com/event/270849)

![SQL PASS Korea 세미나 시즌2 세션 목차](img/sql-pass-korea-seminar-2023-03/onoffmix-table-of-contents.png)

- 등록 및 교류시간 (PM 1:30 ~ 2:00)
- SQL PASS Korea 소개 (발표자 김정선 @ PASS Korea Leader)

- 세션-1: Azure SQL database Geo Replica, 또는 Managed Instance의 Failover group을 통한 CQRS 구현 (50분), 김정우 ㈜클라우드메이트<br>
`(Level 100) Azure SQL database의 Geo Replica, 또는 Managed Instance의 Failover group을 통해 RW와 Readonly를 간편하게 구성할 수 있습니다.
WAS에서 Command(CUD)는 RW로 요청하고, Query(R)는 Readonly로 요청하여 CQRS 패턴을 구현합니다.`

- 세션-2: SQL Server 2022 차세대 Intelligent Query Processing (50분), 김정선 ㈜씨퀄로<br>
`(Level 300)SQL Server 2022의 새로운 기능 중 쿼리 성능 관점에서 핵심이 되는 Intelligent Query Processing(IQP)의 차세대 기능들을 소개합니다. SQL Server 2022로 업그레이드를 준비 중이거나 계획 중인 분들과 SQL Server 기술에 관심 있는 분들에게 도움이 될 것입니다.`

- 세션-3: SQL Server 2022 Engine Innovations (50분), 김영건 (주)씨퀄로<br>
`(Level 200)SQL Server 2022 Engine의 발전된 기능들과 확장된 T-SQL 기능들을 소개합니다. SQL Server 2022로 업그레이드를 준비 중이거나 계획 중인 분들과 SQL Server 기술에 관심 있는 분들에게 도움이 될 것입니다.`

- 이벤트 & Q/A (당일 상황에 따라 변동될 수 있습니다)

# 세션 주요 내용

세션 내용에 대해서 핵심적인 부분만 추려서 설명하자면 Azure SQL Database Geo Replica 와 Failover group에 대한 아키텍처를 비교하는 내용이고,

![](img/sql-pass-korea-seminar-2023-03/Azure-SQL-Database-Geo-Replica-vs-Failover-groups.png)

FOG의 경우 failover 이후에도 기존 {fog-name}.database.windows.net 으로 접속했다면 연결문자열을 업데이트하지 않아도 됩니다.

![](img/sql-pass-korea-seminar-2023-03/Feature-comparison-Azure-SQL-Database-Geo-Replica-VS-Failover-group.png)

Command (insert, update, delete) 와 Query (select) 의 책임을 분리시키는 패턴 중에 Event Sourcing pattern이 있습니다.

![](img/sql-pass-korea-seminar-2023-03/CQRS-Pattern-Event-Sourcing-Pattern.png)

Data Layer 부분에서 listener를 통해서 primary/secondary 간에 read 요청과 write요청이 적절히 배분되도록 구성했습니다.

![](img/sql-pass-korea-seminar-2023-03/Micro-service-Architecture-Basic-Design.png)

Geo Replica의 경우 하나의 db가 이미 다른 FOG에 구성되어 있어도, 이를 또 다른 region에 readonly로 복제할 수 있습니다.

![](img/sql-pass-korea-seminar-2023-03/Azure-SQL-Database의-Geo-Replica-및-Failover-Group-구성-demo.png)

vnet-query-app (10.20.0.0/24) 에서 vnet peering을 통해서 R/W listener와 R/O listener로 SQL을 요청할 수 있게 됩니다.

![](img/sql-pass-korea-seminar-2023-03/Azure-SQL-Managed-Instance의-Failover-Group-구성-demo.png)

오늘의 핵심 Insight!

![](img/sql-pass-korea-seminar-2023-03/Azure-SQL-Managed-Instance의-Failover-Group-demo-구성-insights.png)

# 스냅 사진

![](img/sql-pass-korea-seminar-2023-03/KakaoTalk_20230917_205807945_05_down.jpg)

![](img/sql-pass-korea-seminar-2023-03/KakaoTalk_20230917_205807945_11.jpg)

![](img/sql-pass-korea-seminar-2023-03/KakaoTalk_20230917_205807945_06_down.jpg)

![김정선 이사님](img/sql-pass-korea-seminar-2023-03/KakaoTalk_20230917_205807945_04_down.jpg)

# 발표자료

발표자료는 다음 링크에서 자유롭게 받아가실 수 있습니다.

[CQRS implementation through Azure SQL Database Geo Replica or Failover group of Managed Instance - Jungwoo Kim 2023.03.17.pdf](https://drive.google.com/file/d/1udpMKt6hpttfw1yVBDk3l6tGX8y7wh_R/view?usp=sharing)

# 결론

Azure SQL Database, Managed Instance의 특성와 FOG 구성의 차이점에 대해서 직접 찍먹해보고, 우리 개발팀에서 서비스하는 아키텍처에 적용해보면서 경험할 수 있어서 좋았다.