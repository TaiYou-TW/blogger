---
title: ERC-20 Token Standard 簡介
date: 2020-05-11 09:33:25
tags:
    - erc
    - solidity
---

## ERC-20 與 ERC-721 比較

![](https://i.imgur.com/yJRa1ri.png)

簡單來說，ERC20 是「每個代幣都一樣」；而 ERC721 則是「每個代幣都有其獨特性」

<!-- more -->

## Interface

```Solidity
contract ERC20Interface {
    function totalSupply() public constant returns (uint);
    function balanceOf(address tokenOwner) public constant returns (uint balance);
    function allowance(address tokenOwner, address spender) public constant returns (uint remaining);
    function transfer(address to, uint tokens) public returns (bool success);
    function approve(address spender, uint tokens) public returns (bool success);
    function transferFrom(address from, address to, uint tokens) public returns (bool success);

    event Transfer(address indexed from, address indexed to, uint tokens);
    event Approval(address indexed tokenOwner, address indexed spender, uint tokens);
}
```

總共有 6 個 function 以及 2 個 event。其中 constant 的 function 是唯讀的，所以不會花費 Gas。
Event 只用於記錄，可以視為一般系統上的 log 功能。

```
string public constant name = "Token Name";
string public constant symbol = "SYM";
uint8 public constant decimals = 18; // 18 is the most common number of decimal places
```

另外還有三個需要設定的參數：name、symbol、decimals。name 是 Token 的名字；symbol 是 Token 的代稱（簡稱）；decimals 是 Token 小數最多可以到幾位數，正常為 18，也就是和 Ether 一樣。

## Function 說明

1. totalSupply()，Token 的發行總量。
2. balanceOf(address)，傳入地址的錢包的 Token 數量。
3. allowance(address A, address B)，A 批准給 B 的 Token 量。
4. transfer(address A, uint num)，將數量為 num 的 Token 轉移給 A。
5. approve(address A, uint num)，批准數量為 num 的 Token 轉移給 A，需注意的是，這個 function 只是單純做「批准」這個動作，而沒有進行轉移。若需要轉移則要再呼叫 transferFrom。
6. transferFrom(address, address, uint)，將數量為 num 的 Token 由 A 轉移給 B。

## 注意事項

Solidity 版本 >= 0.4.17

## Ref

-   [ERC20, ERC721 比較](https://medium.com/7sevencoin/erc20%E5%92%8Cerc721%E4%B8%8D%E4%B8%80%E6%A8%A3%E5%9C%A8%E5%93%AA%E8%A3%A1-2e550bb0bea3)
-   [ERC-20 標準 doc](https://eips.ethereum.org/EIPS/eip-20)
