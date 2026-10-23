# BUGFIX — 两字中文数字季度被解析成 NaN（SNaNE05）

## 标题

`pickSeason`（src/release.ts）只支持单字中文数字季度，两字组合（第十二季、第二十一季、第二十季等）解析结果为字符串 `'NaN'`，资源条目标题显示为 `SNaNE05`。

## 证据链

上游 open issue 无对应 bug 报告（仅 dependabot / feature / 兼容 follow-up），走静态实锤 + 最小复现。基线（上游 HEAD `841cdfa`，v0.1.12，`npm test` 118/118 全绿）下，用构建产物 `lib/release.js` 的 `parseRelease` 跑离线复现：

```
"[字幕组] 某番剧 第十二季 - 05 [1080p]"   => heading: "SNaNE05"  episode: 5
"[字幕组] 某番剧 第二十一季 - 03 [1080p]" => heading: "SNaNE03"  episode: 3
"[字幕组] 某番剧 第二季 - 05 [1080p]"     => heading: "S02E05"   （正常）
"[字幕组] 某番剧 第十季 - 05 [1080p]"     => heading: "S10E05"   （正常）
"[字幕组] 某番剧 第2季 - 05 [1080p]"      => heading: "S02E05"   （正常）
复现脚本 exit 1；git stash 回基线复跑逐字一致，证明为上游既有缺陷、非本次引入。
```

## 根因

`pickSeason` 的中文分支正则可捕获 1–3 个中文数字字符，但查表 `map` 只含单字（一…十）；两字组合查不到表后落入 `String(Number(zh[1]))`，`Number('十二')` 为 `NaN`，于是季度值变成字符串 `'NaN'` 并被拼进 heading（`S${pad(season)}E…`）。该 heading 是详情页按字幕组/集数浏览时的条目标题，凡标题含「第十X季」的资源全部中招。

## 改动文件

- `src/release.ts`（唯一逻辑改动）：新增 `chineseSeasonNumber()`，正确解析阿拉伯数字与中文数字（含 `十`、`X十`、`十X`、`X十X` 组合，如 十二=12、二十一=21、三十=30）；无法解析时返回 `undefined`（按无季度处理），绝不再产出 `'NaN'`。
- `package.json`：version 0.1.12 → 0.1.13；repository.url 改指 lhg-dsh-plugins/lhg-dsh-anime-find。上游无版本戳常量、pnpm-lock.yaml 无根包版本字段，无其他待同步项。
- `README.md`：顶部追加 lhg fork 头，上游原文一字未删。
- 新增 `UPSTREAM.md`、`BUGFIX.md`；`LICENSE` 与上游逐字一致。

## 复验结果

- 复现脚本 exit 0：`第十二季` → `S12E05`、`第二十一季` → `S21E03`，原正常用例（第二季/第十季/第2季）输出不变。
- `npm run typecheck` 退出 0。
- `npm test` 118/118 全过，与基线逐项一致，零回归。

## 验证边界

- 离线验证：仅调用 `parseRelease` 纯函数（构建产物），未接真实 DSH 宿主、未调 Mikan/AniBT/AnimeGarden/Bangumi 等外部服务。
- 修复范围限于季度解析；中文数字集数（如「第十二话」）上游本就不解析，不在本版范围。
