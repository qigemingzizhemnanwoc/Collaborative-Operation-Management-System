# 公开版本修复验证（2026-11-07）

补回与原本地已验证文件SHA-256一致的deploy/nginx.conf，未重写代理配置。发布副本Compose配置、前端镜像、后端镜像及完整构建均已成功。项目展示名称统一为企业运营管理系统；内部标识不变。依赖按兼容版本范围修复，生产依赖high/critical为0，剩余告警见dependency-security.md。

本轮独立四服务均healthy；真实HTTP登录、未登录拦截、员工分页、销售防重、库存撤销与流水核对通过。远程CI以最新提交对应运行记录为准。

以下保留原验收记录。

> 发布说明：本报告记录原本地限定验收；公开副本安全清理及重新验证结果以 [发布整理报告](publication-safety.md) 为准。原始日志与本机信息未公开。

# 第四阶段部署验收

## 最终本地验收：通过（2026-11-06）

本节为最新结论；后续各节保留历史排查过程，其中未完成描述已被本节覆盖。验收限于本机隔离部署，不代表公网生产就绪或远程 CI 通过。

### 实际结果

| 项目 | 本轮证据 |
| --- | --- |
| Engine | Docker 29.9.2 可用，Linux 容器实际运行 |
| 镜像 | 官方基础镜像；后端 Maven/Java17 多阶段构建、前端 npm ci/Vite/Nginx 构建成功 |
| 四服务 | `ecs-portfolio` 项目 backend/frontend/mysql/redis 均 running healthy |
| 数据库 | 全新命名卷执行 db.sql 成功，26 张表，手机号和库存流水唯一索引存在；不连接本机业务库 |
| 安全初始化 | 演示账号全部清空旧密码并停用，只有 superadmin 用本地随机强密码初始化为 BCrypt；重启密码不变 |
| 网络与接口 | 127.0.0.1:18080 经 Nginx 访问 SPA、健康接口、实际密码登录、用户信息和员工分页成功；未登录查询被拦截 |
| 库存/Redis | 合成商品初始10，销售2后库存8；即时重复销售被拒绝且仅1笔流水；删除后库存10、2笔流水合计0 |
| 故障健康检查 | 真实停止本项目 Redis，健康接口返回503/DOWN；恢复后重新健康 |
| 持久化 | 重启四容器后重新登录成功，商品、库存及流水仍在，uploads卷标记文件保留 |
| JUnit | 最后一次统一运行42项，失败0、错误0、跳过0：库存16+员工17+Redis7+初始化2；不累计之前重复执行 |
| 远程 CI | 工作流含Java17、MySQL/Redis集成测试、Vue构建、两个Docker镜像构建；未推送、未远程执行 |

证据目录 `docs/verification/deployment/` 保存JUnit逐类统计及HTTP/持久化结果。原始日志不随公开副本上传。JUnit使用已授权的 `warehouse_management_integration_test` 与本次自建16379 Redis实例；容器接口使用独立Compose MySQL/Redis。没有把HTTP检查额外计入42项JUnit。

### 本轮发现并修复

1. 缺少后端就绪检查：新增容器专用 `ContainerHealthController`，实际检查MySQL和Redis，只公开UP/DOWN；Compose等待健康后启动前端。
2. 演示账号固定密码：新增 `002-disable-demo-accounts.sql` 与 `ContainerAdminInitializer`，只初始化空密码且禁用的种子管理员，不重置已有账号。
3. Nginx登录被原回环IP策略拒绝403：专用ingress网络给Nginx固定172.30.94.10，`003-container-login.sql`只允许该精确地址；没有关闭IP校验或放开任意地址。
4. 真实Redis断连时健康检查超过30秒：容器连接/命令超时设为2秒，修复后真实故障返回503。
5. 带浏览器Origin头的同源请求被拒绝：Nginx使用`$http_host`保留端口，且覆盖客户端传入的X-Forwarded-For。修复后带Origin的登录、销售和分页通过。

### 复现与运维边界

- `.env`已被Git忽略，包含本地随机数据库密码和`ADMIN_INITIAL_PASSWORD`；账号为`superadmin`，密码仅在本机文件查看，不复制到报告或提交。
- 本轮命令：`docker compose -p ecs-portfolio up --build -d --wait`；验证 `python deploy/verify-local.py`；重启后 `python deploy/verify-local.py --verify-restart`。Windows若Docker未入PATH，给脚本设置`DOCKER_EXE`。脚本只操作固定本地项目并保留合成记录，不是生产测试脚本。
- `002`只可用于全新空卷初始化，绝不对既有业务库执行。当前容器初始化后才加入的`003`已仅在本次隔离库执行；未来空卷自动执行三份SQL。已有卷不会自动迁移，禁止用`down -v`排错。
- 入口只绑定本机回环。IP白名单在此模式识别Nginx，不是公网客户端级IP策略；ingress使用172.30.94.0/24，其他主机若网段冲突需同步调整Compose及003。不得直接复制到公网部署。
- 数据库/上传卷保留，测试服务当前仍运行。仅停止了测试自行创建的主机Redis进程；未删除Docker/WSL或备份数据。

### 尚未验证或不在范围内

构建中的npm ci报告13项依赖漏洞（3 moderate、10 high）。本轮没有执行自动升级或可利用性分析；列入后续安全处理，不能声称依赖安全审计已通过。

远程GitHub Actions未运行；浏览器页面点击与完整UI交互未自动化（已验证SPA和带Origin的真实接口）；AI知识库路径/模型密钥未配置，容器日志有知识库未加载提示；旧业务账号创建/重置密码策略、演示历史库存追溯、全量ERP功能及生产TLS/备份未扩展。源码原作者已许可公开发布；本发布副本另经安全整理。本轮未修改前三阶段业务算法，也未公开推送。

### 可用于简历的表述

在既有Spring Boot/Vue系统上完成Docker Compose本地工程化：实现前后端多阶段构建、MySQL/Redis健康依赖、强密码管理员初始化和持久化卷；修复反向代理IP白名单与同源转发问题，通过真实容器登录、销售防重和库存流水核对及42项JUnit回归；编写GitHub Actions构建测试工作流（远程运行待验证）。
