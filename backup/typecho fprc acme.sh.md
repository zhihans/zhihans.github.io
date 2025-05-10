修改了一下先前的版本，现在使用两个fprc并且支持https，同时启用了typecho自动安装


![Image](https://github.com/user-attachments/assets/f3a85c5e-aa8c-4551-b7ad-044d5b53a4e5)

compose文件
```yaml
x-frpc-common: &frpc-common
  image: snowdreamtech/frpc:latest
  restart: unless-stopped
  volumes:
    - ./frpc:/etc/frpc
  environment:
    - TZ=Asia/Shanghai

services:
  acme.sh:
    image: neilpang/acme.sh
    container_name: acme.sh
    restart: unless-stopped
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - well_known_tmp:/webroot/.well-known
      - ./acme.sh:/acme.sh
      - ./frpc:/frpc
    command: daemon
  typecho:
    image: joyqi/typecho:nightly-php8.2-alpine
    container_name: typecho
    restart: unless-stopped
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - well_known_tmp:/app/.well-known
      - ./typecho:/app/usr
    environment:
      TYPECHO_INSTALL: 1
      TYPECHO_SITE_URL: https://han.tipai.icu
      TYPECHO_DB_ADAPTER: Pdo_SQLite
      TYPECHO_DB_FILE: data.db
      TYPECHO_DB_NEXT: keep
  frpc:
    <<: *frpc-common
    container_name: typecho_mefrp
    command: ["-c", "/etc/frpc/mefrp.toml"]
  frpc2:
    <<: *frpc-common
    container_name: typecho_openfrp
    command: ["-c", "/etc/frpc/openfrp.toml"]

volumes:
  well_known_tmp:
```


acme.sh 申请证书并安装
```bash
docker exec -it acme.sh ash
acme.sh --register-account -m email@email.com
acme.sh --issue -d han.tipai.icu --webroot /webroot/
acme.sh --install-cert -d han.tipai.icu \
--key-file       /frpc/server.key  \
--fullchain-file /frpc/server.crt
```


fprc配置文件
```toml
serverAddr = "frps.example.com"
serverPort = 2333
user = "user"

[auth]
method = "token"
token = "FrpServerToken"

[[proxies]]
name = "typecho"
type = "http"
localIP = "typecho"
localPort = 80
customDomains = ["han.tipai.icu"]

[[proxies]]
name = "typecho_https"
type = "https"
customDomains = ["han.tipai.icu"]

[proxies.plugin]
type = "https2http"
localAddr = "typecho:80"
crtPath = "/etc/frpc/server.crt"
keyPath = "/etc/frpc/server.key"
```