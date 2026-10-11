# 金信文创商城 · 部署手册

> **版本**：v1.0 · 2026-10-11 · 三端一体部署指南
> **适用**：每个 buy 商城（C 端小程序）+ 管理后台（business-portal）+ 后端 API（Business-API）
> **目标**：让运维 / DevOps 同学按本文可独立完成 dev / test / prod 三套环境的发布与回滚

---

## 0. 仓库 & 端口速查

| 端 | 仓库 | 分支 | 技术栈 | 端口 | 部署路径 |
|---|---|---|---|---|---|
| **C 端小程序** | `gitee.com/beijing_jane_harvest_javaandroidc/Eachbuy_EndUserMini_JXWC` | `master` | 微信小程序原生 + Vant Weapp | — | 通过微信开发者工具上传 |
| **管理后台前端** | `gitee.com/jdcai/business-portal_jxwc.git` | `master` | Vue 2 + vue-cli 4 + Ant Design Vue + JeecgBoot | 3001 (dev) | `/usr/share/nginx/html/eachbuybackstage` |
| **后端 API** | `github.com/BCSCLG/Business-API_jxwc` | `main` | Spring Boot 2.3.5 + JeecgBoot 2.4.2 + MyBatis-Plus | 5000 (gunicorn) / 5001 (部署触发) | `/home/admintalu/application` |
| **DB / Redis / OSS** | — | — | MySQL 5.7+ / Redis 6+ / 阿里云 OSS | 43306 / 6379 | 阿里云 RDS / 自建 |

**核心域名**：
- 生产：`https://www.jdc51.com`
- 小程序：`/eachbuy/wecom/callback`（企微回调）
- 管理后台：`/eachbuybackstage`
- API 网关：`/eachbuy`（业务前缀）

---

## 1. 环境矩阵

| 环境 | 后端 API | 前端域名 | 数据库 | 用途 |
|---|---|---|---|---|
| **dev** | `localhost:5000` | `http://localhost:3001` | 本地 MySQL / Mock 数据 | 开发自测 |
| **test** | `https://test-api.jdc51.com/eachbuy` | `https://test-admin.jdc51.com/eachbuybackstage` | `wcyp_test` 库 | 联调 / UAT |
| **prod** | `https://www.jdc51.com/eachbuy` | `https://www.jdc51.com/eachbuybackstage` | `wcyp.mysql.db.cn` | 生产 |

---

## 2. 前置准备

### 2.1 服务器清单（生产）

| 角色 | 规格 | 数量 | 操作系统 |
|---|---|---|---|
| 应用服务器 | 4C8G | 2 台（双机热备） | CentOS 7.9 / Ubuntu 20.04 |
| 数据库 | RDS MySQL 5.7 高可用版 | 1 实例 | — |
| 缓存 | Redis 6.2 集群 | 1 集群 | — |
| 对象存储 | 阿里云 OSS | 1 个 Bucket | — |
| 监控 | 阿里云 ARMS + 日志服务 | — | — |

### 2.2 域名 / SSL

- 主站：`www.jdc51.com`（已 ICP 备案 + SSL）
- API：`www.jdc51.com/eachbuy/*`
- 回调：`www.jdc51.com/eachbuy/wecom/callback`（企业微信）

### 2.3 凭据管理

⚠️ **生产密钥禁止入库**。所有敏感配置通过环境变量注入。

完整 7 个密钥字段结构见 **`jdc51.sh.example`**（仓库内可入库的模板）：

```bash
# 生产服务器文件：/etc/profile.d/jdc51.sh
# 本地模板文件：./jdc51.sh.example  ← 仓库内可入库
# 本地真实文件：./jdc51.sh          ← 在 .gitignore 中，禁止 commit
#
# 部署步骤：
#   1. 复制仓库内 jdc51.sh.example → 本地 jdc51.sh
#   2. 填入 7 个字段的真实值（从云端控制台获取）
#   3. scp jdc51.sh 到服务器 /etc/profile.d/
#   4. sudo chmod 600 /etc/profile.d/jdc51.sh
#   5. sudo chown root:root /etc/profile.d/jdc51.sh
#   6. 重新登录或 source /etc/profile.d/jdc51.sh
```

**7 个必填字段**：

