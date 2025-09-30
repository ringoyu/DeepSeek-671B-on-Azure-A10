# 通过Nginx实现Ollama API安全访问的详细指南

我来详细指导你如何通过Nginx配置安全的Ollama API访问，包括SSL加密和API密钥认证。

## 1. 环境准备

首先确保你的虚拟机已安装Nginx：

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install nginx apache2-utils

# CentOS/RHEL
sudo yum install nginx httpd-tools
```

## 2. 生成SSL证书

```bash
# 创建SSL目录
sudo mkdir -p /etc/nginx/ssl
cd /etc/nginx/ssl

# 生成自签名证书（生产环境建议使用Let's Encrypt）
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout ollama.key -out ollama.crt \
  -subj "/C=US/ST=State/L=City/O=Organization/CN=your-domain.com"
```

## 3. 创建API密钥管理

### 创建API密钥文件
```bash
# 创建API密钥存储目录
sudo mkdir -p /etc/nginx/api-keys

# 生成API密钥文件（JSON格式）
sudo tee /etc/nginx/api-keys/valid_keys.json > /dev/null <<EOF
{
  "clients": [
    {
      "name": "client-app-1",
      "key": "sk-ollama-$(openssl rand -hex 16)",
      "rate_limit": "10r/s"
    },
    {
      "name": "client-app-2", 
      "key": "sk-ollama-$(openssl rand -hex 16)",
      "rate_limit": "5r/s"
    }
  ]
}
EOF

# 查看生成的密钥
sudo cat /etc/nginx/api-keys/valid_keys.json
```

## 4. 创建Nginx配置

创建Ollama专用的Nginx配置文件：

```bash
sudo tee /etc/nginx/sites-available/ollama-api > /dev/null <<'EOF'
# Ollama API安全代理配置
server {
    listen 80;
    server_name your-domain.com;  # 替换为你的域名或IP
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;  # 替换为你的域名或IP
    
    # SSL证书配置
    ssl_certificate /etc/nginx/ssl/ollama.crt;
    ssl_certificate_key /etc/nginx/ssl/ollama.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    
    # 安全头
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-XSS-Protection "1; mode=block" always;
    
    # API密钥验证
    set $api_key_valid 0;
    
    # 检查X-API-Key头
    if ($http_x_api_key = "sk-ollama-") {
        set $api_key_valid 1;
    }
    # 这里需要根据实际生成的密钥添加更多检查
    # 临时方案：我们会在后面用Lua脚本实现动态验证
    
    location / {
        # 首先尝试静态文件验证，然后使用Lua验证
        access_by_lua_file /etc/nginx/lua/validate_api_key.lua;
        
        # 代理到Ollama
        proxy_pass http://127.0.0.1:11434;
        
        # 代理设置
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # 超时设置
        proxy_connect_timeout 30s;
        proxy_send_timeout 300s;  # 长文本生成可能需要更长时间
        proxy_read_timeout 300s;
        
        # 缓冲区设置
        proxy_buffering off;
        client_max_body_size 100M;
        
        # CORS设置（按需开启）
        # add_header Access-Control-Allow-Origin "*" always;
        # add_header Access-Control-Allow-Methods "GET, POST, OPTIONS" always;
        # add_header Access-Control-Allow-Headers "X-API-Key, Content-Type" always;
        
        # 处理OPTIONS预检请求
        if ($request_method = OPTIONS) {
            add_header Access-Control-Allow-Origin "*";
            add_header Access-Control-Allow-Methods "GET, POST, OPTIONS";
            add_header Access-Control-Allow-Headers "X-API-Key, Content-Type";
            add_header Access-Control-Max-Age 86400;
            return 204;
        }
    }
    
    # 健康检查端点
    location /health {
        access_log off;
        proxy_pass http://127.0.0.1:11434/api/tags;
        proxy_set_header Host $host;
    }
    
    # 拒绝访问敏感路径
    location ~ ^/(\.git|\.env|config) {
        deny all;
        return 404;
    }
}
EOF
```

## 5. 安装和配置Lua脚本用于动态API密钥验证

```bash
# 安装Lua支持
sudo apt install nginx-extras  # Ubuntu
# 或者 sudo yum install nginx-plus  # CentOS

