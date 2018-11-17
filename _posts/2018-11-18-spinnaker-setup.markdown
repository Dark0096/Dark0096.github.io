---
layout: post
title:  "Spinnaker setup command list"
date:   2018-11-18 01:18:00
author: Dark
categories: Spinnaker
tags: spinnaker
---

## Spinnaker helm 을 기반으로 Setting 하기

Ubuntu 환경에서 Spinnaker 를 Kubernetes 기반으로 배포할 수 있는 테스트 환경을 빠르게 만들 수 있도록 Command 를 정리하였습니다.

## Commands

{% highlight bash %}
sudo su

# minikube setup
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube
mv minikube /usr/local/bin
minikube version

# minikube config setup
minikube config set vm-driver none
minikube start --cpus 4 --memory 1572864

# kubectl setup
apt-get update
curl -Lo kubectl https://storage.googleapis.com/kubernetes-release/release/v1.10.0/bin/linux/amd64/kubectl && chmod +x kubectl && sudo cp kubectl /usr/local/bin/ && rm kubectl

apt-get update && sudo apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add - 
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubectl

# helm install
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | cat > /tmp/helm_script.sh \
&& chmod 755 /tmp/helm_script.sh && /tmp/helm_script.sh --version v2.8.2

helm init --upgrade

apt-get install socat
helm install --name deployment-test --timeout 900 stable/spinnaker
export DECK_POD=$(kubectl get pods --namespace default -l "cluster=spin-deck" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward --namespace default spin-deck-5fb57689bd-nnk7j 9000 &
{% endhighlight %}