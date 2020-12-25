---
title: 'MySQL - LOCKS 锁的类型'
date: '2020-11-28'
description: ''
author: 'dashjay'
---

## MySQL 有哪些锁

- 共享/排它锁(Shared and Exclusive Locks)
- 意向锁(Intention Locks)
- 记录锁(Record Locks)
- 间隙锁(Gap Locks)
- 临键锁(Next-key Locks)
- 插入意向锁(Insert Intention Locks)
- 自增锁(Auto-inc Locks

### 1. 共享/排它锁(Sharedand Exclusive Locks)

读取数据加共享锁，读读之间可以并行。排他锁与任何锁都互斥。

### 2. 意向锁(Intention Locks)

- 意向锁是一个表级别的锁
- 分为两种类型
  - 意向共享锁 `select ... lock in share mode;`
  - 意向排他锁 `select ... for update;`

### 3. 记录锁

### 4. 间隙锁

```sql
select * from lock_example where id between 8 and 15 for update;
```

### 5. 临键锁（Next-key Locks）

### 6. 插入意向锁(Insert Intention Locks)

### 7. 自增锁(Auto-inc Locks)

### 总结

按锁的互斥程度来划分，可以分为共享、排他锁；

- 共享锁(S锁、IS锁)，可以提高读读并发；
- 为了保证数据强一致，InnoDB使用强互斥锁(X锁、IX锁)，保证同一行记录修改与删除的串行性；

按锁的粒度来划分，可以分为：

- 表锁：意向锁(IS锁、IX锁)、自增锁；
- 行锁：记录锁、间隙锁、临键锁、插入意向锁；

其中有一些细节

- InnoDB的细粒度锁(即行锁)，是实现在索引记录上的(我的理解是如果未命中索引则会失效)
- 记录锁锁定索引记录；间隙锁锁定间隔，防止间隔中被其他事务插入；临键锁锁定索引记录+间隔，防止幻读；
- InnoDB使用插入意向锁，可以提高插入并发；
- 间隙锁(gap lock)与临键锁(next-key lock)只在RR以上的级别生效，RC下会失效；