# 创建Lua脚本目录
sudo mkdir -p /etc/nginx/lua

# 创建API密钥验证脚本
sudo tee /etc/nginx/lua/validate_api_key.lua > /dev/null <<'EOF'
-- Ollama API密钥验证脚本
local api_key = ngx.var.http_x_api_key

-- 允许的健康检查路径无需认证
if ngx.var.uri == "/health" then
    return
end

-- 检查API密钥是否存在
if not api_key then
    ngx.log(ngx.ERR, "Missing API Key")
    ngx.status = ngx.HTTP_UNAUTHORIZED
    ngx.header.content_type = "application/json"
    ngx.say('{"error": "API key required", "code": "MISSING_API_KEY"}')
    return ngx.exit(ngx.HTTP_UNAUTHORIZED)
end

-- 验证API密钥
local valid_keys = {
    -- 将这些替换为你实际生成的密钥
    ["sk-ollama-替换为你的实际密钥1"] = { name = "client-1", limit = "10r/s" },
    ["sk-ollama-替换为你的实际密钥2"] = { name = "client-2", limit = "5r/s" }
}

local client_info = valid_keys[api_key]

if not client_info then
    ngx.log(ngx.ERR, "Invalid API Key: " .. api_key)
    ngx.status = ngx.HTTP_UNAUTHORIZED
    ngx.header.content_type = "application/json"
    ngx.say('{"error": "Invalid API key", "code": "INVALID_API_KEY"}')
    return ngx.exit(ngx.HTTP_UNAUTHORIZED)
end

-- 设置客户端标识用于日志
ngx.var.client_name = client_info.name

-- 应用速率限制
-- 注意：这需要配置相应的limit_req_zone
ngx.var.rate_limit = client_info.limit

ngx.log(ngx.INFO, "API request from: " .. client_info.name)
EOF
```

## 6. 配置速率限制

在主Nginx配置中添加速率限制：

```bash
sudo tee /etc/nginx/conf.d/rate-limiting.conf > /dev/null <<'EOF'
# 速率限制配置
limit_req_zone $binary_remote_addr zone=api_ip:10m rate=1r/s;
limit_req_zone $http_x_api_key zone=api_key:10m rate=10r/s;

# 错误页面
proxy_intercept_errors on;
error_page 401 /error/401.json;
error_page 429 /error/429.json;

location /error/ {
    internal;
    default_type application/json;
    
    location = /error/401.json {
        return 401 '{"error": "Unauthorized", "message": "Invalid or missing API key"}';
    }
    
    location = /error/429.json {
        return 429 '{"error": "Too Many Requests", "message": "Rate limit exceeded"}';
    }
}
EOF
```

## 7. 启用站点并测试

```bash
# 启用站点
sudo ln -sf /etc/nginx/sites-available/ollama-api /etc/nginx/sites-enabled/

# 测试配置
sudo nginx -t

# 重启Nginx
sudo systemctl restart nginx

# 查看状态
sudo systemctl status nginx
```

## 8. 创建测试脚本

创建API测试脚本：

```bash
tee test_ollama_api.sh > /dev/null <<'EOF'
#!/bin/bash

# 配置信息
API_BASE="https://your-domain.com"  # 替换为你的域名或IP
API_KEY="sk-ollama-替换为你的实际密钥"  # 替换为实际密钥

# 测试健康检查
echo "1. 测试健康检查..."
curl -k -s "$API_BASE/health" | jq .

# 测试API密钥认证（缺少密钥）
echo -e "\n2. 测试缺少API密钥..."
curl -k -s -X POST "$API_BASE/api/generate" \
  -H "Content-Type: application/json" \
  -d '{"model":"deepseek:32b","prompt":"hello"}' | jq .

