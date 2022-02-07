# install jenkins-x for gitlab 十步曲

## 准备gitlab仓库

这是企业内部gitlab域名：gitlab.infra.com

## 准备k8s集群，确认work节点数

## 第一步，准备安装仓库
git clone https://github.com/gitcpu-io/install-jx.git

> 把install-jx仓库提交到内部的gitlab，在k8s master节点上git clone install-jx.git仓库

git clone http://gitlab.infra.com/devopsman/install-jx.git

### 设置访问域名
cd install-jx

vi jx-requirements.yml
```yaml
  ingress:
    domain: "devopsman.io"
```
git add .

git commit -am "change domain"

git push

> 为访问不到镜像做准备，可以执行下面的脚本用来替换，如果你的k8s work节点有多个，请确保每个节点都load到了镜像

cd install-jx/install

./load_images.sh

### <a href="#1">设置ShadowSocks客户端代理</a>

## 第二步，准备oauth personal access token
> 先登录你的个人gitlab账号，然后创建personal access token

### gitlab
> 访问个人资料->访问令牌：/profile/personal_access_tokens

Vbm-XynkoCyC-jyyoDhV

## 第三步，通过安装jx-git-operator来安装jenkins-x

> 务必要安装在命名空间jx-git-operator中

kubectl create ns jx-git-operator

> 下载仓库

git clone https://github.com/jenkins-x/jx-git-operator.git

cd jx-git-operator/charts

> vi values.yaml

- url 就是install-jx.git仓库

- username 就是你的github账号

- password 就是你的personal access token

helm -n jx-git-operator install --set url=https://github.com/gitcpu-io/install-jx.git --set username=rubinus --set password=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 jx-git-operator jx-git-operator

> gitlab

helm -n jx-git-operator install --set url=http://gitlab.infra.com/devopsman/install-jx.git --set username=rubinus --set password=Vbm-XynkoCyC-jyyoDhV jx-git-operator jx-git-operator

> 卸载jx-git-operator

helm -n jx-git-operator uninstall jx-git-operator

> 检查是否安装成功

kubectl -n jx-git-operator get po

kubectl -n jx get po

kubectl -n jx-observability get po

> 检查helm安装，如果install-jx有变动，可以升级

helm -n jx-git-operator list

cd jx-git-operator/charts

helm -n jx-git-operator upgrade --set url=https://github.com/gitcpu-io/install-jx.git --set username=rubinus --set password=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  jx-git-operator jx-git-operator


## 第四步，安装tekton-dashboard

kubectl apply -f tekton-ing.yaml

kubectl apply -f tekton-dashboard-release.yaml

> 配置所有域名解析

kubectl get ing -A

### 这一步不同，内网解析域名，需要给nginx-controller配置externalIPs指向mater节点
```yaml
sepc:
  externalIPs:
  - 10.10.11.100
```


## 第五步，如果Oauth App token过期，可以使用Personal access token来创建

> 安装成功，需要删除 jx-git-operator，使用自己的仓库和prow的配置

kubectl delete ns jx-git-operator

> (可选)准备新的personal access token，来生成lighthouse-oauth-token

kubectl -n jx delete secret lighthouse-oauth-token

kubectl -n jx create secret generic lighthouse-oauth-token --from-literal=oauth=Vbm-XynkoCyC-jyyoDhV


### 重启
kubectl -n jx scale deploy lighthouse-keeper --replicas=0

kubectl -n jx scale deploy lighthouse-foghorn --replicas=0

kubectl -n jx scale deploy lighthouse-tekton-controller --replicas=0

kubectl -n jx scale deploy lighthouse-webhooks --replicas=0

### 重启
kubectl -n jx scale deploy lighthouse-keeper --replicas=1

kubectl -n jx scale deploy lighthouse-foghorn --replicas=1

kubectl -n jx scale deploy lighthouse-tekton-controller --replicas=1

kubectl -n jx scale deploy lighthouse-webhooks --replicas=1


## 第六步，配置tekton pipeline的资源，作为presubmits和postsubmits
cd install-jx/install

kubectl -n jx apply -f ./pipeline

> 在这一步，一定要去修改为正确的secret中的账号密码（dockerhub、github）

kubectl -n jx apply -f ./resources

kubectl -n jx apply -f ./task

> 安装nfs client，nfs-client.yaml中的配置要改成nfs server的相应信息

kubectl -n jx apply -f ./storage


# harbor 单机版搭建

```
安装包下载链接
https://github.com/goharbor/harbor/releases/download/v1.10.10/harbor-offline-installer-v1.10.10.tgz

tar zxvf harbor-offline-installer-v1.10.10.tgz

cd harbor
```

