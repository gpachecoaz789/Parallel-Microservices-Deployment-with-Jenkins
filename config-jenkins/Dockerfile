FROM ubuntu:20.04

ENV KUBE_VERSION=1.18.0
ENV HELM_VERSION=3.5.4
ENV SNYK_VERSION=v1.662.0
ARG JAVA_VERSION=17

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && \
    apt-get install git curl wget gnupg -y

RUN apt-get update && \
    apt-get install -y curl gnupg && \
    echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] http://packages.cloud.google.com/apt cloud-sdk main" | tee -a /etc/apt/sources.list.d/google-cloud-sdk.list && \
    curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key --keyring /usr/share/keyrings/cloud.google.gpg  add - && \
    apt-get update -y && \
    apt-get install google-cloud-sdk -y

RUN wget -q https://github.com/snyk/snyk/releases/download/${SNYK_VERSION}/snyk-linux -O /usr/local/bin/snyk && \
    chmod +x /usr/local/bin/snyk && \
    wget -q https://storage.googleapis.com/kubernetes-release/release/v${KUBE_VERSION}/bin/linux/amd64/kubectl -O /usr/local/bin/kubectl && \
    chmod +x /usr/local/bin/kubectl && \
    wget -q https://get.helm.sh/helm-v${HELM_VERSION}-linux-amd64.tar.gz -O - | tar -xzO linux-amd64/helm > /usr/local/bin/helm && \
    chmod +x /usr/local/bin/helm && \
    chmod g+rwx /root && \
    mkdir /config && \
    chmod g+rwx /config && \
    helm repo add "stable" "https://charts.helm.sh/stable" --force-update

RUN apt install -y openjdk-${JAVA_VERSION}-jdk

RUN apt-get update && \
    apt-get install -y google-cloud-cli-gke-gcloud-auth-plugin

WORKDIR /config

CMD sh
