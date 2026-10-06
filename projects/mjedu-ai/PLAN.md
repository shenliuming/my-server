# 服务器部署规划

状态：规划稿，尚未部署。服务器规格、测试域名和现有 Nginx 配置待确认。

## 1. 部署方式

采用服务器统一 Nginx 入口，每个项目、每个环境独立使用 Docker Compose。
Nginx 站点配置按项目及 `dev`、`test`、`production` 环境拆分，保存在对应环境的 `nginx/` 目录。
Nginx 优先沿用服务器现有安装方式；如果当前由容器运行，需先确认网络和挂载再调整。
下面的端口方案按 Nginx 运行在宿主机设计。

| 服务 | 运行方式 | 职责 |
| --- | --- | --- |
| Nginx | 服务器入口，按项目和环境加载独立站点 | HTTPS、静态网站、API 转发 |
| 用户网站 | Nginx 提供静态构建产物 | 用户页面；前端工程及是否为静态部署待确认 |
| 管理后台网站 | Nginx 提供静态构建产物 | 后台页面；前端工程及是否为静态部署待确认 |
| admin API | Java 17 容器 | `maku-server` 启动，装配 member 等模块 |
| www API | Java 17 容器 | `maku-www` 独立启动，承载用户侧业务 |
| MySQL | 每个环境独立容器 | 同环境 admin、www 使用同一业务库 |
| Redis | 每个环境独立容器 | 同环境 admin、www 按业务配置使用，避免跨环境缓存和锁冲突 |

`member` 是后台应用依赖模块，不另起一个容器。`www` API 与用户网站静态文件是两个部署对象。
如果网站存在 SSR 或 Node 服务，需要增加对应应用容器，不能直接按静态文件部署。

## 2. 入口与端口

正式环境沿用当前域名；开发、测试环境分别使用独立域名，具体名字待确认。
建议每个环境一个域名承载网站、后台及 API，便于沿用现有路径。

| 请求 | Nginx 目标 | 说明 |
| --- | --- | --- |
| `/` | 用户网站静态目录 | SPA 页面需支持刷新及 `/ry/{邀请码}` 等前端路由 |
| `/admin/` | 后台静态目录 | 需核实前端构建的 base 路径 |
| `/admin/api/` | admin API | 优先匹配 API，不能落到后台页面；转发时移除外部前缀 |
| 用户 API 前缀 | www API | 先核实现有前端请求前缀，再配置；不猜测现网路径 |

后台代码路由为 `/ai/invite`，外部请求为 `/admin/api/ai/invite`，当前应用基础配置的 context-path 为 `/`。
落地前需核对实际启动参数是否覆盖该配置，确认 Nginx 前缀处理。
按实际接口配置上传大小、超时、SSE 禁用缓冲以及 WebSocket Upgrade，不对所有请求设置无限超时。

| 环境 | admin 宿主机映射 | www 宿主机映射 | MySQL / Redis |
| --- | --- | --- | --- |
| 正式现状（截图） | `30001 → 8080` | `30002 → 8080` | `40001 → 3306` / `40002 → 6379` |
| 测试建议 | `127.0.0.1:31001 → 8080` | `127.0.0.1:31002 → 8080` | 默认不发布宿主机端口 |
| 正式目标 | 保留端口号，改为本机监听 | 保留端口号，改为本机监听 | 按现有客户端依赖评估后收口 |

测试端口需确认未占用。对外网站统一使用 80/443，运维通过 SSH。
应用容器通过 Compose 服务名 `mysql:3306`、`redis:6379` 连接依赖，不使用动态容器 IP。
若需要桌面数据库工具，单独配置本机端口并通过 SSH 隧道访问。
正式环境网络变更需在测试完成及确认现网依赖后安排窗口。

## 3. Git 仓库目录

以下是目标结构，文件随实际搭建补充，不创建空配置。

```text
my-server/
  README.md
  projects/
    mjedu-ai/
      README.md
      PLAN.md
      scripts/                # 构建、发布、备份；执行时明确指定环境
      dev/
        README.md
        compose.yaml
        .env.example
        config/               # 开发应用配置模板
        nginx/                # 开发站点配置模板
      test/
        README.md
        compose.yaml
        .env.example
        config/               # 应用配置模板
        nginx/                # 测试站点配置模板
      production/
        README.md
        compose.yaml
        .env.example
        config/               # 正式应用配置模板
        nginx/                # 正式站点配置模板
```

站点配置分别管理域名、静态文件路径、API 转发目标及该站点的证书路径。
服务器部署时，将对应配置发布到 Nginx 的站点加载目录，并使用包含项目和环境的文件名，例如 `mjedu-ai-test.conf`。
每个配置只声明所属站点的 server 块；主配置负责加载已启用的站点。证书申请与续期说明放在所属环境目录。

