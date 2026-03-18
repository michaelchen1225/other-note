# SSL 憑證更換 - LetsEncrypt

## 目錄

* [How LetsEncrypt works](#how-letsencrypt-works)

## How LetsEncrypt works

* 目標：確認你是該網域的擁有者 (以下將以 www.example.com 為例)

* 方法：[ACME 協定](https://letsencrypt.org/zh-tw/docs/challenge-types/)。

    1. 由 ACME client (例如在伺服器安裝certbot) 來向 ACME server (例如 LetsEncrypt) 申請憑證。

    2. 根據申請域名的不同，驗證過程可分為兩種：
    
        * 單一域名憑證：ACME server 會要求 ACME client 在 www.example.com 上放置一個特定的檔案 (例如 http://www.example.com/.well-known/acme-challenge/12345)，以確認你是該網域的擁有者。

            > **./well-known**：這是一個公認的網頁路徑(由[RFC 8615](https://www.rfc-editor.org/info/rfc8615)定義)，目的是在不同 http server 之間，提供該 http server 相關的一些資訊。

        * 泛域名憑證：ACME server 會要求 ACME client 在 DNS 上新增一個特定的 TXT 紀錄 (例如 _acme-challenge.www.example.com)，以確認你是該網域的擁有者。

            > **泛域名(Wildcard domain)**：例如 *.example.com，代表 example.com 以及所有以 .example.com 結尾的子域名，例如 www.example.com、api.example.com 等等。



* 實務上可使用 letsencrypt 的 [Certbot](https://certbot.eff.org/) 來自動化這個過程，Certbot 會幫你處理 ACME client 的部分，並且在驗證成功後，幫你把憑證放到指定的位置。




