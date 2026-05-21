### Install marzban master
> 安装marzban主服务器
```
sudo bash -c "$(curl -sL https://github.com/Gozargah/Marzban-scripts/raw/master/marzban.sh)" @ install
marzban cli admin create --sudo
#升级最新xray-core，默认xray-core 兼容有问题
marzban-node core-update 
```

> 登录marzban,设置配置
```
{
  "log": {
    "loglevel": "warning"
  },
  "routing": {
    "rules": [
      {
        "ip": [
          "geoip:private"
        ],
        "outboundTag": "BLOCK",
        "type": "field"
      }
    ]
  },
  "inbounds": [
    {
      "tag": "VLESS+TCP+REALITY+8082",
      "listen": "0.0.0.0",
      "port": 8082,
      "protocol": "vless",
      "settings": {
        "clients": [],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "tcpSettings": {},
        "realitySettings": {
          "show": false,
          "dest": "www.cloudflare.com:443",
          "xver": 0,
          "serverNames": [
            "www.cloudflare.com",
            "www.paypal.com",
            "www.aliexpress.com",
            "www.digitalocean.com",
            "www.vultr.com"
          ],
          "privateKey": "RQzEpI4qID2ns_u4bysQUZiBJuiqnH6Txytd6HGXlJs",
          "shortIds": [
            "3e8a11e952589ee0"
          ],
          "fingerprint": "chrome"
        }
      }
    },
    {
      "tag": "Shadowsocks TCP",
      "listen": "0.0.0.0",
      "port": 1080,
      "protocol": "shadowsocks",
      "settings": {
        "clients": [],
        "network": "tcp,udp"
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "tag": "DIRECT"
    },
    {
      "protocol": "blackhole",
      "tag": "BLOCK"
    }
  ]
}

```

### Install marzban node
> 安装marzban node
```
sudo bash -c "$(curl -sL https://github.com/Gozargah/Marzban-scripts/raw/master/marzban-node.sh)" @ install
# CN2 节点
ufw allow allow proto tcp <marzban master ip> to any port 62050
ufw allow allow proto tcp <marzban master ip> to any port 62051
ufw allow 56080/tcp

# 非CN2 节点
ufw allow allow proto tcp <marzban master ip> to any port 62050
ufw allow allow proto tcp <marzban master ip> to any port 62051
ufw allow 43080/tcp

#升级最新xray-core，默认xray-core 兼容有问题
marzban-node core-update 
```

