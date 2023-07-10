---
# Common-Defined params
title: "SQL PASS Korea 세미나 발표 - Azure SQL database의 Geo Replica와 Managed Instance의 Failover group을 통한 CQRS 구현"
date: "2023-03-17"
description: "SQL PASS Korea 커뮤니티의 초청을 받아 컨퍼런스 발표"
#images: ['images\KakaoTalk_20230616_154117362_05.jpg']
categories:
  - "DB"
  - "Story"
tags:
  - "Azure SQL"
  - "PASS Korea"
  - "Failover"
  - "Failover group"
#menu: side # Optional, add page to a menu. Options: main, side, footer

# Theme-Defined params
#thumbnail: "images\20230616_154117362_05_600_450.jpg" # Thumbnail image
lead: "SQL PASS Korea 커뮤니티의 초청을 받아 컨퍼런스 발표" # Lead text
comments: true # Enable Disqus comments for specific page
authorbox: true # Enable authorbox for specific page
pager: true # Enable pager navigation (prev/next) for specific page
toc: true # Enable Table of Contents for specific page
mathjax: true # Enable MathJax for specific page
sidebar: "right" # Enable sidebar (on the right side) per page
widgets: # Enable sidebar widgets in given order per page
  - "recent"
  - "taglist"
---

# 도입

그렇다. 발표할 일 있으면 해야지 했다가 막상 하게 되면 또 걱정되고 무섭다. 여튼 나는 DB 전문가는 아니지만 무장적 해보기로 했다.

<!--![](images/ss%202023-06-16%20145300.png)-->

https://rokag3-gb.github.io/posts/2023-03-17-sql-pass-korea-seminar-azure-sql-failover-group/images/SQL%20PASS%20Korea%20Seminar%20Season%202%20%EB%B0%9C%ED%91%9C%202023.03.17.pdf

