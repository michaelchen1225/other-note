# SSL 憑證更換 - LetsEncrypt

## 目錄

* [How LetsEncrypt works](#how-letsencrypt-works)
* [Lab](02-lab.md)
* [Prod](03-prod.md)

## How LetsEncrypt works

* 目標：確認你是該網域的擁有者 (以下將以 www.example.com 為例)

* 方法：[ACME 協定](https://letsencrypt.org/zh-tw/docs/challenge-types/)。

    1. 由 ACME client (例如在伺服器安裝certbot) 來向 ACME server (例如 LetsEncrypt) 申請憑證。

    2. 根據申請域名的不同，驗證過程可分為兩種：
    
        * **HTTP-01 challenge**：用來申請單一域名憑證，ACME server 會要求 ACME client 在 www.example.com 上放置一個特別的檔案，以確認你是該網域的擁有者。這個特別的檔案會放在 webroot 目錄下的 **.well-known/acme-challenge/** 路徑中，ACME server 會透過 HTTP 請求來驗證這個檔案是否存在 & 內容是否正確。

            > **./well-known**：這是一個公認的網頁路徑(由[RFC 8615](https://www.rfc-editor.org/info/rfc8615)定義)，目的是在不同 http server 之間，提供該 http server 相關的一些資訊。

        * **DNS-01 challenge**：用來申請泛域名憑證(萬用憑證)，ACME server 會要求 ACME client 在 DNS 上新增一個特定的 TXT 紀錄，以確認你是該網域的擁有者。

            > **泛域名(Wildcard domain)**：星號(*)開頭的 domain，例如 *.example.com，可代表 example.com 以及所有以 .example.com 結尾的子域名，例如 www.example.com、api.example.com 等等。



* 實務上可使用 letsencrypt 的 [Certbot](https://certbot.eff.org/) 來自動化這個過程，Certbot 會幫你處理 ACME client 的部分，並且在驗證成功後，幫你把憑證放到指定的位置。

* 以上是關於 LetsEncrypt 的基本運作原理，實際操作的細節可參考：
  * [Lab](02-lab.md)：在本地環境模擬整個申請流程，適合沒有正式 domain 的環境、或者想要先熟悉流程的人。
  * [實務操作](03-prod.md)：筆者個人在正式環境中更換、renew 憑證的筆記。