| 字段 | 来源 |
|---|---|
| `MYSQL_PASSWORD` | 阿里云 RDS 控制台 / 自建 MySQL 配置 |
| `REDIS_PASSWORD` | Redis 实例配置 |
| `OSS_ACCESS_KEY_ID` | 阿里云 RAM → AccessKey（`LTAI` 开头）|
| `OSS_ACCESS_KEY_SECRET` | 阿里云 RAM → AccessKey Secret |
| `WECOM_CORP_SECRET` | 企业微信管理后台 → 应用 → 密钥 |
| `WECHAT_PAY_KEY` | 微信支付商户平台 → API 安全 → APIv3 密钥 |
| `DRUID_PASSWORD` | 自行设定（用于 `/actuator/druid` 监控登录）|

**验证**：

```bash
env | grep -E "MYSQL_PASSWORD|REDIS_PASSWORD|OSS_ACCESS_KEY|WECOM_CORP|WECHAT_PAY|DRUID_PASSWORD"
# 应输出 7 行非空值
```

**紧急撤销**（密钥泄露时）：见 `jdc51.sh.example` 顶部注释。

本地开发复制 `application-dev.yml.example` → `application-dev.yml` 填入测试值。

---

## 3. 后端 API 部署（最复杂）

### 3.1 部署脚本（已存在）

```bash
# 服务器路径
/home/admintalu/application

# 重启脚本（已部署在服务器）
sh /home/admintalu/application/Restart-eaechbuy51-module-system-prod.sh
```

### 3.2 完整部署步骤

```bash
# ====== 步骤 1：登录服务器 ======
ssh admintalu@192.168.2.14   # chatbot001（生产）

# ====== 步骤 2：备份当前版本（必须） ======
cd /home/admintalu/application
TS=$(date +%Y%m%d_%H%M%S)
tar czf /backup/api-${TS}.tar.gz jeecg-boot-module-system/
echo "✅ 已备份到 /backup/api-${TS}.tar.gz"

# ====== 步骤 3：拉取 release 包或 git 拉代码 ======
# 方式 A：CI/CD 包（推荐，prod 环境）
cd /tmp
wget https://ci.jdc51.com/api-${CI_BUILD_TAG}.tar.gz -O api-new.tar.gz

# 方式 B：git 拉取（test 环境）
cd /home/admintalu/source/Business-API_jxwc
git fetch --tags
git checkout v1.2.3
mvn -B clean package -DskipTests

# ====== 步骤 4：停止旧服务 ======
sh /home/admintalu/application/Restart-eaechbuy51-module-system-prod.sh stop

# ====== 步骤 5：替换 jar ======
cp -r jeecg-boot-module-system/target/jeecg-boot-module-system-*.jar \
      /home/admintalu/application/jeecg-boot-module-system/

# ====== 步骤 6：加载新环境变量 ======
source /etc/profile.d/jdc51.sh

# ====== 步骤 7：启动服务 ======
nohup java -jar \
  -Xms2g -Xmx4g -XX:+UseG1GC \
  -Dspring.profiles.active=prod \
  -Dlogging.level.root=INFO \
  jeecg-boot-module-system-*.jar \
  --server.port=5000 \
  > /home/admintalu/logs/api.log 2>&1 &

# ====== 步骤 8：健康检查 ======
sleep 15
curl -s http://localhost:5000/eachbuy/actuator/health
# 期望：{"status":"UP"}

# ====== 步骤 9：检查启动日志 ======
tail -100 /home/admintalu/logs/api.log | grep -E "Started|ERROR"
```

### 3.3 数据库迁移

```bash
# 1. 备份（永远先备份）
mysqldump -u root -h wcyp.mysql.db.cn -P 43306 \
  --single-transaction --routines --triggers \
  wcyp_prod > /backup/db-${TS}.sql

# 2. 跑迁移（Flyway 自动管理）
# 启动时 Flyway 会自动执行 db/migration/V*.sql

# 3. 验证
mysql -h wcyp.mysql.db.cn -u readonly -p wcyp_prod \
  -e "SELECT version, description, success FROM flyway_schema_history ORDER BY installed_rank DESC LIMIT 5"
```

### 3.4 部署后冒烟测试

```bash
# 健康检查
curl https://www.jdc51.com/eachbuy/actuator/health

# 登录测试
curl -X POST https://www.jdc51.com/eachbuy/sys/login \
  -H "Content-Type: application/json" \
  -d '{"username":"smoke_test","password":"test123","captcha":"test"}'

# 模拟企微回调
curl -X POST https://www.jdc51.com/eachbuy/wecom/callback \
  -H "Content-Type: application/xml" \
  -d '<xml>...</xml>'
```

