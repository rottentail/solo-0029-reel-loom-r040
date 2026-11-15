实验全部完成，工作区保持干净。以下是完整的定位与影响分析报告。

---

# 声画织场「媒体目录恢复不完整」事故复现与分析报告

## 1. 实验设置（生产代码零改动，`git status` 全程干净）

在专属临时数据目录 `/tmp/loom-incident` 运行真实服务（`LOOM_DATA` 指向该目录，venv 安装于 `/tmp`，工作区未动一行代码）。关键对象：

| 对象 | ID / 摘要 |
| --- | --- |
| 共享来源媒体 | `asset_c5a7acb5…`，blob `ee2422e6…b67ce`（= 上传文件的 sha256，内容寻址） |
| 交付一（带印本成片） | 任务 `render_989099c2…` → 成片 `artifact_b6d325ea…`（blob `d32c393c…`）→ 印本 `imprint_0848f605…`（ZIP blob `69a798b8…`） |
| 交付二（冻结任务） | 任务 `render_86df5306…`，入队时冻结指纹 `a5533a2c…`，切片引用同一来源 |
| 对照 B（全新完整目录） | `/tmp/loom-control`，同流程全绿（conformance valid、印本 verify valid、reconcile consistent） |
| 对照 A（事故前完整快照） | `/tmp/loom-before`，移走字节**之前** `cp -a` 的同一数据库+全部字节 |

## 2. 事故时间线（全部经真实 HTTP 与进程信号完成）

1. 交付一渲染成功 → conformance `valid:true` → 印本打包并 `verify valid:true`（基线健康）。
2. 交付二入队（快照已冻结），轮询确认进入 `running` 后 `kill -9` 服务，模拟中断。
3. 快照完整目录到 `/tmp/loom-before`（对照 A）。
4. **仅移走两条交付共同引用的来源 blob 文件** `blobs/ee/2422e6…` → 隔离区；数据库、成片 blob、印本 blob 全部保留。
5. 重启服务，开始取证。

## 3. 定位：缺失点被三处独立证据交叉锁定

- **资源核对** `GET /resources`（E2）：`consistent:false`，`missing:["ee2422e6…"]` —— 精确指出缺失哈希，无 `bad_size`、无未登记字节。
- **任务事件**（E4/E17）：交付二事件链 `queued→started→recovered→retried→started→failed`，失败载荷 `{"error":"not_found","detail":"blob does not exist","context":{"id":"ee2422e6…"}}` —— 错误对象直接携带缺失哈希。
- **引用关系追踪** `GET /tickets/{id}/lineage`（E12）：血缘图在 DB 层完整，来源 blob 节点仍被 `content` 边引用 —— 图没坏，**只有图指向的一坨字节不在盘上**。`GET /blobs/ee2422e6…` → 404，而 `GET /blobs/d32c393c…`（成片）→ 200（E13），影响是选择性的，不是全库损坏。

机制根因（代码锚点）：`blobs.resolve()` 在文件缺失时抛 404（`reel_loom/blobs.py:52-55`）；渲染时才以 `verify=True` 解析来源字节（`rendering.py:23`、`planner.py:36`），而**入队冻结只读数据库元信息**（`planner.py:11-29`），`assets.get(ready=True)` 也只查 DB 状态（`assets.py:36`）。所以「登记在册」与「字节在盘」是两层事实，事故正是两层脱节。

## 4. 影响分析：两类能力必须分开判读

### 4.1 已发布字节的可验证性 —— 不受来源缺失影响（交付一真正健康）

| 证据 | 结果 |
| --- | --- |
| E5 `GET /tickets/render_9890…` | `succeeded` —— 对取回而言是**真健康**，不是假象 |
| E6 下载成片 | 200，ETag= blob 哈希，下载体 sha256 与登记一致 |
| E7/E8 | Range → 206；ETag 复验 → 304 |
| E9 conformance | `valid:true`，全解码 55 视频帧 + 104 音频帧（只读成片字节，`conformance.py:18`） |
| E10 印本 verify | `valid:true` —— ZIP 自含清单与媒体副本，逐成员重哈希（`imprints.py:71-95`），**完全不需要来源** |
| E11 印本下载 | 200，sha256 与印本 blob 一致 |

### 4.2 来源可重算性 —— 凡需重读来源字节的路径全部阻断

| 证据 | 结果 |
| --- | --- |
| E14 对未完成任务打包印本 | 409 `imprint_state`（状态机拒绝，与字节无关） |
| E15–E16 重试交付二 | 重新入队（attempt 2）→ 渲染时 404 `not_found`，点名缺失哈希 |
| E18 资产状态 | DB 仍 `ready` —— **DB 层的 ready 不是来源可用性的保证** |
| E19/E20 signal、contact-sheet | 均 404 `not_found`（分析需解码来源） |
| E21 事故后新建剪辑卷+渲染单 | **被接受并入队**，渲染时才失败 —— 冻结时不验字节，故障延迟到执行期暴露 |
| E22 retire → restore | retire 成功（纯 DB）；restore 404 —— 「恢复先核对摘要」（`assets.py:58`）挡住了无字节恢复 |

