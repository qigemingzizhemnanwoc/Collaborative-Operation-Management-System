# 依赖安全复核

本轮仅应用现有版本范围内的npm修复，未使用force或自动跨主版本升级。修复后Vue构建通过。

生产依赖审计无high/critical，保留ECharts moderate警报GHSA-fgmj-fm8m-jvvx。公告限定Lines系列默认HTML tooltip场景；当前图表使用bar/pie/line（并非lines），未发现该触发配置。后续若使用lines必须重新评估，不能据此宣称依赖无漏洞。

开发依赖仍有Vite/esbuild警报（开发服务器相关），最终容器由Nginx提供静态文件，不运行Vite开发服务器。开发服务器不得暴露到不可信网络；主版本升级需独立兼容性验证，本轮未强制升级。

公告：https://github.com/advisories/GHSA-fgmj-fm8m-jvvx