---

## 4. 管理后台前端部署

### 4.1 构建

```bash
# ====== 本机构建 ======
cd D:\sourcecode\opensource\BCSCLG\eachwcyp\gitee\business-portal_jxwc

# 安装依赖（仅首次或依赖变更）
yarn install --registry=https://registry.npmmirror.com

# 测试环境构建
yarn build --mode test

# 生产环境构建
yarn build --mode prod
# 输出：dist/

# ====== 或 CI/CD 触发 ======
# 详见 .gitlab-ci.yml / Jenkinsfile
```

### 4.2 Nginx 部署

```bash
# ====== 上传到服务器 ======
scp -r dist/* admintalu@192.168.2.14:/tmp/eachbuybackstage-new/

# ====== 服务器侧替换 ======
ssh admintalu@192.168.2.14 << 'EOF'
  # 备份当前版本
  TS=$(date +%Y%m%d_%H%M%S)
  mv /usr/share/nginx/html/eachbuybackstage /backup/portal-${TS}

  # 部署新版本
  mv /tmp/eachbuybackstage-new /usr/share/nginx/html/eachbuybackstage
  chown -R nginx:nginx /usr/share/nginx/html/eachbuybackstage

  # 验证
  ls -la /usr/share/nginx/html/eachbuybackstage/ | head
EOF

# ====== 重载 Nginx（零停机） ======
ssh admintalu@192.168.2.14 "nginx -t && nginx -s reload"
```

### 4.3 Nginx 关键配置（已存在 `部署到服务器/nginx.conf`）

```nginx
# 管理后台 SPA（history 模式）
location /eachbuybackstage/ {
    alias /usr/share/nginx/html/eachbuybackstage/;
    try_files $uri $uri/ /eachbuybackstage/index.html;
    index index.html;

    # 缓存策略
    location ~* \.(js|css|png|jpg|svg|ico|woff2?)$ {
        expires 7d;
        add_header Cache-Control "public, immutable";
    }
    location = /eachbuybackstage/index.html {
        add_header Cache-Control "no-cache, no-store, must-revalidate";
    }

    # gzip
    gzip on;
    gzip_types text/plain application/json application/javascript text/css;
}

# API 反向代理
location /eachbuy/ {
    proxy_pass http://127.0.0.1:5000/eachbuy/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 60s;
    proxy_send_timeout 60s;
}

# 营销落地页（公开访问，无须鉴权）
location /eachbuybackstage/marketing/ {
    alias /usr/share/nginx/html/eachbuybackstage/marketing/;
}

# 企微回调（POST 接收 + 验签）
location /eachbuy/wecom/callback {
    proxy_pass http://127.0.0.1:5000/eachbuy/wecom/callback;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}

# 微信小程序支付回调
location /eachbuy/wechat/pay/callback {
    proxy_pass http://127.0.0.1:5000/eachbuy/wechat/pay/callback;
}
```

---

## 5. 微信小程序端部署

### 5.1 配置修改

```bash
cd D:\sourcecode\opensource\BCSCLG\eachwcyp\gitee\Eachbuy_EndUserMini_JXWC

# 修改配置（按环境）
# appid：
#   dev:    wx569b119af852c5a0（测试号）
#   test:   wx569b119af852c5a0（测试号）
#   prod:   wx...（正式 appid，详见微信公众平台）

# request 域名（project.config.json → setting）
# dev:   https://test-api.jdc51.com
# prod:  https://www.jdc51.com
```

### 5.2 上传步骤

```bash
# ====== 方式 A：微信开发者工具（手工） ======
# 1. 打开微信开发者工具 → 导入项目 → 选本仓库根目录
# 2. 右上角"详情" → 确认 appid / 域名白名单
# 3. 点击右上角"上传" → 填写版本号（如 1.2.3）+ 项目备注
# 4. 登录微信公众平台 https://mp.weixin.qq.com
# 5. 版本管理 → 找到刚上传的版本 → 提交审核 → 发布

# ====== 方式 B：CI/CD 自动化（miniprogram-ci） ======
# 详见 .gitlab-ci.yml，调用 miniprogram-ci 自动上传
```

### 5.3 提审清单

- [ ] 体验账号（测试用）
- [ ] 类目匹配（电商 / 文创）
- [ ] 用户协议 + 隐私政策（小程序内可见 URL）
- [ ] 小程序截图（5 张，至少 1 张运营截图）
- [ ] 版本说明 + 测试账号
- [ ] 备案号截图

