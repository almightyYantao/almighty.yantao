---
title: hugo + github自动化部署静态博客
date: 2023-04-18 
tags: ['hugo','nginx','自动化部署','github']  
featured_image: img.png
---

## 背景
在使用hugo + nginx搭建好博客后，文章可以通过ftp上传到服务器，然后在服务器上再编译成网页，或者本地搭建的hugo环境，编译好网页再上传到服务器，这样做虽然也可以，但是很麻烦，如果每次都这么发布文章，肯定玩几次就不想弄了。

使用webhook就能实现自动部署，其实原理很简单。理想状态就是我把我的myblog项目托管到github，然后每次我写完文章直接push到github仓库，webhook监听到我的push，给我的服务器发送一个http请求，服务器收到请求后执行本地shell脚本，自动拉取最新的仓库代码，然后执行hugo编译成网页。这样就实现自动部署啦!
## 服务器环境配置

### node
```bash
cd ~
wget https://nodejs.org/dist/v14.17.3/node-v14.17.3-linux-x64.tar.xz
tar -xvf node-v14.17.3-linux-x64.tar.xz
cd node-v14.17.3-linux-x64/bin
ln -s /root/node-v14.17.3-linux-x64/bin/node /usr/bin/
ln -s /root/node-v14.17.3-linux-x64/bin/npm /usr/bin/
```
我是centos系统，直接安装编译好的二进制文件了，之前试过自己编译，要等好久就放弃了  
安装完后可以执行以下命令检查：
```bash
node -v
npm -v
```

### pm2
```bash
npm install pm2@latest -g
ln -s /root/node-v14.17.3-linux-x64/bin/pm2 /usr/bin/
pm2 -v
```

## 项目代码托管到github
在常用的本地机器上安装git，myblog 目录作为 git 仓库，push 到 github 进行备份。由于 public 目录下的静态网页完全可由其余文件自动生成，因此仓库可以排除 public 目录。

```
touch .gitignore && echo "/public" >> .gitignore
git add .
git commit -m "初次提交"
git branch -m main
git push -u origin main
```

## Github配置SSH Key
GitHub配置SSH Key的目的是为了帮助我们在通过git拉取代码时，不需要繁琐的验证用户名密码，如果需要验证，我们的自动脚本就挂了。  
首先检查是否存在sshkey
```bash
cd ~/.ssh
ls
# 或者
ll
# 看是否存在 id_rsa 和 id_rsa.pub文件，如果存在，说明已经有SSH Key
```
如果没有则执行如下命令，然后回车到底
```bash
ssh-keygen
```
最后获取sshkey填入github配置中（点击右上角头像，settings，找到ssh点进去取个名字复制下即可。） 
```bash
cat ~/.ssh/id_rsa.pub
```

## 配置shell脚本
```shell
#!/bin/bash

cd /var/www/yantao/public
git pull origin master
echo "deploy success!"
exit 0
```

## 编写js脚本
先安装一个第三方插件
{{< github-auto name="rvagg/github-webhook-handler" >}}
```bash
npm install github-webhook-handler --save
```
然后在webhook目录下创建一个github-webhook.js文件，写入以下内容，就是监听github的webhook请求
```shell
var http = require('http')
var createHandler = require('github-webhook-handler')
var exec = require('child_process').exec
var handler = createHandler({ path: '/webhook', secret: 'mysecret' })

http.createServer(function (req, res) {
  handler(req, res, function (err) {
    res.statusCode = 404
    res.end('no such location')
  })
}).listen(7777)

console.log("github Hook Server running at http://0.0.0.0:7777/webhook");

handler.on('error', function (err) {
  console.error('Error:', err.message)
})

handler.on('push', function (event) {
  console.log('Received a push event for %s to %s',
    event.payload.repository.name,
    event.payload.ref)
    exec('sh ./hugo-deploy.sh', function (error, stdout, stderr) {
        if(error) {
            console.error('error:\n' + error);
            return;
        }
        console.log('stdout:\n' + stdout);
        console.log('stderr:\n' + stderr);
    });
})
```
记住这个地址http://你的域名:7777/webhook这就是webhook要发送过来的地址，下一步需要用到。
{{< notice warn >}}
两个脚本最好放在一个目录下面，然后目前因为整体的方式是通过SSH拉取代码下来，你可以改成拉取所有的文件，然后在服务器里面hugo编译，也可以本地hugo，然后服务器只拉取public的文件，这个随意，看着改就行。
{{< /notice >}}

## 启动脚本
```bash
pm2 start github-webhook.js
pm2 startup
```

## github配置webhook

打开托管仓库
{{< github-auto name="almightyYantao/almighty.yantao" >}}
点击右上角Settings按钮，选择Webhooks，点击右上角Add wehook  
`Payload URL` ：http://你的域名:7777/webhook  
`Content type`：application/json  
`Secret`：mysecret  
到此，所有配置就结束了，可以在本机上push到仓库，服务器就会自动编译网站了。(#.#)