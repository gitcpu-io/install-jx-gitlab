FROM cnych/helm-kubectl-curl-git-jq-yq

COPY ./docker/gitlab.infra.local.pem /usr/local/share/ca-certificates/

RUN update-ca-certificates

RUN apk add --update --no-cache ca-certificates

