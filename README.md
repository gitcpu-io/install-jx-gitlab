# install jenkins-x for gitlab 十步曲

## 准备工作...

- 准备gitlab仓库 <a href="#4">安装gitlab</a>

> 这是企业内部gitlab域名：gitlab.infra.com

- 准备k8s集群，确认work节点数 <a href="#2">安装docker</a> <a href="#3">安装k8s</a>

> kubectl get node

- 准备镜像仓库harbor

> <a href="#1">单机安装harbor</a>

- 如需翻墙，请越狱：

> <a href="#1">设置ShadowSocks客户端及代理</a>


## 第一步，准备安装仓库

git clone http://gitlab.infra.com/devopsman/install-jx.git

> 需要fork这个仓库，fork后如果是 yourname/install-jx，注意下面目录也是yourname

### 设置访问域名
cd install-jx

vi jx-requirements.yml
```yaml
  ingress:
    domain: "devopsman.io"
```

> 为访问不到镜像做准备，可以执行下面的脚本用来替换，如果你的k8s work节点有多个，请确保每个节点都load到了镜像

cd install-jx/install

./load_images.sh


## 第二步，准备机器账号，并设置头像与昵称
> 使用gitlab的root账号创建一个新用户账号，并设置access token

- 账号实例：

devopsman-jx-bot

- 账号access token

_5z5BDqPTeam-mPELbMs

## 第三步，准备替换

搜索：config-root目录中

- 第一处搜索下面的，替换成自己的安装仓库
  http://gitlab.infra.com/devopsman/install-jx.git ==> 自己的仓库地址

- 第二处
  http://gitlab.infra.com  ==> 自己的仓库主域名

- 第三处
  devopsman-jx-bot ==> 自己仓库的账号

> 修改完成后，提交变更到代码仓库

git add .

git commit -am "change domain and bot account"

git push


## 第四步，通过安装jx-git-operator来安装jenkins-x

> 务必要安装在命名空间jx-git-operator中

kubectl create ns jx-git-operator

> 下载仓库

git clone https://github.com/jenkins-x/jx-git-operator.git

> 安装jx-git-operator

cd jx-git-operator/charts

helm -n jx-git-operator install --set url=http://gitlab.infra.com/devopsman/install-jx.git --set username=devopsman-jx-bot --set password=_5z5BDqPTeam-mPELbMs jx-git-operator jx-git-operator

> 卸载jx-git-operator

helm -n jx-git-operator uninstall jx-git-operator

> 检查是否安装成功

kubectl -n jx-git-operator get po

kubectl -n jx get po

kubectl -n jx-observability get po

> 检查helm安装，如果install-jx有变动，可以升级

helm -n jx-git-operator list

cd jx-git-operator/charts

helm -n jx-git-operator upgrade --set url=http://gitlab.infra.com/devopsman/install-jx.git --set username=devopsman-jx-bot --set password=_5z5BDqPTeam-mPELbMs jx-git-operator jx-git-operator

> 安装成功，需要删除 jx-git-operator

kubectl delete ns jx-git-operator

## 第五步，安装tekton-dashboard

kubectl apply -f tekton-ing.yaml

kubectl apply -f tekton-dashboard-release.yaml

> 配置所有域名解析

kubectl get ing -A

### 这一步不同，内网解析域名，需要给nginx-controller配置externalIPs指向mater节点

kubectl -n nginx edit deploy ingress-nginx-controller

```yaml
sepc:
  externalIPs:
  - 10.10.11.100
```

### http://dashboard-jx.devopsman.io/账号密码
kubectl -n jx get secret jx-basic-auth-user-password -oyaml

对结果进行base64 decode，比如：

echo YWRtaW4=|base64 -d

## 第六步，如果token过期，可以使用Personal access token来创建

> (可选)准备新的personal access token，来生成lighthouse-oauth-token

kubectl -n jx delete secret lighthouse-oauth-token

kubectl -n jx create secret generic lighthouse-oauth-token --from-literal=oauth=_5z5BDqPTeam-mPELbMs


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


## 第七步，配置tekton pipeline的资源，作为presubmits和postsubmits
cd install-jx/install

kubectl -n jx apply -f ./pipeline

> 在这一步，一定要去修改为正确的secret.yaml中的地址、账号、密码（dockerhub、github）

kubectl -n jx apply -f ./resources

kubectl -n jx apply -f ./task

> 安装nfs client，nfs-client.yaml中的配置要改成nfs server的相应信息

kubectl -n jx apply -f ./storage


## 第八步，配置Prow

> 以jx-demo为例，作为gitops的代码仓库

