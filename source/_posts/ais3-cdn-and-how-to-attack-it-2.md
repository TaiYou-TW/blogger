---
title: AIS3 2022 — CDN 技術所帶來的攻擊面向（下）
date: 2022-10-10 19:50:00
tags:
    - ais3
    - web security
    - cdn
---

Note: 這篇是下篇，還沒看過[上篇](https://fongyehong.top/blog/2022/10/10/ais3-cdn-and-how-to-attack-it-1/)的可以先看一下再回來這篇！

## 1. How to Attack & Defend

### 1.1. Post Exploitation - CF-Connecting-IP

還有另一種常見的問題，就是取得 IP 的實作方式所產生的漏洞。由於你的網站都是 CDN 的 Server 代替使用者送出請求，那你要怎麼取得使用者真實 IP 呢？這時 CDN 通常就會在 Request Header 中塞入一個自定義的 Header 來傳遞，原理與 Proxy 的 `X-Forwarded-For` Header 差不多（有興趣的話，推薦延伸閱讀：[如何正確的取得使用者 IP](https://devco.re/blog/2014/06/19/client-ip-detection)），例如 CF 就會使用 `CF-Connecting-IP` 這個 Header 來傳遞使用者 IP，我們就可以從這個 Header 中取得使用者的 IP。

<!-- more -->

#### 1.1.1. Post Exploitation

在講這個攻擊方式前，我們先講一下什麼是 Post Exploitation，有人好像會翻成「後利用」或「後攻擊」，我個人不喜歡翻成中文，因為 SEO 問題，你用中文 Google 的話永遠都找不到你想要的文章 XDD，總之 Post Exploitation 就是在你攻擊成功目標後，後續進行的提權、擴散、深入、開後門之類的操作，這邊的情境就是在我們得知 IP 之後，就可以開始進行我們的 Post Exploitation 了。

#### 1.1.2. Bypass IP Validation

OK，回到前面講的，我們現在就算躲在 CDN 後面，還是可以得到使用者的 IP，聽起來蠻好的。但，如果今天不是 CDN 告訴你這個資訊，而是一個透過前述各種方法得到你的 IP 的駭客呢？他得到你的 Server 的真實 IP 後，直接存取你的 Server，並自己偽造一個 Header： `CF-Connecting-IP:127.0.0.1`，這時就有機會繞過 IP 驗證直接登入後台了。

#### 1.1.3. Defense

這邊沒有特別有效的防禦方法，基本上還是需要有一個重要的概念：「永遠不要相信使用者」，因為這個 Header 是使用者可控的，因此不能完全相信它，只能當作參考，不過還是有幾種可以減緩風險的作法：

-   同前，白名單僅允許 CDN 的 IP，封鎖有心人士繞過 CDN 後直接存取 Server 或 Application。
-   敏感資訊或管理系統等的存取驗證不能僅用 IP 白名單來做，最好同時使用多種驗證方式（帳號密碼、OTP 等)。

### 1.2. Post Exploitation - Bypass IP Deny

從前述的防禦方法，攻擊者又可以延伸出新的防禦方式，假設今天網頁使用了白名單，並只限定 CDN 的 IP 存取，這樣是否就萬無一失了呢？

#### 1.2.1. Bypass throught Host Headr Rewrite

答案是否定的，讓我們看看以下情境：

1. A 網站使用了 CDF 的 WAF，且白名單內只有 CDN 的 IP。
2. B 駭客申請了一個新的 domain C，綁上 A 網站的真實 IP，並使用 CDN 代理。
3. B 在存取 C 時 Rewrite Host Header，讓 CDN 以為它在存取 C 而不是 A，這時就不會有 WAF。

但值得注意的一點是，這個功能（Rewrite Host Headr）不是每個 CDN 業者都有提供，例如 Mico 的簡報中有寫到 Cloudflare 有此功能，但實際查證後，發現雖然有，但因為僅限 Rewirte 成自己 Dmain 下的 subdomain，因此無法執行此攻擊（可見 References 2.，Cloudflare 官方使用範例是 Rewirte 成自己所有的 S3 Bucket），因此適用性沒有前面那麼高，但依舊是一個隱患。

而提出此攻擊的作者是基於 CloudFront ，他也有釋出一個工具：[cdn-proxy](https://github.com/RyanJarv/cdn-proxy)，可以用來執行此攻擊。

#### 1.2.2. Defense

作者的同篇文章裡也有提到防禦方法，這邊簡單介紹，詳細一樣可以到 References 中觀看：

1. for CloudFront，可以依照[官方文件](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/restrict-access-to-load-balancer.html)中的建議處理，在 CloudFront 中設定一個秘密的 HTTP Header，並讓 Load Balancer 只接收擁有該 Header 的 Request（另外還有 [S3 Bucket 的版本](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)），避免惡意使用者利用。
2. for Cloudflare，一樣參考[官方文件](https://developers.cloudflare.com/fundamentals/get-started/task-guides/origin-health/enterprise/#secure-origin-connections) 裡面有一些建議，主要建議使用 [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/) 或 [Authenticated origin pull](https://developers.cloudflare.com/ssl/origin-configuration/authenticated-origin-pull/)（請注意這個驗證有三種不同的選項，Zone-Level Authenticated Origin Pull using Cloudflare certificate 是沒辦法防禦這個攻擊的）來防禦。

至於其他 CDN，雖然可能不一定有類似的功能，但基本上就是以一個概念去做防禦：「始終對 CDN 進行嚴謹的驗證」。

### 1.3. Web Cache Deception & Poisoning

CDN 加速的其中一種方法，就是透過 Cache 來達成的，例如使用者 A 先存取了一張圖片，這時 CDN 上沒有這張圖片的 Cache，因此會到 Server 上抓取並 Cache 在 CDN 上，這時下一個使用者存取這張圖片時，CDN 就可以直接將這張圖片傳給使用者，而不用再到 Server 去重新要一次圖片。

#### 1.3.1. Cache Control Headers

簡單介紹一下 Cache Control Headers，主要是兩個：

1. Vary：定義如何選擇需 Cache 的內容，例如 Server 可能會依據 User-Agent 去回傳不同的內容（例如電腦版或手機板），以及依據壓縮演算法不同也有不同的內容，這種情況就會把 Header 定義為 `Vary: User-Agent, Accept-Encoding`。也就是若有完全相同的 Header 的 Cache 存在時，才會取用該 Cache，否則就會去取得新的資源。
2. Cache-Control：定義 Cache 行為，例如是否需要 Cache、Cache 有效時間、儲存在哪、是否需要跟後端驗證等等，也可以設定這個快取是否能共享給他人、或是僅能給當下的使用者使用，例如：個人帳號的頁面，你絕對不會想要這個頁面被 Cache 起來並共享給其他人吧？但一些錯誤的設定很有可能就會讓這件事情發生，因此開發者不可不慎。另外，此 Header 有個常見的誤區要注意，有些人會因為不希望內容被 Cache，就設定為 `no-cache`，但這個值的意義不是不設定 Cache，而是在使用 Cache 前須向後端進行驗證，如果不希望有 Cache 的話應設為 `no-store`。

#### 1.3.2. Default Cache Files

這邊簡單談一下預設會被 Cache 的檔案副檔名，但實際上還是要視 CDN 業者而定，不過基本上大部分的都會以以下為主：

1. 「不會」被 Cache：HTML、PHP、ASPX 等，通常來說較為動態的網頁或資源。
2. 「會」被 Cache：JS、CSS、JPG、MP3、MP4 等，靜態的檔案或資源。

#### 1.3.3. Web Cache Deception Attack

當使用者訪問一個不存在的資源時，假設出現以下情境：

1. Server 回覆了一個錯誤的頁面。
2. 該頁面含有個人或敏感資訊。
3. Cache Server 把該頁面當作靜態資源 Cache 起來。
4. 該靜態資源為可被其他使用者看見的 Cache。

這時若有其他人存取了這個 Cache，就可以在未經授權的情況下竊取他人資訊。也就是惡意使用者可以把一個 <http://server.com/myaccount/home/attack.css> 的連結傳給受害者，這時 Cache Server 就會對該頁面建立一個 Cache，接著攻擊者也存取該連結，Cache Server 就會把該 Cache 傳給攻擊者，也就可以獲得受害者的資訊。

而除了頁面上直接可以看見的資訊外，有的甚至會包含 Session ID 或其他 Cookie 資訊等，也就是可以讓惡意使用者完全控制使用者帳戶，是十分嚴重的一個漏洞。

2017 年時，就有一個研究員針對 Paypal 實現了這個攻擊，基本上就是以上述的概念實現的，詳細可以看[此篇文章](https://www.bleepingcomputer.com/news/security/web-cache-deception-attack-tricks-servers-into-caching-pages-with-personal-data/)了解更多。

#### 1.3.4. Web Cache Poisoning Attack

與前一項攻擊類似，但這項攻擊是攻擊者主動去汙染 Cache，讓後續存取的使用者受到攻擊，例如某些內容可能與 Header 的值相關聯，或是圖片的 base url 是 Header 的某個值等等，這時攻擊者就可以控制 Header 來改變使用者的靜態資源，甚至進行 XSS 等攻擊。

這邊後續有非常多的進階操作，這邊我十分推薦去讀一下 PortSwigger 上的[此篇文章](https://portswigger.net/research/practical-web-cache-poisoning)，裡面有非常多關於 Web Cache Poisoning 的內容、說明、防禦等，十分精彩。

#### 1.3.5. Defense

1. 只 Cache 指定資料夾或路徑內的靜態檔案。
2. 確實做好 Cache Rule，尤其是網站路由架構異動時，例如只有在 HTTP Header 允許時才做 Cache，可以從根本解決 Web Cache Deception 的問題。
3. 由 Content Type 決定是否 Cache，避免 Cache 一些動態或是不該被 Cache 的資源。
4. 禁止惡意路徑，例如 `/index.php/bad.css`。
5. 部署 WAF，增加惡意請求的難度。
6. 設定 Cache Header 時注意不該誤入前述的常見錯誤，以免將私人 Cache 公開。
7. 避免從 Header 或 Cookie 等地方獲取輸入，但通常較難預防或得知其他層面或框架等是否有使用，可以使用 [Param Miner](https://portswigger.net/bappstore/17d2949a985c4b7ca092728dba871943) 來做簡單的檢測。
8. 在 Cache 層對輸入做清洗或是將其加入 Cache Key 中，以防惡意使用者可以影響或是存取到他人的 Cache。（但有些 CDN 可能會把這項功能放在進階或企業用戶中）
9. 不管網站或 CDN 是否有 Cache，需要注意部分的使用者端也會自行處理 Cache，因此切記要注意一些可以透過控制 HTTP Header 達成的 XSS 或其他 Client-side 漏洞。

### 1.4. Cache Poisoned Denial Of Service (CPDoS)

目前由於 DDoS 的成本越來越低，導致發生的頻率與規模越來越高，前陣子 Cloudflare 才剛發布 2600 萬 requests/s 的紀錄，怕。而同樣的，攻擊者也可以透過前述的 Cache Poisoning 來達成 DoS 的攻擊。

以下三種攻擊的原理皆為讓 Cache Server 把錯誤的頁面 Cache 起來，導致正常使用者後續都只能得到該錯誤頁面，進而達到 DoS 的效果；而流量放大則是透過 CDN 大部分為流量計價的原理，用大量的流量迅速達到網站的預算上限而停止服務。

前三種攻擊的成立前提為 Server 在錯誤頁面沒有指定或錯誤地指定了 Cache 機制，導致 Cache Server 將錯誤頁面 Cache 起來，讓後續使用者無法正常存取，詳細可以看[這篇文章](https://cpdos.org/)。

#### 1.4.1. HTTP Header Oversize (HHO)

攻擊情境如下：

1. 攻擊者傳送 Header 中含有非常長的值的 Request。
2. Cache Server 中沒有 Cache，因此向 Server 送出 Request。
3. Server 無法 Handle 過大的 Header，Return 500 或其他錯誤頁面。
4. Cache Server 將 3. Cache 起來，並 Return 給 1.。
5. 後續，一般使用者發出一般的 Request，這時 Cache Server 擁有 Cache，因此直接 Return 如同 4. 的頁面。

#### 1.4.2. HTTP Meta Character (HMC)

攻擊情境如下：

1. 攻擊者傳送 Header 中含有 `\n`、`\r`、`\a` 等控制字元的 Request。
2. Cache Server 中沒有 Cache，因此向 Server 送出 Request。
3. Server 無法 Handle 該 Header，Return 500 或其他錯誤頁面。
4. Cache Server 將 3. Cache 起來，並 Return 給 1.。
5. 後續，一般使用者發出一般的 Request，這時 Cache Server 擁有 Cache，因此直接 Return 如同 4. 的頁面。

#### 1.4.3. HTTP Method Override (HMO)

有些 Proxy、Load Balancer、Firewall 等中間層會使用擋掉沒辦法或不希望收到的 HTTP Method，我們可以藉由自己在 Header 中塞入 `X-HTTP-Method-Override`，這時就可以繞過中間層的限制，對網站成功送出意料外的 Method，這時若對方無法處理，可能就會出現錯誤頁面，達到與前述類似的效果。

攻擊情境如下：

1. 攻擊者傳送 Header 中 `X-HTTP-Method-Override: DELETE`（或其他不應該有的 Method） 的 Request。
2. Cache Server 中沒有 Cache，因此向 Server 送出 Request。
3. Server 無法 Handle 該 Header，Return 500 或其他錯誤頁面。
4. Cache Server 將 3. Cache 起來，並 Return 給 1.。
5. 後續，一般使用者發出一般的 Request，這時 Cache Server 擁有 Cache，因此直接 Return 如同 4. 的頁面。

#### 1.4.4. Amplification Attack with CDN

DDOS 有一種很常見的方法就是流量放大，但至於如何達到就有十分多種選擇，在 CDN 上也有一些方法可以達成，不過此處主要是針對 CDN 節點本身，而不是 CDN 背後的 Server，那接下來我們就來看一下這些流量放大的攻擊技巧，不過通常目前大牌的提供商都有自行處理或緩解此類行為，不一定適用：

1. 自身循環放大：將 Server 的 source IP 設為 CDN 節點本身 IP，就可以讓 CDN 節點進行無限循環，進而放大流量。
2. CDN 互相映射：由於前者相當容易被防禦（只要禁止將 Source IP 設為節點 IP 即可），因此我們可以利用其他家的 CDN，透過將兩個節點的 IP 互相映射，達到循環放大的效果。
3. 多 CDN 映射：由於兩者也容易被擋住（偵測 Source 與 Destination 相同），我們可以繼續增加防禦的難度，也就是加入不同的節點，三個、四個、甚至更多個，讓這些節點循環映射之下，不僅可以放大流量，更增加防禦的難度。

而攻擊 CDN 的節點可以有什麼效果呢？通常 CDN 的節點上會有不止一個服務，我們就可以透過阻斷該節點，達到阻斷其他網站的效果，可以利用類似 [you get signal](https://www.yougetsignal.com/tools/web-sites-on-web-server/) 的服務來查詢，輸入 CDN 節點 IP 來得知哪些 domain 架在此節點上：

![image-20221009163723766](https://s2.loli.net/2022/10/09/Z4YCkTAo9pMnBQP.png)

#### 1.4.5. Defense

-   選擇具有威信的 CDN 服務廠商，並確認計價行為，避免被 DDOS 時產生大量的費用或服務中斷
-   遵循 RFC 標準開發，並對可 Cache 的狀態設白名單，避免 Cache 到錯誤頁面
-   確保 Server 架構與 CDN 不衝突
-   異常頁面加入 `Cache-COntrol: no-store`
-   與提供商簽訂 SLA（服務水準協議），明定如 DDOS 時發生時應如何收費等，確保權益

### 1.5. Domain Fronting

Domain Fronting 就是可以利用修改 Host Header，來達成 Domain 與實際訪問的網站不同的效果，通常會用於在已控制的目標上想回連 C2 Server 或是回傳資訊時，發現目標限制外連的 IP，就可以透過此方法繞過限制，實務上像 Telegram、Signal、Tor 都曾經使用此方法來逃過一些封鎖，但目前 CDN 業者可能基於一些政治或其他因素，逐漸不再支援此 Feature（或 Bug XD?）。

至於為什麼可以繞過限制，則是因為我們可以透過 HTTPS 外連，這時候 Header 以及 Body 會被加密，檢查機制就無法得知 Host 的資訊。所以我們只要找到一個該目標所信任的網站 IP，就可以透過把我們的 Server 架在同一個 CDN 節點上，來達到繞過限制的效果，如下圖：

![img](https://media.kasperskydaily.com/wp-content/uploads/sites/92/2019/04/08132118/domain-fronting-scheme.png)

### 1.6. HTTP Desync Request Smuggling

這也是一個利用「不一致」所達成的攻擊手法，首先需要了解一下 HTTP 的持久連接（HTTP persistent connection）：

![img](https://upload.wikimedia.org/wikipedia/commons/thumb/d/d5/HTTP_persistent_connection.svg/langzh-450px-HTTP_persistent_connection.svg.png)

由於可以減少延遲、使用較少資源等優點，以致 HTTP 1.1 開始預設會使用此方法，所以這個 connection 就會被重複利用，而 CDN 就會接收多個使用者的封包後，再轉送給 Server：

![Reverse proxy working as intended](https://portswigger.net/cms/images/3b/af/c28234336ee3-article-revproxy.svg)

這個攻擊簡單來說，就是攻擊者可以謊稱自己封包的長度，讓自己的封包穿插別人的封包，或是讓別人的封包穿插到自己的封包，而成因就是 CDN 及 Server 的行為不一致。（可以看 PortSwigger 的[這篇](https://portswigger.net/research/http-desync-attacks-request-smuggling-reborn)，很詳細 推推( •̀ ω •́ )✧）

#### 1.6.1. Attack

一般封包的長度會以兩個 Header 來描述，分別是：

1. `Content-Length`：舊有描述方式，用於一次傳遞整個封包時。
2. `Transfer-Encoding`：HTTP 1.1 引進，可以將封包切割成多個，較有彈性。

這時可以想像以下幾種情況：

1. 同時傳兩種會怎樣？

    這點 [RFC 2616](https://www.rfc-editor.org/rfc/rfc2616#page-34) 有定義（4.4 Message Length 倒數第二段），一起傳時必須忽略 Content-Length，聽起來蠻合理的，畢竟新的優先度比較高很正常。但 CDN 跟 Server 可能有不一樣的行為（總有人不照規範走 😶），例如一個讀 `Content-Length`，一個讀 `Transfer-Encoding`，就會導致前面提到封包穿插的問題。

2. 同時傳兩個一樣的會怎樣？

    RFC 2616 沒想過有這種情況 🤔 所以沒有標準規範的情況下，實作時就會有很大的差異空間，所以跟前面一樣，若 CDN 與 Server 取不同的 Header 也會有一樣的問題。

#### 1.6.2. Defense

1. 選擇會對 Header 做檢查、正規化的 CDN 提供商。
2. 避免前後端點封包結束方式不同。
3. 升級至 HTTP/2。
4. 確認伺服器實作方式是否遵守 RFC 規範。

## 2. Summary

打完發現這篇超長，只好分兩篇了 😶。在這堂課學到蠻多 CDN 攻擊的知識，很多攻擊技巧也非常酷，讓我大開眼界 XD，沒想到這些很方便的功能或是服務可以被這樣利用。雖然很多攻擊技巧都是好幾年前的，大廠牌的 CDN 提供商也大部分都已經修正了，不過有些小型或是知名度較低的廠商可能就尚未修正或是修正的不完全，所以可能就會讓攻擊者有機可趁，因此選用服務商時還是盡量選用口碑好或大廠牌的，貴是有它的道理的（？）。

## 3. References

1. [Bypassing CDN WAFs with alternate domain routing](https://blog.apnic.net/2022/05/19/bypassing-cdn-wafs-with-alternate-domain-routing/)
2. [Using Page Rules to rewrite Host Headers - Cloudflare](https://support.cloudflare.com/hc/en-us/articles/206652947-Using-Page-Rules-to-rewrite-Host-Headers)
3. [RSAC 2019: camouflage for the attackers' communication channel | Kaspersky official blog](https://www.kaspersky.com/blog/domain-fronting-rsa2019/26352/)[RSAC 2019: camouflage for the attackers' communication channel | Kaspersky official blog](https://www.kaspersky.com/blog/domain-fronting-rsa2019/26352/)
4. [HTTP 持久連接 - 維基百科，自由的百科全書 (wikipedia.org)](https://zh.wikipedia.org/zh-tw/HTTP持久连接)
5. [HTTP Desync Attacks: Request Smuggling Reborn | PortSwigger Research](https://portswigger.net/research/http-desync-attacks-request-smuggling-reborn)
