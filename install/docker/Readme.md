# 重新打镜像，添加gitlab私有仓库证书到容器中

cd install-jx/install

## 构建git-init

docker build -f docker/git-init/Dockerfile -t harbor.devopsman.io/devopsman/git-init:v0.21.0 .

## 构建golangci-line
docker build -f docker/golangci-lint/Dockerfile -t harbor.devopsman.io/devopsman/golangci-lint:v1.44 .

## 构建commit-infra
docker build -f docker/commit-infra/Dockerfile -t harbor.devopsman.io/devopsman/commit-infra:v1.1 .


## 构建buildpacks，把harbor私有镜像库证书放到容器中
docker build -f docker/buildpacks/Dockerfile -t harbor.devopsman.io/devopsman/buildpacks-ca:20 .
