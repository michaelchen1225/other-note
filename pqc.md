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