# n8n_service

這個服務使用 Docker Compose 來部署 [n8n](https://n8n.io/) 工作流自動化平台及其所需的 PostgreSQL 資料庫。

## 環境配置

本服務由以下兩個容器組成：

1. **PostgreSQL 資料庫**：
   - 版本：PostgreSQL 16
   - 用途：為 n8n 提供資料儲存

2. **n8n 工作流自動化平台**：
   - 映像：docker.n8n.io/n8nio/n8n
   - 端口：5678
   - 用途：提供工作流自動化服務

## 環境變數

啟動服務前需要配置以下環境變數：

- `POSTGRES_USER`：PostgreSQL 超級用戶名稱
- `POSTGRES_PASSWORD`：PostgreSQL 超級用戶密碼
- `POSTGRES_DB`：n8n 使用的資料庫名稱
- `POSTGRES_NON_ROOT_USER`：n8n 連接資料庫的普通用戶名稱
- `POSTGRES_NON_ROOT_PASSWORD`：n8n 連接資料庫的普通用戶密碼
- `SUBDOMAIN`：n8n 服務的子域名
- `DOMAIN_NAME`：n8n 服務的主域名

## 持久化儲存

服務使用兩個 Docker 卷來保證資料持久化：

- `db_storage`：PostgreSQL 資料庫資料儲存
- `n8n_storage`：n8n 工作流和設定檔案儲存

## 網絡配置

n8n 服務配置為使用 HTTPS 協議，網址為 `https://${SUBDOMAIN}.${DOMAIN_NAME}/`。

## Nginx 反向代理

本服務使用 Nginx 作為反向代理，提供 SSL 加密和域名訪問：

- **配置文件**：`nginx/nginx-site.confg`
- **域名**：n8n.site.com
- **SSL**：使用 Let's Encrypt 提供的 SSL 證書
- **代理設置**：
  - 將來自 443 端口的 HTTPS 請求轉發到 n8n 服務的 5678 端口
  - 配置了 WebSocket 支援 (Upgrade 和 Connection 標頭)
  - 設置了正確的請求頭以保證 n8n 正常運行
  - HTTP 請求會被自動重定向到 HTTPS

### Nginx 配置摘要
```nginx
# 主要 HTTPS 設定
server {
    server_name n8n.site.com;
    listen 443 ssl;
    
    # 代理設置
    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header Origin https://$host;
        # 其他代理設定...
    }
    
    # SSL 配置
    ssl_certificate /etc/letsencrypt/live/n8n.site.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/n8n.site.com/privkey.pem;
}

# HTTP 轉 HTTPS 重定向
server {
    listen 80;
    server_name n8n.site.com;
    return 301 https://$host$request_uri;
}
```

## 啟動方式

確保已配置好所有環境變數後，執行以下命令啟動服務：

```bash
docker-compose up -d
```

啟動後可通過 `https://n8n.site.com/` 安全地訪問 n8n 界面。

## 安全注意事項

- 服務中設定 `N8N_SECURE_COOKIE=false`，若需更高安全性可考慮修改
- 使用非 root 用戶連接資料庫以提高安全性
- 透過 Nginx 啟用 SSL 加密，確保資料傳輸安全
- 所有 HTTP 流量都會自動重定向到 HTTPS 