Git 保存配置模板、版本约定、发布脚本和迁移记录。
真实 `.env`、证书私钥、数据库备份、日志、JAR、前端构建产物和数据目录不提交。
现有业务仓库配置含真实连接信息，不能直接复制进这个公有仓库。

## 4. 服务器目录

```text
/srv/
  my-server/                        # 部署仓库 checkout
  apps/mjedu-ai/
    dev/                            # 开发环境独立目录
    test/
      .env                          # 服务器本地凭据
      config/                       # 本地实际应用配置
      nginx/                        # 本环境实际站点配置
      releases/<版本>/              # JAR 和前端构建产物
      current                       # 当前发布版本
      data/mysql/                   # MySQL 持久化
      data/redis/                   # Redis 持久化
      logs/                         # 应用文件日志（如有）
      backups/
    production/                     # 相同结构，完全独立
```

Nginx 实际站点配置按现有安装目录加载，HTTPS 私钥位于服务器本地受限目录。
Compose 必须使用明确的运行目录挂载绝对路径，避免不同启动目录导致数据写错位置。
所有 MySQL/Redis 数据均持久化到环境专属目录，应用升级不删除数据目录。

## 5. 环境隔离与配置

- Compose 项目名固定为 `mjedu-ai-dev`、`mjedu-ai-test`、`mjedu-ai-production`，分别创建网络及服务。
- admin、www 保留容器内 8080；通过各自映射端口区分。
- MySQL、Redis、数据库账号、目录和密钥在各环境独立。其他业务项目同样按项目隔离。
- 同机部署还需限制测试容器 CPU、内存及日志容量；网络隔离不能消除资源争抢和主机故障风险。
- 根据服务器实际空闲资源设置 JVM 堆、容器内存上限、MySQL 缓冲池和 Redis 内存上限，给堆外内存与操作系统留余量。
- 启动时显式选择环境并加载外部配置。当前两个应用基础配置默认 `dev`；www 暂无现成 test 配置，需补齐完整测试配置。
- 测试使用测试支付、短信及第三方服务配置；没有沙箱的外部服务，先明确测试账号、费用及可触发操作。
- 测试数据使用初始化数据或脱敏导入，避免测试任务操作正式数据。
- 镜像固定已验证版本，应用版本关联业务仓库 commit 和构建产物校验值，不使用 `latest` 作为发布依据。

## 6. 构建、发布与回滚

1. 本地或构建机从指定 commit 分别构建 `maku-server`、`maku-www`；网站从各自前端仓库构建。
2. 产物上传到环境专属 `releases/<版本>/`，记录业务 commit、构建时间及校验值。
3. 数据库变更先备份、核对 SQL 和兼容性，再按顺序执行并记录；新库初始化和升级迁移分开管理。
4. 使用 Compose 更新指定应用服务，保持 MySQL、Redis 及其数据目录独立。
5. 检查应用启动、登录、邀请码注册与赠豆、后台列表，以及数据库/Redis 连通性。
6. 前端切换到对应版本；Nginx 配置改动先做语法检查，通过后 reload。
7. 保留近期发布版本；失败时切回上一版应用及前端，并重建对应容器使挂载生效。

旧发布脚本直接重启现有正式容器，不能用于测试环境。新脚本必须明确环境和目标服务。
应用回滚不自动撤销 SQL；破坏性迁移必须提前设计恢复方案。
首次采用单实例发布，应用更新可能短暂不可用；需要无中断发布时再补双实例切换。

## 7. 运维与搭建顺序

- 容器配置重启策略、健康检查和日志轮转；依赖启动完成后应用仍需能重连。
- 先确认 Redis 是纯缓存还是还保存任务等数据，再确定持久化、淘汰策略及备份方式。
- MySQL 初期建议每日一致性备份、保留 7～14 天并同步到服务器之外；仅每日备份意味着最多约一天数据损失，按业务恢复目标调整。
- 正式切换前完成一次恢复演练；备份不能只确认文件存在。
- 先做磁盘空间、服务健康、备份失败告警，资源有需求时再增加监控平台。

落地顺序：盘点现网与资源 → 搭建测试 MySQL/Redis → 部署两个 API → 发布两个网站 → 接入测试 Nginx/HTTPS → 功能回归和恢复验证 → 记录并逐步规范正式环境。

## 参考

- [Docker Compose 网络与服务名](https://docs.docker.com/compose/how-tos/networking/)
- [Docker 端口发布与本机绑定](https://docs.docker.com/engine/network/port-publishing/)
- [Docker Compose 生产部署](https://docs.docker.com/compose/how-tos/production/)
- [Docker 日志配置](https://docs.docker.com/engine/logging/configure/)
- [Nginx proxy_pass 路径处理](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass)
- [Nginx WebSocket 转发](https://nginx.org/en/docs/http/websocket.html)
