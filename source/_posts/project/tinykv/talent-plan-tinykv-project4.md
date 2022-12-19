---
title: Talent plan Tinykv | project4 | Transaction
categories:
  - project
  - tinykv
tags:
  - tinykv
  - raft
mathjax: false
date: 2022-01-10 01:00:01
---


# Project4

## Project4a

#### 实验目的

实现多版本并发控制

#### 实验思路

本部分实现一个多版本并发控制(MVCC)，简单来说，就是在底层DB存储多个版本的KV。

在本次实验中，底层有三个列族，`default`保存数据，`lock`存储锁，`write`记录更改的操作。`lock`使用key访问，`default`利用key和被写入的事务的时间戳访问，`write`利用key和事务被提交时写入的时间戳访问。

在本次实验中，我们需要完成get、delete、write数据和锁的函数，以及`CurrentWrite()`返回当前事务commit的写入，`MostRecentWrite()`返回给定key的最近一次commit。

需要注意以下几点;
1. delete和write操作，需要将操作封装为`storage.Modify`，然后附加到`txn.writes`数组中

2. `GetValue`首先从`write`列族中，寻找出距离此事务最近的版本的commit，然后从DB中读取到此写入的开始时间戳，再从`default`列族读取数据

## Project4b

#### 实验目的

利用project4a实现的mvcc，完成`KvGet `，`KvPrewrite`和`KvCommit`函数。

#### 实验思路

`KvGet`首先利用请求信息，创建一个`mvccTxn`对象，再判断key有没有上锁，如果检测到key被其他事务锁住，则返回`KeyError`。然后再利用`GetValue`获取值。

`KvPrewrite`时，需要对请求中的key，跳过那些已经被lock和已经被更晚的事务commit的key。然后利用mvcc中的函数，写入数据并上锁。

`KvCommit`需要判断`lock`列族的锁时间戳是否与当前事务的写入时间一致，一致的情况下，从DB中删除此锁，并再`write`列族记录commit。

## Project4c

#### 实验目的

实现`KvScan`，`KvCheckTxnStatus`，`KvBatchRollback`和`KvResolveLock`，以及`scanner.go`。

#### 实验思路

`scanner.go`是`KvScan`的一个底层，它用来迭代DB，返回对应的key和value。`KvScan`利用`scanner`对象，从DB中Scan数据。

`KvCheckTxnStatus` 检查超时，删除过期的锁并回滚，但是如果发现`MostRecentWrite`的事务为回滚事务或commit时间大于当前lock的时间，那么说明现有的锁为新事务的锁，直接返回。

`KvBatchRollback`对所有需要回滚的key，如果发现`MostRecentWrite`的事务为回滚事务或commit时间大于当前时间戳，说明已经回滚。回滚时需要从`lock`列族删除锁，从`default`删除key的value，还需要从`write`列族写入回滚操作。

`KvResolveLock`首先检查key的lock是否存在，然后按照请求，调用`KvBatchRollback`或`KvCommit`函数完成回滚或commit操作。
