# Linux安装
## 安装RabbitMQ
### 1、下载
官网下载地址：[https://www.rabbitmq.com/download.html](https://www.rabbitmq.com/download.html)
这里我们选择的版本号（注意这两个版本要求）
`rabbitmq-server-3.8.8-1.el7.noarch.rpm`
GitHub：[https://github.com/rabbitmq/rabbitmq-server/releases/tag/v3.8.8](https://github.com/rabbitmq/rabbitmq-server/releases/tag/v3.8.8)
`erlang-21.3.8.21-1.el7.x86_64.rpm`
加速下载：[https://packagecloud.io/rabbitmq/erlang/packages/el/7/erlang-21.3.8.21-1.el7.x86_64.rpm](https://packagecloud.io/rabbitmq/erlang/packages/el/7/erlang-21.3.8.21-1.el7.x86_64.rpm)
### 2、安装
使用`xftp`工具上传到`/usr/local/rabbitmq`下
使用以下命令安装
```shell
rpm -ivh erlang-21.3.8.21-1.el7.x86_64.rpm
yum install socat -y
rpm -ivh rabbitmq-server-3.8.8-1.el7.noarch.rpm
```
### 3、启动
```shell
# 启动服务
systemctl start rabbitmq-server
# 查看服务状态
systemctl status rabbitmq-server
# 开机自启动
systemctl enable rabbitmq-server
# 停止服务
systemctl stop rabbitmq-server
# 重启服务
systemctl restart rabbitmq-server

```
![image.png](https://cdn.nlark.com/yuque/0/2023/png/35204765/1685857814289-6d5abd64-77ef-420c-b593-d543845274c5.png#averageHue=%23100c09&clientId=u254f9e44-ad8f-4&from=paste&height=395&id=u854ed9c7&originHeight=395&originWidth=805&originalType=binary&ratio=1&rotation=0&showTitle=false&size=53038&status=done&style=none&taskId=u1f5a360c-841a-4a0e-912f-ee855e8c28e&title=&width=805)
可以看到 `active（running）`即安装成功
## Web管理界面及授权操作
### 1、安装
默认情况下是没有web端的客户端插件，需要安装才可以生效
```shell
rabbitmq-plugins enable rabbitmq_management
```
安装完毕后，重启服务
```shell
systemctl restart rabbitmq-server
```
然后访问`ip:15672`，使用默认账号密码（guest）登录。此时出现权限问题，需要添加一个远程登录的用户！
### 2、添加用户
```shell
# 创建账号和密码
rabbitmqctl add_user admin 123456

# 设置用户角色
rabbitmqctl set_user_tags admin administrator

# 为用户添加资源权限
# set_permissions [-p <vhostpath>] <user> <conf> <write> <read>
rabbitmqctl set_permissions -p "/" admin ".*" ".*" ".*"
```
用户级别：

1. **administrator**：可以登录控制台、查看所有信息、可以对 rabbitmq 进行管理
2. **monitoring**：监控者 登录控制台，查看所有信息
3. **policymaker**：策略制定者 登录控制台，指定策略
4. **managment**：普通管理员 登录控制台

再次登录，用 admin 用户
> 命令

关闭应用的命令为：rabbitmqctl stop_app
清除的命令为：rabbitmqctl reset
重新启动命令为：rabbitmqctl start_app
# Docker安装
官网：[https://registry.hub.docker.com/_/rabbitmq/](https://registry.hub.docker.com/_/rabbitmq/)
```shell
docker run -id --name myrabbit -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=123456 -p 15672:15672 rabbitmq:3-management

```
