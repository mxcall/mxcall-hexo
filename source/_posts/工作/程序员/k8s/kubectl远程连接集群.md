---
title: kubectl远程连接集群
date: 2020-05-27 08:00:00
toc: true
categories: [工作, 程序员, k8s]
tags: [k8s, docker]

---


# kubectl远程连接集群
以前一直以为只有拥有 k8s集群节点才能操作集群(kubectl), 后来开发过相关restful API才知道, 集群远程也是可以连的, 实际上也就是master节点监听了指定HTTP端口号, 程序通过给端口号发命令, 来获得各种信息;

所以, 我们申请相关机器权限的时候, 可以找任何一台机器, 只要它与k8s主节点是网络互通的, 就可以在上面执行相关kubectl命令, 这样可以避免人员误操作把集群机器搞宕机!

## 相关链接: 
官网: https://k8smeetup.github.io/docs/tasks/tools/install-kubectl/
版本列表: https://storage.googleapis.com/kubernetes-release/release/stable.txt
第三方参考文档: https://dylanyang.top/post/2019/05/22/kubectl%E8%BF%9C%E7%A8%8B%E8%BF%9E%E6%8E%A5%E9%9B%86%E7%BE%A4/

`kubectl下载链接: `
mac: https://storage.googleapis.com/kubernetes-release/release/v1.9.0/bin/darwin/amd64/kubectl
linux: https://storage.googleapis.com/kubernetes-release/release/v1.9.0/bin/linux/amd64/kubectl


## 操作步骤
1. 修改所下载的 kubectl 二进制文件为可执行模式
chmod +x ./kubectl
2. 将 kubectl 可执行文件放置到系统 PATH 目录下。
sudo mv ./kubectl /usr/local/bin/kubectl
3. 拷贝已经配置好的config, 从集群中主节点后有kubectl权限的节点拷贝config文件
~/.kube/config
4. 配置到自己的机器中

```
mkdir $HOME/.kube
cd $HOME/.kube
rz config
```

## 检验集群
执行命令
kubectl cluster-info

