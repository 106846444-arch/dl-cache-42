# dl-cache-42 —— 116NIT下载夸克版 · 机房直传夸克

本仓库是「116NIT下载夸克版」软件的 **GitHub 机房中转端**。
NIT(③页) 与 UP(④页) 共用同一个 workflow，**软件只负责发指令和搬书签**，
真正的「下载 + 上传夸克」全部在机房（GitHub Actions runner）完成。

## 唯一的 workflow：`.github/workflows/fetch-upload-quark.yml`

| 输入 | 说明 |
|---|---|
| `items_b64` | base64(JSON数组)。每项 `{"name":"文件名","url":"直链","page_url":"文件页URL","referer":"可选"}`。NIT 传直链，UP 传文件页 URL（机房自己换直链） |
| `folder` | 夸克目标文件夹名（NIT→`NIT`，UP→`UP`），不存在会自动建（幂等） |
| `cookie` | 站点登录 Cookie（UP 换直链时用；NIT 不需要） |
| `threads` | 每个文件的下载并发连接数，默认 8 |

**输出**：日志里两行 `RESULT_B64 <文件名> <base64>`，分别是
`dl_result.json`（下载结果，`skipped` 为下载阶段跳过的条目）和
`up_result.json`（上传结果，`ok`/`bad`，分卷时 `ok` 里带 `parts` 列表）。
base64 是必须的 —— Secret 的值本身是 JSON，GitHub 会把出现在其中的
`{ } [ ]` 在日志里打码成 `***`，直接打 JSON 软件根本没法解析。

## 必需的 Secret

`QUARK_AUTH_JSON` = 本机 `quark_skill/workbuddy/config.json` 的完整内容。
CLI 固定从 `dirname(scripts)/<profile>/config.json` 读它。
**本机重新扫码登录夸克后，必须重新推送这个 Secret**，否则机房上传会 token 失效。

## 大文件会自动分卷

夸克对「单次上传会话」有隐性上限（实测 440MB/110片 成功，
3.68GB/880片 报 `ACK timeout`）。所以：

- **> 1000MB**：切成 500MB 的卷分别上传（各卷独立成功）
- **≤ 1000MB**：先整传；整传失败自动降级为分卷

分卷落在目标文件夹下的**同名子文件夹**里，卷名 `原名.part001`、`原名.part002`……
用户下载全部卷后本地合并：`copy /b "原名.part*" "原名"`。

## 不再使用 Release / tag

旧架构是「机房下载 → 发 Release → 本地镜像拉回 → 本地上传」。
现在文件不经过 GitHub，**仓库里不需要 Release 和 tag**（历史上产生的已清理干净）。
