# 301
智能301跳转系统 - 完整项目文档

📋 项目概述

项目名称

智能301跳转系统 - 基于健康检测的自动容灾域名跳转方案

项目简介

本系统实现了一个智能的301跳转服务，当用户访问固定入口域名时，系统会自动生成随机三级域名，经过健康检测确认可用后，将用户301跳转到该随机域名。整个过程自动完成，支持HTTPS、自动容灾和证书自动续期。

核心特性

· ✅ 智能域名生成：自动生成8位随机字符串作为三级域名
· ✅ 健康检测机制：DNS解析检测 + 端口连通性检测
· ✅ 自动容灾：不可用域名自动重试，最多10次，支持降级备用域名
· ✅ HTTPS支持：全站HTTPS加密，Let's Encrypt证书自动续期
· ✅ 高可用架构：Nginx反向代理 + Python Flask服务
· ✅ 一键部署：Docker Compose + Systemd 双部署方案

---

🏗️ 系统架构

整体架构图

```
用户访问 https://abc.a1cc.site/test
         ↓ (301)
健康检测服务 http://check.twziranfeng.xyz:8080
         ↓ (生成随机域名)
检测：随机字符串.twziranfeng.xyz
         ↓ (检测通过，301)
https://随机字符串.twziranfeng.xyz/test
         ↓ (200)
显示欢迎页面
```

技术栈

组件 技术 版本
反向代理 Nginx 1.24+
后端服务 Python Flask 3.11+
DNS解析 dnspython 2.6+
SSL证书 Let's Encrypt Certbot
进程管理 Systemd -
容器化 Docker + Docker Compose 可选

域名规划

域名 用途 SSL证书
abc.a1cc.site 用户固定访问入口 独立证书
check.twziranfeng.xyz 健康检测服务API 泛域名证书
*.twziranfeng.xyz 随机域名目标服务 泛域名证书
backup.twziranfeng.xyz 降级备用服务 泛域名证书

---

📁 项目文件结构

```
/www/wwwroot/test/
├── health_check.py          # Python健康检测服务
├── index.html               # 前端测试工具页面
├── requirements.txt         # Python依赖
├── nginx.conf              # Nginx配置文件
├── docker-compose.yml      # Docker编排文件
├── Dockerfile              # Docker镜像构建
├── deploy.sh               # 一键部署脚本
└── README.md               # 项目说明文档
```

---

💻 核心代码实现

