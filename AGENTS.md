# Repository guidance

## Local demo budget-utilization fix

- This checkout starts from v0.7.6 (`605d0028d80becc95f7f9ce3084ec0f00122d772`) and is used by the adjacent `sure-demo/` local deployment.
- Reports use `Budget#percent_of_budget_spent`, the same monthly spending-limit calculation used by the budget/planning UI. Category allocations are not the monthly spending limit.
- An uninitialized budget has no utilization value. Calculation failures must propagate instead of being silently displayed as missing budget data.
- Run report/budget regression tests against the isolated database in `../sure-demo/compose.test.yml`; never point Rails tests at the demo database.
- The local derived image is defined in `../sure-demo/Dockerfile.fix`. Keep its controller identical to the tested source; no live-only container edits.
- Report expense totals are gross outflows; budget actual spending nets inflows within each top-level category and floors each category at zero. Genuine refunds can therefore make the totals differ.
- Transfers whose two legs the demo generator creates are known pairs and must go through `Transfer#confirm!`: ordinary movements and card payments are excluded, loan principal remains an expense, and receiving legs are funds movement. This does not change the pending status of real auto-match suggestions.


## Financial metric contracts

- Report cash-flow breakdowns and exports share `IncomeStatement#reportable_transactions`: posted reportable transactions only, with the existing finance-account, tax-account, kind and activity-label rules. Trades are portfolio activity, not household income.
- Forward balance materialization must share holdings' actual-data horizon, ending no later than the calculation's current date. Keep future ledger entries, but do not materialize their dates without matching holdings. Preserve the cross-day regression tests.
- Missing FX in the audited aggregate paths raises a dated conversion error. Zero balances need no FX. The adjacent deployment's fixed EUR/USD and GBP/USD rates are isolated synthetic fixtures, never real-market data.
- Demo parent allocations include their children; monthly limits sum roots only. Demo investment accounts must have their documented subtypes.
- Current source/deployment evidence and metric definitions live in `../sure-demo/FINANCIAL_AUDIT.md`; preserve existing report-budget and known-transfer fixes.

## Working in the repository

- Read [architecture and conventions](docs/llm-guides/architecture.md) before changing code, and the relevant [task guides](docs/llm-guides/README.md).
- Rails code lives in `app/`; keep business logic in models, concerns and POROs, with thin controllers. JavaScript is in `app/javascript/`, components in `app/components/`, assets in `app/assets/`, and configuration in `config/`.
- Minitest tests mirror `app/` under `test/`; fixtures are in `test/fixtures/`. Migrations and schema are in `db/`, scripts in `bin/`, shared libraries in `lib/`.
- Use `Current.user` and `Current.family`, never `current_user` or `current_family`. Preserve family tenancy and existing authorization boundaries.
- Prefer built-in Rails patterns and established dependencies. New dependencies need a strong technical or business reason. Keep changes focused, readable and consistent with nearby code.
- Keep comments focused on non-obvious intent, constraints, and current behavior—not implementation diaries or issue/PR history. Put that history in commits and PR descriptions. Update or remove stale comments when changing code.
- Ruby uses two-space indentation, `snake_case` methods/variables and `CamelCase` classes. JavaScript uses `lowerCamelCase` variables/functions and `PascalCase` classes; follow Biome. Keep domain logic out of ERB.
- Never commit secrets; use environment variables and `.env.local` for local configuration.
- Do not start `rails server`, touch `tmp/restart.txt`, run `rails credentials`, or automatically run migrations. Setup and database commands in the [development guide](docs/llm-guides/development.md) are for explicitly requested environment work.
- New migrations use the current Rails migration version; leave historical migration versions intact.

## Tests and pull requests

