- Machine: MacOS

- Installation[1]

    ``` bash
    brew tap v2ray/v2ray

    brew install v2ray-core

    # update config.json

    brew services start v2ray-core
    ```


- Configuration example


```
{
    "log": {
        "loglevel": "warning"
    },
    "inbounds": [
        {
            "listen": "127.0.0.1",
            "port": "1081",
            "protocol": "http"
        }
    ],
    "outbounds": [
        {
            "protocol": "vless",
            "settings": {
                "vnext": [
                    {
                        "address": "DOMAINNAME", // 换成你的域名或服务器 IP（发起请求时无需解析域名了）
                        "port": 443,
                        "users": [
                            {
                                "id": "UUID", // 填写你的 UUID
                                "encryption": "none",
                                "level": 0
                            }
                        ]
                    }
                ]
            },
            "streamSettings": {
                "network": "ws",
                "security": "tls",
                "tlsSettings": {
                    "serverName": "DOMAINNAME" // 换成你的域名
                },
                "wsSettings": {
                    "path": "/websocket" // 必须换成自定义的 PATH，需要和服务端的一致
                }
            }
        }
    ]
}
```

# Reference

1. [v2ray/homebrew-v2ray](https://github.com/v2ray/homebrew-v2ray)
