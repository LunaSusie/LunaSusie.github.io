---
title: 使用gitlab构建基于docker的CI
tags: [gitlab, docker, aspnetcore]
categories: 持续集成
---
----------

我测试项目使用的是aspnetcore项目，所以项目中的坑是针对net的，但是gitlab部分应该是通用的。
本文中git仓库，CI等这些系统都使用了gitlab自带的，并且开启了gitlab的dockerc仓库功能。

本来是有想法是部署到公网，但是由于部署gitlab需要的服务器要求太高，再加上gitlab-runner的运行，本人腾讯云服务器配置太低，所以使用的是虚拟机配合本机去实现，然后就是gitlab很消耗内存，需要大内存，我虚拟机配置之后加上runner，运行状态占用了5G内存。


本文只是做了部署的一些记录，不保证你也能部署成功，如果过程中有其他问题，欢迎大家可以留言交流。

推荐目录浏览，因为本文实在太长了。

本文目前为止，只是部署完成了，对于后续CI项目还没有完成......


![git项目界面](https://www.github.com/loveshullf/Notes/raw/img/小书匠/使用gitlab构建基于docker的CI-2018-4/1524012232084.jpg)

![docker仓库](https://www.github.com/loveshullf/Notes/raw/img/小书匠/使用gitlab构建基于docker的CI-2018-4/1524012665989.jpg)

![CI界面](https://www.github.com/loveshullf/Notes/raw/img/小书匠/使用gitlab构建基于docker的CI-2018-4/1524012721018.jpg)

# 整体环境规划
* 本机（开发机）：win10系统
	* 开启Hyper-v
	* 安装docker for windows
	* vs2017
	* 使用自签名证书的话要导入证书
	* hosts设置域名：gitlab.luna.cn 映射gitlab服务器上
* gitlab服务器：Centos 7.3
	* docker
	* docker-compose
    * 导入CA证书   
    * 设置hosts设置域名：gitlab.luna.cn 映射本机ip
* gitlab-runner：Centos7.3 （我把它部署到了gitlab服务器上了，规范的做法是分开部署）
	* docker
	* docker-compose

# 准备工作
## CA证书
我提供一个我用的证书，证书的域名为：gitlab.luna.cn
创建两个文本，然后复制下面代码，改文件名即可。
gitlab.luna.cn.crt
```bash
-----BEGIN CERTIFICATE-----
MIIDWzCCAkOgAwIBAgIBATANBgkqhkiG9w0BAQsFADAZMRcwFQYDVQQDEw5naXRs
YWIubHVuYS5jbjAgFw0xODA0MTMxNjU2MDBaGA8yMTE4MDQxMzE2NTYwMFowGTEX
MBUGA1UEAxMOZ2l0bGFiLmx1bmEuY24wggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAw
ggEKAoIBAQDfNr+I2/H7blGFJfpoN4DM3Z4eRolH439ElKkdIj99XiXQ0U2mIm57
LaFORPGPh0ESqT4qKWSeIyQvPInBb97ACbVj30iqBoQPC919bHOjQpebp7L/8YMH
ZMqA7Njaq+a397d6bbtMTKX0nixmTSqqrohJhK6vvdKgjLhK1neLDmGw9wJwh40W
IuOV8u/gQqGGnKmzJieMufowTcSPm9nM0rACYpUAdGCpMp1MFxpkRKd62LKzx36j
uaXKmwiLo9RsKlPGoZbyOjZhkEw2F0TQDSno5CKJIsO+GqK2Piu2o55bmhe8DOWv
+v3BfGaM0Ne5injmCE+SUs7baInS7TqFAgMBAAGjgaswgagwDwYDVR0TAQH/BAUw
AwEB/zAdBgNVHQ4EFgQUtX0LUdEaIwJnFSIIVNwNBRX6Js8wCwYDVR0PBAQDAgGG
MDsGA1UdJQQ0MDIGCCsGAQUFBwMBBggrBgEFBQcDAgYIKwYBBQUHAwMGCCsGAQUF
BwMEBggrBgEFBQcDCDAZBgNVHREEEjAQgg5naXRsYWIubHVuYS5jbjARBglghkgB
hvhCAQEEBAMCAPcwDQYJKoZIhvcNAQELBQADggEBACYKfo4KacODb202yW+GFuf1
4jzUtM7xVp9KfDbKfitpYJETNDouDoauZRmjJHHqTOMXlnVQjkgC01o5t/QcCT/X
siV+/5aqj+44Ww3POd9CizVId1luWC44UZ+tJxJmQqgn5fckH7SE5k8+tZrzQgZa
wbqE37mnCauWMTcLrUxEy36m4yQjrn5XCk2LRZt60kU2q/CbOyR7FCzF3Dv0+99r
OfmA52THHlFxyoP4E20cEH/bVZ11wyTopCR9B7yzQykkJ3jeNylwEOWPnhDOk4+t
ZRedShv32tHM+XT9wuQ0+YgV+jjcM3PfqfhDSunln/i0dKHnkMRZyqLbSE4pOo8=
-----END CERTIFICATE-----
```
gitlab.luna.cn.key
```bash
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA3za/iNvx+25RhSX6aDeAzN2eHkaJR+N/RJSpHSI/fV4l0NFN
piJuey2hTkTxj4dBEqk+KilkniMkLzyJwW/ewAm1Y99IqgaEDwvdfWxzo0KXm6ey
//GDB2TKgOzY2qvmt/e3em27TEyl9J4sZk0qqq6ISYSur73SoIy4StZ3iw5hsPcC
cIeNFiLjlfLv4EKhhpypsyYnjLn6ME3Ej5vZzNKwAmKVAHRgqTKdTBcaZESnetiy
s8d+o7mlypsIi6PUbCpTxqGW8jo2YZBMNhdE0A0p6OQiiSLDvhqitj4rtqOeW5oX
vAzlr/r9wXxmjNDXuYp45ghPklLO22iJ0u06hQIDAQABAoIBAGX8pdbqZ83xwd2M
VLV5Zqg0OiKrJ95o26WCJyLgmxG1CqI2f7wAz2oIl0MjzRs/OURFf9nTv91hQQ80
Idz4OFaWGQLg6lqFT6FwUmsUOmHF829zWB4JQ00FiGEP1qVTFb/It1SA/qsF+m2i
N7cmWvBRfoPY09gIa0xf/3RyOXyW4ynzxN6OGUmXK7uuGmjJsG+yYUcIkpi2nVb9
WhCB06sNZ/wLWhg9hHZTaiC/DQ3WmFp2pZCI8qoY5jD3rx0yKJmqrShYkphKHYuM
9AnyRoMLTUJZIzHuiysbQMq50qunlEIz9mQdAvKH+ObP+AqXJSpyQ/11Xk45YqQy
aJwP2wECgYEA783ihXtg7vS1Ymj7IRN07i0oXm16M+j7wgq/PpI/FJqL7S2Es3Ot
3lqEsX4V5/HLgAwsH2kcNelkqmSzLE6CwxktmFqj16MBxGkteI5T0O8QE4+KcZFh
M3JIMkuwPjHoQP9SoDfxblVOm01y0RqDR39Speh3ZQxZU/eg7hjiU+0CgYEA7koF
yxzLUAbjB2jMHsMxejpyk2qFOgGjW2Poy38kPapeID7q9dUSUXDgqnPZ0hsrv0I5
8jrh+OK4+Nn9MtpJ2XtP116Jc6hvUOMplZALiU8oCU5IwCAGh6oVOiDfENTe7jRV
wZYiYdBa5cfhprp3SThx1gCQVja1SJJ77E7u3fkCgYEApoYsVVE2IPnhs3L/YRqn
ynWlYN1ZTQ7vNPJNl9/q2h3wKUXArvUXuh7VooPSJn1sOYE6ap2NL4rhksnW+l+S
wnSLiw72U9oocgIvx1XesmowmcTF+NNh0l378KFKxAXYKLqk4Am5KEspCQOhRb/J
hi7Ob9OchZkrtvlw0aaKFIkCgYEAtUcI60EHhuUGV7+o8YorHMJUIcO6gKt4W/FA
y3b42hS+sKdM1iH3Yo+Nyv6Bae6TtFesf5O+Dzpj36Tuk34vCk1eKwjXZm5v6Mg3
/Xjs3dOjMJkmjUqPzSteJK+XI1XeFrcnujL+Cw2X6RDLoKxgTQqsx1H8fCn4dbJC
pj5SR/kCgYA3dJD4p66xs7NxreppPCiuJvdTcK5GNseLy0ZK/EIJhAaZd5eB43h2
TwBWieY8hDEgeqO52Y4lwLVEPFLfnDvVhy5Vt21RY4DkubrLVrT9Qfi9FXoZDv1K
FopMObRAdaaxh9h5SHjIUg6ABf5owQsB1y+cmaGYHIDkLttOmXVJEA==
-----END RSA PRIVATE KEY-----
-----BEGIN CERTIFICATE-----
MIIDWzCCAkOgAwIBAgIBATANBgkqhkiG9w0BAQsFADAZMRcwFQYDVQQDEw5naXRs
YWIubHVuYS5jbjAgFw0xODA0MTMxNjU2MDBaGA8yMTE4MDQxMzE2NTYwMFowGTEX
MBUGA1UEAxMOZ2l0bGFiLmx1bmEuY24wggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAw
ggEKAoIBAQDfNr+I2/H7blGFJfpoN4DM3Z4eRolH439ElKkdIj99XiXQ0U2mIm57
LaFORPGPh0ESqT4qKWSeIyQvPInBb97ACbVj30iqBoQPC919bHOjQpebp7L/8YMH
ZMqA7Njaq+a397d6bbtMTKX0nixmTSqqrohJhK6vvdKgjLhK1neLDmGw9wJwh40W
IuOV8u/gQqGGnKmzJieMufowTcSPm9nM0rACYpUAdGCpMp1MFxpkRKd62LKzx36j
uaXKmwiLo9RsKlPGoZbyOjZhkEw2F0TQDSno5CKJIsO+GqK2Piu2o55bmhe8DOWv
+v3BfGaM0Ne5injmCE+SUs7baInS7TqFAgMBAAGjgaswgagwDwYDVR0TAQH/BAUw
AwEB/zAdBgNVHQ4EFgQUtX0LUdEaIwJnFSIIVNwNBRX6Js8wCwYDVR0PBAQDAgGG
MDsGA1UdJQQ0MDIGCCsGAQUFBwMBBggrBgEFBQcDAgYIKwYBBQUHAwMGCCsGAQUF
BwMEBggrBgEFBQcDCDAZBgNVHREEEjAQgg5naXRsYWIubHVuYS5jbjARBglghkgB
hvhCAQEEBAMCAPcwDQYJKoZIhvcNAQELBQADggEBACYKfo4KacODb202yW+GFuf1
4jzUtM7xVp9KfDbKfitpYJETNDouDoauZRmjJHHqTOMXlnVQjkgC01o5t/QcCT/X
siV+/5aqj+44Ww3POd9CizVId1luWC44UZ+tJxJmQqgn5fckH7SE5k8+tZrzQgZa
wbqE37mnCauWMTcLrUxEy36m4yQjrn5XCk2LRZt60kU2q/CbOyR7FCzF3Dv0+99r
OfmA52THHlFxyoP4E20cEH/bVZ11wyTopCR9B7yzQykkJ3jeNylwEOWPnhDOk4+t
ZRedShv32tHM+XT9wuQ0+YgV+jjcM3PfqfhDSunln/i0dKHnkMRZyqLbSE4pOo8=
-----END CERTIFICATE-----

```
## 准备虚拟机系统：安装Centos7.3
安装的时候尤其要注意：
* 开启网络，不开启，后续需要进到虚拟机开启，比较麻烦。
* 设置时区，不设置的话，时间会错误。
* 设置密码，root的密码，不说你也知道。
* 最小安装，就是默认，不要去选其他的乱七八糟的预装系统。
最后，要保证虚拟机能联网，主机能ping通虚拟机就可以了。

## 3.设置Centos的host
1.获取Centos主机的IP
```bash
[root@gitlab etc]# ip addr
```
2.找eth0的地址，我这里找到ip为：172.24.162.122
3.设置hosts
```bash
[root@gitlab etc]# vi /etc/hosts
```
4.添加一条记录 
```bash
172.24.162.122   gitlab.luna.cn
```
5.你也可以把hostname也一起改了。
5.退出保存即可。
6.ping一下域名，看一下是否可以通。
## 设置windows的host
1.打开C:\Windows\System32\drivers\etc\hosts文件
2.添加记录172.24.162.122   gitlab.luna.cn
3.最好重启。
4.尝试ping一下域名，是否能通，能通就行。
## docker安装
获取docker官方源
```bash
yum -y install yum-utils

sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```
如果获取不到源，可以用阿里源，vi /etc/yum.repos.d/docker-ce.repo替换掉内容为下面的
```bash
[docker-ce-stable]

name=Docker CE Stable - $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/$basearch/stable
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-stable-debuginfo]
name=Docker CE Stable - Debuginfo $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/debug-$basearch/stable
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-stable-source]
name=Docker CE Stable - Sources
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/source/stable
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-edge]
name=Docker CE Edge - $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/$basearch/edge
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-edge-debuginfo]
name=Docker CE Edge - Debuginfo $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/debug-$basearch/edge
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-edge-source]
name=Docker CE Edge - Sources
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/source/edge
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-test]
name=Docker CE Test - $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/$basearch/test
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-test-debuginfo]
name=Docker CE Test - Debuginfo $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/debug-$basearch/test
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-test-source]
name=Docker CE Test - Sources
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/source/test
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg
```
安装docker
```bash
#刷新yum源
yum makecache fast
#安装docker-ce
yum -y install docker-ce
```
最后最好配置一下docker仓库源，不然下载镜像会很慢。我配置的阿里云的源。

## docker-compose安装
这里有点坑，我推荐的是安装一个python3和python2共存环境，然后从github上直接下载docker-compose
因为我网上看到的是python3支持yml文件的中文注释。

[github上docker-compose最新版本地址](https://github.com/docker/compose/releases)

以下是参考命令，不要复制全部运行，会有问题
安装python3，并且和Python2共存
```bash
mv /usr/bin/python2.7 /usr/bin/python2.7.5　　# 保留默认版本python为python2.7.5
ln -s /usr/bin/python2.7.5 /usr/local/bin/python2.7.5　　# 创建软连接

#yum修改
ll /usr/bin/yum*　　# 查看/usr/bin/目录下所有yum文件（7个）头部
vi /usr/bin/yum*　　# 修改/usr/bin/目录下所有yum文件（7个）头部
#!/usr/bin/python —> #!/usr/bin/python2.7.5　
vi /usr/libexec/urlgrabber-ext-down　　# 修改/usr/libexec/目录下 urlgrabber-ext-down头部
#!/usr/bin/python —> #!/usr/bin/python2.7.5

#firewall也要修改
ll /usr/bin/firewall* #查看，2个
vi /usr/bin/firewall* #修改
vi /usr/sbin/firewalld #修改

#编译环境
yum -y groupinstall 'Development Tools' 
yum -y install zlib-devel bzip2-devel openssl-devel ncurses-devel

#下载python3
yum -y install wget
wget https://www.python.org/ftp/python/3.6.2/Python-3.6.2.tgz 

#编译，就不一步一步解释了
tar zxvf Python-3.6.2.tgz
cd Python-3.6.2
./configure
make all
make install
make clean
make distclean
rm -rf /usr/bin/python
rm -rf /usr/bin/python3
rm -rf /usr/bin/python3.6
ln -s /usr/local/bin/python3.6 /usr/bin/python
ln -s /usr/local/bin/python3.6 /usr/bin/python3
ln -s /usr/local/bin/python3.6 /usr/bin/python3.6
/usr/bin/python -V
/usr/bin/python3 -V
/usr/bin/python3.6 -V
rm -rf /usr/local/bin/python
rm -rf /usr/local/bin/python3
ln -s /usr/local/bin/python3.6 /usr/local/bin/python
ln -s /usr/local/bin/python3.6 /usr/local/bin/python3
python -V
python3 -V
python3.6 -V
```
下载docker-compose
```bash
curl -L https://github.com/docker/compose/releases/download/1.21.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```
如果可以使用docker-compose命令基本算安装成功了吧。
## window下安装docker for windows就可以了，包含docker-compose
## Centos7.3导入CA证书
```bash
yum install ca-certificates
update-ca-trust force-enable
#拷贝证书，foo.crt只是例子
cp foo.crt /etc/pki/ca-trust/source/anchors/
update-ca-trust extract
```
## window10导入CA证书
直接打开.crt证书

![安装证书](https://www.github.com/loveshullf/Notes/raw/img/小书匠/使用gitlab构建基于docker的CI-2018-4/1524000104742.jpg)

![导入证书到本机计算机](https://www.github.com/loveshullf/Notes/raw/img/小书匠/使用gitlab构建基于docker的CI-2018-4/1524000178679.jpg)

![选择导入位置](https://www.github.com/loveshullf/Notes/raw/img/小书匠/使用gitlab构建基于docker的CI-2018-4/1524000330723.jpg)

![导入](https://www.github.com/loveshullf/Notes/raw/img/小书匠/使用gitlab构建基于docker的CI-2018-4/1524000361878.jpg)

![完成](https://www.github.com/loveshullf/Notes/raw/img/小书匠/使用gitlab构建基于docker的CI-2018-4/1524000406659.jpg)

导入之后打开证书应该是这样的

![验证导入成功](https://www.github.com/loveshullf/Notes/raw/img/小书匠/使用gitlab构建基于docker的CI-2018-4/1524000473733.jpg)
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
docker-compose.yml文件，其中的其他配置自己去摸索gitlab官方吧
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
域名为： gitlab.luna.cn 能访问，说明一切正常，不能运行，而我再说明里没有说，请google。
然后
1.需要创建一个管理员密码
2.管理员账号为root，用设置的密码登录。
3.如果第一次使用gitlab，你可以看看每个菜单，如果没有兴趣，就可以关闭了。

# Centos上配置gitlab-runner
gitlab-runner是一个很大的坑，我不知道我能不能说的清楚，不对的地方，希望大家指正。

我使用的是gitlab-runner的docker镜像，而且使用的是docker in docker 模式。

我要做的是先运行这个镜像，然后把配置文件映射出来，然后停止并且删除容器。
这样做的原因是，需要先注册，然后gitlab才认得这个runner，然后我就可以修改这个配置文件，用docker-compose来启动这个runner。

这样就关键是可以重用这个已经注册过的runner。
这样就关键是可以重用这个已经注册过的runner。
这样就关键是可以重用这个已经注册过的runner。

运行gitlab-runner，并且映射出配置文件，文件名为config.toml
```bash
  #运行镜像
  docker run -d --name gitlab-runner-bate --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \#这个配置可以让runner联通本机docker
  -v /root/gitlab-runner/config:/etc/gitlab-runner \
  gitlab/gitlab-runner:latest
  
  #执行注册
   docker exec -it gitlab-runner-bate gitlab-runner register
```
创建一个文件夹gitlab-runner，存放配置，然后新建一个docker-compose.yml文件，一样传到服务器，然后docker-compose up。
```treeview
gitlab-runner
  config
     config.toml
  docker-compose.yml
```
config.toml
```
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
docker-compose.yml
```
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

额，总觉得这个还是说的不是很明白，最后贴几个官方文档给大家参考，也给自己做下备忘

[gitlab-runner的配置](https://docs.gitlab.com/runner/configuration/)
[在容器中运行GitLab Runner](https://docs.gitlab.com/runner/install/docker.html#docker-image-installation-and-configuration)

# aspnet项目dockerfile的编写
对于项目我先不讲那么多，有时间补充


检测数据库机制？

[dotnet-docker例子](https://github.com/dotnet/dotnet-docker/tree/master/samples)
关于docker文档的一些链接
[github上的docker](https://github.com/docker/)
[github上的docker-compose](https://github.com/docker/compose)

[docker文档](https://docs.docker.com/)
[dockerfile文件编写文档](https://docs.docker.com/engine/reference/builder/)
[docker-compose文档](https://docs.docker.com/compose/)
[docker-compose命令行文档](https://docs.docker.com/compose/reference/overview/)
[docker-compose文件编写文档](https://docs.docker.com/compose/compose-file/)
# 测试项目gitlab-ci.yml编写

```



```






[GitLab CI/CD文档](https://docs.gitlab.com/ce/ci/README.html)
[.gitlab-ci.yml配置](https://docs.gitlab.com/ce/ci/yaml/README.html)
[GitLab CI/CD 预定义变量 ](https://docs.gitlab.com/ce/ci/variables/README.html)
[gitlab-CI关于docker集成](https://docs.gitlab.com/ce/ci/docker/using_docker_images.html)


# 关于部署中的一些说明，或者说问题
## Centos上怎么验证证书是否导入成功
我也不懂，希望有知道的指教
## gitlab的git仓库无法上传
因为在docker中我把gitlab镜像的22端口转到了主机1022端口上，有四个解决方案；
1.在gitlab中设置ssh端口到1022上，系统会把，链接变成 gitlab.luna.cn:1022/用户名/项目名称XXXX.git。
2.你可以什么都不用改，直接使用这样的链接 gitlab.luna.cn:1022/用户名/项目名称XXXX.git 来上传项目。
3.使用scoretree软件，在.ssh目中添加一个config，原理就是改变ssh的端口，这样做我发现vs的git插件不能用。
```
hostName gitlab.luna.cn
port 1022
```
4.网上看到的，可以在centos上设置docker和docker镜像共用22端口，我没有尝试。
## Centos安装python3为系统默认之后,不兼容的问题
目前我发现的有两个firewall，yum。
解决方法是把对python的引用指定python2.7.5 。
## 关于windows的hosts之后重启
这也是一个坑，我遇到的是不重启，ping的通但是网页打不开。但是也不一定，你也可以拼一下人品。
## 虚拟机Centos联网问题
Centos安装过程中要开启网络，不然后续要自己开启网络，比较麻烦。
## Windows与Centos的通讯
我使用的是Xshell，Xftp。
## 关于使用CA证书
关于CA证书，只是在docker仓库的时候用到了，按道理来说也不是必须的，但是gitlab集成docker仓库的时候就已经集成了CA证书，并且推荐配置CA证书，如果不配置，感觉会更复杂也没必要，所以还是配置上了。
## 自签名证书的坑
自签名证书证书，是需要在你的本机和服务器都配置的。
## 关于使用域名
使用了CA证书，必须要在ip和域名中选择一个绑定，想到域名可能更通用，所以使用了域名。
## 使用了gitlab镜像
使用gitlab镜像更容易安装部署gitlab。
## 关于gitlab-runner部署在gitlab服务器
确实，这不应该这样做，官方也不推荐这样做，但是资源有限。下一步会分离。