---

## 6. 第三方依赖接入

### 6.1 阿里云 OSS（图片 / 文件）

```yaml
# application-prod.yml
oss:
  endpoint: oss-cn-beijing.aliyuncs.com
  bucket: jxwc
  access-key-id: ${OSS_ACCESS_KEY_ID}
  access-key-secret: ${OSS_ACCESS_KEY_SECRET}
```

回调域名：`https://www.jdc51.com/eachbuy/file/oss/callback`

### 6.2 微信支付

```
商户号：wcyp.mysql.db.cn（生产）
API v3 密钥：${WECHAT_PAY_KEY}
证书序列号：${WECHAT_PAY_CERT_SERIAL}
回调地址：/eachbuy/wechat/pay/callback
```

### 6.3 企业微信

```
CorpID：ww10exxxxx
应用 AgentID：10000xx
回调 URL：https://www.jdc51.com/eachbuy/wecom/callback
Token：chatbot
EncodingAESKey：${WECOM_ENCODING_AES_KEY}
```

详见 `部署到服务器/部署.md`。

### 6.4 短信 / 邮件 / 快递 100

详见 `运维第三方支付清单.md`。

---

## 7. 启动 / 重启 / 停止

### 7.1 后端 API

```bash
# 启动
sh /home/admintalu/application/Restart-eaechbuy51-module-system-prod.sh start

# 重启（先停后启，零停机靠蓝绿）
sh /home/admintalu/application/Restart-eaechbuy51-module-system-prod.sh restart

# 停止
sh /home/admintalu/application/Restart-eaechbuy51-module-system-prod.sh stop

# 查看进程
ps -ef | grep jeecg-boot-module-system | grep -v grep

# 查看端口
netstat -anop | grep :5000

# 查看日志（最新）
tail -f /home/admintalu/logs/api.log

# 查看错误（最近 1 小时）
grep -E "ERROR|Exception" /home/admintalu/logs/api.log | tail -100
```

### 7.2 前端（纯静态）

```bash
# 前端由 Nginx 服务，无需重启应用
# 修改文件后仅需 nginx -s reload

# 验证
curl -I https://www.jdc51.com/eachbuybackstage/
```

### 7.3 Redis

```bash
systemctl status redis
systemctl restart redis
redis-cli -h 127.0.0.1 -p 6379 -a ${REDIS_PASSWORD} ping
```

---

## 8. 监控与告警

### 8.1 应用监控

| 指标 | 阈值 | 告警渠道 |
|---|---|---|
| JVM 内存使用率 | > 80% | 钉钉群 + 短信 |
| API 5xx 错误率 | > 1%（5 分钟） | 钉钉群 |
| 接口响应时间 P99 | > 3s | 钉钉群 |
| 数据库连接池使用率 | > 85% | 钉钉群 |
| Redis 连接数 | > 10000 | 钉钉群 |

### 8.2 健康检查脚本（推荐部署到 Zabbix / Prometheus）

```bash
#!/bin/bash
# /usr/local/bin/check_api_health.sh
ENDPOINT="https://www.jdc51.com/eachbuy/actuator/health"
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" -m 5 $ENDPOINT)
if [ "$HTTP_CODE" != "200" ]; then
  echo "ALERT: API health check failed, HTTP=$HTTP_CODE"
  exit 1
fi
echo "OK"
```

### 8.3 日志聚合

```bash
# 实时错误日志（最近 24h）
find /home/admintalu/logs -name "*.log" -mtime -1 -exec \
  grep -E "ERROR|Exception" {} \; | tail -100

# 按接口统计错误数
grep -oE 'GET /eachbuy/[a-z/]+' /var/log/nginx/access.log | \
  sort | uniq -c | sort -rn | head -20
```

---

## 9. 回滚方案

### 9.1 后端 API 回滚

```bash
# 1. 停当前版本
sh /home/admintalu/application/Restart-eaechbuy51-module-system-prod.sh stop

# 2. 恢复上一个版本（保留 5 个历史版本）
cd /home/admintalu/application
LATEST_BACKUP=$(ls -t /backup/api-*.tar.gz | sed -n '2p')
echo "回滚到: $LATEST_BACKUP"
tar xzf $LATEST_BACKUP -C ./

# 3. 启动
nohup java -jar ... jeecg-boot-module-system-*.jar > /home/admintalu/logs/api.log 2>&1 &

# 4. 健康检查
sleep 15 && curl -s http://localhost:5000/eachbuy/actuator/health
```

