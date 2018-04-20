---
title: 使用gitlab构建基于docker的持续集成（二）
tags: [gitlab, docker, aspnetcore]
categories: 持续集成
date: 2018/4/20
updated: 2018/4/20
---
----------
# Centos配置gitlab镜像并且启动
在本机上创建目录结构，然后上传到虚拟机去运行，使用的软件为Xshell，Xftp，记得把CA证书带上。

目录结构：
```treeview
gitlab
	config
		ssl
			gitlab.luna.cn.crt
			gitlab.luna.cn.key
	logs
	docker-compose.yml
```
docker-compose.yml文件，要说明的是这里的环境变量配置，是可以使用gitlab配置文件中的配置的，不过后续使用配置文件也可以。
```yml
version: '3'
services:
    gitlab:
      image: 'twang2218/gitlab-ce-zh:10.6.2'
      restart: unless-stopped
      hostname: 'gitlab.luna.cn'
      container_name: gitlab1062
      environment:
        TZ: 'Asia/Shanghai'
        GITLAB_OMNIBUS_CONFIG: |
          external_url 'http://gitlab.luna.cn/'
          registry_external_url 'https://gitlab.luna.cn'
          gitlab_rails['time_zone'] = 'Asia/Shanghai'
          # gitlab_rails['smtp_enable'] = true
          # gitlab_rails['smtp_address'] = "smtp.exmail.qq.com"
          # gitlab_rails['smtp_port'] = 465
          # gitlab_rails['smtp_user_name'] = "xxxx@xx.com"
          # gitlab_rails['smtp_password'] = "password"
          # gitlab_rails['smtp_authentication'] = "login"
          # gitlab_rails['smtp_enable_starttls_auto'] = true
          # gitlab_rails['smtp_tls'] = true
          # gitlab_rails['gitlab_email_from'] = 'xxxx@xx.com'
      ports:
        - '80:80'
        - '443:443'
        - '1022:22'
      volumes:
        - ./data:/var/opt/gitlab
        - ./config:/etc/gitlab
        - ./logs:/var/log/gitlab
```

上传到/root文件夹下

![gitlab文件夹上传](https://www.github.com/loveshullf/Notes/raw/img/小书匠/使用gitlab构建基于docker的CI-2018-4/1524002559155.jpg)

Xshell中运行docker-compose up，等待....
# Centos配置防火墙
我列出配置需要用到的命令，主要打开http，https，docker-registry这几个服务和1022端口。
1022端口是为了ssh使用的。
```bash
firewall-cmd --list-ports #查看打开的端口
firewall-cmd --list-services #查看打开的服务
firewall-cmd --get-services  #查看可以打开的服务

firewall-cmd --add-service=xxx --permanent  #添加服务
firewall-cmd --add-port=xx/tcp --permanent #添加端口

firewall-cmd --reload      #打开操作之后要重载
```
# windows上访问gitlab
域名为： gitlab.luna.cn 能访问，说明一切正常。
然后需要做的是：
1.需要创建一个管理员密码
2.管理员账号为root，用设置的密码登录。
3.如果第一次使用gitlab，你可以看看每个菜单，如果没有兴趣，就可以关闭了。

# Centos上配置gitlab-runner
gitlab-runner是一个很大的坑，我不知道我能不能说的清楚，不对的地方，希望大家指正。

安装官方的说法，runner需要先注册，从而让gitlab认识这个runner，从而触发runner运行job。

这里我使用的是runner的docker镜像。

1.首先创建一个文件夹用来放runner的配置文件，这里是/root/gitlab-runner。
2.然后是启动runner镜像，执行注册。注册的参数除了第一个和第二个参数之外，其他参数都可以后面配置，不用太纠结。
```bash
  #运行镜像
  docker run -d --name gitlab-runner-bate --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \#这个配置可以让runner联通本机docker
  -v /root/gitlab-runner/config:/etc/gitlab-runner \
  gitlab/gitlab-runner:latest
  
  #执行注册
   docker exec -it gitlab-runner-bate gitlab-runner register
```
3.注册之后，目录结构应该是这样的，然后配置文件应该是这样的。
```treeview
gitlab-runner
  config
     config.toml
  docker-compose.yml
```
config.toml
```yml
concurrent = 1
check_interval = 0

[[runners]]
  name = "test"
  url = "http://gitlab.luna.cn/"
  token = "040a9c35d32e1c9a912c006c8a10a6"
  executor = "docker"
  [runners.docker]
    tls_verify = false
    image = "docker:stable"
    privileged = false
    disable_cache = false
    volumes = ["/var/run/docker.sock:/var/run/docker.sock", "/cache"]
    shm_size = 0
  [runners.cache]
```
4.runner是否配置成功，可以用管理员登录gitlab去查看，成功的话，是可以看到的。
5.然后创建一个docker-compose来启动，就不用敲那么长docker run命令了。
docker-compose.yml
```yml
version: '3'
services:
    runner:
      image: 'gitlab/gitlab-runner:latest'
      container_name: gitlab-runner
      restart: always 
      volumes: 
        - ./config:/etc/gitlab-runner
        - /var/run/docker.sock:/var/run/docker.sock
```

最后贴几个官方文档给大家参考，也给自己做下备忘。

[gitlab-runner的配置](https://docs.gitlab.com/runner/configuration/)
[在容器中运行GitLab Runner](https://docs.gitlab.com/runner/install/docker.html#docker-image-installation-and-configuration)

