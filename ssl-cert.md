# SSL 憑證更換

* Solution：LetsEncrypt (its free!)

## How LetsEncrypt works

* 目標：確認你是該網域的擁有者 (以下將以 www.example.com 為例)

* 方法：透過[ACME 協定](https://zh.wikipedia.org/zh-tw/%E8%87%AA%E5%8B%95%E6%86%91%E8%AD%89%E6%9B%B4%E6%96%B0%E7%92%B0%E5%A2%83/)來驗證。

    1. 由 ACME client (例如在伺服器安裝certbot) 來向 ACME server (例如 LetsEncrypt) 申請憑證。

    2. 根據申請域名的不同，驗證過程可分為兩種：
    
        * 單一域名憑證：ACME server 會要求 ACME client 在 www.example.com 上放置一個特定的檔案 (例如 http://www.example.com/.well-known/acme-challenge/12345)，以確認你是該網域的擁有者。

            > **./well-known**：這是一個公認的網頁路徑(由[RFC 8615](https://www.rfc-editor.org/info/rfc8615)定義)，目的是在不同 http server 之間，提供該 http server 相關的一些資訊。

        * 泛域名憑證：ACME server 會要求 ACME client 在 DNS 上新增一個特定的 TXT 紀錄 (例如 _acme-challenge.www.example.com)，以確認你是該網域的擁有者。

            > **泛域名(Wildcard domain)**：例如 *.example.com，代表 example.com 以及所有以 .example.com 結尾的子域名，例如 www.example.com、api.example.com 等等。




> 實務上可使用 letsencrypt 的 [Certbot](https://certbot.eff.org/) 來自動化這個過程，Certbot 會幫你處理 ACME client 的部分，並且在驗證成功後，幫你把憑證放到指定的位置。


## 實作

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

