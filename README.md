---
layout: default
title: Home
nav_order: 1
---

# 清华OS实习开发日志

## 实习时间：

2026年1月20日-2026年4月20日

## 实习任务目标：

对比AlloyStack与StarryOS，为StarryOS设计系统模块按需加载，提升其内核性能。

附：AlloyStack仓库链接：https://github.com/tanksys/AlloyStack

## 工作实施计划：

自2026.1.25始

1. week1--week2 阅读StarryOS代码并运行，了解StarryOS系统启动和模块加载逻辑。
2. 尝试构建模块按需加载逻辑：
   * 裁剪与系统非耦合模块并启动内核
   * 构建按需加载系统
   * 构建测试应用，测试系统按需加载卸载模块功能
   * 测试性能
3. 如果构建按需加载模块功能成功，后面尝试在Axvisor中迁移该功能，支持多核甚至多系统场景。

## 日志记录

