### Install marzban master
> 安装marzban主服务器
```
sudo bash -c "$(curl -sL https://github.com/Gozargah/Marzban-scripts/raw/master/marzban.sh)" @ install
marzban cli admin create --sudo
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
> 升级xray-core，默认xray-core和太新的的xray-core，marzban兼容有问题
```
# 使用v25.8.3版本
marzban core-update 
```
### Install marzban node
> 安装marzban node
```
sudo bash -c "$(curl -sL https://github.com/Gozargah/Marzban-scripts/raw/master/marzban-node.sh)" @ install
ufw allow 62050/tcp
ufw allow 62051/tcp
ufw allow 8082/tcp
```
> 升级xray-core，默认xray-core和太新的的xray-core，marzban兼容有问题
```
# 使用v25.8.3版本
marzban-node core-update 
```
