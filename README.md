# Secret Scan Triage · GitHub Action

> Your scanner reports 28 "leaks". This action tells you **which ones are real** — and fails the build only when something actually needs rotating.

```yaml
- name: Generate scanner report
  run: gitleaks detect --report-format json --report-path gitleaks.json || true

- name: Triage it
  uses: zjdyzww/secret-triage-action@v1
  with:
    report: gitleaks.json
    fail-on-real: 'true'
```

That's the whole integration. No account, no token, no upload — the triage runs on the runner.

## What it does

1. Clones the triage tool (`zjdyzww/ai-code-health`, MIT-less but public source you can read).
2. Reads **your** scanner's report: gitleaks JSON / SARIF / any text with `path:line`.
3. Classifies every finding into 6 buckets:
   `test fixture` · `scanner rule file` · `doc/template` · `build artifact` · `detector source` · `knowledge snapshot`
4. Writes `secret-triage-report.md`, puts a summary in the Job Summary, and sets outputs `real` / `triaged`.
5. Fails only if `real > 0` and `fail-on-real` is `true`.

## Inputs

| Input | Default | Meaning |
|---|---|---|
| `report` | *(required)* | Path to the scanner report |
| `repo-root` | `.` | Root used to resolve paths |
| `fail-on-real` | `true` | Fail the step on real findings |
| `output` | `secret-triage-report.md` | Where the markdown goes |

## Outputs

| Output | Meaning |
|---|---|
| `real` | findings judged real |
| `triaged` | findings excluded by context |

## Measured on real public repos

| Sample | Files | Candidates | Real |
|---|---|---|---|
| 9 security-tool repos | 240 | 33 (all their own rule files) | **0** |
| 10 mainstream libraries | 2,242 | 2 (test fixtures) | **0** |

Data: `github.com/zjdyzww/secret-fp-census`

## What this is NOT

- Not a scanner. Bring your own (gitleaks / Semgrep / custom).
- Not a pentest, not an audit, no compliance claim.
- 6 pattern families only — a clean run does not mean your repo is safe.
- **"Could not run" is exit 3, not a failure** — the tool refuses to call a broken environment a security problem.

## Why this exists

Because "your scanner reported X" is the least useful sentence in security. Knowing which of the X matter is the whole job.

## 已验证在真实 runner 上可运行（不是"应该能跑"）

工作流：`zjdyzww/ai-code-health` 的 `self-test-action.yml`（本仓库自己当第一个用户）
运行结果：**success**
```
断言通过：真泄露 1 条、语境排除 2 条，报告含正确条目
```
输入是刻意构造的三条：`rules/aws.yml`（规则文件→排除）、`tests/redact.spec.ts`（测试夹具→排除）、
`src/leak.js`（真密钥形态→保留）。断言 hard-coded 为 `real=1 triaged=2`，**不达标就红**。

证据链接：`https://github.com/zjdyzww/ai-code-health/actions/workflows/self-test-action.yml`
