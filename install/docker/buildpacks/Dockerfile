FROM heroku/buildpacks:20

USER root

COPY ./docker/harbor.devopsman.io.crt /usr/local/share/ca-certificates/

RUN update-ca-certificates

ENV GOPROXY=https://goproxy.cn,direct
