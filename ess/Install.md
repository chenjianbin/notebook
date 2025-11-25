# 设置dns (替换your.tld为你的域名)
```
# 如果使用cloudflare, 除了mrtc外均可开启proxy进行cdn加速
your.tld               A       your server ip    
matrix.your.tld        A       your server ip
account.your.tld       A       your server ip
admin.your.tld         A       your server ip
chat.your.tld          A       your server ip
mrtc.your.tld          A       your server ip
```

---
# 开放ess所需访问端口
### 根据使用的云厂商使用security group开放以下端口
```
TCP 80
TCP 443
TCP 30881
UDP 30882
```

### ubuntu 服务器开放端口命令
```
ufw allow http
ufw allow https
ufw allow 30881/tcp
ufw allow 30882/udp
```
---
# 初始化容器环境
### 安装k3s (使用其他容器环境，可忽略该步骤)
``` 
curl -sfL https://get.k3s.io | sh -
```

### 设置kubeconfig (使用其他容器环境，可忽略该步骤)
``` 
mkdir ~/.kube
export KUBECONFIG=~/.kube/config
sudo k3s kubectl config view --raw > "$KUBECONFIG"
chmod 600 "$KUBECONFIG"
chown "$USER:$USER" "$KUBECONFIG"
echo "export KUBECONFIG=~/.kube/config" >> ~/.bashrc
``` 

### 安装helm
``` 
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
``` 

### 设置ess相关配置
``` 
kubectl create namespace ess
mkdir ~/ess-config-values
```
---
# 设置证书自动管理
```
helm repo add jetstack https://charts.jetstack.io --force-update

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.17.0 \
  --set crds.enabled=true

kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-private-key
    solvers:
      - http01:
          ingress:
            class: traefik
EOF

curl -L https://raw.githubusercontent.com/element-hq/ess-helm/main/charts/matrix-stack/ci/fragments/quick-setup-letsencrypt.yaml -o ~/ess-config-values/tls.yaml
```

# 安装ess
### 下载hostnames.yaml配置
```
curl -L https://raw.githubusercontent.com/element-hq/ess-helm/refs/heads/main/charts/matrix-stack/ci/fragments/quick-setup-hostnames.yaml -o ~/ess-config-values/hostnames.yaml
```
### 替换hostnames.yaml中域名，将下面命令中<your domain>换成你的域名
```
sed -i 's/your.tld/<your domain>/g' ~/ess-config-values/hostnames.yaml
```
### (optional)如需支持自动token注册，编辑~/ess-config-values/hostnames.yaml，在matrixAuthenticationService下添加配置
```
  additional:
    auth.yaml:
      config: |
        account:
          password_registration_enabled: true  # 启用密码注册
          registration_token_required: true    # 需要注册 token
          password_registration_email_required: false  # 不需要邮箱验证
          password_change_allowed: true        # 允许用户修改密码

```
### 安装启动ess
```
helm upgrade --install --namespace "ess" ess oci://ghcr.io/element-hq/ess-helm/matrix-stack -f ~/ess-config-values/hostnames.yaml -f ~/ess-config-values/tls.yaml --wait
```
### 注册管理用户
``` 
kubectl exec -n ess -it deploy/ess-matrix-authentication-service -- mas-cli manage register-user admin
kubectl exec -n ess -it deployment/ess-matrix-authentication-service -- mas-cli manage promote-admin admin
```
