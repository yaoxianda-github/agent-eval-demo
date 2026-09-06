# agent-eval demo —— CI 质量门禁 · 合并阻断验证

演示仓库：把 [ai-agent-eval](https://github.com/yaoxianda-github/ai-agent-eval) 的
`agent-eval ci` 无头门禁接入 GitHub Actions，验证**核心任务包通过率不达标自动阻断代码合并**链路。

## 链路

```
PR / push
  └─ GitHub Actions: agent-eval ci --gate demo（minimal-react @ deepseek-chat）
       ├─ 通过 → job 绿 → required check 通过 → 允许合并
       └─ 不达标 → 退出码 1 → job 红 → branch protection 阻断合并
  └─ dorny/test-reporter：JUnit 报告评论到 PR
  └─ allure-report-action：生成 Allure 报告
```

## 仓库内容

| 路径 | 说明 |
|---|---|
| `tasks/` | 评测集（21 任务，与主仓库一致） |
| `ci/gate.yaml` | `demo`（4 任务 × 2 runs，快）与 `core`（10 任务 × 3 runs，正式卡口） |
| `.github/workflows/agent-eval.yml` | 门禁 workflow |

## 使用方法

1. 仓库 Settings → Secrets and variables → Actions → 添加 `DEEPSEEK_API_KEY`
2. Settings → Branches → 为 `main` 添加保护规则，勾选 **Require status checks**，
   选择 `agent-eval demo gate`
3. 提交 PR：门禁自动运行并评论 JUnit 结果；不达标时合并按钮被阻断

## 验证记录

| 日期 | 场景 | 结果 |
|---|---|---|
| 2026-09-06 | 正常 PR（门禁 PASS） | 放行 |
| 2026-09-06 | 故意改坏 spec（门禁 FAIL） | 阻断合并 |
