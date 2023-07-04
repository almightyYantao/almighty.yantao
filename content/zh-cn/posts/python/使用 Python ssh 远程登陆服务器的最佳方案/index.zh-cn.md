---
title: 使用 Python ssh 远程登陆服务器的最佳方案
date: 2023-04-19
tags: ['python','ssh']
featured_image: img.png
---

在使用 Python 写一些脚本的时候，在某些情况下，我们需要频繁登陆远程服务去执行一次命令，并返回一些结果。  
在 shell 环境中，我们是这样子做的。  
```bash
sshpass -p ${passwd} ssh -p ${port} -l ${user} -o StrictHostKeyChecking=no xx.xx.xx.xx "ls -l"
```
然后你会发现，你的输出有很多你并不需要，但是又不去不掉的一些信息（也许有方法，请留言交流），类似这样
```bash
host: xx.xx.xx.xx, port: xx 
Warning: Permanently added '[xx.xx.xx.xx]:xx' (RSA) to the list of known hosts. 
Login failure: [Errno 1] This server is not registered to rmp platform, 
please confirm whether cdn server. 
total 4 
-rw-r--r-- 1 root root 239 Mar 30 2018 admin-openrc
```
对于直接使用 shell 命令，来执行命令的，可以直接使用管道，或者将标准输出重定向到文件的方法取得执行命令返回的结果

## 1、使用 subprocess
若是使用 Python 来做这件事，通常我们会第一时间，想到使用 os.popen，os.system，commands，subprocess 等一些命令执行库来间接获取 。  
但是据我所知，这些库获取的 output 不仅只有标准输出，还包含标准错误（也就是上面那些多余的信息）  
所以每次都要对 output 进行的数据清洗，然后整理格式化，才能得到我们想要的数据。  
用 `subprocess` 举个例子  
```python
import subprocess ssh_cmd = "sshpass -p ${passwd} ssh -p 22 -l root -o StrictHostKeyChecking=no xx.xx.xx.xx 'ls -l'" status, output = subprocess.getstatusoutput(ssh_cmd) 
# 数据清理，格式化的就不展示了
<code...>
```

通过以上的文字 + 代码的展示 ，可以感觉到 ssh 登陆的几大痛点
-   **痛点一**：需要额外安装 sshpass（如果不免密的话）
-   **痛点二**：干扰信息太多，数据清理、格式化相当麻烦
-   **痛点三**：代码实现不够优雅（有点土），可读性太差
-   **痛点四**：ssh 连接不能复用，一次连接仅能执行一次
-   **痛点五**：代码无法全平台，仅能在 Linux 和 OSX 上使用

## 2、使用 paramiko
带着最后一丝希望，我尝试使用了 `paramiko` 这个库，终于在 `paramiko` 这里，找回了本应属于 Python 的那种优雅。  
你可以通过如下命令去安装它
```bash
python3 -m pip install paramiko
```
### 方法1：基于用户名和密码的 sshclient 方式登录
```python
import paramiko

ssh = paramiko.SSHClient()
# 允许连接不在know_hosts文件中的主机
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())

# 建立连接
ssh.connect("xx.xx.xx.xx", username="root", port=22, password="you_password")

# 使用这个连接执行命令
ssh_stdin, ssh_stdout, ssh_stderr = ssh.exec_command("ls -l")

# 获取输出
print(ssh_stdout.read())

# 关闭连接
ssh.close()
```
### 方法2：基于用户名和密码的 transport 方式登录
```python
import paramiko

# 建立连接
trans = paramiko.Transport(("xx.xx.xx.xx", 22))
trans.connect(username="root", password="you_passwd")

# 将sshclient的对象的transport指定为以上的trans
ssh = paramiko.SSHClient()
ssh._transport = trans

# 剩下的就和上面一样了
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
ssh_stdin, ssh_stdout, ssh_stderr = ssh.exec_command("ls -l")
print(ssh_stdout.read())

# 关闭连接
trans.close()
```
### 方法3：基于公钥密钥的 SSHClient 方式登录
```python
import paramiko

# 指定本地的RSA私钥文件
# 如果建立密钥对时设置的有密码，password为设定的密码，如无不用指定password参数
pkey = paramiko.RSAKey.from_private_key_file('/home/you_username/.ssh/id_rsa', password='12345')

# 建立连接
ssh = paramiko.SSHClient()
ssh.connect(hostname='xx.xx.xx.xx',
            port=22,
            username='you_username',
            pkey=pkey)

# 执行命令
stdin, stdout, stderr = ssh.exec_command('ls -l')

# 结果放到stdout中，如果有错误将放到stderr中
print(stdout.read())

# 关闭连接
ssh.close()
```
### 方法4：基于密钥的 Transport 方式登录
```python
import paramiko

# 指定本地的RSA私钥文件
# 如果建立密钥对时设置的有密码，password为设定的密码，如无不用指定password参数
pkey = paramiko.RSAKey.from_private_key_file('/home/you_username/.ssh/id_rsa', password='12345')

# 建立连接
trans = paramiko.Transport(('xx.xx.xx.xx', 22))
trans.connect(username='you_username', pkey=pkey)

# 将sshclient的对象的transport指定为以上的trans
ssh = paramiko.SSHClient()
ssh._transport = trans

# 执行命令，和传统方法一样
stdin, stdout, stderr = ssh.exec_command('df -hl')
print(stdout.read().decode())

# 关闭连接
trans.close()
```
以上四种方法，可以帮助你实现远程登陆服务器执行命令，如果需要复用连接：一次连接执行多次命令，可以使用 **方法二** 和 **方法四**  

用完后，记得关闭连接。

### 实现 sftp 文件传输

同时，`paramiko` 做为 ssh 的完美解决方案，它非常专业，利用它还可以实现 `sftp` 文件传输。
```python
import paramiko

# 实例化一个trans对象# 实例化一个transport对象
trans = paramiko.Transport(('xx.xx.xx.xx', 22))

# 建立连接
trans.connect(username='you_username', password='you_passwd')

# 实例化一个 sftp对象,指定连接的通道
sftp = paramiko.SFTPClient.from_transport(trans)

# 发送文件
sftp.put(localpath='/tmp/11.txt', remotepath='/tmp/22.txt')

# 下载文件
sftp.get(remotepath='/tmp/22.txt', localpath='/tmp/33.txt')
trans.close()
```
## 参考链接
-   [https://github.com/paramiko/paramiko](https://github.com/paramiko/paramiko)
-   [http://docs.paramiko.org](http://docs.paramiko.org/)
-   [https://www.liujiangblog.com/blog/15/](https://www.liujiangblog.com/blog/15/)