- Use Minitest and fixtures for behavioral tests, with Mocha and VCR where needed. RSpec/rswag is for OpenAPI documentation only. Follow the [testing guide](docs/llm-guides/testing.md).
- Run `bin/rails test` and ensure it is green before pushing. Before opening a PR, run **all** checks in the [pre-PR checklist](docs/llm-guides/development.md#before-opening-a-pull-request): full Rails tests, applicable system tests, Ruby and ERB lint, Biome and Brakeman. Only create the PR when all required checks pass.
- Commits use imperative subjects of at most 72 characters, with rationale and issue references where relevant. Target `main` with small, cohesive changes.
- PRs explain the problem, resulting behavior and validation; link issues and include screenshots for UI changes and migration notes when applicable. Ensure CI passes and the branch is up to date before requesting review; see [CONTRIBUTING.md](CONTRIBUTING.md).

## UI changes

When touching ERB, view components or CSS, follow the [design system guide](docs/llm-guides/design-system.md):

- Use functional tokens from `app/assets/tailwind/sure-design-system.css`, such as `bg-container`, `text-primary`, `border-primary`, `bg-warning/10` and `text-destructive`. No raw Tailwind palette classes or hex literals.
- Check `app/components/DS/` first for alerts, badges, buttons, disclosures, dialogs and inputs. Use existing `DS::*` primitives.
- If the same hand-built shape appears at least twice in a diff with no DS equivalent, propose a new `DS::*` primitive before the second copy lands.
- Use the `icon` helper, never `lucide_icon` directly; no raw SVG outside DS primitives. Use `t()` for user-facing strings and scale tokens instead of arbitrary pixel values when a scale token fits.
- Adding styles to `app/assets/tailwind/sure-design-system.css` or `app/assets/tailwind/application.css` requires explicit permission.
- Reviewers escalate DS reuse and repeated-shape violations to close/rewrite; token and icon/SVG/localization/scale violations are request-changes.

## API changes

Adding or modifying `app/controllers/api/v1/` endpoints requires Minitest behavioral coverage and corresponding **documentation-only** rswag specs in `spec/requests/api/v1/`. Reusable schemas belong in `spec/swagger_helper.rb`; regenerate `docs/api/openapi.yaml` with `RAILS_ENV=test bundle exec rake rswag:specs:swaggerize` after changes.

**Post-commit API consistency:** after every API endpoint commit, follow the [API checklist](docs/llm-guides/api-endpoint-consistency.md): Minitest coverage, no behavioral assertions in rswag, and the shared `X-Api-Key` authentication pattern in those tests/specs.

## Provider and feature work

- Read [provider sync guidance](docs/llm-guides/providers.md) when changing imports, pending transactions, FX metadata or diagnostics. Use `DebugLogEntry.capture(...)` for support-relevant failures and partial responses, with provider/source metadata and family/account-provider context where available.
- For securities providers, follow [adding a securities provider](docs/llm-guides/adding-a-securities-provider.md).
- For feature rollout, follow [preview-feature gating](docs/llm-guides/gating-a-preview-feature.md); for goals, read the [Goals guide](docs/llm-guides/goals.md).

## Local commit secret-guard exceptions

- The local ecc-guard hook flags credential-named variables assigned API/fixture lookup expressions, the public demo-key constant, and the UUID lock owner. The initial eleven lines have narrow trailing `ecc-guard-ignore` comments after inspecting their full contents.
- Keep the guard enabled. Do not replace these annotations with repository-wide rule or file ignores. Their before/after byte provenance and independent positive/negative control are in `../sure-demo/evidence/guard-annotation-provenance.json` and `guard-independent-verification.json`.
- The zh-TW `email_confirmations.new.invalid_token` message has one additional line-scoped false-positive annotation; its YAML data is unchanged. Provenance and independent controls are in `../sure-demo/evidence/full-guard-annotation-provenance.json` and `full-guard-independent-verification.json`.
- These annotations change no executable behavior. Original test/runtime source hashes remain historical evidence; `../sure-demo/evidence/commit-source-sha256.txt` identifies the annotated commit version.

## 手動儲蓄投入確認

- 手動帳戶有尚未過期的 `manual_save` 投入時，更新餘額須建立／更新當日對帳，不得改寫期初餘額；以正向增加額套用既有金額、日期容差配對，保留轉帳類投入的確認流程。
- 對帳失敗不可更新帳戶快取或儲存未完成的對帳紀錄。固定目標分配額不因投入確認自動增加；進度仍按已指定資金計算。
- 修正及示範重播紀錄見 `sure-demo/MANUAL_SAVINGS_FIX.md`（來源 checkout 請參閱相鄰部署目錄）。

- 首次當日對帳須使用呼叫端已知的更新前餘額，不能依賴可能缺少／過期的每日餘額。已有同日對帳時，優先使用對帳本身更新前金額，避免重複配對。

## 缺匯率與完整測試

- 主要財務彙總缺匯率仍須拋出 Money::ConversionError，不得代入 0 或 1。共用帳戶側欄在呈現邊界明確顯示缺少的幣別／日期、記錄日誌並列出可見帳戶的原幣餘額，不應遮蔽頁面原本的驗證錯誤。
- 缺匯率的側欄不可快取；只用 cache_if 在成功時快取，且保留成功／失敗／恢復、跨家庭可見範圍與非匯率例外傳播測試。
- 完整測試只使用獨立 Compose project 的資料庫與 Redis；一次只執行一個 Rails 測試程序，採用 compose.test.yml 的較保守 GC 設定。原始 2026-11-13 中斷紀錄不可改寫成通過；後續診斷與完整測試結果另存 sure-demo/FULL_TEST_DIAGNOSIS.md（來源 checkout 參閱相鄰部署目錄）。
