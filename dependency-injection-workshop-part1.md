---
layout: post
title: "Dependency Injection and AOP 介紹與工作坊心得（上）"
date: 2020-08-03
tags: [dependency-injection]
draft: true
category: article
---

去年九月參加 91 的 [DI 與 AOP 進階實戰](https://dotblogs.com.tw/hatelove/2018/11/14/201905-dependency-injection-and-aspect-oriented-programming) ，直到最近才有機會完整複習。該課程早期梯次只有支援 C#，因為平時工作不會用到 C#，我刻意改用 Python 做練習，並且在練習完畢後，對公司同事做分享。

DI 是 Dependency Injection 的縮寫，由外部注入目標物件所需的依賴，使得物件內部不再直接依賴於系統。舉例來說，物件方法如果直接連接資料庫，那麼測試該方法就必然涉及真實系統，將較為危險、耗時，而其後如果需要更改資料來源（比如從資料庫改為檔案系統），也勢必需要修改該物件檔案，將違反開放-封閉原則（即對擴充開放，對修改封閉）。

- 補 Open-close 補連結
- 補圖

AOP 是 Aspect-oriented Programming 的縮寫，切面導向的程式設計，主要目的是提高程式模組化程度，手段是將業務邏輯拆分並封裝成較小的模組，予以抽象化，各模組依賴其他抽象類，而不直接依賴實體類。接續上一段的例子，上述連接資料庫的邏輯可以封裝成一個類，並且抽象化，目標物件則依賴此抽象類。當關注點是目標物件時，就不須關心資料怎麼取得，不論資料來源變了，或是資料庫連結類的實作有改變，都不會影響此時關注點內的業務邏輯。

- 補圖

這篇文章分為上下兩篇，上篇介紹 DI 與 AOP 的內涵，並搭配範例說明。下篇紀錄與課心得和後續內部分享的心得。

<!--more-->

#### 依賴注入



