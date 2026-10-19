# AGENTS.md

本仓库是笔试交付仓库。以下是 AI 助手在本仓库工作时**必须遵守**的协作规则，优先级高于任何默认行为。

## 零、开工第一步：建 GitHub public 仓库并推送（先于一切编码）

本机 `gh` CLI 已登录 GitHub 账号 `wqhflywhite`（git 走 SSH，已验证可用）。开工后**第一件事**是执行：

```bash
gh repo create <仓库名> --public --source . --push
```

- 仓库名：题目有要求用题目要求的，没有就用 `huashun-exam`。
- 若 `gh` 不可用，退化为：告诉我在 GitHub 网页手动建 public 仓库，然后你执行 `git remote add origin git@github.com:wqhflywhite/<仓库名>.git && git push -u origin main`。
- 之后每个 commit 都及时 `git push`，不要攒到最后一次性推。
- 建仓完成后把仓库 URL 报给我。

## 零点五、题目硬性技术要求（笔试方指定）

- **持久化存储一律用 MySQL，禁止使用 SQLite**（包括「本地先跑通」的临时方案也不行）。Go 侧用 `database/sql` + MySQL 驱动或 GORM，连接串走环境变量/配置文件，不要硬编码。
- 若题目要求容器化部署：提供 `docker-compose.yml`，应用 + MySQL 8 一键起，应用要等 MySQL healthy 再连（healthcheck + depends_on condition 或重试逻辑），并附初始化 schema（`init.sql` 或应用启动时自动迁移）。
- DSN、账号密码用环境变量注入，compose 里只允许出现示例口令，README 写清如何改。

## 零点六、验证环境：远程笔记本 Docker（本机无 Docker/MySQL）

本机（Windows 台式机）没有 Docker 和 MySQL，**一切需要 MySQL/容器的验证都在笔记本上执行**：

- 入口：`ssh flywhite@192.168.1.4`（免密已配好，BatchMode 可用，Docker 29.1.3）。SSH 不通时先让我检查代理/TUN 是否接管了局域网流量，不要自己改网络配置。
- **独立命名，严禁动现有容器**：笔试专用容器/compose project 统一前缀 `hs-`（如 `hs-mysql`、`docker compose -p hs-exam`），端口避开已占用项。`docker ps` 里已有的容器（infra-*、deploy-*、interview-deploy 栈等）一个都不许停、不许改。
- 标准验证流：① 直接用已常驻的 `hs-mysql`（见下）→ ② 本机 `go build ./...` → ③ 测试 DSN 指向 `192.168.1.4:3307` 跑 `go test ./...` → ④ 需要验证 compose 时，把仓库 scp 到笔记本临时目录（如 `~/hs-exam-verify/`）用 `docker compose -p hs-exam up -d --build` 实测一键起，验证后 `down -v` 清理。
- **MySQL 已提前备好（2026-10-19 实测全通）**：笔记本上容器 `hs-mysql`（mysql:8.0，8.0.46）常驻运行、`--restart unless-stopped`，**root 空口令**（笔试一次性场景定稿），预建库 `hs_exam`，端口 `192.168.1.4:3307`。本机 DSN：`root@tcp(192.168.1.4:3307)/hs_exam`，Go 空口令直连建表读写已验证；go-sql-driver/mysql 已在本地模块缓存。若容器异常重启即可：`docker start hs-mysql`；如题目要求别的库名，`docker exec hs-mysql mysql -uroot -e "CREATE DATABASE <库名>"`。
- **Redis 同样已备好（2026-10-19 实测全通）**：容器 `hs-redis`（redis:8）常驻、无密码，端口 `192.168.1.4:6380`；本机 go-redis/v9 直连 PING + XADD/XLEN（Stream）已验证，模块已在本地缓存。题目若涉及缓存/排行榜/轻量队列直接用；**Redis Stream 可作为轻量消息队列兜底**（XADD/XREADGROUP/消费者组）。
- **MQ（Kafka/RabbitMQ）决策：不提前部署**。笔试时长短、题目显式要求 MQ 概率低；同流程面经候选人的自选栈也没用真 MQ。若题目明确要求，现场经 daocloud 镜像源拉取，并在动手前与我确认。
- 笔记本上拉 Go 模块走 `GOPROXY=https://goproxy.cn,direct`；GitHub 拉取走 gh-proxy.com。
- 仓库文件（含 compose）里**严禁出现真实凭据**，只放环境变量占位与示例口令；验证用的空口令不落任何文件。
- 笔试全部结束后清理：`docker rm -f hs-mysql` 及 `hs-` 前缀资源、`~/hs-exam-verify/`。

## 一、核心协作模式：分步确认，禁止一把梭

**绝对不要拿到题目就一次执行到底。** 必须按以下节奏工作：

1. **先拆后做**：拿到需求后，第一步是输出一份「子目标拆解清单」（例如：① 数据模型与存储 ② 核心接口 ③ 业务逻辑 ④ 测试 ⑤ README），并给出你对需求的理解。等我确认拆解合理之前，不写任何代码。
2. **逐个确认细节**：每开始一个子目标前，先和我沟通该子目标的实现细节——技术选型、接口签名、文件结构、关键取舍，给出你的建议方案和备选方案，**等我明确回复后再动手**。
3. **完成即停**：每完成一个子目标，立即停下来汇报：做了什么、验证结果（编译/测试输出）、下一步打算做什么。等我回复「继续」或调整意见后，再进入下一个子目标。
4. **有歧义先问**：需求不清楚、有多种合理做法、或我的指令与本文档冲突时，先提问，不要自行假设。

## 二、验证纪律

- 每个子目标完成后必须实际跑验证：编译通过 + 相关测试通过（Go 项目即 `go build ./... && go test ./...`，其他语言按对应工具链），把真实输出贴给我看。
- 测试没绿不算完成，不许说「基本完成」。
- 全部完成后，在干净目录重新 clone/构建一次，确认仓库自包含可跑。

## 三、Git 纪律

- **每个子目标一个 commit**，不要把多个子目标揉进一次提交；commit message 用中文说明性短句，格式 `<type>: <说明>`（feat/fix/docs/test/chore）。
- 严禁提交：密钥、`.env`、编译产物（`*.exe`、二进制）、IDE 配置。仓库根目录维护好 `.gitignore`。
- commit 前先让我看一眼改动摘要。

## 四、工程质量

- README 必须包含：项目简介、运行方式（一条命令能跑起来）、设计取舍说明、已知不足。
- 代码风格保持简洁可读，不堆注释，不过度设计；先保主干正确，再考虑加分项。
- 引入任何第三方依赖前先说明理由并征得同意。

## 五、沟通语言

全程使用中文交流。