### 4.3 状态判读细则（不一概而论）

- `succeeded` 的两条任务：**交付检索维度健康**（字节可下载、可验规、可打包验证），但**再派生维度受损** —— 血缘指向的来源没了，无法据此重渲染。
- `failed(worker_interrupted)`：**不是损坏**，是设计的恢复语义（`tickets.py:103-112`），快照完好可重试。
- `failed(not_found)`：环境性失败，同样不污染冻结快照；attempt 递增只是计数。
- 服务级 `/health` 全程 `ready`/`database:ok` —— 服务健康不反映字节层完整性，只有 `/resources` 的 reconcile 暴露了事故。

## 5. 重试与清理如何改变可恢复机会

- **重试是安全但无谓的（在字节回归之前）**：E15–E16 证明重试复用原冻结快照、清错误、attempt+1，然后再次在同一哈希上 404。它不破坏任何恢复机会（快照、登记、事件历史都保留），但也绝不自愈；盲目重试循环只会稀释事件日志。
- **清理（sweep）不伤这条缺失来源**：E23 `dry_run` 显示 `protected_count:4, candidates:[]` —— 只要 blob 仍被 assets/artifacts/imprints 引用，sweep 不动其登记（`resources.py:10`），恢复门始终敞开。真正会关闭恢复机会的是：丢失隔离区里那份字节（唯一副本）、或将来对**无引用** blob 执行 `dry_run:false`（会删登记+文件）。
- **第二条自愈路径（内容寻址）**：重新上传逐字节相同的来源会重写缺失文件并复用既有资产（`assets.py:19-22`、`blobs.put` 的 `INSERT OR IGNORE`）—— 若能从上游重新生成/取回相同字节，不依赖隔离副本也能愈合。注意成片**不能**反推来源（有损转码+裁切），来源字节原则上不可替代。

## 6. 有证据支持的恢复顺序与阻断条件

实际执行并验证的顺序（R1–R9 全绿）：

1. **止损**：暂停重试循环；不要执行 `sweep dry_run:false`（虽本次证明引用受保护，但无引用字节可能是恢复原料）。
2. **定位**：`GET /resources` 取缺失哈希 → 用 lineage / 资产 blob 字段映射到受影响交付。
3. **归还字节**：隔离副本回移（或重传相同内容），sha256 与登记一致（R1）。
4. **核对**：`GET /resources` → `consistent:true`（R2）—— 这是继续的前提。
5. **恢复资产状态**：`POST restore`（摘要校验通过，R3 → `ready`）。
6. **重试冻结任务**：交付二 attempt 3 → `succeeded`（R4）；事故期新稿 attempt 2 → `succeeded`（R5）。
7. **补交付动作**：交付二印本打包并 `verify valid:true`（R6/R9）；signal 分析恢复（R7）。终态 4 任务全 `succeeded`、7 blob、reconcile 一致（R8）。

阻断条件（均有证据）：字节未回归就重试 → 再次 404（E16）；字节未回归就 restore → 404（E22）；任务未成功就打包 → 409（E14）；字节缺失期新派工 → 入队成功、渲染失败（E21）。

## 7. 对照验证：排除配置/编码器误判

- **对照 B（全新目录）**：同一样例、同一 ffmpeg（imageio-ffmpeg 7.0.2）、同版本代码，全流程绿 —— 环境基线无问题。
- **对照 A（事故前快照，同一数据库）**：C-A1 健康检查与事故库逐字节同源；C-A2 reconcile `consistent:true`；**同一个重试请求**在事故目录 404、在快照目录 `succeeded`（C-A3）；**同一个 signal 请求**在事故目录 404、在快照目录返回静音区间 `[1.01s–1.51s]`（C-A4）。唯一变量就是那份被移走的 blob 文件。
- **同目录恢复后**：R4–R8 全绿，再次证明故障与配置、编码器无关，纯粹是来源字节缺失。

## 8. 附带观察（非本次事故主线，供后续加固）

- 被 kill 的渲染在 `scratch/loom-*/media.mp4` 留下半成品，reconcile 只扫 `blobs/` 不扫 `scratch/`（E2 的 `unregistered_blobs` 为空时该文件已存在）—— 核对覆盖面的一个盲区。
- API 怪癖：无 body 的 POST（cancel/retry）缺 `Content-Length` 会被 411 拒绝，客户端需显式发送空 body。
- 证据与样例全部保留在 `/tmp/loom-evidence/`（各 HTTP 响应）、`/tmp/loom-quarantine/`（已清空，字节已归还）、`/tmp/loom-{incident,before,control}/`（三份目录可复查）。