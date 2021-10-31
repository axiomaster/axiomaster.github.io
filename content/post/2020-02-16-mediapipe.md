---
layout:     post
title:      mediapipe
subtitle:   
date:       2020-02-16
author:     axiomaster
header-img: img/post-bg-desk.jpg
catalog: true
tags:
    - ML
---

Google推出的将机器学习/深度学习进行产品化的框架，主要是其中类似tensorflow中图编辑模式。但我们只是看中了人家训练好的手部跟踪的demo。

使用bazel编译系统，但bazel这坑货又不支持proxy。那就需要搞明白所有的编译规则，然后把包挨个离线下下来，然后使用local_repo的方式进行编译。