http://gitlab.infra.com/devopsman/jx-demo.git

### config的ConfigMap
cd install-jx/install

kubectl -n jx delete cm config

kubectl -n jx create cm config --from-file=config.yaml

### plugins的ConfigMap
cd install-jx/install

kubectl -n jx delete cm plugins

kubectl -n jx create cm plugins --from-file=plugins.yaml

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

http://gitlab.infra.com/devopsman/jx-demo-infra.git

> 或是配置ssh的仓库

ssh://git@gitlab.infra.com/devopsman/jx-demo-infra.git

跳过ssl验证，添加任何一个ssh的私匙

## 第十步，配置访问jx-demo

cd install-jx/install

kubectl apply -f jx-demo-ing.yaml

kubectl get ing


# <a name="0">harbor 单机版搭建</a>

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
USER root
####这个ca.crt来自harbor仓库,当是https时，需要把ca.crt放到这个基础镜像中
COPY ca.crt /usr/local/share/ca-certificates/
RUN update-ca-certificates

```
docker build -t harbor.devopsman.io/devopsman/buildpacks-ca:20 .

docker push harbor.devopsman.io/devopsman/buildpacks-ca:20

## 如果需要导入证书到服务器上（每个节点）

### Mac 导入证书
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ca.crt

重启docker

### Centos导入自签证书
cp ca.crt /etc/pki/ca-trust/source/anchors/

cp harbor.devopsman.io.cert /etc/pki/ca-trust/source/anchors/

> 或使用scp到其它节点上

scp ca.crt root@10.10.11.104:/etc/pki/ca-trust/source/anchors/

scp harbor.devopsman.io.cert root@10.10.11.104:/etc/pki/ca-trust/source/anchors/

> 最后执行update

update-ca-trust

### Ubuntu导入自签证书
cp ca.crt /usr/local/share/ca-certificates/

update-ca-certificates

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

```shell
[Service]
Environment="HTTP_PROXY=http://10.10.11.103:8118"

Environment="HTTPS_PROXY=http://10.10.11.103:8118"

Environment="NO_PROXY=localhost,127.0.0.0/8,10.0.0.0/24,harbor.cloud2go.cn,harbor.devopsman.io,docker.io,docker.com"

```

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


# <a name="2">安装docker</a>

sudo yum install -y yum-utils

sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

> 列表显示versoin

yum list docker-ce --showduplicates | sort -r
```shell
docker-ce.x86_64            3:20.10.9-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.8-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.7-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.6-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.5-3.el7                     docker-ce-stable
```

> 安装最新

yum install -y docker-ce docker-ce-cli containerd.io

> 安装指定version

yum install -y docker-ce-20.10.8 docker-ce-cli-20.10.8 containerd.io

> 启动

systemctl enable docker && systemctl start docker && systemctl status docker

> 卸载

yum remove -y docker-ce docker-ce-cli containerd.io

rm -rf /var/lib/docker

rm -rf /var/lib/containerd

# ===<a name="3">安装k8s</a>===

## 每台节点执行
* 准备源

vi /etc/yum.repos.d/kubernetes.repo

```shell
[kubernetes]
name=Kubernetes Repo
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
enabled=1
```

yum clean all && yum makecache && yum repolist

### 设置路由

modprobe br_netfilter

sysctl -w net.bridge.bridge-nf-call-iptables=1

sysctl -w net.bridge.bridge-nf-call-ip6tables=1

echo "net.bridge.bridge-nf-call-iptables=1" > /etc/sysctl.d/k8s.conf

echo net.ipv4.ip_forward = 1 >> /etc/sysctl.conf && sysctl -p


## 每台节点执行
yum install -y kubeadm-1.21.6 kubelet-1.21.6 kubectl-1.21.6 --disableexclude s=kubernetes

> 让kubelet开机启动

systemctl enable kubelet

> 查看当前版本需要的组件version

kubeadm config images list

> 在master上执行升级计划

kubeadm upgrade plan

kubeadm upgrade apply v1.21.7

### x86-64 架构
```shell
docker pull k8simage/kube-apiserver:v1.21.6
docker pull k8simage/kube-controller-manager:v1.21.6
docker pull k8simage/kube-scheduler:v1.21.6
docker pull k8simage/kube-proxy:v1.21.6
docker pull k8simage/pause:3.4.1
docker pull k8simage/etcd:3.4.13-0
docker pull k8simage/coredns:1.7.0

docker pull weaveworks/weave-kube:2.8.1

