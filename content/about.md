---
title: "关于"
layout: "single"
---

这里是我在折腾 homelab 和云原生基础设施过程中的踩坑记录。

## 资质

**RHCA**（红帽认证架构师，基础设施方向）
[验证链接](https://www.redhat.com/rhtapps/services/verify?certId=240-064-477) · 证书编号 `240-064-477`

- EX442 — Linux 性能调优
- EX436 — 高可用集群（Pacemaker/Corosync/GFS2）
- EX374 — Ansible Automation Platform
- EX260 — Ceph 云存储
- EX358 — 服务治理与自动化

## 现在在搞什么

- **可观测平台**：基于 Prometheus + Loki + Grafana 的监控告警链路，已接入钉钉通知
- **K8s 实验环境**：用 Ansible 从裸机拉起 3 节点集群，计划把监控栈迁上去
- **博客即作品集**：把排查过程写成文章，面试时有东西讲

## 记录原则

- **从实际问题出发**：不背八股，只记自己踩过的坑和怎么爬出来的
- **可复现**：环境信息、版本号、配置都保留，半年后自己能看懂
- **闭环**：现象 → 排查 → 根因 → 验证，每篇都走完这个流程

## 相关项目

- [homelab-observability](https://github.com/wamudus/node-observability-baseline)：Docker Compose 一键部署的监控告警基线

---

*最后更新：2026-04-24*
