## Production - config & renewal

### nginx config

* **try_files**：明確的定義如果找不到 acme 驗證檔就回傳 404，以免被其他 location block (例如 proxy_pass) 給攔截了，丟出 502、403 這種其他的 error code。

* **allow all**：確保 ACME server 的驗證請求能夠順利到達 nginx，並且不會被其他的存取控制規則給阻擋。

```text
server {
    listen 80;
    server_name prod.example.com;

    location /.well-known/acme-challenge/{
            root /var/www/certbot;
            try_files $uri $uri/ =404;
            allow all;
    }

    location / {
        proxy_pass http://backend:8080;
    }
}

server {
    listen        443 ssl;

    server_name prod.example.com;
    add_header Cache-Control no-cache;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    root /data/dmp/dmp-web;
    client_max_body_size 110M;
    ssl_certificate      /data/certbot_data/conf/live/prod.example.com/fullchain.pem;
    ssl_certificate_key  /data/certbot_data/conf/live/prod.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;

    if ($request_uri ~* .*[\x00-\x1F]+.*) {return 435;}

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### Cerbot 參數

```bash
docker run -it --rm \
        --name certbot \
        -v /var/www/certbot:/var/www/certbot/:rw,Z \
        -v /data/certbot_data/conf:/etc/letsencrypt/:rw,Z \
        certbot:latest \
        certbot certonly \
        -v \
        --non-interactive \
        --agree-tos \
        --webroot -w /var/www/certbot/ \
        --register-unsafely-without-email \
        -d prod.example.com
```

### 一些踩過的坑

由於正式環境可能會有比較多權限方面的限制，可能導致 certbot 再驗證時出問題，可以參考以下方式提前預防：

#### 提前建立 `/var/www/certbot/.well-known/acme-challenge/`，並且把權限調整好：

```bash
sudo mkdir -p /var/www/certbot/.well-known/acme-challenge/
sudo chmod -R 755 /var/www/certbot
```

#### 啟動 nginx 後，隨便在 `/var/www/certbot/.well-known/acme-challenge/` 放一個測試檔案，確保可以透過 HTTP 連到：

```bash
echo "test" | sudo tee /var/www/certbot/.well-known/acme-challenge/test
```
> 然後用瀏覽器或 curl 來測試


#### SELinux 問題 (CentOS 環境)

* 先確認 SELinux 有沒有開啟：

```bash
getenforce
```

* 如果資安政策允許的話，就直接關閉：

```bash
setenforce 0
```

* 如果不允許整個關閉 SELinux，就把 certbot 會存取的路徑開放給 httpd 存取：

```bash
chcon -Rt container_file_t /var/www/certbot # webroot 
chcon -Rt container_file_t /data/certbot_data/conf # 本機放置憑證的地方，會掛載到 certbot
```

* 也可以在掛載時加入 `:Z` 參數，讓 docker 自動幫你處理 SELinux 的權限問題：

```bash
docker run -it --rm \
        --name certbot \
        -v /var/www/certbot:/var/www/certbot/:rw,Z \
        -v /data/certbot_data/conf:/etc/letsencrypt/:rw,Z \
(底下省略)
```


* renew script

```bash
#!/bin/bash

# Configuration
CERT_FILE="/data/certbot_data/conf/live/dhub.hinet.net/fullchain.pem"
# Corrected image tag as per your update
CERTBOT_IMAGE="harbor.cht.com.tw:30725/p4u-project/shared/aud/docker-certbot:28.3.3-dind"
VOLUMES="-v /data/certbot_data/conf:/etc/letsencrypt -v /data/certbot_data/www:/var/www/certbot"

# 1. Check expiration using openssl
EXP_DATE=$(openssl x509 -enddate -noout -in "$CERT_FILE" | cut -d= -f2)
EXP_SECONDS=$(date -d "$EXP_DATE" +%s)
NOW_SECONDS=$(date +%s)
THRESHOLD_SECONDS=$(( 30 * 24 * 3600 )) # 30 days in seconds

# 2. Logic check
if [ $(( EXP_SECONDS - NOW_SECONDS )) -le $THRESHOLD_SECONDS ]; then
    echo "$(date): Certificate expires soon. Starting Docker renewal..."
    
    # Run Certbot and capture all output (stdout and stderr) into a variable
    # Removed --quiet so we can actually see what happened in the log if it fails
    RENEW_OUTPUT=$(docker run --rm $VOLUMES $CERTBOT_IMAGE certbot renew 2>&1)
    EXIT_CODE=$?
    
    # 3. Handle results
    if [ $EXIT_CODE -eq 0 ]; then
        echo "$(date): Renewal successful."
        echo "Certbot Output: $RENEW_OUTPUT"
        echo "Reloading Nginx daemon..."
        systemctl reload nginx
    else
        echo "$(date): ERROR - Certbot renewal failed with exit code $EXIT_CODE."
        echo "-------------------------------------------"
        echo "FULL CERTBOT ERROR OUTPUT:"
        echo "$RENEW_OUTPUT"
        echo "-------------------------------------------"
        exit 1
    fi
else
    echo "$(date): Certificate is still valid ($(( (EXP_SECONDS - NOW_SECONDS) / 86400 )) days left). No action taken."
fi
```


```
# 1. 啟用 TLS 1.2 與 1.3 (修復 PQC 無法運作的問題)
ssl_protocols TLSv1.2 TLSv1.3;

# 2. 指定密鑰交換曲線 (設定 PQC 混合模式為第一優先) ssl_ecdh_curve X25519MLKEM768:X25519:secp384r1:prime256v1;

# 3. 修復 WebInspect 弱點：套用報告官方建議的強加密套件 (僅對 TLS 1.2 有效)
# 這行設定徹底排除了 CBC、SHA1 以及金鑰長度過短的弱點演算法
ssl_ciphers
ECDHE-ECDSA-AES128-GCM-SHA256
ECDHE-RSA-AES128-GCM-SHA256
ECDHE-ECDSA-AES256-GCM-SHA384
ECDHE-RSA-AES256-GCM-SHA384
ECDHE-ECDSA-CHACHA20-POLY1305
ECDHE-RSA-CHACHA20-POLY1305
DHE-RSA-AES128-GCM-SHA256
DHE-RSA-AES256-GCM-SHA384
DHE-RSA-CHACHA20-POLY1305;

# 強制優先使用伺服器端決定的加密套件
ssl_prefer_server_ciphers on;
```