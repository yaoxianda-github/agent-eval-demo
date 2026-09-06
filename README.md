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
  └─ Allure results 随 artifact 上传（本地 `allure generate results/allure-results` 出报告）
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

### 端到端结果（2026-09-06，minimal-react @ deepseek-chat）

| 阶段 | 场景 | 结果 | 关键证据 |
|---|---|---|---|
| 1 | main 分支门禁跑绿（Run #8/#9，修复 scripts 目录定位 + Pillow/python 软链后） | ✅ PASS | 4 任务 × 2 runs，gate 0.75 ≥ 0.75，约 1 分钟 / 约 ¥0.09 |
| 2 | 配置 branch protection：main 要求 `agent-eval demo gate`，且对管理员生效 | ✅ 生效 | Settings → Branches → rule 82797648；check 旁标 `Required` |
| 3 | **失败阻断**：PR #1 改坏 T106/T305/T306 校验（`feat: 调整超时上限与工龄条件`、`订单总额判定偏移+999`） | ✅ 阻断 | T305/T306 FAIL → gate 0.50 < 0.75 → check 红（1 failing）→ 合并按钮 `aria-disabled=true`，无法合并 |
| 4 | **修复放行**：PR #1 还原校验（`fix: 还原 T106/T305/T306 校验逻辑`） | ✅ 放行 | 门禁绿（succeeded 1m49s）→ PR 状态 `Ready to merge`，合并按钮恢复可用 |
| 5 | JUnit 报告 | ✅ | `agent-eval ci --junit-xml` 输出 checkpoint 级 testcase；test-reporter 发布 check（fail-on-error=false，阻断语义由 ci 退出码承担） |

### 门禁成本（demo gate = T106/T205/T305/T306 × 2 runs）

| Actions Run | 提交 | 结果 | 耗时 | token（in/out） | 成本 |
|---|---|---|---|---|---|
| #8 | abea172（+pillow） | PASS（T205 仍挂） | ~60s | 40,475 / 3,903 | ≈ ¥0.093 |
| #9 | da3361f（python 软链修复） | PASS | ~60s | — | ≈ ¥0.09 |
| #10 | 43842e9（PR 首版，改坏 T305） | PASS（0.75 恰好达标） | 50s | 32,281 / 3,598 | ≈ ¥0.075 |
| #11 | dacd4e2（PR 二版，改坏 T306） | **FAIL（gate 0.50）** | ~1m | — | ≈ ¥0.08 |
| #12 | 6251477（还原，PR 放行版） | PASS | 1m49s | — | ≈ ¥0.09 |

> 说明：单次 demo 门禁 ≈ 1 分钟、成本 < ¥0.1；正式卡口用 `ci/gate.yaml` 的 `core`（10 任务 × 3 runs，预计 3–5 分钟、约 ¥0.5–1.0）。
