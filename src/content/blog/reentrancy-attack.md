---
author: Flaneur
pubDatetime: 2023-02-08T16:14:00Z
title: 图解 Solidity 安全：重入攻击（reentrancy attack）
postSlug: reentrancy-attack
featured: true
draft: false
tags:
  - Solidity
  - security
  - Hack
ogImage: "https://res.cloudinary.com/dewu7okpv/image/upload/v1675828572/blog/canary/%E6%97%A0%E6%A0%87%E9%A2%98-2023-01-29-1431_s4awrl.png"
description: 重入攻击是指恶意合约在对目标合约的提款调用执行过程中，不等执行结束再通过调用该合约的函数对其进行二次攻击，从而破坏原本的执行流程，达到攻击目的的技术。
---

Solidity 中的重入攻击会反复从智能合约中提取资金并将其转移到未经授权的合约中，直到被攻击合约的资金耗尽。

下面的文章解释了重入攻击的原理、两种不同类型的重入攻击，以及 Solidity 开发人员可以采取的预防措施，以保护智能合约免受以太坊和 Solana 区块链漏洞的侵害。

## Table of contents

## 什么是重入攻击（reentrancy attack）?

重入攻击是指恶意合约在对目标合约的提款调用执行过程中，不等执行结束再通过调用该合约的函数对其进行二次攻击，从而破坏原本的执行流程，达到攻击目的的技术。

![](https://res.cloudinary.com/dewu7okpv/image/upload/v1675764495/blog/canary/%E6%97%A0%E6%A0%87%E9%A2%98-2023-01-29-1431_zxqhdp.png)

## 重入攻击（reentrancy attack）是如何工作的？

![](https://res.cloudinary.com/dewu7okpv/image/upload/v1675828572/blog/canary/%E6%97%A0%E6%A0%87%E9%A2%98-2023-01-29-1431_s4awrl.png)

1. Attack 合约先去调用 Bank.deposit() 转入部分资金。
2. 之后 Attack 合约调用 Bank.withdraw() 判断是否有能够提取的资金，然后将资金转移到 Attack 合约。
3. Bank.withdraw() 判断提款人的余额是否满足提款要求，并进行转账，然后去更新余额。
4. 当 Attack 合约收到资金，就会再次执行 Bank.withdraw() 去提取资金，导致上一次的 Bank.withdraw() 执行搁置不能进行余额更新操作，由于此时余额还没更新，可以再次提款形成递归。

### 代码示例

```solidity
contract Bank {
    mapping(address => uint256) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external payable {
        require(balances[msg.sender] > 0, "Not enough balance");

        (bool success, ) = msg.sender.call{value: balances[msg.sender]}("");
        require(success, "Failed to transfer ETH");

        balances[msg.sender] = 0;
    }
}
```

```solidity
contract Attack {
    Bank public bank;

    constructor(Bank _bank) {
        bank = _bank;
    }

    receive() external payable {
        bank.withdraw();
    }

    function attack() external payable {
        bank.deposit{value: 1 ether}();
        bank.withdraw();
    }
}
```

## 重入攻击（reentrancy attack）有哪些不同类型？

重入攻击有两种类型：**单一函数重入攻击**和**跨函数重入攻击**。

### 单一函数重入攻击

当受攻击合约的函数与恶意合约试图递归调用的函数相同时，就是单次重入攻击（与上述例子相同）。与跨函数重入攻击相比，单一函数重入攻击更简单、更容易预防。

### 跨函数重入攻击

只有当受攻击合约的函数与另一个对攻击者有利的函数共享状态时，跨函数重入攻击才是可行的。跨函数重入攻击更难检测和预防。

## Solidity 重入攻击（reentrancy attack）案例

以下著名的案例进一步说明了黑客如何利用区块链协议中的漏洞进行重入攻击。

### DAO (2016)

> DAO 是一个去中心化的自治组织。 它的目标是将组织的规则和决策机制编纂成法典，消除对文件和人员进行治理的需要，创建一个分散控制的结构。

2016 年 6 月 17 日，The DAO 遭到黑客攻击，360 万以太币（5000 万美元）在第一次重入攻击中被盗。以太坊基金会发布了一个重要更新来回滚黑客攻击。 这导致以太坊被分叉为以太坊经典和以太坊。

### Lendf.me Protocol (2020)

2020 年 4 月，一个黑客利用 Lendf.me 协议的重入攻击窃取了 2500 万美元，Lendf.me 协议是一种用于在以太坊网络上进行借贷操作的去中心化金融协议。

协议开发人员忽略了一个漏洞，即 ERC-777 代币包含一个回调函数，该函数会在发送或接收资金时通知用户。黑客通过将恶意智能合约设置为接收方并利用该漏洞耗尽 Lendf.me 协议 99.5% 的资金。

### Cream Finance Hack (2021)

2021 年 10 月，一名黑客通过对该协议的“闪电贷”功能进行重入攻击，窃取了价值超过 1.3 亿美元的 ERC-20 和 CREAM 流动性协议 (LP) 代币。利用的根本原因是 AMP 错误地集成到 CREAM 金融协议中。

## 怎样预防重入攻击（reentrancy attack）

### Checks, Effects, Interactions

Checks, Effects, Interactions（CEI）是防止重入攻击的基本原则。

- Checks 是指判断是否满足条件
- Effects 是指调用导致的状态修改
- Interactions 是指函数或合约之间的交易

以 CEI 的顺序来执行是防止重入攻击的关键，对于上面的例子来说，应当先改变取款人的余额再去调用转账交易。

```solidity
function withdraw() external payable {
    require(balances[msg.sender] > 0, "Not enough balance");
    balances[msg.sender] = 0; // 操作提前
    (bool success, ) = msg.sender.call{value: balances[msg.sender]}("");
    require(success, "Failed to transfer ETH");
}
```

### 重入锁

通常的解决方案是加入 [mutex](https://en.wikipedia.org/wiki/Mutual_exclusion) ,智能合约重入锁是一种在智能合约中使用的锁定机制，用于防止多个交易同时修改智能合约的状态，从而保证智能合约的一致性。在易受重入攻击的函数调用上下文放置一个布尔锁，初始状态为 false，在它易受重入攻击的函数执行开始之前设置为 true，然后在终止后设置回 false。

![](https://res.cloudinary.com/dewu7okpv/image/upload/v1675842309/blog/canary/2_tusx7i.png)

```solidity
contract Bank {
    mapping(address => uint256) public balances;
    bool private locks;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external payable {
        require(!locks, "Balances locked!");
        locks = true;

        require(balances[msg.sender] > 0, "Not enough balance");
        (bool sent, ) = msg.sender.call{value: balances[msg.sender]}("");
        require(sent, "failed to transfer ETH");

        balances[msg.sender] = 0;

        locks = false;
    }
}

```

### Pull Payment

更安全的端到端交易是使用 Pull Payment 支付方法。 Pull Payment 流程要求使用中介托管来发送资金，避免与潜在的恶意合约直接交互。

## 参考文章

1. [https://consensys.github.io/smart-contract-best-practices/attacks/reentrancy/](https://consensys.github.io/smart-contract-best-practices/attacks/reentrancy/)
2. [https://www.alchemy.com/overviews/reentrancy-attack-solidity](https://www.alchemy.com/overviews/reentrancy-attack-solidity)
3. [https://dev.to/peterblockman/smart-contract-security-an-illustrated-guide-to-re-entrancy-attack-2f3p](https://dev.to/peterblockman/smart-contract-security-an-illustrated-guide-to-re-entrancy-attack-2f3p)
