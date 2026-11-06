# 企业协同运营管理系统 · Java 后端二次开发

基于 Spring Boot、MyBatis-Plus、MySQL 和 Redis 的企业协同运营管理系统二次开发项目，保留原有 Vue 3 前端。围绕数据一致性、数据库接口、故障降级与可部署性开展有测试依据的改造，用于 Java 后端实习技术展示。

## 来源与作者归属

原项目：[shanqiu127 / Collaborative-Operation-Management-System](https://github.com/shanqiu127/Collaborative-Operation-Management-System)。维护者已获得原作者的二次开发及公开发布许可，保留原作者归属；授权私人沟通记录不公开。详见 [NOTICE](NOTICE.md)。

**原项目已有**：认证与角色/部门权限、用户和部门管理、采购/销售/退货、库存、供应商和商品、统计页面、公告消息、工作要求、审批与 AI 助手相关代码。以下四项是本次二次开发，不能把整个系统描述为个人从零原创。

## 四项二次开发

| 方向 | 实际改造与证据 |
| --- | --- |
| 库存并发与事务一致性 | 保留条件更新防超卖；来源单据行锁串行化退货额度检查；条件状态更新防止重复删除变更库存；库存流水与业务同事务，唯一索引限制重复事件；真实 MySQL 并发/回滚验证 |
| 员工数据库与 API | 查询与显式历史修补分离；UNION ALL 合并普通员工和只读管理员并在数据库分页；Bean Validation；逻辑删除兼容的手机号唯一索引；跨表回滚及真实 Sa-Token 角色/部门授权验证 |
| Redis 防重复提交 | 在原 AOP + SET NX/TTL 实现上优化 SHA-256 指纹、用户/接口/参数隔离及有界本地降级；真实 Redis 并发、断连恢复及 MySQL 联合故障测试 |
| Docker 与 CI 配置 | Vue/Nginx、Java17、MySQL、Redis 四服务 Compose；健康依赖、强密码管理员初始化、持久化卷；真实容器登录/库存业务与重启验证；GitHub Actions 构建测试工作流 |

阅读：[库存验收](docs/stock-acceptance.md) · [员工验收](docs/employee-api-acceptance.md) · [Redis验收](docs/redis-acceptance.md) · [部署验收](docs/deployment-acceptance.md)。

## 技术栈与架构

Java 17、Spring Boot 3.3.5、MyBatis-Plus 3.5.5、MySQL 8、Redis、Sa-Token、Bean Validation、JUnit 5、Maven；Vue 3、Vite、Element Plus、Axios；Docker Compose、Nginx、GitHub Actions。

```mermaid
flowchart LR
    Browser[浏览器] --> Nginx[Vue 静态资源 / Nginx]
    Nginx --> API[Spring Boot / Sa-Token / Controller]
    API --> Service[业务 Service / Spring 事务]
    Service --> MySQL[(MySQL / 条件更新 / 唯一约束)]
    API --> AOP[防重复提交 AOP]
    AOP --> Redis[(Redis SET NX + TTL)]
```

MySQL 是业务一致性的边界；Redis 短时间防重不等于持久业务幂等。本地降级不能保证多实例统一防重，切换窗口的限制详见 Redis 报告。

## 功能页面与截图

保留登录、员工/用户/部门、商品/供应商、采购/销售/退货、库存统计与协同页面。当前公开副本不包含未经隐私确认的截图或附件，避免泄露真实用户资料。可按下方步骤在本地使用合成数据浏览页面；没有伪造项目截图或线上演示链接。

## Docker 快速运行

需要可运行 Linux 容器的 Docker Engine 与 Docker Compose。无需连接已有业务数据库。

1. 将 `.env.example` 复制为本地 `.env`（已有文件不要覆盖）。
2. 先填写本地应用数据库用户名 `DB_USERNAME`，再填写三个**各不相同、随机生成**的强密码：`MYSQL_ROOT_PASSWORD`、`DB_PASSWORD`、`ADMIN_INITIAL_PASSWORD`。管理员密码16–64位，含字母和数字。不要提交或分享 `.env`。
3. 运行：

```bash
docker compose -p ecs-portfolio up --build -d --wait
docker compose -p ecs-portfolio ps
```

4. 浏览 `http://localhost:18080`，账号 `superadmin`，密码为自己填写的 `ADMIN_INITIAL_PASSWORD`。没有公开固定初始密码。

数据库和 Redis 不发布主机端口；前端只绑定本机回环。数据库与上传目录使用命名卷。首次空卷运行结构及最小合成种子数据；管理员初始处于禁用状态，启动时用环境变量生成 BCrypt 密码。重启不会重置已有密码。

公开副本去除了原始演示人员、手机号、邮箱、历史业务记录和密码散列。管理员新增账号使用不可预测的随机初始值，由有权限管理员通过已有“设置密码”功能设置后再交付使用。

```bash
docker compose -p ecs-portfolio logs --tail 100 backend
docker compose -p ecs-portfolio restart
```

不要用 `down -v` 排错。`db.sql` 仅用于全新数据库，不可对已有业务库执行；增量迁移见 [docs/migrations](docs/migrations)，执行前自行备份并审查历史冲突。专用 Nginx 网段为 `172.30.94.0/24`，若冲突，需同步修改 Compose 和 `deploy/003-container-login.sql`。此配置仅供本机展示，未完成公网 TLS 与生产安全部署。

## 测试与验证

此前限定范围统一回归：**42 项通过，失败0、错误0、跳过0**（库存16、员工17、Redis7、容器初始化2）。这是已有真实运行结果，不是覆盖率或性能提升声明。安全摘要：[JUnit汇总](docs/verification/deployment/junit-summary.json)。

[发布整理报告](docs/publication-safety.md) 单独记录发布副本重新验证结果，不能以原副本的结果代替清理后的验证。

集成测试必须显式设置 `PURCHASE_TEST_JDBC_URL`、`PURCHASE_TEST_USERNAME`、`PURCHASE_TEST_PASSWORD`，仅允许本机专用 `warehouse_management_integration_test` 数据库；Redis 使用独立测试实例，`REDIS_TEST_SERVER` 指向可执行文件。禁止使用正式数据库/Redis。

```bash
./back/mvnw -f back/pom.xml -Dtest=ContainerAdminInitializerTest,RedisDuplicateIntegrationTest,EmployeeAcceptanceTest,EmployeeDirectoryIntegrationTest,EmployeeReadOnlyTest,EmployeeValidationTest,EmployeeServiceTest,AuthServiceTest,UserManageServiceTest,SalesConcurrencyIntegrationTest,PurchaseStockFlowIntegrationTest,*DeleteTest,PurchaseServiceTest test
```

[GitHub Actions](.github/workflows/verify.yml) 已配置 Java17、临时 MySQL/Redis、指定 JUnit 集合、Vue 构建和两个镜像构建；**远程工作流尚未验证通过**，不展示未经验证的通过徽章。不是所有已有测试都纳入上述限定集合。

## 已知限制

- 本机限定验收不是生产验收；没有性能提升数据，也没有完整浏览器页面自动化。
- 历史库存不伪造补录流水；公开 SQL 只提供空业务表和明确合成的基础数据。
- Redis 故障切换可能额外放行，最终依赖数据库约束及事务；不承诺所有新建单据请求的持久幂等。
- AI 功能源码保留，但未发布私人知识库、模型密钥或会话，默认关闭；无需 AI 即可运行主要业务。
- 依赖安装曾报告13项 npm 漏洞（3 moderate、10 high），尚未完成可利用性分析和升级；代码公开不表示适合直接公网部署。
- 不包含原 Git 历史、本机凭据、原始日志、数据库备份、私人截图、上传附件或 AI 工具记录。
