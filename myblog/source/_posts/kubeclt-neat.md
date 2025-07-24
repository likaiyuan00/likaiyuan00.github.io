---
title: kubeclt-neat
date: 2025-07-24 14:23:29
tags:
categories: k8s
---
# kubeclt-neat使用
* 如果部署的yaml丢失，可以使用kubeclt-neat精简后直接使用导入新的环境，默认的文件有多余的信息是不能直接使用的
```shell
yum -y install bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
wget https://github.com/itaysk/kubectl-neat/releases/download/v2.0.3/kubectl-neat_linux_amd64.tar.gz
tar -zxvf kubectl-neat_linux_amd64.tar.gz
mv kubectl-neat /usr/local/bin/
kubectl get deploy my-deployment -o yaml | kubectl neat > current-config.yaml
kubectl apply -f current-config.yaml
diff current-config.yaml new-config.yaml
```
