---
# Common-Defined params
title: "Grand open"
date: "2015-11-02"
description: "Example article description"
images: ['img/giftcard.png']
categories:
  - "story"
tags:
  - "Grand open"
menu: main # Optional, add page to a menu. Options: main, side, footer

# Theme-Defined params
thumbnail: "img/avatar.jpg" # Thumbnail image
lead: "rokag3-gb.github.io 블로그를 개설하였습니다." # Lead text
comments: true # Enable Disqus comments for specific page
authorbox: true # Enable authorbox for specific page
pager: true # Enable pager navigation (prev/next) for specific page
toc: true # Enable Table of Contents for specific page
mathjax: true # Enable MathJax for specific page
sidebar: "left" # Enable sidebar (on the right side) per page
widgets: # Enable sidebar widgets in given order per page
  - "recent"
  - "taglist"
---

# 대제목

Hello World! 나는 GarlicBread 입니다.

rokag3-gb.github.io 블로그를 개설하였습니다.

{{< openinnewtab href="https://rokag3-gb.github.io/" title="markdown 안에서 openinnewtab을 활용해서 하이퍼링크를 새 창으로 여는 테스트 입니다." >}}

![](img/GitHub-Logo-history-500x428.jpg)

## 중제목 - SQL syntax highlight

{{< highlight sql "linenos=table, hl_lines=8 18-21" >}}
// SQL syntax highlight sample
select  getdate();

select  *
from  dbo.Deal
where 거래일 between '2013-01-01' and '2013-12-31';
{{< /highlight >}}

## 중제목 - C#.NET syntax highlight

C# 소스코드 샘플 입니다.

{{< highlight "C#" "linenos=table" >}}
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
            // To customize application configuration such as set high DPI settings or default font,
            // see https://aka.ms/applicationconfiguration.
            ApplicationConfiguration.Initialize();

            Application.Run(new Form1());
        }
    }
}
{{< / highlight >}}

### 소제목

![](img/GitHub-Logo-history-500x428.jpg)