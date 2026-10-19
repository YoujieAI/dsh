# YoujieAI/dsh：Youjie 维护的 dsh 分叉

上游是 [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)（dsh）。上游的 `CONTRIBUTING.md`
写明暂不接受外部 pull request，所以 Youjie 对 dsh 的补丁和修改提交在这个分叉里，同时持续跟上游。

## 分支与标签

| 引用 | 内容 | 规则 |
| --- | --- | --- |
| `master` | 上游 `master` 的镜像 | 只快进，不在上面提交 |
| `youjie`（默认分支） | 上游 + Youjie 的补丁与修改 | 上游更新以 merge 并入，不 rebase，已有提交号不变 |
| `dsh-v*` 标签 | 上游发布标签 | 原样镜像，不改不删 |

## 同步

[YoujieAI/dsh-sync](https://github.com/YoujieAI/dsh-sync) 每 6 小时运行一次，也可以手动触发：

1. 把 `master` 快进到上游 `master`；
2. 镜像上游的 `dsh-v*` 标签；
3. 把 `master` 合并进 `youjie`。

上游改写了历史（`master` 不能快进）或者合并有冲突时，什么都不推送，在 dsh-sync 开 issue 等人处理。
同步用只对本仓库有写权限的 deploy key，不用任何个人凭据。

本仓库的 GitHub Actions 是关闭的：上游的工作流（CI、发布等）不在这里运行。

## 补丁

基线是 `dsh-v0.2.1-rc.2`（`687f282e8`）。每个补丁一个提交，问题、复现、修复、风险、测试和已做的验证都写在
`youjie/patches/` 下的对应文件里。

| # | 提交 | 内容 | 说明 |
| --- | --- | --- | --- |
| 1 | `4da796a855` | llm-pi-ai: keep the model's text when pi-ai could not parse a tool call's arguments | [`01-pi-ai-keep-unparsable-tool-args.md`](youjie/patches/01-pi-ai-keep-unparsable-tool-args.md) |
| 2 | `c9f5f1d0b6` | apps/cli: declare the pinned zod so npm installs one copy, and make the install-layout gate see it | [`02-cli-pin-zod-dedupe.md`](youjie/patches/02-cli-pin-zod-dedupe.md) |
| 3 | `41ea5af309` | app-boot: report a profile patch whose target row does not exist | [`03-warn-unmatched-patch-row.md`](youjie/patches/03-warn-unmatched-patch-row.md) |
| 4 | `29faa0e831` | acp: offer the `_meta` of `session/new` and `session/resume` to plugins (optional) | [`04-acp-session-meta-event.md`](youjie/patches/04-acp-session-meta-event.md) |
| 5 | `3b7fe0d594` | llm-retry: state the billing effect of the default retry count and which adapters honour `Retry-After` | [`05-llm-retry-document-defaults.md`](youjie/patches/05-llm-retry-document-defaults.md) |

这些补丁没有提交给上游，因为上游目前不收外部 PR。如果上游以后收了等价的修复，合并时会冲突，按上游的实现解决即可。

## 怎么改

- 从 `youjie` 拉分支，PR 合回 `youjie`；一个补丁一个提交，并在 `youjie/patches/` 里写说明。
- 不要在 `master` 上提交，也不要改动上游的标签。

## 天工怎么用

天工的 `agent-kernel/` 目前仍然从 npm 安装 `@deepseek-ai/dsh@0.2.1-rc.2`（锁在 `agent-kernel/pnpm-lock.yaml`），
这些补丁针对的行为由天工的 profile 与插件规避。直接使用本分支构建的包需要另外发布，目前还没有做。
