---
title: 연습 삼아 작성해보는 첫번째 포스트
date: 2015-09-19 14:40:55
tags: [first, test posting, hexo, Fluid theme]
index_img: img/avatar_Jayden.jpg
---

# 대제목 - 첫번째 대제목

## 중제목 - 블로그 개설

Hello World!

rokag3-gb.github.io 블로그를 개설하였습니다.

Hugo에서 Hexo로 변경했고, 테마 적용하는 부분부터 모르겠군요.

Fluid 테마를 적용하고 처음 생성한 포스트 입니다.

[이 하이퍼링크를 누르면 현재 페이지에서 바로 이동합니다.](https://rokag3-gb.github.io/)

## markdown 연습

### 이미지

![](img/GitHub-Logo-history-500x428.jpg)

### 인용구, 리스트, 체크박스, 폰트, code block

> 인용구 테스트  
> 인용구 테스트 두번째 줄

리스트 & 체크박스 테스트
- [x] 항목 1번
  - [ ] 1.1 세부항목
  - [x] 1.2 세부항목
- [x] 항목 2번
  - 2.1 세부항목
  - 2.2 세부항목
    - 2.2.1 블라블라
    - 2.2.2
  - [ ] 2.3 세부항목
- [ ] 항목 3번
  - [ ] 3.1 이렇게 체크박스를 리스트 안에 넣을 수 있습니다.

---

_이탤릭_ 은 underscore(_)를 앞뒤로 묶어주고,  
**볼드체** 은 ** 을 앞뒤로 묶어주고,  
~~취소선~~ 은 ~~ 을 앞뒤로 묶어주면 됩니다.

`code block` 은 grave(물결표시 아래 작은따옴표) 로 묶어주면 됩니다.

### C#.NET syntax highlight

C# 소스코드 샘플 입니다.

```c#
// C#.NET syntax highlight sample
namespace myapp
{
    internal static class Program
    {
        /// <summary>
        ///  The main entry point for the application.
        /// </summary>
        [STAThread]
        static void Main()
        {
            ApplicationConfiguration.Initialize();

            Application.Run(new Form1());
        }
    }
}
```

### SQL syntax highlight

```
/* SQL syntax highlight sample */
select  getdate();

-- 한줄 주석
select  *
from  dbo.Deal
where 거래일 between '2013-01-01' and '2013-12-31';
{{< /highlight >}}
```

### table

|col1|col2|col3|
|--|--|--|
|value1|value2|value3|

흠 table은 조금 별로군요.

``` bash
$ hexo generate; hexo server
```

### 코드 조각

몇가지 코드 조각을 넣는 테스트 입니다.

```sql
select  getdate();

select  *
from    dbo.Tenant
where   isActive = 1;
```

### Table

마크다운에서 Table 삽입 테스트 입니다.

|column A|column B|
|--|--:|
|A|1000000|
|B|300000|
|C|1300000|
|D|2006000|

### 하이퍼링크

https://hexo.fluid-dev.com/posts/hello-fluid/

https://fluid.s3.bitiful.net/hello-fluid/cover.png?w=480&fmt=webp