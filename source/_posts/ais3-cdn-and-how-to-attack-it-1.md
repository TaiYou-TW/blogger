---
title: AIS3 2022 — CDN 技術所帶來的攻擊面向（上）
date: 2022-10-10 19:42:00
tags:
    - ais3
    - web security
    - cdn
---

## 1. Introduction

距離今年 AIS3 已經結束一個月了，還都收到了第二張結業證書，一定是提醒我該開始來補坑(๑•ั็ω•็ั๑) 今年是第一次參加 AIS3，參加的感想用一句話來說的話，就是「這真的是免費可以參加的嗎」（X），講師都是各路業界大神，真的是一個非常棒的經驗，也很感謝所有籌辦的教授、工作人員等，他們多年來的努力耕耘，讓台灣的資安界能越來越好 (_´▽`_)

這系列文章預計會分享一些網頁安全或是我比較感興趣的課程，不僅希望可以幫助一些也有興趣的同好，也希望透過筆記讓自己能夠溫故知新~~（趁自己還沒忘記之前）~~。

第一篇讓我直接跳到第二天的上午課程，是由 Mico 帶來的「CDN 技術所帶來的攻擊面向」，這篇會介紹 CDN 以及相關的攻擊議題，由於這堂課內容太豐富了，所以會分成兩篇。另外，這是我第一次聽 Mico 講課，不得不稱讚一下口條蠻好的，而且內容也十分淺顯易懂，人還很有趣，大推 👍

完稿後補充：結果中間太忙，打完上跟下兩篇已經是課程結束後兩個月了 XDD，不過這篇太長了，多花點時間也應該是可以接受的（吧？

<!-- more -->

## 2. CDN Introduction

CDN 的全名是 Content Delivery Network，中文通常翻成「內容傳遞網路」，由於現在十分普及，所以不管是藍隊或是紅隊，都需要對它有一定的了解。

而 CDN 是什麼呢？顧名思義就是傳遞內容的網路，而這些「內容」通常就是音樂、圖片、影片這些靜態檔案，這個網路可以高效的把這些內容傳給使用者，而至於為什麼這個網路可以做到高效，就是透過部署大量的節點在各地，當使用者想要存取「內容」時，就會讓離使用者最近的節點（或是最快）來提供，而不是從原始提供者的伺服器，聽起來可能很模糊，讓我們看張圖：

![CDN meme](https://i.imgur.com/TjaWIyJ.png)

不對，是這張：

![CDN vs single server(source: wikipedia)](https://upload.wikimedia.org/wikipedia/commons/thumb/2/26/NCDN_-_CDN.svg/1280px-NCDN_-_CDN.svg.png)

左邊為單一節點模式（Single Server），右邊就是 CDN 的模式，因此 CDN 最直觀的價值就是讓連上你網站的使用者可以快速的得到他們想要的內容，而網站回應速度同時影響到 SEO、UX 等，也可以降低總體成本、防禦部分攻擊，所以可說是十分重要的，以下簡單比較一下雙方：

### 2.1. Single Server

-   單一伺服器：伺服器僅有一個，流量乘載上限即為該骨幹的上限。
-   「無」離線快取能力：當伺服器無法正常運作時，使用者就無法存取。
-   防禦成本高：需自行建置防禦系統，就算能自行建置安全且高可用性的系統，其極限也不高（例如：DDOS 攻擊，就算 Server 可以處理流量，但骨幹可能無法承受）。
-   流量浪費：在正常的網站瀏覽中，絕大多數的流量都是為了處理靜態資源，這樣重複存取靜態資源會造成大量的流量浪費。

### 2.2. CDN

-   全球的近端伺服器：伺服器分佈於全球（實際取決於 CDN 業者），流量會分散給各地的伺服器承擔。
-   「有」離線快取能力：就算主伺服器無法存取，但各地的節點可以繼續提供快取的資源。
-   防禦成本較低：由於使用者會先存取節點，而不是主伺服器，因此 CDN 業者可以統一在平台端部署防禦機制（WAF 或流量清洗等機制），相較起來防禦效果較好，成本也較低。
-   快取重用：由於靜態資源會先被快取，因此可以達到加速且節省流量的效果。
-   隱藏真實 IP：由於 CDN 可以將使用者流量轉送到實際的主機，因此 domain 只需將 CDN 業者的 IP 暴露在外即可，可以減少受攻擊可能性。
-   瀏覽加速：若 CDN 擁有可用的快取時，可以直接回傳給使用者；若沒有快取時，會由 CDN 去原始 Server 查詢，再由 CDN 返回給使用者，而由於通常 CDN 業者的頻寬、吞吐量、速度等網路環境都一定比使用者來得優秀，因此這時就算內容沒有快取，只要是透過 CDN 去拿到的資料還是會比使用者直連 Server 來得快速，就彷彿你請了個奧運國手代替你跑腿（？）。
-   異地備援：當其中一個節點故障時，

## 3. How to Attack & Defend

簡介完了 CDN，看起來很棒對吧？可以加速又可以防禦，那是不是託管給 CDN 就萬無一失了呢？當然沒有這麼好的事情，接下來我們會看一些打法，以及相對應的防禦措施：

### 3.1. Find Real IP

剛剛提到可以隱藏真實 IP，就是為了減少遭受被攻擊的可能性，因此反過來說，只要我們找到原始 IP，就可以直接 bypass WAF 攻擊其 Server，因此這章節會探討如何找到真實 IP：

#### 3.1.1. If it using CDN?

首先第一步，先確認這個網站是否正在使用 CDN：

-   A Record：可以用 Whois 之類的方法（或 Windows 的 `nslookup`、Linux 的 `dig` 等指令）從 domain 找到 A Record 對應的 IP，再把 IP 丟到 Whois 就可以查到該 IP 為 CDN 業者或是私人所擁有的。
-   Multi ping：由於 CDN 的特性，我們從不同地區存取同一個 domain 會被導到不同的伺服器，我們也可以反過來利用這個特性來測試，可以簡單地用一些線上工具做到（e.g. [Ping Test](https://tools.keycdn.com/ping)）。如下圖，可以看到我們從不同地區所發出的 ping 所得到的 IP 會是不一樣的。
    ![Multi ping](https://i.imgur.com/C0XWJ8D.png)
-   `/cdn-cgi/trace`(Cloudflare only)：在目標網站存取 `/cdn-cgi/trace`，若目標網站為使用 CF 的話，會有一個資訊頁面，例如：<https://www.cloudflare.com/cdn-cgi/trace>
-   Parse Error：利用 URL encoding 的方式造成頁面噴錯，這時有機會看到 CF 的錯誤頁面，例如：正常情況下網址可能為 `http://target.com/%20`，若我們輸入 `http://target.com/%`，就可能讓 Web Server 解析時，以為這個編碼還沒結束，導致解析錯誤而噴出 5XX 的錯誤。

#### 3.1.2. Find IP

##### 3.1.2.1. 工具或服務

-   字典檔掃描：有時網站會有一些 subdomain，而有可能因為成本考量，subdomain 並沒有託管在 CDN 上，而 subdomain 的 IP 可能會是與目標相近的 Server（甚至同一台），因此我們可以先用字典檔掃出 subdomain 再用旁敲側擊的方式攻入。可以利用一些別人寫好的腳本，例如 [Knockpy](https://github.com/guelfoweb/knock) 或是下面的 Sublist3r。
-   DNS 歷史紀錄：DNS 紀錄有時可以找到一些蛛絲馬跡，例如一開始可能是直接綁主機 IP，後來才綁在 CDN 上，我們一樣可以用一些線上工具掃出來，例如：[VirusTotal](https://www.virustotal.com/gui/home/url)。
-   X509v3 Subject Alternative Name(X509v3 SAN)：一種公鑰證書的擴充標準，總之可以在憑證裡面填寫一些備用名稱，包括 Email、IP、URI、DNS 等，所以在記錄裡面有機會看到 IP，一樣有線上工具：[crt.sh](https://crt.sh/)。
    ![x509v3 SAN(source: wikipedia)](https://upload.wikimedia.org/wikipedia/commons/d/d2/Ssl_com_ev_uc_certificate.jpg)
-   [BuiltWith](https://builtwith.com)：這個服務可以查詢一些網站的各項資訊，其中有一個地方可以查到 IP 的歷史紀錄，如下圖所示。
    ![BuiltWith](https://i.imgur.com/Qj2QgYQ.png)
-   [CloudFlair](https://github.com/christophetd/CloudFlair)：一個用 SSL 證書的資訊去尋找網站 IP 的 python 工具。（CF only）
-   [theHarvester](https://github.com/laramies/theHarvester)：一個強大的 OSINT python 工具，可以收集 name、email、IP、subdomain、URL 等多項資訊。
-   Other OSINT search engine：一些常見的特徵搜尋引擎也可以用來找蛛絲馬跡，例如：ZoomEye、FOFA、SHODAN 等，可以利用 CDN 的一些特徵 header 去做搜尋。
-   [Sublist3r](https://github.com/aboul3la/Sublist3r)：一個用來掃描 subdomain 的 python 工具。
-   [dnsdumpster](https://dnsdumpster.com/)：一個用來掃描 domain 資訊的服務，會用圖表的方式方便觀看。
-   [CrimeFlare](https://github.com/zidansec/CloudPeler)：一個用來找 CF 背後的 IP 的資訊的工具。

##### 3.1.2.2. 功能探測

網站中有時會有一些外部的服務，例如連結預覽，這些功能就會讓伺服器對外部發出 Reqeust，也就表示你可以在自己的網站上收到來自對方 Server 的 Request，也就是可以拿到對方 IP；另外，如果對方有寄信（忘記密碼？）的功能，也有可能對方會使用同一個 Server 當作 Mail Server 去寄送信件，也可以從此得到一些資訊。

##### 3.1.2.3. 其他

-   針對客服或後台 XSS
-   Webshell

#### 3.1.3. Defense

IP 被揭露後，乍看不會有什麼嚴重的危害，但這的確是攻擊的第一步，所以能夠好好隱藏起來的話，確實可以降低風險，以下列出一些防禦建議：

-   白名單僅允許 CDN 的 IP（CDN IP 為公開的，因此就算被得知 IP，白名單擋住外界還是可以降低風險）。
-   Subdomain 或其他服務（無 CDN 的）的 IP 最好與欲防禦的主機 IP 差距夠大，以防對方得知旁站的 IP 後，只需要掃描相鄰的 IP 即可找到目標 IP。
-   用第三方服務或平台取代部分功能，避免洩漏過多伺服器資訊給有心人士。
-   使用新的 IP 綁 CDN，避免被有心人士從 DNS 紀錄中找到蛛絲馬跡。

## 4. Summary

這篇先簡單介紹了 CDN 的原理，以及如何使用一些簡單的工具、服務來確認、尋找隱藏在 CDN 背後的 Server IP，下一篇就會聚焦在實際攻擊及防禦的各種手法，有興趣的請移駕[下篇](https://fongyehong.top/blog/2022/10/10/ais3-cdn-and-how-to-attack-it-2/)👍
