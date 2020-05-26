---
title: 免费好用的图床Gitee
date: 2020-05-26 08:00:18
toc: true
categories: [娱乐, 软件使用]
tags: [Hexo]

---

# 免费好用的图床Gitee

一直在找一个稳定的(不会跑路)的图床, 发现可以白嫖的都已经失效了( 包括 '微博图床', 'Imgur图床'等)...

突发奇想,  能否使用Github当图床呢?  在网上搜寻了一番, 方案可行, 而且还有更优解, `使用Gitee(国内版github)当图床`;

使用Gitee当图床, 总结有如下优势:

- 快(对比github)
- 稳(有提交记录, 删了都能给你找回来)
- 方便(用picgo可以做到一键上传后自动返回markdown的语法链接)



## 相关链接

picgo官网

https://molunerfinn.com/PicGo/

picgo使用说明

https://picgo.github.io/PicGo-Doc/zh/guide/

picgo压缩插件

https://github.com/JuZiSang/picgo-plugin-compress


tinypng (在线压缩,  500张/月,用完了可以再去develop申请api, 有时候develop-api不稳定)

https://tinypng.com/dashboard/api

node官网

https://nodejs.org/en/



## 安装

### 准备工作

1. 安装node (高版本自带 npm) 

   [官网下载](https://nodejs.org/en/)

   PS: 踩坑了, 注意macos不要用brew安装, 会一直提示安装失败...

2. 设置各种代理, 也可尝试不设置, 最终目的是 `npm install xx ` 能执行成功

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

3. 安装picgo的 npm工具 ,这一步是为了顺利安装  gitee插件和compress插件 

   ```
   npm install picgo -g
   ```

   

### 安装步骤

1. [下载](https://molunerfinn.com/PicGo/) 安装picgo, 

   直接下一步下一步就好了, picgo仅仅是上传图片的客户端, 重点是它的配置 !!

2. 安装gitee插件和 compress插件

   picgo仅仅是上传图片的客户端, 因此重点是要配置用它传到哪儿去, 我们要传到gitee, 所以需要gitee插件;

   picgo右键->插件设置-> 搜索关键词(gitee-upload 和 compress)

   坑: 若没有执行 过` npm install picgo -g ` , 这一步很难安装成功

   ![](https://gitee.com/matrixcall/bed001/raw/master/img/2020/05/20200526110803.png)

   

3. 等待安装完成, 安装成功后需要`重启`生效



### 图床设置

#### gitee图床设置

repo: 输入gitee仓库地址, 如果你的gitee仓库链接为https://gitee.com/matrixcall/bed001, 则这里写 `matrixcall/bed001`

token: 在gitee的 个人设置->私人令牌- 中添加

![](https://gitee.com/matrixcall/bed001/raw/master/img/2020/05/20200524154228.png)



#### compress插件设置

在弹出的表单中选择 compress=tinypng 即可, 表单说明参考了插件文档

https://github.com/JuZiSang/picgo-plugin-compress

![](https://gitee.com/matrixcall/bed001/raw/master/img/2020/05/20200526111717.png)





## 使用

参考picgo使用文档 , 方便快捷

https://picgo.github.io/PicGo-Doc/zh/guide/

![](https://raw.githubusercontent.com/Molunerfinn/test/master/picgo/picgo-2.0.gif)



## Q & A

Q: 为什么要用compress ? 
A: 因为 gitee只能上传小于1m的图片用来预览; github 好像没有这个限制, 不过慢, 所以还是压缩一下

Q: 仓库满了怎么办?
A: 再开个仓库...

Q: compress插件禁用后找不到了?
A: 去Picgo的json配置文件里面找, 把插件的false 改为true 即可;



