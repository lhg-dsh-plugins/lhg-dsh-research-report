# BUGFIX — 0.3.30 → 0.3.31（lhg fork）

## 标题

`registerClaims` 的冲突检测漏比 `citations`：同一 claim id 仅修改数据集引用（path/value/tolerance）时被当作幂等重复静默接受，旧引用被永久保留并继续驱动数值核验——与该方法自述的「different bindings is refused loudly（绑定不同即大声拒绝）」契约矛盾。

## 证据链

上游 open issue 仅有 7 条 dependabot 依赖升级 PR，无 bug 报告；基线（HEAD `8181c93`）`pnpm typecheck` 退出 0、`pnpm test` 115/115 全绿，无现成失败测试。本缺陷由静态审查定位 + 最小复现坐实。

最小复现（纯 Node 直接加载 `src/ledger.ts`，离线）：先登记 claim `c1`（text/evidenceIds/dataset 固定，citations 为 `[{id:'n1', path:'rows[0].nav', value:100}]`），随后：

- 原样重复登记 → 幂等成功（对照，符合预期）；
- 仅改 text → 抛 `ID_CONFLICT`（对照，符合预期）；
- **仅把 citation 的 value 改为 999 → 未抛错、静默返回旧记录，`getClaim('c1')` 读回的 citation value 仍为 100**（复现脚本 exit 1）。

根因：`registerClaims` 的 `same` 比对只含 `text`、`evidenceIds`、`dataset` 三项，独漏 `citations`，而 `citations` 正是登记后由 `verifyRegistration` 取出、交给 `ctx.dataQuality.verifyCitations` 的核验绑定。

## 根因

`src/ledger.ts` `EvidenceLedger.registerClaims` 的冲突/幂等判定不完整：claim 的四类绑定字段（text、evidenceIds、dataset、citations）中 citations 未纳入比对，变更被误判为「同一登记」而丢弃。

## 改动文件

- `src/ledger.ts`（唯一逻辑改动）：新增模块级 `sameCitations()` 辅助函数（按字段值逐条比对，忽略源对象键序；`tolerance` 缺省按 `null` 处理），并将其并入 `registerClaims` 的 `same` 判定。错误信息与错误码不变（仍为 `ID_CONFLICT` / "different text or bindings"）。
- 配套同步（非逻辑）：`package.json`（version 0.3.30 → 0.3.31、repository.url 改落点仓）、`src/version.ts`（`VERSION` → `0.3.31`）、`CHANGELOG.md`（新增 0.3.31 条目）、`README.md`（顶部追加 fork 头）、新增 `UPSTREAM.md` / 本文件。`LICENSE` 与上游逐字一致。

## 复验结果

- 主复现：同一脚本修复后 exit 0——citations 变更被以 `ID_CONFLICT` 拒绝，存储绑定保持不变。
- 附加用例：citation 对象键序打乱的等价登记仍幂等通过（不误报冲突）；仅改 `tolerance`（0.1 → 0.2）被拒绝；整表删去 citations 被拒绝。
- 回归：`pnpm typecheck` 退出 0、`pnpm lint`（oxlint）退出 0、`pnpm test` 115/115 全绿——与基线（115/115）逐项一致，零回归。
- `node scripts/check-readme-sync.mjs` 在打包后复跑通过（fork 头为引用块、未新增 `##` 章节，五语种章节结构不变）。

## 验证边界

- 未调用真实 `ctx.dataQuality` 服务：citations 的下游影响以源码数据流为准（`assembleDetailed` 将登记的 citations 传入 `verifyRegistration` → 桥接核验），复现只覆盖 ledger 层这一缺陷本身。
- 上游基线中 `src/version.ts` 的 `VERSION`（`0.3.27`）与 package.json（`0.3.30`）不一致，属上游既有漂移、不是本版修复目标；本 fork 仅按 fork 惯例将两者一并对齐到 0.3.31。
- 修复只堵「静默吞掉」这一条路：冲突时抛错、要求换新 claim id，与 text/evidenceIds/dataset 变更的既有行为一致；不提供原地更新绑定的能力（登记不可变是该 ledger 的既定设计）。
