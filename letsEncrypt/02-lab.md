## Labs

* **情境**：在「沒有正式 domain name」的情況，在本地體驗一下 letsencrypt 的流程。

### Overview

Lab 會使用 **Docker** 跑以下 3 個容器：

| 容器名稱 | 角色 | 核心任務 |
|--------|------|--------|
| Pebble | CA 認證單位 | 扮演 Let's Encrypt 伺服器，負責發放 Token 與簽發證書。 |
| Certbot | ACME client | 負責向 Pebble 申請證書，並處理挑戰驗證（Challenge）。 |
| Nginx | Web server | 對外提供 80 port（ACME驗證用）與 443 port（加密連線）。 |

流程概述：

1. 準備「驗證用」的 nginx config
2. 把三個容器跑起來，分別扮演 CA 認證單位、ACME client、Web server 的角色。
3. 使用 certbot 來申請憑證。成功申請到的憑證會放在 certbot container 的 `/etc/letsencrypt/live/${domain_name}/` 目錄下。
4. 把憑證放到「正式」的 nginx config 上，並且測試看看。
 

### 環境準備

1. [安裝 Docker](https://docs.docker.com/engine/install/)

2. 建立 Lab 目錄 & 必要的子目錄：

```bash
mkdir -p pebble-lab && cd pebble-lab

# 建立關鍵目錄
# webroot：會掛載到 nginx container 上，作為 ACME 驗證用的目錄。
# letsencrypt：會掛載到 certbot container 上，作為憑證放置的目錄。
mkdir -p webroot letsencrypt

# (Optional) 調整權限，確保 docker container 可以讀寫這些目錄。
sudo chmod -R 777 webroot letsencrypt
```

3. 下載 pebble 設定檔：

```bash
curl -o pebble-config.json https://raw.githubusercontent.com/letsencrypt/pebble/main/test/config/pebble-config.json
```


1. 準備｢驗證用」的 nginx config：

```bash
domain_name="www.example.com"

echo "server {
    listen 80;
    listen [::]:80;
    server_name ${domain_name};

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
}" >nginx_validate.conf
```


2. 把一個 nginx 容器跑起來，作為 ACME client 的驗證用：

```bash
cert_path="./certbot_data"

mkdir -p $cert_path/www
mkdir -p $cert_path/conf

docker run -d --rm \
    --name nginx \
    -p 80:80 \
    -v $(pwd)/nginx_validate.conf:/etc/nginx/conf.d/nginx_validate.conf:ro \
    -v $cert_path/www:/var/www/certbot/:ro \
    -v $cert_path/conf:/etc/nginx/ssl/:ro \
    nginx:latest
```

> 因為只是用來驗證來取得憑證，所以這個容器很簡單，跑完就刪除了。

3. 檢查 nginx 是否成功啟動：

```bash
docker ps
docker logs -f nginx
```

4. 使用 certbot 來申請憑證：

```bash
docker run --rm \
    --name certbot \
    -v $cert_path/www:/var/www/certbot/:rw \
    -v $cert_path/conf:/etc/letsencrypt/:rw \
    certbot/certbot:latest certonly \
    -v \
    -n \
    --agree-tos \
    --webroot \
    --webroot-path /var/www/certbot/ \
    -m example@mail.com \
    -d $domain_name
```

5. 查看憑證內容：

```bash
openssl x509 -in ./certbot_data/conf/live/${domain_name}/fullchain.pem -text -noout
```



6. 把憑證放到「正式」的 nginx config 上：


```bash
echo "server {
    listen 80 default_server;
    listen [::]:80 default_server;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://\$host\$request_uri;
    }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    ssl_certificate /etc/nginx/ssl/live/${domain_name}/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/live/${domain_name}/privkey.pem;
    ssl_session_timeout 1d;
    ssl_session_cache shared:MozSSL:10m;
    ssl_session_tickets off;

    ssl_dhparam /etc/nginx/ssl/live/${domain_name}/dhparam;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305;
    ssl_prefer_server_ciphers off;

    add_header Strict-Transport-Security "max-age=63072000" always;

    ssl_stapling on;
    ssl_stapling_verify on;

    ssl_trusted_certificate /etc/nginx/ssl/live/${domain_name}/chain.pem;
    resolver 8.8.8.8;
}" >nginx_prod.conf
```

> 比較重要的是「location /.well-known/acme-challenge/」的部分，方便下次 renew 憑證。

7. 測試看看：

```bash
docker run -d --rm \
    --name nginx \
    -p 80:80 \
    -p 443:443 \
    -v $(pwd)/nginx_prod.conf:/etc/nginx/conf.d/nginx_prod.conf:ro \
    -v $cert_path/www:/var/www/certbot/:ro \
    -v $cert_path/conf:/etc/nginx/ssl/:ro \
    nginx:latest
```

## 憑證更新

```
cert_path="./certbot_data"
docker run --rm \
    --name certbot \
    -v $cert_path/www:/var/www/certbot/:rw \
    -v $cert_path/conf:/etc/letsencrypt/:rw \
    certbot/certbot:latest renew
```