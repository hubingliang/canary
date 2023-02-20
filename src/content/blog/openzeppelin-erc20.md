---
author: Flaneur
pubDatetime: 2023-02-08T16:14:00Z
title: OpenZeppelin ERC20 源码解读
postSlug: openzeppelin-erc20
featured: true
draft: false
tags:
  - Solidity
  - OpenZeppelin
ogImage: ""
description: 本文对 OpenZeppelin ERC20 的源码做简单解析，基于 OpenZeppelin v4.8.0 版本。
---

本文对 [OpenZeppelin ERC20 的源码](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol) 做简单解析，基于 OpenZeppelin v4.8.0 版本。

## Table of contents

## 变量

```solidity
// 储存代币与地址的映射
mapping(address => uint256) private _balances;

// 储存地址与地址所授权代币的映射
mapping(address => mapping(address => uint256)) private _allowances;

// 代币总供给
uint256 private _totalSupply;

// 代币名称 例如 "MyToken"
string private _name;

// 代币符号 例如 “HIX”
string private _symbol;
```

## 事件

### Transfer

```solidity
/**
 * @dev 当代币（value）从一个帐户（from）转移到另一个账户（to）时触发
 *
 * 注意: 需要判断 0 地址的情况
 */
event Transfer(address indexed from, address indexed to, uint256 value);
```

### Approval

```solidity

/**
 * @dev 当代币（value）从一个账户 (owner) 授权给另一账户 (spender）时触发
 */
event Approval(address indexed owner, address indexed spender, uint256 value);
```

## 函数

### constructor

```solidity
/**
 * @dev 初始化代币名称和符号
 *
 * 默认精度为18位，如果要更改的话你应该重载这个函数。
 *
 * 这两个值都不可变: 它们都在初始化的时候设置好
 */
constructor(string memory name_, string memory symbol_) {
    _name = name_;
    _symbol = symbol_;
}
```

### totalSupply

```solidity
/**
  * @dev 返回代币总供给量
  */
function totalSupply() public view virtual override returns (uint256) {
    return _totalSupply;
}
```

### balanceOf

```solidity
/**
 * @dev 返回 account 账户拥有的代币
 */
function balanceOf(address account) public view virtual override returns (uint256) {
    return _balances[account];
}
```

### transfer

```solidity
/**
 *
 * - `to` 不能为 0 地址
 * - 调用者的余额至少有 amount 多的代币
 */
function transfer(address to, uint256 amount) public virtual override returns (bool) {
    address owner = _msgSender();
    _transfer(owner, to, amount);
    return true;
}
```

```solidity
/**
 * @dev 从 from 转移 amount 个代币给 to
 *
 * 触发 Transfer 事件
 *
 * 要求:
 *
 * - `from` 不能为 0 地址
 * - `to` 不能为 0 地址
 * - `from` 用者的余额至少有 amount 多的代币
 */
function _transfer(
    address from,
    address to,
    uint256 amount
) internal virtual {
    // 校验两个地址都是非 0 地址
    require(from != address(0), "ERC20: transfer from the zero address");
    require(to != address(0), "ERC20: transfer to the zero address");

    // 触发转账之前的 hook
    _beforeTokenTransfer(from, to, amount);

    // 判断调用者的余额至少有 amount 多的代币
    uint256 fromBalance = _balances[from];
    require(fromBalance >= amount, "ERC20: transfer amount exceeds balance");

    // 使用 uncheck 不校验溢出情况节省 gas 并进行转账操作
    unchecked {
        _balances[from] = fromBalance - amount;
        // 排除溢出的情况: 余额由 totalSupply 限制，先减再增的操作可以保证不出现溢出情况
        _balances[to] += amount;
    }

    // 触发 Transfer 事件
    emit Transfer(from, to, amount);

    // 触发转账之后的 hook
    _afterTokenTransfer(from, to, amount);
}
```

### allowance

```solidity
/**
 * 返回`owner` 给 `spender` 的授权额度
 * 允许通过调用 {transferFrom} 来代表 `owner` 进行消费。 默认值为 0
 * 当调用 {approve} 或 {transferFrom} 时，此值会更改。
 */
function allowance(address owner, address spender) public view virtual override returns (uint256) {
    return _allowances[owner][spender];
}
```

### approval

```solidity
/**
 * 授权 amount 数量的代币可以被 spender 支配。
 *
 * NOTE: 如果 `amount` 是 `uint256` 的最大值, allowance 不会被 transferFrom 更新，这相当于设置了无限授权额度。
 *
 * 要求:
 *
 * - `spender` 不能为 0 地址
 */
function approve(address spender, uint256 amount) public virtual override returns (bool) {
    address owner = _msgSender();
    _approve(owner, spender, amount);
    return true;
}
```

```solidity
/**
 * 授权 amount 数量的代币可以被 spender 支配。
 *
 * 触发 Approval 事件
 *
 * 要求:
 *
 * - `owner` 不能为 0 地址
 * - `spender` 不能为 0 地址
 */
function _approve(
    address owner,
    address spender,
    uint256 amount
) internal virtual {
  
    // 校验两个地址都是非 0 地址
    require(owner != address(0), "ERC20: approve from the zero address");
    require(spender != address(0), "ERC20: approve to the zero address");

    // 授权额度
    _allowances[owner][spender] = amount;

    // 触发 Approval 事件
    emit Approval(owner, spender, amount);
}
```