# 测试错误API密钥
echo -e "\n3. 测试错误API密钥..."
curl -k -s -X POST "$API_BASE/api/generate" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: invalid-key" \
  -d '{"model":"deepseek:32b","prompt":"hello"}' | jq .

# 测试正确API密钥
echo -e "\n4. 测试正确API密钥..."
curl -k -s -X POST "$API_BASE/api/generate" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $API_KEY" \
  -d '{
    "model": "deepseek:32b",
    "prompt": "请用简短的话介绍一下人工智能",
    "stream": false
  }' | jq .

echo -e "\n测试完成！"
EOF

chmod +x test_ollama_api.sh
```

## 9. Python客户端示例

创建Python客户端示例：

```python
# ollama_client.py
import requests
import json

class OllamaClient:
    def __init__(self, base_url, api_key):
        self.base_url = base_url
        self.headers = {
            "X-API-Key": api_key,
            "Content-Type": "application/json"
        }
    
    def generate_text(self, model, prompt, stream=False):
        """生成文本"""
        data = {
            "model": model,
            "prompt": prompt,
            "stream": stream
        }
        
        response = requests.post(
            f"{self.base_url}/api/generate",
            headers=self.headers,
            json=data,
            verify=False  # 仅用于自签名证书，生产环境应使用有效证书
        )
        
        if response.status_code == 200:
            return response.json()
        else:
            raise Exception(f"API Error: {response.status_code} - {response.text}")
    
    def chat(self, model, messages, stream=False):
        """聊天接口"""
        data = {
            "model": model,
            "messages": messages,
            "stream": stream
        }
        
        response = requests.post(
            f"{self.base_url}/api/chat",
            headers=self.headers,
            json=data,
            verify=False
        )
        
        if response.status_code == 200:
            return response.json()
        else:
            raise Exception(f"API Error: {response.status_code} - {response.text}")

# 使用示例
if __name__ == "__main__":
    client = OllamaClient(
        base_url="https://your-domain.com",  # 替换为你的域名
        api_key="sk-ollama-替换为你的实际密钥"  # 替换为实际密钥
    )
    
    # 测试文本生成
    try:
        result = client.generate_text(
            model="deepseek:32b",
            prompt="请用简短的话解释机器学习"
        )
        print("生成结果:", result.get("response", "No response"))
    except Exception as e:
        print(f"错误: {e}")
```

## 10. 监控和日志

配置Nginx日志以监控API使用：

```bash
# 在Nginx配置中添加详细的日志格式
sudo tee /etc/nginx/conf.d/logging.conf > /dev/null <<'EOF'
log_format api_log '[$time_local] $remote_addr "$http_x_api_key" '
                   '"$request" $status $body_bytes_sent '
                   '"$http_referer" "$http_user_agent" '
                   'rt=$request_time uct="$upstream_connect_time" '
                   'uht="$upstream_header_time" urt="$upstream_response_time"';

access_log /var/log/nginx/api_access.log api_log;
EOF
```

## 部署完成后的检查清单

1. ✅ Nginx配置语法检查：`sudo nginx -t`
2. ✅ SSL证书路径正确
3. ✅ API密钥已更新为实际值
4. ✅ Ollama服务在11434端口运行
5. ✅ 防火墙已开放80/443端口
6. ✅ 测试脚本能够正常工作

## 故障排除

如果遇到问题，检查以下位置：

```bash
# 检查Nginx错误日志
sudo tail -f /var/log/nginx/error.log

# 检查API访问日志
sudo tail -f /var/log/nginx/api_access.log

# 检查Ollama服务状态
sudo systemctl status ollama  # 如果使用systemd管理

# 测试本地Ollama访问
curl http://localhost:11434/api/tags
```

这样就完成了通过Nginx实现安全的Ollama API访问！你现在可以通过HTTPS和API密钥安全地调用DeepSeek 32B模型了。