## 1.生成自签名证书 (可不操作，不使用证书 !!!)

### 首先创建证书存放目录：

mkdir -p /data/cert && cd /data/cert

### 创建 CA 证书：

```shell
openssl genrsa -out ca.key 4096

openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=harbor.devopsman.io" \
 -key ca.key \
 -out ca.crt

```
### 创建服务端证书
```shell
openssl genrsa -out harbor.devopsman.io.key 4096

openssl req -sha512 -new \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=harbor.devopsman.io" \
    -key harbor.devopsman.io.key \
    -out harbor.devopsman.io.csr

```

### Generate an x509 v3 extension file.

```shell
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=harbor.devopsman.io
EOF

```

### Use the v3.ext file to generate a certificate for your Harbor host
```shell
openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in harbor.devopsman.io.csr \
    -out harbor.devopsman.io.crt

> 如果不需要可以不用v3

openssl x509 -req -sha512 -days 3650 \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in harbor.devopsman.io.csr \
    -out harbor.devopsman.io.crt

```

### 转化crt to cert for docker
openssl x509 -inform PEM -in harbor.devopsman.io.crt -out harbor.devopsman.io.cert

### 转化为pem(如果需要)
openssl x509 -in ca.crt -out ca.pem

openssl x509 -in harbor.devopsman.io.cert -out cert.pem

openssl rsa -in harbor.devopsman.io.key -text > key.pem

### 复制到docker使用的目录下
mkdir -p /etc/docker/certs.d/harbor.devopsman.io

cp /data/cert/ca.crt /etc/docker/certs.d/harbor.devopsman.io/

cp /data/cert/harbor.devopsman.io.cert /etc/docker/certs.d/harbor.devopsman.io/

cp /data/cert/harbor.devopsman.io.key /etc/docker/certs.d/harbor.devopsman.io/

> 通过scp复制到其它节点

cd /etc/docker/certs.d/

scp -r harbor.devopsman.io root@10.10.11.100:/etc/docker/certs.d/


## 3.修改 harbor.yml
```
hostname = harbor.devopsman.io

https:
  port: 443
  certificate: /data/cert/harbor.devopsman.io.crt
  private_key: /data/cert/harbor.devopsman.io.key

```

## 安装docker-compose
curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

chmod +x /usr/local/bin/docker-compose

docker-compose --version

## 安装Harbor
```

./install.sh --with-chartmuseum --with-notary --with-clair

docker-compose ps

docker-compose up -d

docker-compose down

```

## 配置k8s coredns的configmap，添加自定义域名解析
kubectl -n kube-system edit cm coredns

```yaml
        hosts {
            10.10.11.105  harbor.devopsman.io
            fallthrough
        }
```

### 重启docker
systemctl restart docker

docker login harbor.devopsman.io

docker logout harbor.devopsman.io

> 账号密码

admin/Harbor12345

## 为builder的buildpacks重新打镜像
vi Dockerfile
```dockerfile
FROM heroku/buildpacks:20

####这个ca.crt来自harbor仓库,当是https时，需要把ca.crt放到这个基础镜像中
COPY ca.crt /etc/ssl/certs/

RUN ls -la /cnb

```
docker build -t harbor.devopsman.io/devopsman/buildpacks-ca:20 .

docker push harbor.devopsman.io/devopsman/buildpacks-ca:20

## 如果需要导入证书到服务器上（每个节点）

### Centos导入自签证书
cp ca.crt /etc/pki/ca-trust/source/anchors/

cp harbor.devopsman.io.cert /etc/pki/ca-trust/source/anchors/

> 使用scp到其它节点上

scp ca.crt root@10.10.11.104:/etc/pki/ca-trust/source/anchors/

scp harbor.devopsman.io.cert root@10.10.11.104:/etc/pki/ca-trust/source/anchors/

ln -s /etc/pki/ca-trust/source/anchors/ca.crt /etc/ssl/certs/ca.crt

update-ca-trust

### Ubuntu导入自签证书
cp ca.crt /usr/local/share/ca-certificates/

update-ca-certificates


## 第七步，配置Prow

> 以jx-demo为例，作为chatops的代码仓库

https://github.com/gitcpu-io/jx-demo.git

### config的ConfigMap
cd install-jx/install

kubectl -n jx delete cm config

kubectl -n jx create cm config --from-file=config.yaml

### plugins的ConfigMap
cd install-jx/install

kubectl -n jx delete cm plugins

kubectl -n jx create cm plugins --from-file=plugins.yaml

## 第八步 搞定显示color label（如果是对接的github）

