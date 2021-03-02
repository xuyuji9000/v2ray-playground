- Server configuration 

```
{
  "inbounds": [
    {
      "port": 28602,
      "listen":"127.0.0.1",
      "protocol": "vmess",
      "settings": {
        "clients": [
          {
            "id": "CLIENT_ID_HERE",
            "alterId": 64
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "wsSettings": {
        "path": "/ray"
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "settings": {}
    }
  ]
}
```

- Client configuration

```
{
  "routing" : {
    "name" : "all_to_main",
    "domainStrategy" : "AsIs",
    "rules" : [
      {
        "type" : "field",
        "outboundTag" : "gcp-tw-ws",
        "port" : "0-65535"
      }
    ]
  },
  "inbounds" : [
    {
      "listen" : "127.0.0.1",
      "protocol" : "http",
      "settings" : {
        "timeout" : 0
      },
      "tag" : "httpinbound",
      "port" : 8001
    }
  ],
  "dns" : {
    "servers" : [
      "localhost"
    ]
  },
  "outbounds" : [
    {
      "sendThrough" : "0.0.0.0",
      "mux" : {
        "enabled" : false,
        "concurrency" : 8
      },
      "protocol" : "vmess",
      "settings" : {
        "vnext" : [
          {
            "address" : "WWW.DOMAINNAME.COM",
            "users" : [
              {
                "id" : "CLIENT_ID_HERE",
                "alterId" : 64,
                "security" : "auto",
                "level" : 0
              }
            ],
            "port" : 443
          }
        ]
      },
      "tag" : "gcp-tw-ws",
      "streamSettings" : {
        "network" : "ws",
        "wsSettings" : {
          "path" : "\/ray"
        },
        "security" : "tls"
      }
    }
  ]
}
```