```

> 换镜像tag
```shell
docker tag k8simage/kube-apiserver:v1.21.6 k8s.gcr.io/kube-apiserver:v1.21.6
docker tag k8simage/kube-controller-manager:v1.21.6 k8s.gcr.io/kube-controller-manager:v1.21.6
docker tag k8simage/kube-scheduler:v1.21.6 k8s.gcr.io/kube-scheduler:v1.21.6
docker tag k8simage/kube-proxy:v1.21.6 k8s.gcr.io/kube-proxy:v1.21.6
docker tag k8simage/pause:3.4.1 k8s.gcr.io/pause:3.4.1
docker tag k8simage/etcd:3.4.13-0 k8s.gcr.io/etcd:3.4.13-0
docker tag k8simage/coredns:1.7.0 k8s.gcr.io/coredns/coredns:v1.8.0

docker tag weaveworks/weave-kube:2.8.1 ghcr.io/weaveworks/launcher/weave-kube:2.8.1

```

### 创建config配置文件
vi kubeadm-config.yaml

```yaml
apiServer:
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: "10.10.11.100:6443"
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
kind: ClusterConfiguration
kubernetesVersion: v1.21.6
networking:
  podSubnet: 10.20.0.0/16
  serviceSubnet: 10.21.0.0/16
  dnsDomain: cluster.local
scheduler: {}
```

## 集群init，在master节点上运行

kubeadm init --config=/root/kubeadm-config.yaml --v=5

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

### 在各个节点上执行join
kubeadm reset

kubeadm join 10.10.11.100:6443 --token lcc52d.8bnpy71m8t7f63ui --discovery-token-ca-cert-hash sha256:f86b59da19d0eab7f6c11975599eed38212f140a2ca0232ba944a34d401a0f86

## 仅master安装weave
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

# 或者使用cilium网络插件

##不用kube-proxy组件，在master节点上运行

kubeadm init --config=/root/kubeadm-config.yaml --skip-phases=addon/kube-proxy --v=5

> 运行下面三行

mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config

> 在其它node节点上执行，加入集群

kubeadm join 10.10.11.100:6443 --token gnzgvi.s8psrfgzy48q192k --discovery-token-ca-cert-hash sha256:3b3f79542a1da45f8d81d05df0300dc5ffb5972fe0fb05a3c3ec1d8a2efe2270

## 安装cilium网络插件（内核最好>5.1）

> 安装helm

curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

> 安装helm（二进制）

wget https://get.helm.sh/helm-v3.7.2-linux-amd64.tar.gz

tar zxvf helm-v3.7.2-linux-amd64.tar.gz

cp linux-amd64/helm /usr/local/bin

> 添加helm仓库

```shell
helm repo add cilium https://helm.cilium.io/

helm install cilium cilium/cilium --version 1.11.1 \
--namespace kube-system \
--set kubeProxyReplacement=strict \
--set endpointRoutes.enabled=true \
--set hostServices.protocols=tcp
--set k8sServiceHost=10.10.11.100 \
--set k8sServicePort=6443 \
--set hubble.relay.enabled=true \
--set hubble.ui.enabled=true  \
--set prometheus.enabled=true \
--set operator.prometheus.enabled=true \
--set hubble.enabled=true \
--set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,http}"

```
> 确认网卡TX值

ethtool -g ens192

> 加载bpf内核模块

modprobe  xt_bpf

> 卸载

helm -n kube-system uninstall cilium

## 安装prometheus和grafana
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.10.5/examples/kubernetes/addons/prometheus/monitoring-example.yaml

kubectl -n cilium-monitoring get po

## 安装cilium cli
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz{,.sha256sum}

sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin

rm cilium-linux-amd64.tar.gz{,.sha256sum}

cilium status

## 为节点打标

kubectl label node devopsman104 node-role.kubernetes.io/worker=worker


# <a name="4">安装gitlab </a>

```shell
docker run -d --restart always --name gitlab --privileged --hostname harbor.devopsman.io --ulimit sigpending=62793 --ulimit nproc=131072 --ulimit nofile=60000 --ulimit core=0 -p 80:80 -p 443:443 -p 22:22 -e GITLAB_OMNIBUS_CONFIG="nginx['redirect_http_to_https'] = true; " -v /srv/gitlab-ce/conf:/etc/gitlab -v /srv/gitlab-ce/logs:/var/log/gitlab -v /srv/gitlab-ce/data:/var/opt/gitlab gitlab

docker run -d --restart always -p 5432:5432 --name postgres -e POSTGRES_DB=gitlab -e POSTGRES_USER=gitlab -e POSTGRES_PASSWORD=gitlab -v /var/lib/cloudtogo/postgressql:/var/lib/postgresql/data postgres

```