cd install-jx/install

> 创建label-config的ConfigMap

kubectl -n jx apply -f labels-cm.yaml

> 运行job

kubectl -n jx delete -f label_sync_job.yaml

kubectl -n jx apply -f label_sync_job.yaml

kubectl -n jx delete -f label_sync_cron_job.yaml

kubectl -n jx apply -f label_sync_cron_job.yaml

## 第九步，安装argocd
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

> 备用

cd install-jx/install

kubectl -n argocd apply -f argocd-install.yaml

> 创建argocd ingress

kubectl -n argocd apply -f argocd-ing.yaml

> 可以通过http访问

kubectl -n argocd edit deploy argocd-server

在 cmd argocd-server下面添加flag：--insecure

> argocd的初始密码

kubectl -n argocd get secret argocd-initial-admin-secret -oyaml

echo xxxxxxx |base64 -d

> 登录后修改密码，另外创建argocd-oauth的secret给pipeline使用(如果需要)

kubectl -n argocd apply -f ./resources/secret.yaml

### 登录argocd后，创建app

> 通过jx-demo-infra这个配置仓库

https://github.com/gitcpu-io/jx-demo-infra.git

> 或是配置ssh的仓库

ssh://git@gitlab.infra.com/devopsman/jx-demo-infra.git

跳过ssl验证，添加任何一个ssh的私匙

### 配置访问jx-demo

cd install-jx/install

kubectl apply -f jx-demo-ing.yaml

kubectl get ing



# 安装 ShadowSocks 客户端，在其中一台内网机器上（10.10.11.103）

## <a name="1">安装 ShadowSocks</a>
yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm

cd /etc/yum.repos.d/

curl -O https://copr.fedorainfracloud.org/coprs/librehat/shadowsocks/repo/epel-7/librehat-shadowsocks-epel-7.repo

yum install -y shadowsocks-libev

## 添加配置文件

vi /etc/shadowsocks-libev/config.json

```json
{
    "server": "你的ShadowSocks server IP",
    "server_port": 8888,
    "local_port": 1080,
    "password": "密码",
    "timeout": 60,
    "method": "加密算法"
}
```

## 启动shadowsocks服务
systemctl enable --now shadowsocks-libev-local

systemctl status shadowsocks-libev-local

curl --socks5 127.0.0.1:1080 http://httpbin.org/ip

## 配置代理
yum install -y privoxy

vi /etc/privoxy/config

```shell
#forward-socks5t / 127.0.0.1:1080 . #全局
forward-socks5t google.com 0.0.0.0:1080 .  #google.com转发
forward-socks5t gcr.io 0.0.0.0:1080 . #gcr.io转发
```

systemctl enable privoxy  && systemctl start privoxy  && systemctl status privoxy

## 以下开始在其它内网节点配置代理

echo -e "export http_proxy=http://10.10.11.103:8118" >> /etc/profile

echo -e "export https_proxy=http://10.10.11.103:8118" >> /etc/profile

echo -e "export no_proxy=127.0.0.1,localhost,10.0.0.0/24" >> /etc/profile

source /etc/profile

curl www.google.com

备注：如果不需要用代理了，把 /etc/profile 里的配置注释即可。

## Docker代理

mkdir -p /etc/systemd/system/docker.service.d

vi /etc/systemd/system/docker.service.d/http-proxy.conf

[Service]
Environment="HTTP_PROXY=http://10.10.11.103:8118"

Environment="HTTPS_PROXY=http://10.10.11.103:8118"

Environment="NO_PROXY=localhost,127.0.0.0/8,10.0.0.0/24,harbor.cloud2go.cn,harbor.devopsman.io,docker.io,docker.com"


systemctl daemon-reload && systemctl restart docker && systemctl status docker

systemctl show --property=Environment docker

### 测试拉取gcr.io
docker pull gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/entrypoint:v0.27.0@sha256:b8a0bed8e402138f7b14b44115719f44460255497132b8a8233e710692ef6930

### 确认镜像的sha256
docker images --digests

## 配置 yum 代理（如果需要）
vim /etc/yum.conf

proxy=http://10.10.11.103:8118

## 配置 wget 代理（如果需要）
vi /etc/wgetrc

use_proxy = on

https_proxy=http://10.10.11.103:8118

http_proxy=http://10.10.11.103:8118

ftp_proxy=http://10.10.11.103:8118

## 配置git代理（如果需要）
git config --global http.proxy http://10.10.11.103:8118

git config --global https.proxy http://10.10.11.103:8118

### 取消git代理
git config --global --unset http.proxy

git config --global --unset https.proxy
