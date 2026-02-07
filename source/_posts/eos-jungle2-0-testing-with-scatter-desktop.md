---
title: EOS Jungle2.0 Testnet with Scatter Desktop
date: 2020-05-11 09:38:04
tags:
    - eos
    - scatter
---

此篇旨在紀錄筆者透過 Scatter 桌面版使用 Jungle2.0 Testnet 的過程及教學

<!-- more -->

## Generate key

1. 打開 Scatter，並點選 Wallet > Generate Key
   ![](https://i.imgur.com/oKJ3yeP.png)

2. 選擇 EOSIO
   ![](https://i.imgur.com/lueA2wS.png)

3. 選擇 Key
   ![](https://i.imgur.com/FXg23Rp.png)

4. 此時 Scatter 即會幫你生成一個 Private Key，請自行保管好
   ![](https://i.imgur.com/aUjkbB5.png)

5. 按 Back 回到 Wallet 的頁面 (步驟 1.)，並將紅色框框中的 Public Key 複製下來
   ![](https://i.imgur.com/bgycJfo.png)

## Create an account

1. [點此](https://monitor.jungletestnet.io/#account)註冊一個帳號

2. 輸入 Account name，並將剛才的 Public Key 貼上在下面兩個欄位
   ![](https://i.imgur.com/GLgU91k.png)

## Get Free Tokens

1. [點此](http://monitor.jungletestnet.io/#faucet)並輸入剛才的帳號名稱，就可以獲得 1 Token

## Add Jungle 2 Network in Scatter

1. 回到 Scatter，並點選 Networks > Add Custom Network
   ![](https://i.imgur.com/NLKUF1I.png)

2.  - Name 中輸入一個可辨識的名稱

-   Host 輸入 jungle2.cryptolions.io
-   Protocol 為 https
-   Port 為 443
-   Chain ID 為 e70aaab8997e1dfce58fbfac80cbbb8fecec7b99cf982a9444273cbc64c41473
    最後按下 Add 即可
    ![](https://i.imgur.com/oUufXBu.png)

## Check Network Setting

1. 最後回 Wallet 頁面，確認是否確實連上測試鏈。只要兩個紅框處分別呈現帳號名稱，及剛剛拿到的 1 個 Free Token，即為成功連結測試鏈。

![](https://i.imgur.com/XyWA4LK.png)
