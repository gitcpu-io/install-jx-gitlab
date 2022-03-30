FROM gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/git-init:v0.21.0

COPY ./docker/gitlab.infra.local.pem /usr/local/share/ca-certificates/

RUN update-ca-certificates
