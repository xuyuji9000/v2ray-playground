- Prepare a machine at Google Cloud Taiwan Region

    Machine Size: f1-micro

    OS: Ubuntu 20.04 LTS

- Download V2ray from here[2]

- Installation commands[3]

  ``` bash
  wget https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh
  sudo bash ./install-release.sh

  # Related files
  #installed: /usr/local/bin/v2ray
  #installed: /usr/local/bin/v2ctl
  #installed: /usr/local/share/v2ray/geoip.dat
  #installed: /usr/local/share/v2ray/geosite.dat
  #installed: /usr/local/etc/v2ray/config.json
  #installed: /var/log/v2ray/
  #installed: /var/log/v2ray/access.log
  #installed: /var/log/v2ray/error.log
  #installed: /etc/systemd/system/v2ray.service
  #installed: /etc/systemd/system/v2ray@.service
  ```

- Prepare TLS certificate[5]

  ``` bash
  # Issue certificate
  # - Need configure DNS to current server public IP
  # - Use `sudo su` to get root permission
  acme.sh --issue \
          --domain DOMAINNAME \
          --standalone


  # Install certificate
  acme.sh --install-cert \
          --domain DOMAINNAME \ 
          --cert-file /usr/local/etc/v2ray/DOMAINNAME/cert.pem \
          --key-file /usr/local/etc/v2ray/DOMAINNAME/key.pem \
          --fullchain-file /usr/local/etc/v2ray/DOMAINNAME/fullchain.pem \
  ```

  > Certificate original location: `/root/.acme.sh/DOMAIN_NAME`

- renew certificate: 

  ``` bash
  acme.sh --renew \
  -d DOMAINNAME \
  --ecc
  ```


- Prepare configuration [4]

  Get configuration file scaffold and configure parameters.
  
  > Generate uuid: `v2ctl uuid`


  ```
  {
    "log": {
        "loglevel": "warning"
    },
    "inbounds": [
        {
            "port": 443,
            "protocol": "vless",
            "settings": {
                "clients": [
                    {
                        "id": "UUID", // 填写你的 UUID
                        "level": 0,
                        "email": "Your Email"
                    }
                ],
                "decryption": "none",
                "fallbacks": [
                    {
                        "dest": 80
                    },
                    {
                        "path": "/websocket", // 必须换成自定义的 PATH
                        "dest": 1234,
                        "xver": 1
                    }
                ]
            },
            "streamSettings": {
                "network": "tcp",
                "security": "tls",
                "tlsSettings": {
                    "alpn": [
                        "http/1.1"
                    ],
                    "certificates": [
                        {
                            "certificateFile": "/pth/to/fullchain.pem", // 换成你的证书，绝对路径
                            "keyFile": "/path/to/key.pem" // 换成你的私钥，绝对路径
                        }
                    ]
                }
            }
        },
        {
            "port": 1234,
            "listen": "127.0.0.1",
            "protocol": "vless",
            "settings": {
                "clients": [
                    {
                        "id": "UUID", // 填写你的 UUID
                        "level": 0,
                        "email": "Your Email"
                    }
                ],
                "decryption": "none"
            },
            "streamSettings": {
                "network": "ws",
                "security": "none",
                "wsSettings": {
                    "acceptProxyProtocol": true, // 提醒：若你用 Nginx/Caddy 等反代 WS，需要删掉这行
                    "path": "/websocket" // 必须换成自定义的 PATH，需要和上面的一致
                }
            }
        }
    ],
    "outbounds": [
        {
            "protocol": "freedom"
        }
    ]
}
  ```


# Reference 

1. [ WebSocket + TLS + Web](https://guide.v2fly.org/advanced/wss_and_web.html)

2. [v2fly/v2ray-core releases](https://github.com/v2fly/v2ray-core/releases)


3. [v2fly/fhs-install-v2ray](https://github.com/v2fly/fhs-install-v2ray)


4. [v2fly/v2ray-examples](https://github.com/v2fly/v2ray-examples)

5. [acmesh-official/acme.sh](https://github.com/acmesh-official/acme.sh)