### 9.2 前端回滚

```bash
# 1. 切回上一个版本目录
mv /usr/share/nginx/html/eachbuybackstage /usr/share/badstage-now
mv /backup/portal-${TS_LAST_GOOD} /usr/share/nginx/html/eachbuybackstage

# 2. 重载 Nginx
nginx -s reload
```

### 9.3 数据库回滚

```bash
# 1. 切换到只读模式（应用侧配合）
# 2. 恢复备份
mysql -h wcyp.mysql.db.cn -u root -p wcyp_prod < /backup/db-${TS}.sql
# 3. 应用侧回滚（参考 9.1）
```

### 9.4 小程序回滚

- 登录微信公众平台 → 版本管理 → 选上一个稳定版本 → 撤回 / 重新发布

---

## 10. 上线检查清单

### 10.1 上线前

- [ ] 后端 jar 已构建且本地通过冒烟测试
- [ ] 前端已构建且部署到 staging
- [ ] 数据库迁移脚本已通过 dry-run
- [ ] 配置文件环境变量已确认
- [ ] 第三方接口（OSS/微信/企微）已切换到生产密钥
- [ ] 已通知相关团队（运营 / 客服）
- [ ] 已准备回滚方案（备份位置已记录）

### 10.2 上线中

- [ ] 按顺序：DB 迁移 → 后端 → 前端 → Nginx reload
- [ ] 每步执行后做健康检查
- [ ] 监控告警已开启
- [ ] 至少 2 人在线协同（一人操作，一人盯监控）

### 10.3 上线后

- [ ] 持续观察日志 30 分钟（关注 ERROR）
- [ ] 跑完整冒烟测试（登录 / 下单 / 支付 / 退款）
- [ ] 监控指标正常（CPU / 内存 / 错误率）
- [ ] 通知运营 / 客服上线完成
- [ ] 更新 `运维第三方支付清单.md`（如新增第三方）

---

## 11. 常见问题

### Q1：启动报 OOM
```bash
# 调整 JVM 参数
-Xms2g -Xmx4g
# 或检查是否有内存泄漏
jmap -histo:live <pid> | head -30
```

### Q2：502 Bad Gateway
```bash
# 检查后端进程
ps -ef | grep java
# 检查端口
netstat -anop | grep :5000
# 检查 Nginx 配置
nginx -t
```

### Q3：Nginx 静态资源 404
```bash
# 确认部署路径
ls -la /usr/share/nginx/html/eachbuybackstage/
# 确认 try_files 配置
grep "try_files" /etc/nginx/nginx.conf
```

### Q4：数据库迁移失败
```bash
# 1. Flyway 历史表查看
SELECT * FROM flyway_schema_history ORDER BY installed_rank DESC LIMIT 10;
# 2. 修复后重新跑（不删除已成功记录）
# 3. 极端情况：手动 UPDATE flyway_schema_history SET success=1 WHERE version='xxx'
```

### Q5：企业微信回调验签失败
```bash
# 1. 确认 Token 一致：chatbot
# 2. 确认 EncodingAESKey 一致：jWmYm7qr5nMoAUwZRjGtBxmz3KA1tkAj3ykkR6q2B2C
# 3. 确认回调 URL 可公网访问
curl -X POST https://www.jdc51.com/eachbuy/wecom/callback
```

---

## 12. 相关文档索引

- **`部署到服务器/部署.md`**：企业微信配置详情
- **`部署到服务器/publish.md`**：原发布流程 + 重启脚本路径
- **`部署到服务器/nginx.conf`**：完整 Nginx 配置
- **`运维第三方支付清单.md`**：第三方资源费用清单
- **`pushEachwcypCodeTogitee.bat`**：5 仓库批量推送脚本（本地开发机用）
- **`../gitee/Business-API_jxwc/SECRETS.md`**：密钥管理指南（⚠️ 撤销清单）
- **`../gitee/business-portal_jxwc/DEV.md`**：本地开发环境启动
- **`../AI_MEMO.md`**：AI 服务集成备忘

---

## 附录：变更记录

| 版本 | 日期 | 作者 | 变更 |
|---|---|---|---|
| v1.0 | 2026-10-11 | AI 整理 | 首版 · 覆盖三端部署全流程 |

---

> 📌 **本手册任何环节出错，先看 11. 常见问题；还不行，联系运维 lead。**
> 📌 **生产部署必须有 2 人协同；单人禁止发布。**