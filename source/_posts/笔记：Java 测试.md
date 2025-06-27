---
title: 笔记：Java 测试
date: 2025-05-12
categories:
  - Java
  - 测试框架
tags: 
author: 霸天
layout: post
---

```
@Test
@Transactional
@Rollback(value = false) // juint 默认回滚事务，所以想提交事务就设置为false这是在juint 中，其他的可以不用
public void testSendMessageInTx() {
    // 1、发送第一条消息
    rabbitTemplate.convertAndSend(EXCHANGE_NAME, ROUTING_KEY, "I am a dragon(tx msg ~~~01)");

    // 2、抛出异常
    log.info("do bad:" + 10 / 0);

    // 3、发送第二条消息
    rabbitTemplate.convertAndSend(EXCHANGE_NAME, ROUTING_KEY, "I am a dragon(tx msg ~~~02)");
}
```
