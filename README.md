# pi-study

跟着源码学 **Pi**（[earendil-works/pi](https://github.com/earendil-works/pi)）—— 一个 TypeScript 写的 AI Agent 框架。

目标不是「看懂就算」，而是**从零把它重新搭一遍**：先跑通原仓库，再按最小粒度拆功能、逐个复刻，最后得到一份目录结构对得上的实现。过程中产生的所有笔记、实验代码、教程都沉淀在这个仓库里。

## 目录结构

```text
pi-study/
├── AGENTS.md            # 学习工作区的背景说明书（给 AI 协作者看的上下文入口）
├── c-pi/                # 学习自留地：笔记 + 实验 + 复刻代码，本仓库的主体
│   ├── docs/            # 主线文档站：Pi 怎么设计与实现（VitePress）
│   ├── wiki/            # 支线知识库：读源码撞到的基础知识，一个主题一个站
│   │   ├── shell/       #   ├ pi-test.sh 陪读教程
│   │   └── js-ts/       #   └ JS/TS 语法扫盲
│   ├── labs/            # 所有动手实验的落脚点（可随时整体删除，命令会重建）
│   └── package.json     # npm workspaces 入口，统一驱动上面几个站
└── pi22/                # 上游仓库的 fork，只读参考，已被 .gitignore 忽略
```

> `pi22/` 是为了对照和运行原始代码而克隆的，不参与本仓库的版本管理；本仓库只保存**自己写的东西**。

## 快速开始

需要 Node.js 18+ 和 npm。

```bash
cd c-pi
npm install          # 一次装好 docs 与 wiki/* 各站的依赖（workspaces）

npm run dev:docs     # 主线：Pi 学习笔记
npm run dev:shell    # 支线：pi-test.sh 陪读教程
npm run dev:js-ts    # 支线：JS/TS 语法扫盲
```

构建静态站点：把 `dev:` 换成 `build:`，产物在各站的 `.vitepress/dist`。

想跑上游原始仓库，先自行克隆（本仓库不含它）：

```bash
git clone https://github.com/earendil-works/pi.git pi22
```

## 阅读顺序

1. **先跑起来** —— `c-pi/docs/01-run-pi.md`：环境检查、装依赖、源码启动、首次登录。
2. **再看懂启动链路** —— `c-pi/docs/02-run-to-ui.md`：`./pi-test.sh` 执行之后到底发生了什么。
3. **补基础** —— 卡在 Shell 或 JS/TS 上时去 `c-pi/wiki/` 对应的支线站，读到哪补到哪，不提前学。
4. **动手** —— 每讲笔记里的实验命令都落在 `c-pi/labs/`，命令用绝对路径，粘贴即可跑。

功能全景与复刻路线图：`c-pi/docs/pi-功能全景图.md`。

## 这份笔记的写法

- **落到行号**：讲代码必须标出「文件路径:行号」并引用原文，方便直接跳过去核对。
- **最小粒度**：一次只讲一个点，一个点一篇笔记，能独立跑通再往下走。
- **先全局后局部**：动手复刻前，先把原仓库的功能清单、运行方式、目录结构摸清楚。
- **可丢弃的实验**：`labs/` 里的东西都是临时的，删掉不影响任何文档。

## 进度

- [x] 工作区初始化与背景说明（`AGENTS.md`）
- [x] `c-pi/docs` 初始化为 VitePress 文档站
- [x] 跑通原始仓库并记录启动链路
- [x] 支线：`pi-test.sh` 陪读教程（13 讲）、JS/TS 语法扫盲（12 讲）
- [ ] 梳理功能清单与功能点地图
- [ ] 按最小粒度开始复刻，逐步对齐原仓库目录结构

## 说明

仓库里的 `pi22/`、`node_modules/`、各站构建产物与缓存均不入版本管理，规则见 [`.gitignore`](./.gitignore)。
