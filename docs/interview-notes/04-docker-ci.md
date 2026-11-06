> 发布说明：本报告记录原本地限定验收；公开副本安全清理及重新验证结果以 [发布整理报告](../publication-safety.md) 为准。原始日志与本机信息未公开。

# Docker与CI：第四阶段本地验收

## 最终实现与证据（2026-11-06）

以下是最新结果，文末旧记录仅作为排查历史。本轮独立Compose四服务健康，镜像实际构建、HTTP业务、重启持久化已通过；最后一次JUnit运行42项全部通过，远程Actions没有运行。

### 代码与请求链

- `back/Dockerfile`：Maven/Java17编译，JRE非root运行；容器专用properties替换本地配置，避免把开发密码打进镜像。
- `front/Dockerfile`、`deploy/nginx.conf`：npm ci/Vite产物交给Nginx；`/api`转发backend:8080并保留原Host及端口。浏览器 → Nginx → Sa-Token/IP拦截 → Controller/Service → MySQL；防重切面访问Redis。
- `compose.yaml`：MySQL/Redis不发布主机端口，前端只发布127.0.0.1:18080；后端实际依赖检查成功后前端才启动。数据库与上传目录分别有命名卷。
- `ContainerHealthController`：`/api/health/ready`运行SELECT 1和Redis PING，成功200/UP、依赖失败503/DOWN，不输出内部异常。Redis连接及命令超时2秒避免故障检查长时间挂起。
- `002-disable-demo-accounts.sql`：只在空卷初始化阶段清空演示密码、停用演示账号。`ContainerAdminInitializer`仅匹配禁用且空密码的superadmin，用环境变量强密码生成BCrypt；重启不覆盖旧密码。两项H2测试验证这些边界，真实MySQL启动/重启也验证成功。
- `003-container-login.sql`：只为专用Nginx的172.30.94.10/32开放登录策略；没有关闭原IP校验。本机模式不等于公网真实客户端IP过滤。
- `.github/workflows/verify.yml`：临时MySQL/Redis、Java17、42项测试、Vue及前后端镜像构建，无发布步骤。编写了工作流不等于远程CI已通过。

### 真实验证

证据见`docs/verification/deployment/`。运行`python deploy/verify-local.py`，验证SPA、含Origin的HTTP、密码登录、未登录拦截、员工分页、商品初始10→销售2后8→重复销售拒绝→删除恢复10，两笔库存流水合计0。脚本只对固定本地项目运行，保留合成数据。

运行`docker compose -p ecs-portfolio restart`，等待健康后执行`python deploy/verify-local.py --verify-restart`，验证账号、数据库记录和uploads卷标记未丢失。只重启，不使用`down -v`。本轮还真实停止/恢复隔离Redis，验证健康接口503及恢复。

### 面试问题与回答依据

1. **为什么容器Running还不算部署成功？** 本次四服务健康后仍遇到登录403；必须进一步验证权限链、反向代理和实际业务写入。
2. **为什么用服务名连接数据库？** backend容器的localhost不是MySQL；Compose网络通过mysql/redis服务名解析。
3. **为什么出现同源请求403？** 原Nginx `$host`丢掉18080端口，后端将浏览器Origin判为跨域；改为`$http_host`并以真实Origin头复测。
4. **健康检查为什么还要配超时？** Redis默认超时让一次故障请求超过30秒；本次容器配置2秒超时后返回503，不能只看进程是否存在。
5. **管理员初始化为什么不每次重设密码？** UPDATE同时限定空密码和禁用状态，已有账号重启不会再次匹配，避免配置变化意外覆盖密码。
6. **卷和容器有什么区别？** 本次重启后数据库和上传文件仍在；卷保存业务数据，禁止通过删卷解决初始化错误。初始化SQL只在空卷运行，已有库需另行审查迁移。
7. **Redis重启后还能保证持久业务幂等吗？** 不能，Redis防重是短时间窗口，数据库事务/唯一约束仍承担数据一致性；本阶段不改变前三阶段机制。
8. **42项测试能证明什么？** 16库存+17员工+7Redis+2初始化的限定场景；容器HTTP和持久化另列证据，不重复计数，也不代表全部ERP或浏览器UI都被测试。

### 可亲自练习与已知限制

阅读`.env`取得本地管理员密码（不要发到聊天或提交），访问http://localhost:18080。按脚本顺序观察库存变化；只在本次Compose项目停止Redis，观察健康状态再恢复。修改健康检查前先保留现有配置和测试结果。

尚未执行远程CI或浏览器页面自动化。AI知识库/模型配置未完成；npm ci报告13项依赖漏洞（3 moderate、10 high），本轮未做漏洞可利用性分析或自动升级，不代表公网安全验收。公开发布已获原作者许可；其他安全问题仍需按已知限制处理。
