# MemoryV1.3-Breathe

**给 DeepSeek Harness 桌面端加一个本地、可审计、可纠正的个人记忆容器。** 记忆以 Markdown 和 JSON 保存在本机。你可以查看记住了什么、确认模型提出的候选记忆，也可以撤销衰减。

[安装与回滚](INSTALL.md) · [公开预览版下载](https://github.com/TeddyHelena/Memory-V1.3-Breathe-1120/releases/tag/v0.16.3%2Bbreathe.1120.3) · [变更记录](CHANGELOG.md) · [English overview](#english-overview)

> 公开安装包为 `0.16.3+breathe.1120.3`。仓库 `main` 的包清单为 `0.16.3+breathe.1120.4`。项目处于预览阶段，接口、数据格式和界面可能变化。维护者记录的实测环境为 macOS + DSH 0.2.2-rc.3。

## 一条记忆怎样进入上下文

```mermaid
flowchart LR
    inference["模型提出的记忆"] --> candidate["本地候选区"]
    candidate -->|"默认由人工确认"| memory["正式记忆<br/>Markdown + JSON"]
    memory -->|"按当前场景检索"| context["按需纳入模型上下文"]
    memory -->|"达到半衰期"| decay["降层显示"]
    decay -->|"撤销降层"| memory
```

- **写入前可确认**。模型推断先进入候选区，默认由人工确认。待确认的条目不会进入检索或常驻注入。可选的 `autoPromote` 默认关闭。
- **使用时可追溯**。容器中的 `injection-ledger.jsonl` 记录记忆注入，便于追查条目的投递情况。
- **衰减后可恢复**。Breathe 按半衰期降层，保留原文，支持撤销。
- **在界面中查看和整理**。用列表、知识球或关系图浏览记忆，也可以搜索、编辑或手动组织记忆链。链操作有记录并支持回滚。

记忆容器保存在本机。选中的记忆可随上下文交给 DSH 配置的模型通道。如果使用远程模型，请按该模型服务的数据处理规则判断哪些内容适合进入上下文。备份文件也可能包含个人信息。

## 安装预览版

已有 DSH 记忆容器配置的 macOS 用户可以按 [安装指南](INSTALL.md) 下载并校验公开 ZIP、备份现有容器和 Profile，再切换插件。ZIP 已包含构建产物。

## 开发与许可

源码位于 [`src/`](src/)，维护脚本索引见 [`scripts/INDEX.md`](scripts/INDEX.md)。包清单中的构建与测试脚本有各自的开发依赖，安装发布包时无需运行。

本项目由 Yang Tianyu 原创并维护，以 [MIT License](LICENSE) 发布。Tabler Icons 的许可及使用边界见 [NOTICE.md](NOTICE.md)。请勿使用项目名称或作者名义为衍生品背书，或将项目包装为他人原创作品。

## English overview

MemoryV1.3-Breathe is a local personal memory plugin for DeepSeek Harness desktop. It stores memories as Markdown and JSON, holds model-inferred candidates for human review by default, and records memory injections in a local ledger. Breathe can lower a memory’s display tier as it ages while retaining the original entry. Browse memories in list, knowledge sphere, or graph views, and organize related entries into chains.

Selected memories can enter the context sent to the model configured in DSH. If that model is remote, its provider’s data handling rules apply to the request.

Start with the [installation and rollback guide](INSTALL.md) and the [public preview ZIP](https://github.com/TeddyHelena/Memory-V1.3-Breathe-1120/releases/tag/v0.16.3%2Bbreathe.1120.3). The documented test environment is macOS with DSH 0.2.2-rc.3.