[](https://rokag3-gb.github.io/posts/2023-03-17-sql-pass-korea-seminar-azure-sql-failover-group/images/SQL%20PASS%20Korea%20Seminar%20Season%202%20%EB%B0%9C%ED%91%9C%202023.03.17.pdf)

# 발표 자료

1

<html>
<object data="https://rokag3-gb.github.io/posts/2023-03-17-sql-pass-korea-seminar-azure-sql-failover-group/images/SQL%20PASS%20Korea%20Seminar%20Season%202%20%EB%B0%9C%ED%91%9C%202023.03.17.pdf" type="application/pdf" width="700px" height="700px">
    <embed src="https://rokag3-gb.github.io/posts/2023-03-17-sql-pass-korea-seminar-azure-sql-failover-group/images/SQL%20PASS%20Korea%20Seminar%20Season%202%20%EB%B0%9C%ED%91%9C%202023.03.17.pdf">
        <p>This browser does not support PDFs. Please download the PDF to view it: <a href="https://rokag3-gb.github.io/posts/2023-03-17-sql-pass-korea-seminar-azure-sql-failover-group/images/SQL%20PASS%20Korea%20Seminar%20Season%202%20%EB%B0%9C%ED%91%9C%202023.03.17.pdf">Download PDF</a>.</p>
    </embed>
</object>
</html>

2

<html>
<embed type="application/pdf" src="https://rokag3-gb.github.io/posts/2023-03-17-sql-pass-korea-seminar-azure-sql-failover-group/images/SQL%20PASS%20Korea%20Seminar%20Season%202%20%EB%B0%9C%ED%91%9C%202023.03.17.pdf" width="100%" height="9000px">
</html>

3

---

4

<html>
<iframe
      class="slide"
      src="https://rokag3-gb.github.io/posts/2023-03-17-sql-pass-korea-seminar-azure-sql-failover-group/images/SQL%20PASS%20Korea%20Seminar%20Season%202%20%EB%B0%9C%ED%91%9C%202023.03.17.pdf"
      width="50%"
      height="500px"
    ></iframe>
</html>

5
```
{% pdf https://rokag3-gb.github.io/posts/2023-03-17-sql-pass-korea-seminar-azure-sql-failover-group/images/SQL%20PASS%20Korea%20Seminar%20Season%202%20%EB%B0%9C%ED%91%9C%202023.03.17.pdf %}
```

{% pdf https://rokag3-gb.github.io/posts/2023-03-17-sql-pass-korea-seminar-azure-sql-failover-group/images/SQL%20PASS%20Korea%20Seminar%20Season%202%20%EB%B0%9C%ED%91%9C%202023.03.17.pdf %}

<html>
{% pdf https://rokag3-gb.github.io/posts/2023-03-17-sql-pass-korea-seminar-azure-sql-failover-group/images/SQL%20PASS%20Korea%20Seminar%20Season%202%20%EB%B0%9C%ED%91%9C%202023.03.17.pdf %}
</html>

6

```
{% pdf ./images/SQL%20PASS%20Korea%20Seminar%20Season%202%20%EB%B0%9C%ED%91%9C%202023.03.17.pdf %}
```

{% pdf ./images/SQL%20PASS%20Korea%20Seminar%20Season%202%20%EB%B0%9C%ED%91%9C%202023.03.17.pdf %}

<html>
{% pdf ./images/SQL%20PASS%20Korea%20Seminar%20Season%202%20%EB%B0%9C%ED%91%9C%202023.03.17.pdf %}
</html>

7

<iframe src="https://rokag3-gb.github.io/posts/2023-03-17-sql-pass-korea-seminar-azure-sql-failover-group/images/SQL%20PASS%20Korea%20Seminar%20Season%202%20%EB%B0%9C%ED%91%9C%202023.03.17.pdf" style="width:100%; height:600px;" frameborder="0"></iframe>

8

<html>
<iframe src="https://rokag3-gb.github.io/posts/2023-03-17-sql-pass-korea-seminar-azure-sql-failover-group/images/SQL%20PASS%20Korea%20Seminar%20Season%202%20%EB%B0%9C%ED%91%9C%202023.03.17.pdf" style="width:100%; height:600px;" frameborder="0"></iframe>
</html>

9

```html
<p>This is an example of HTML code.</p>
<div class="my-div">This is a div element.</div>
\```

위와 같이 작성하면 코드 블록이 생성되며, Hexo는 해당 코드를 그대로 HTML로 인식하여 블로그에서 코드를 표시할 것입니다.

여러 줄의 코드 블록을 생성하려면 백틱을 세 개 붙이면 됩니다. 아래는 예시입니다:

10

````html
<p>This is an example of HTML code.</p>
<div class="my-div">This is a div element.</div>
\````

11

```html
<iframe src="https://rokag3-gb.github.io/posts/2023-03-17-sql-pass-korea-seminar-azure-sql-failover-group/images/SQL%20PASS%20Korea%20Seminar%20Season%202%20%EB%B0%9C%ED%91%9C%202023.03.17.pdf" style="width:100%; height:600px;" frameborder="0"></iframe>
```

12

````html
<iframe src="https://rokag3-gb.github.io/posts/2023-03-17-sql-pass-korea-seminar-azure-sql-failover-group/images/SQL%20PASS%20Korea%20Seminar%20Season%202%20%EB%B0%9C%ED%91%9C%202023.03.17.pdf" style="width:100%; height:600px;" frameborder="0"></iframe>
````

13

<iframe src="https://www.slideshare.net/slideshow/embed_code/key/ux3JY2awIl63Qc?hostedIn=slideshare&page=upload" width="476" height="400" frameborder="0" marginwidth="0" marginheight="0" scrolling="no"></iframe>

14

[](https://www.slideshare.net/JungwooKim41/sql-pass-korea-seminar-season-2-azure-sql-database-geo-replica-managed-instancefailover-group-cqrs)

https://recruit.hmglobal.com/careers/list.asp

15

https://present.do/documents/64aa85c910ab9a5ae55bad48

16

<iframe src="https://present.do/documents/64aa85c910ab9a5ae55bad48" style="width:100%; height:600px;" frameborder="0"></iframe>

17

---
`eof`