---
title: 【玩转EOS】转账、抵押等操作需要消耗多少net和cpu资源？
date: 2018-07-21 23:16:24
tags: cpu net
---

在EOS网络里面进行转账操作时，相信不少人都会遇到过因net或cpu资源不足，导致转账失败的情况。本文就结合实际的转账操作，分析一下转账EOS大致需要多少net和cpu资源。

## 写在前面
本文仅从实际使用者的角度，通过重复多次执行相同的操作，观察net和cpu资源的使用情况，从而统计分析出不同转账方式，所消耗的资源数量。下面会先给出分析结果，文末会附上试验过程数据。

## 资源消耗统计分析

下面表格展示了 **转账** 和 **抵押** 操作平均消耗的资源。**转账时所带memo不同，消耗的资源也不同，而转账金额的多少不影响消耗** 。

|操作方式|命令示意|net消耗(KiB)|cpu消耗(ms)|
|--|--|--|--|
|【转账】不带memo|cleos transfer 账户A 账户B '0.0001 EOS' ''|0.124|0.884|
|【转账】带短memo|cleos transfer 账户A 账户B '0.0001 EOS' '感谢！'|0.133|1.018|
|【转账】带最长memo|cleos transfer 账户A 账户B '0.0001 EOS' '1111…<256个1>'|0.374|1.306|
|【转账】带中文memo|cleos transfer 账户A 账户B '0.0001 EOS' '测测测…<85个测>'|0.374|1.046|
|【抵押】换net和cpu|cleos system delegatebw 账户A 账户A "0.01 EOS" "0.01 EOS" |0.140|2.7671|

> 转账eos所带memo规定不能超过 256 bytes。以utf-8编码的中文字符占 3 bytes，所以在memo中的中文字符不能超过85个。

> 抵押eos换取的资源数量是不固定的，会根据全网资源使用情况而变化，可以通过[这里](https://explorer.eoseco.com/)查看实时的资源成本

## 试验数据

1【转账】不带memo
```
cleos transfer 账户A 账户B '0.0001 EOS' ''
```
![image](https://user-images.githubusercontent.com/8636635/43038566-051bd878-8d4e-11e8-8dac-1b4f7d7ce163.png)

2【转账】不同金额
```
cleos transfer 账户A 账户B '0.1 EOS' ''
```
![image](https://user-images.githubusercontent.com/8636635/43038788-9d812aec-8d52-11e8-8185-5c138839ccda.png)

3【转账】带短memo
```
cleos transfer 账户A 账户B '0.0001 EOS' '感谢！'
```
![image](https://user-images.githubusercontent.com/8636635/43038747-dc6a3f1a-8d51-11e8-9236-762ba9a89285.png)

4【转账】带最长memo
```
cleos transfer 账户A 账户B '0.0001 EOS' '1111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111'
```
![image](https://user-images.githubusercontent.com/8636635/43038580-72199d20-8d4e-11e8-9bc4-00747ab10fe9.png)

5【转账】带中文memo
```
cleos transfer 账户A 账户B '0.0001 EOS' '测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测测'
```
![image](https://user-images.githubusercontent.com/8636635/43038677-706d87b4-8d50-11e8-88da-ffc4573441e3.png)

6【抵押】换net和cpu
```
cleos system delegatebw 账户A 账户A "0.01 EOS" "0.01 EOS"
```
![image](https://user-images.githubusercontent.com/8636635/43038799-fd30d078-8d52-11e8-923a-a9a82abb7b02.png)