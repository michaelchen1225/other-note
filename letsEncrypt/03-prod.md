## 憑證更新

```
cert_path="./certbot_data"
docker run --rm \
    --name certbot \
    -v $cert_path/www:/var/www/certbot/:rw \
    -v $cert_path/conf:/etc/letsencrypt/:rw \
    certbot/certbot:latest renew
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