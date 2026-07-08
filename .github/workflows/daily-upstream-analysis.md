---
name: Daily Upstream Sync + Request-Flow Analysis
description: 每天检查上游 anomalyco/opencode 是否有更新;若有则快进同步本 fork 的 dev,并用 GitHub Copilot 分析 OpenCode 的请求流程与构造的请求 Header,产出一个 GitHub Issue。
on:
  schedule:
    - cron: "0 0 * * *"
  workflow_dispatch:
    inputs:
      force_analysis:
        description: "即使上游无更新也强制运行分析(用于手动测试)"
        type: boolean
        default: false

engine: copilot

# 仅当 sync job 检测到上游有更新、或手动触发并勾选 force_analysis 时,才运行分析 agent
# (否则整个 agent job 被跳过,零 AI 成本、不产生任何 Issue)。
if: ${{ needs.sync.outputs.updated == 'true' || github.event.inputs.force_analysis == 'true' }}

permissions:
  contents: read

network: defaults

timeout-minutes: 25

checkout:
  ref: dev
  fetch-depth: 50

tools:
  bash:
    - "grep"
    - "rg"
    - "cat"
    - "ls"
    - "find"
    - "head"
    - "tail"
    - "sed"
    - "awk"
    - "wc"
    - "git log"
    - "git show"
    - "git diff"

jobs:
  sync:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    outputs:
      updated: ${{ steps.sync.outputs.updated }}
    steps:
      - name: Sync fork dev from upstream (anomalyco/opencode)
        id: sync
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          set -euo pipefail
          # 记录同步前 dev 的 SHA
          before="$(gh api "repos/${GITHUB_REPOSITORY}/git/ref/heads/dev" --jq .object.sha)"
          echo "dev before sync: $before"
          # 用 GitHub 的 merge-upstream(fast-forward only,不加 --force,分叉时会失败)
          # 从本 fork 的 parent(anomalyco/opencode)同步 dev 分支。
          if gh repo sync "${GITHUB_REPOSITORY}" --branch dev 2> sync_err.txt; then
            after="$(gh api "repos/${GITHUB_REPOSITORY}/git/ref/heads/dev" --jq .object.sha)"
            echo "dev after sync:  $after"
            if [ "$before" != "$after" ]; then
              echo "上游有更新,已快进同步:$before -> $after"
              echo "updated=true" >> "$GITHUB_OUTPUT"
            else
              echo "已是最新,无更新。"
              echo "updated=false" >> "$GITHUB_OUTPUT"
            fi
          else
            echo "gh repo sync 失败(通常是无可同步或已分叉),跳过本次分析:"
            cat sync_err.txt || true
            echo "updated=false" >> "$GITHUB_OUTPUT"
          fi

safe-outputs:
  create-issue:
    title-prefix: "[opencode 请求流程分析] "
    labels: [analysis, automated]
    max: 1
---

# OpenCode 请求流程与请求 Header 分析

你正在分析 **opencode** 代码仓库(已检出到当前工作目录,分支 `dev`,并已与上游 `anomalyco/opencode` 完成同步)。本次运行仅在检测到上游有更新时才会执行分析。

## 任务

详细梳理 opencode 向 LLM provider 发起请求的**完整请求流程**,以及为每个 provider **构造的 HTTP 请求 Header**,产出一份结构化的中文分析报告,并作为 GitHub Issue 发布。

## 分析范围与建议起点

优先阅读以下位置(可用 `grep`/`rg` 进一步定位符号与调用点):

- `packages/llm/` —— 核心请求 / 路由 / 传输层
  - `src/route/`:`client.ts`、`executor.ts`、`auth.ts`、`transport/http.ts`、`transport/websocket.ts`
  - `src/providers/`(如 `anthropic.ts` 等)、`src/protocols/`(如 `anthropic-messages.ts`、`utils/bedrock-auth.ts`)
  - `AGENTS.md`、`DESIGN.md`
- `packages/opencode/src/session/llm/`:`request.ts`、`native-request.ts`、`native-runtime.ts`
- `packages/opencode/src/provider/`:`provider.ts`、`auth.ts`

以仓库实际代码为准,不要臆测;若某 provider/协议在仓库中不存在则不必强行覆盖。

## 报告内容要求(作为 Issue 正文,使用中文)

1. **请求流程(时序)**:从会话 / prompt 到最终对 provider 发出的 HTTP 请求,分步说明关键环节(provider/model 解析、鉴权凭据获取、请求体构造、协议适配、传输发送、错误重试)。每一步标注关键文件与函数(`path:line` 或 `path` + 符号名)。适当使用 mermaid 时序图。
2. **请求 Header 明细**:按 provider / 协议(覆盖仓库中实际存在者,例如 Anthropic、OpenAI / OpenAI 兼容、Bedrock、GitHub Copilot 等)分组,用表格列出构造的 Header:`Header 名称` | `取值来源(代码位置)` | `作用 / 说明`。重点覆盖鉴权类 Header(`Authorization`、`x-api-key`、`anthropic-version`、`User-Agent` 等)。
3. **鉴权机制**:各 provider 如何获取并注入凭据(API key / OAuth / Bedrock SigV4 等),给出对应代码位置。
4. **关键代码位置索引**:列出本次分析引用的主要文件与行号。
5. **本次变更提示**:运行 `git log --oneline -20` 查看近期同步进来的提交;若其中包含与请求流程 / Header 相关的变更,请在报告开头简要说明受影响之处。

## 输出方式

- 只读分析,**不要修改任何代码**。
- 通过 `create-issue` 安全输出发布报告:标题形如「OpenCode 请求流程与请求 Header 分析(YYYY-MM-DD)」,正文为上述 Markdown 报告。
