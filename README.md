# @macily.dots/dsh-mnemon-source-runtime

`dsh-mnemon-source-runtime` 的一个分支，给运行时记忆加上 **pinned 写保护**。依赖上游 `dsh-mnemon` 提供其余组件（策略、Documents、Memory Spaces）。

## 致谢

本项目基于 [omdsh-dev/dsh-mnemon](https://github.com/omdsh-dev/dsh-mnemon) 的 `dsh-mnemon-source-runtime` 包修改而来。上游采用 MIT 许可证，版权归 dsh-external contributors 所有（见 [LICENSE](LICENSE)）。pinned 的原始需求来自上游 [issue #515](https://github.com/omdsh-dev/dsh-mnemon/issues/515)。本分支新增的是对模型/维护通道的只读拦截，不属于上游功能。

## 它解决什么

上游的 pinned 只保证"容量整理不归档 + 每轮快照必现"，但模型仍能用 `mnemon_runtime_memory` 替换一条已固定条目的内容、而 pin 标记还留着。本分支在 `prepareMutation` 里加了硬拦截：

- 模型通道与后台维护通道对 pinned 条目的 `replace` / `remove` 直接拒绝，错误含 `Do NOT retry` 避免模型反复重试；
- 记忆系统页面的管理通道仍可解锁、编辑；
- 拦截发生在任何文件写入之前，失败时 `memories.json` 字节不变。

## 安装

先确保已安装官方 `dsh-mnemon`（DSH Starter 默认带）。本包**未发布到 npm**，需要从 GitHub 源码安装：

```sh
git clone https://github.com/MacilyDots/dsh-mnemon-source-runtime.git
```

然后在 DSH 插件页：**插件 → 添加插件 → 本地目录 → 选刚 clone 的仓库根目录**。

安装后重启 DSH。本包自带的 `cordis.patch.yml` 会自动完成：

- 新增一个独立的运行时候选 Source（entry id `guard-runtime`），使用全新数据目录，不碰你现有的记忆；
- 禁用官方 runtime，避免两个工作记忆源同时注入和双写；
- 禁用技能管理，记忆空间只保留本地 `mnemon-native` provider；
- 用 scoped 策略选中本 Source。

打开记忆系统，看到 **Runtime Memory (Guard)** 即表示生效。

## 数据

- 数据目录由 entry id 决定：`~/.mnemon/sources/source%3Aguard-runtime/`，全新且独立。
- 不读取、不迁移你现有的 `~/.mnemon/runtime`。要复用旧数据须显式配置 `dataDir` / `userDataDir` 绝对路径（并发安全未验证，一般不需要）。

## 升级

上游 `dsh-mnemon` 升级后，若 entry 列表变化，`cordis.patch.yml` 里的覆盖项需要重新核对（patch 是整块替换 config，不深合并）。源码改动本身不受上游升级影响，因为这是一个独立包。

## 验证记录

- `tests/controller.spec.ts`：6 条 pinned 核心测试通过（模型/维护通道拦截、planMaintenance、compact、user 目标、`add` 触发压缩回归）；唯一失败项 `never follows a link at the local archive` 是 Windows 符号链接权限（EPERM），与本改动无关，上游同样失败。
- 隔离 headless 环境：bundle 正常装载，`mnemon_*` 工具齐全，组合配置无 loader 报错。
- 桌面端实测：对 pinned 条目发起 `replace` 返回精确拒绝文本，`memories.json` SHA-256 前后一致。

## 许可

MIT。保留上游 `dsh-mnemon-source-runtime` 的版权声明（见 [LICENSE](LICENSE)）。
