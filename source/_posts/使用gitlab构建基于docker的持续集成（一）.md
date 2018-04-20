---
title: 使用gitlab构建基于docker的持续集成（一）
tags: [gitlab, docker, aspnetcore]
categories: 持续集成
date: 2018/4/20
updated: 2018/4/20
---
----------
# 开篇

关于CI，我的简单理解就是，项目的一键编译+测试+部署发布，然后可以重复利用这个过程，你只要一提交代码，项目就发布完成了。

而docker，可以给编译，测试，部署提供环境支撑，简单来说，就是比传统CI更方便，简化了CI过程中的环境问题。

本来是有想法是部署到公网，让大家试一下效果的，但是由于部署gitlab非常消耗，我的小服务器抗不住，只好把过程记录下来，给大家一个参考，也给自己做个备忘。

先贴几张效果图吧，有个整体印象。

![git项目界面](https://www.github.com/loveshullf/Notes/raw/img/小书匠/使用gitlab构建基于docker的CI-2018-4/1524012232084.jpg)

![docker仓库](https://www.github.com/loveshullf/Notes/raw/img/小书匠/使用gitlab构建基于docker的CI-2018-4/1524012665989.jpg)

![CI界面](https://www.github.com/loveshullf/Notes/raw/img/小书匠/使用gitlab构建基于docker的CI-2018-4/1524012721018.jpg)

本文大致是分三部分，基础准备，Centos主机上的配置，win主机的配置，最后发布测试。

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
* gitlab-runner：Centos7.3 （我把它部署到了gitlab服务器上了，规范的做法是应该分开部署）
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
## 虚拟机系统：安装Centos7.3
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
## Centos上docker安装
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

## Centos上docker-compose安装
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


# 补充
最后科普一个Centos开机启动docker的方法，网上找来的怎么做开机启动脚本
1.chmod +x /etc/rc.d/rc.local
2.vi  /etc/rc.d/rc.local 添加/root/script/autostart.sh
3.vi /root/script/autostart.sh 添加/bin/systemctl start docker