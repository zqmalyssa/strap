---
layout: post
title: 调度任务
tags: [java, code]
author-id: zqmalyssa
---

Java项目开发的一些调度任务的总结

#### 一些对比

有Spring Schedule，qurtz，powerjob，xxl-job，Elastic-job，power-job等等


Spring Schedule必须有@Scheduled注解
支持cron、不支持持久化、开发非常非常非常简单、能分布式吗

quartz：支持cron、支持持久化、开发复杂，支持分布式


#### powerjob
