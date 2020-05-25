---
title: Picgo图床配置说明
date: 2020-05-24 15:32:18
toc: true
categories: [娱乐, 软件使用]
tags: [Hexo]

---

# Picgo图床配置说明



## 链接

picgo官网

https://github.com/PicGo/Awesome-PicGo

picgo-plugin-compress

https://github.com/JuZiSang/picgo-plugin-compress


tinypng(在线压缩,  500张/月,用完了可以再去develop申请api, 有时候develop-api不稳定)

https://tinypng.com/dashboard/api





## 安装

**所有插件配置完毕后, 记得重启才能生效**!!!!





### 准备工作

1. 安装node 和 npm (官网下载, 不要brew安装)

2. 重点是npm 要用代理或者fq

   ```python
   # 设置加速
   npm config set registry https://registry.npm.taobao.org
   # 取消加速
   npm config set registry https://registry.npmjs.org/
   # 设置代理
   npm config set proxy http://127.0.0.1:1888
   npm config set https-proxy http://127.0.0.1:1888
   # 取消代理
   npm config delete proxy
   npm config delete https-proxy
   
   # 本机命令代理
   export https_proxy=http://127.0.0.1:1888 http_proxy=http://127.0.0.1:1888 all_proxy=socks5://127.0.0.1:1288
   ```

3. 安装picgo , 同时在设置里面打开日志, 可以观察插件启动情况, 用于排错



### 个人配置

![](https://gitee.com/matrixcall/bed001/raw/master/img/2020/05/20200524154228.png)



<u>Picgo系统配置</u>

- 设置日志文件(success)
- 修改快捷键(取消避免冲突)
- 设置代理(暂时不知道有没有)
- 关闭更新
- 时间戳重命名
- 开启上传提示



### 插件配置

**所有插件配置完毕后, 记得重启才能生效**!!!!

<u>gitee-uploader 1.1.2</u>

- 不用配置, 因为有界面



<u>compress 1.2.0</u>

- compress= tinypng
- key= 自己申请, 或者留空用作者的也可以, 不过会有500张/月的限制
- nameType= none





## 其它
Q: 为什么要用compress ? 
A: 因为 gitee只能上传小于1m的图片用来预览; github 好像没有这个限制, 不过慢, 所以还是压缩一下

Q: 仓库满了怎么办?
A: 再开个仓库...

Q: compress插件禁用后找不到了?
A: 去Picgo的json配置文件里面找, 把插件的false 改为true 即可;



