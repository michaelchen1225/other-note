## Labs

* **情境**：在「沒有正式 domain name」的情況，在本地體驗一下 letsencrypt 的流程。

### Overview

Lab 會使用 **Docker** 跑以下 3 個容器：

| 容器名稱 | 角色 | 核心任務 |
|--------|------|--------|
| pebble | CA 認證單位 | 扮演 Let's Encrypt 伺服器，負責發放 Token 與簽發證書。 |
| certbot | ACME client | 負責向 Pebble 申請證書，並處理挑戰驗證（Challenge）。 |
| nginx | Web server | 對外提供 80 port（ACME驗證用）與 443 port（加密連線）。 |

流程概述：

> 目的：幫 `my-test-domain.com` 申請一張憑證。

1. 準備「驗證用」的 nginx config
2. 把三個容器跑起來，分別扮演 CA 認證單位、ACME client、Web server 的角色。
3. 使用 certbot 來申請憑證。成功申請到的憑證會放在 certbot container 的 `/etc/letsencrypt/live/${domain_name}/` 目錄下。
4. 把憑證放到「正式」的 nginx config 上，並且測試看看。
 

### Step 1：環境準備

1. [安裝 Docker](https://docs.docker.com/engine/install/)

2. 建立 Lab 目錄 & 必要的子目錄：

* **pebble-lab/webroot**：作為 nginx 的 webroot 目錄，會掛載到 nginx container 中的 /var/www/html，讓 nginx 可以從這裡回應 ACME server 的驗證請求。
* **pebble-lab/letsencrypt**：用來放置 certbot 生成的憑證，同時被 certbot 和 nginx 兩個容器掛載。certbot 會先把憑證放在這裡，這樣 nginx 可以從這裡讀取憑證來提供 HTTPS 連線。

```bash
mkdir -p ~/pebble-lab/webroot ~/pebble-lab/letsencrypt
cd pebble-lab

# (Optional) 調整權限，確保 docker container 可以讀寫這些目錄。
sudo chmod -R 777 webroot letsencrypt
```

3. 建立本次 Lab 會用到的 Docker network：

```bash
docker network create acme-net
```

### Step 2：準備設定檔

下載 pebble 設定檔：

```bash
curl -o pebble-config.json https://raw.githubusercontent.com/letsencrypt/pebble/main/test/config/pebble-config.json
```

建立 nginx 設定檔：

```bash
echo 'server {
    listen 80;
    server_name my-test-domain.com;
    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }
    location / {
        return 200 "HTTP OK - Waiting for ACME...\n";
    }
}' > nginx.conf
```

* **listen 80**：nginx 會監聽 80 port，之後會負責回應 ACME server(Pebble) 的驗證請求。
* **server_name my-test-domain.com**：設定 server name 為 `my-test-domain.com`，也就是要申請憑證的 domain name。
* ""location /.well-known/acme-challenge/**：這個 section 的效果是「請求 `http://my-test-domain.com/.well-known/acme-challenge/xxx` 時，nginx 會從 **/var/www/html/**.well-known/acme-challenge/ 這個路徑下尋找對應的檔案來回應」，這是 ACME server 驗證用的路徑。
* **location /**：這個 section 的效果是「請求 `http://my-test-domain.com/` 時，nginx 會回應「HTTP OK - Waiting for ACME...」(預設的回應內容)。

建立 `docker-compose.yml`，定義 `pebble`、`certbot`：

```yaml
services:
  pebble:
    image: ghcr.io/letsencrypt/pebble:latest
    command: -config /test/config/pebble-config.json -strict
    volumes:
      - ./pebble-config.json:/test/config/pebble-config.json
    environment:
      - PEBBLE_VA_ALWAYS_VALID=1
    networks:
      - acme-net

  certbot:
    image: certbot/certbot
    volumes:
      - ./letsencrypt:/etc/letsencrypt
      - ./webroot:/var/www/html
    depends_on:
      - pebble
    entrypoint: "/bin/sh -c 'sleep infinity'"
    networks:
      - acme-net

networks:
  acme-net:
    external: true
```

### Step 3： 啟動容器

```bash
docker run -d --name my-nginx \
  -p 80:80 -p 443:443 \
  --network acme-net \
  --network-alias my-test-domain.com \
  -v $(pwd)/nginx.conf:/etc/nginx/conf.d/default.conf \
  -v $(pwd)/webroot:/var/www/html \
  -v $(pwd)/letsencrypt:/etc/letsencrypt \
  nginx:alpine
```

> **--network-alias my-test-domain.com**：讓 nginx container 在 acme-net 這個 network 中有一個別名叫 `my-test-domain.com`，這樣 pebble container 就可以透過這個別名來訪問 nginx。(用來模擬真實情況的 DNS 解析)

```bash
docker compose up -d
```

### Step 4：申請憑證

```bash
# 執行 Certbot 申請 (Webroot 模式)
docker compose exec certbot certbot certonly \
  --webroot -w /var/www/html \
  --server https://pebble:14000/dir \
  -d my-test-domain.com \
  --email admin@test.com --agree-tos --no-eff-email --no-verify-ssl
```

* **--webroot -w /var/www/html**：讓 certbot 放置驗證用的檔案在 `/var/www/html`。
* **--server https://pebble:14000/dir**：設定 ACME server 的位置，這裡指向 pebble container。
* **-d my-test-domain.com**：指定要申請憑證的 domain name。
* **--agree-tos**：同意 Let's Encrypt 的服務條款。
* **--no-eff-email**：不提供電子郵件給 EFF。
* **--no-verify-ssl**：不檢查 ACME server 的安全性。(畢竟 pebble 是個測試用的 ACME server，沒有正式的憑證)

> Production 環境使用的參數會稍微不同，後面會再補充。

上面的指令如果成功的話，會看到：

```text
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/my-test-domain.com/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/my-test-domain.com/privkey.pem
This certificate expires on 2026-03-26.
These files will be updated when the certificate renews.
```

可以在 ./letsencrypt/live/my-test-domain.com/ 看到剛剛申請到的憑證：

```bash
openssl x509 -in ./letsencrypt/live/my-test-domain.com/fullchain.pem -text -noout
```

### 修改 nginx config 來使用憑證

添加 HTTPS 的相關設定：

```bash
echo 'server {
    listen 80;
    server_name my-test-domain.com;
    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }
    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name my-test-domain.com;
    ssl_certificate /etc/letsencrypt/live/my-test-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/my-test-domain.com/privkey.pem;
    location / {
        return 200 "HTTPS Success! Certificate is active.\n";
    }
}' > nginx.conf

sudo chmod -R 755 ./letsencrypt

docker exec my-nginx nginx -t && docker exec my-nginx nginx -s reload
```


### Step 5：測試

修改 `/etc/hosts`，讓 `my-test-domain.com` 指向 localhost：

```bash
echo "127.0.0.1 my-test-domain.com" | sudo tee -a /etc/hosts
```

現在可以透過瀏覽器或 curl 來測試 HTTPS 是否成功：

```bash
curl -k https://my-test-domain.com
```
```text
HTTPS Success! Certificate is active.
```
> 用 `-k` 參數是因為 pebble 的憑證是自簽的，沒有被正式的 CA 簽署，所以會被瀏覽器或 curl 視為不安全連線。


### Final Step：清理環境

```bash
docker compose down
docker rm -f my-nginx
docker network rm acme-net
```