1. Python健康检测服务 (health_check.py)

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
智能301跳转健康检测服务
"""

from flask import Flask, redirect, request, jsonify
import random
import string
import socket
import dns.resolver
import logging
from datetime import datetime
import time
import os

app = Flask(__name__)

# 配置参数
BASE_DOMAIN = 'twziranfeng.xyz'      # 泛解析域名
MAX_ATTEMPTS = 10                     # 最大尝试次数
CHECK_TIMEOUT = 2                     # 检测超时（秒）
PORT = 8080                           # 服务端口
TEST_PAGE_PATH = '/www/wwwroot/test/index.html'

# 日志配置
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


def generate_random_domain(length=8):
    """生成随机三级域名"""
    chars = string.ascii_lowercase + string.digits
    random_str = ''.join(random.choices(chars, k=length))
    return f"{random_str}.{BASE_DOMAIN}"


def check_domain_available(domain):
    """检测域名是否可用（DNS + 端口）"""
    try:
        # DNS解析检测
        answers = dns.resolver.resolve(domain, 'A')
        if not answers:
            return False
        
        ip = str(answers[0])
        logger.debug(f"DNS解析成功: {domain} -> {ip}")
        
        # 端口检测（优先443，其次80）
        for port in [443, 80]:
            try:
                sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
                sock.settimeout(CHECK_TIMEOUT)
                result = sock.connect_ex((ip, port))
                sock.close()
                if result == 0:
                    logger.debug(f"端口{port}可达: {domain}")
                    return True
            except Exception as e:
                logger.debug(f"端口{port}检测异常: {e}")
        
        return False
        
    except Exception as e:
        logger.debug(f"检测异常 {domain}: {e}")
        return False


def find_available_domain():
    """查找可用的随机域名"""
    for attempt in range(1, MAX_ATTEMPTS + 1):
        candidate = generate_random_domain()
        logger.info(f"尝试 #{attempt}: 检测 {candidate}")
        
        if check_domain_available(candidate):
            logger.info(f"✅ 找到可用域名: {candidate}")
            return candidate, attempt
        
        time.sleep(0.1)
    
    # 降级备用域名
    fallback = f"backup.{BASE_DOMAIN}"
    logger.warning(f"⚠️ 使用降级域名: {fallback}")
    return fallback, MAX_ATTEMPTS


@app.route('/')
def redirect_to_healthy():
    """主跳转接口"""
    original_path = request.args.get('path', '/')
    if not original_path.startswith('/'):
        original_path = '/' + original_path
    
    logger.info(f"收到请求: {request.remote_addr} -> path={original_path}")
    
    target_domain, attempts = find_available_domain()
    target_url = f"https://{target_domain}{original_path}"
    
    logger.info(f"301跳转: {target_url} (尝试{attempts}次)")
    return redirect(target_url, code=301)


@app.route('/health')
def health_check():
    """健康检查接口"""
    return jsonify({
        'status': 'ok',
        'timestamp': datetime.now().isoformat(),
        'base_domain': BASE_DOMAIN,
        'max_attempts': MAX_ATTEMPTS
    })


@app.route('/test-page')
def test_page():
    """前端测试工具页面"""
    try:
        if os.path.exists(TEST_PAGE_PATH):
            with open(TEST_PAGE_PATH, 'r', encoding='utf-8') as f:
                return f.read()
        return "<h1>404</h1><p>测试页面不存在</p>", 404
    except Exception as e:
        return f"<h1>Error</h1><p>{e}</p>", 500


if __name__ == '__main__':
    logger.info(f"启动健康检测服务 - {BASE_DOMAIN}")
    app.run(host='0.0.0.0', port=PORT, debug=False, threaded=True)
```

2. Nginx完整配置 (nginx.conf)

```nginx
# ============================================
# 智能301跳转系统 - 全站HTTPS配置
# ============================================

# 入口域名 HTTPS
server {
    listen 443 ssl http2;
    server_name abc.a1cc.site;
    
    ssl_certificate /etc/letsencrypt/live/abc.a1cc.site/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/abc.a1cc.site/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
    
    location / {
        return 301 http://check.twziranfeng.xyz:8080/?path=$request_uri;
    }
}

# 入口域名 HTTP -> HTTPS
server {
    listen 80;
    server_name abc.a1cc.site;
    return 301 https://abc.a1cc.site$request_uri;
}

# 健康检测服务 HTTPS
server {
    listen 443 ssl http2;
    server_name check.twziranfeng.xyz;
    
    ssl_certificate /etc/letsencrypt/live/twziranfeng.xyz/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/twziranfeng.xyz/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# 泛域名目标服务 HTTPS
server {
    listen 443 ssl http2;
    server_name ~^(?<subdomain>.+)\.twziranfeng\.xyz$;
    
    ssl_certificate /etc/letsencrypt/live/twziranfeng.xyz/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/twziranfeng.xyz/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    
    location / {
        default_type text/html;
        charset utf-8;
        return 200 "<!DOCTYPE html><html><head><title>Welcome</title></head>
                    <body><h1>Welcome to $subdomain.twziranfeng.xyz</h1>
                    <p>✅ HTTPS随机域名生效！</p>
                    <p>访问时间: $time_local</p></body></html>";
    }
}
```

3. Systemd服务配置

```ini
# /etc/systemd/system/smart-redirect.service
[Unit]
Description=Smart 301 Redirect Health Check
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/www/wwwroot/test
ExecStart=/www/wwwroot/test/venv/bin/python /www/wwwroot/test/health_check.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

4. Docker Compose配置

```yaml
version: '3.8'

services:
  health-check:
    build: .
    container_name: smart-redirect
    restart: always
    ports:
      - "8080:8080"
    volumes:
      - ./logs:/app/logs
    environment:
      - BASE_DOMAIN=twziranfeng.xyz
      - MAX_ATTEMPTS=10
      - CHECK_TIMEOUT=2

  nginx:
    image: nginx:alpine
    container_name: redirect-nginx
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - health-check
```

---

🚀 部署指南

前提条件

· 服务器：Ubuntu 22.04+ / CentOS 8+
· 域名：已配置DNS泛解析 *.twziranfeng.xyz
· 端口开放：80、443、8080
· Cloudflare账号（用于DNS API）

快速部署（Systemd方案）

```bash
# 1. 创建项目目录
mkdir -p /www/wwwroot/test && cd /www/wwwroot/test

# 2. 克隆/上传代码文件
# 将 health_check.py、index.html 等文件放到此目录

# 3. 创建虚拟环境
python3 -m venv venv
source venv/bin/activate
pip install flask dnspython

# 4. 配置Nginx
cp nginx.conf /etc/nginx/sites-available/twziranfeng
ln -sf /etc/nginx/sites-available/twziranfeng /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx

# 5. 配置Systemd服务
cp smart-redirect.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable smart-redirect
systemctl start smart-redirect

# 6. 申请SSL证书
# abc.a1cc.site 证书
certbot --nginx -d abc.a1cc.site

# 泛域名证书
certbot certonly --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  -d twziranfeng.xyz -d "*.twziranfeng.xyz"
```

DNS配置（Cloudflare）

类型 主机记录 记录值 代理状态
A abc.a1cc.site 服务器IP 已代理（橙色云）
A check.twziranfeng.xyz 服务器IP 已代理（橙色云）
A *.twziranfeng.xyz 服务器IP 已代理（橙色云）
A backup.twziranfeng.xyz 服务器IP 已代理（橙色云）

SSL证书申请（Cloudflare API）

```bash
# 1. 创建API Token凭证
cat > /etc/letsencrypt/cloudflare.ini << EOF
dns_cloudflare_api_token = your_api_token
EOF
chmod 600 /etc/letsencrypt/cloudflare.ini

# 2. 申请泛域名证书
certbot certonly --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  -d twziranfeng.xyz -d "*.twziranfeng.xyz"

# 3. 测试自动续期
certbot renew --dry-run
```

---

🧪 测试验证

1. 命令行测试

```bash
# 测试入口域名
curl -I https://abc.a1cc.site/test

# 测试健康检测服务
curl https://check.twziranfeng.xyz/health

# 测试泛域名
curl https://test123.twziranfeng.xyz/

# 测试完整跳转流程
curl -L https://abc.a1cc.site/dashboard
```

2. 浏览器测试

访问以下URL验证功能：

· https://abc.a1cc.site/test - 入口跳转测试
· https://check.twziranfeng.xyz/test-page - 测试工具页面
· https://check.twziranfeng.xyz/health - 健康检查API

3. 压力测试（可选）

```bash
# 使用ab工具进行压力测试
ab -n 1000 -c 10 https://abc.a1cc.site/test
```

---

📊 监控与运维

日志查看

```bash
# Nginx访问日志
tail -f /var/log/nginx/abc_ssl_access.log
tail -f /var/log/nginx/twziranfeng_ssl_access.log

# Python服务日志
journalctl -u smart-redirect -f

# 证书续期日志
tail -f /var/log/letsencrypt/letsencrypt.log
```

服务管理

```bash
# 重启服务
systemctl restart smart-redirect
systemctl reload nginx

# 查看状态
systemctl status smart-redirect
systemctl status nginx

# 查看资源占用
htop
netstat -tlnp | grep -E "80|443|8080"
```

证书管理

```bash
# 查看证书信息
certbot certificates

# 手动续期
certbot renew

# 测试续期
certbot renew --dry-run
```

---

🔧 故障排查

问题 可能原因 解决方法
访问显示404 Nginx配置未生效 nginx -t && systemctl reload nginx
跳转不生效 Cloudflare缓存 清除Cloudflare缓存，开启开发模式
证书错误 证书未申请或过期 certbot renew
Python服务停止 进程崩溃 systemctl restart smart-redirect
DNS检测失败 泛解析未生效 检查Cloudflare DNS配置
中文乱码 字符编码问题 添加 charset utf-8;

---

📈 性能优化建议

1. 使用Gunicorn替代Flask开发服务器

```bash
pip install gunicorn
gunicorn -w 4 -b 0.0.0.0:8080 health_check:app
```

1. 添加Redis缓存检测结果

```python
# 缓存可用域名，减少重复检测
cache.set(domain, True, timeout=300)
```

1. 配置Nginx缓存静态资源

```nginx
location ~* \.(jpg|png|css|js)$ {
    expires 30d;
    add_header Cache-Control "public, immutable";
}
```

1. 启用Nginx Gzip压缩

```nginx
gzip on;
gzip_types text/html text/css application/json;
```

---

🔒 安全加固

1. 配置防火墙

```bash
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 22/tcp
ufw enable
```

1. 限制API访问频率（Nginx）

```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
location /health {
    limit_req zone=api burst=20;
    proxy_pass http://127.0.0.1:8080/health;
}
```

1. 隐藏Nginx版本

```nginx
server_tokens off;
```

---

📝 项目总结

技术亮点

· ✅ 零维护随机域名池：无需预设域名列表，动态生成
· ✅ 智能健康检测：DNS+端口双重检测，确保域名可用
· ✅ 自动容灾降级：10次重试 + 备用域名兜底
· ✅ 全站HTTPS：Let's Encrypt自动证书管理
· ✅ 双部署方案：Systemd + Docker Compose

适用场景

· 防封禁域名跳转系统
· 负载均衡入口服务
· 多节点容灾架构
· 动态CDN调度系统

扩展方向

· 支持多地域检测（就近跳转）
· 增加域名访问质量评分
· 集成Redis缓存优化性能
· 开发Web管理控制台

---

📄 许可证

MIT License

---

📞 联系方式

如有问题或建议，请通过以下方式联系：

· Email: mmm1abc@163.com
· GitHub: [项目仓库地址]

---

项目状态： ✅ 生产就绪
最后更新： 2026-04-23
版本： v2.0 (HTTPS版本)

---

本文档涵盖了项目的完整技术细节、部署流程和运维指南。请根据实际环境调整配置参数。
