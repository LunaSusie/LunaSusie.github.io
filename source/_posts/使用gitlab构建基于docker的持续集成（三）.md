---
title: 使用gitlab构建基于docker的持续集成（三）
tags: [gitlab, docker, aspnetcore]
categories: 持续集成
---
----------

# 构建发布思路：
1.构建单元测试镜像，上传镜像仓库。
2.拉取测试镜像，执行单元测试。
3.构建发布镜像，上传镜像仓库。
4.使用docker-compose编排发布镜像和数据库镜像，打包发布，我把项目也发布到了gitlab.luna.cn服务器
# aspnetcore 下的dockerfile编写
![我的aspnetcore目录结构](https://www.github.com/loveshullf/Notes/raw/img/小书匠/使用gitlab构建基于docker的持续集成（三）-2018-4/1524192992754.jpg)
dockerfile的编写，参考了dotnet官方的例子，是用分层配置，这样只要写一个dockerfile就可以了。
```dockerfile
FROM microsoft/dotnet:2.0-sdk AS build
WORKDIR /app

# copy csproj and restore as distinct layers
COPY *.sln .
COPY User.Api/*.csproj ./User.Api/
COPY User.Api.UnitTest/*.csproj ./User.Api.UnitTest/
RUN dotnet restore

# copy everything else and build app
COPY . .
WORKDIR /app//User.Api
RUN dotnet build


FROM build AS testrunner
WORKDIR /app/User.Api.UnitTest
ENTRYPOINT ["dotnet", "test", "--logger:trx"]


FROM build AS publish
WORKDIR /app/User.Api
RUN dotnet publish -c Release -o out

FROM microsoft/aspnetcore:2.0 AS runtime
WORKDIR /app
ENV ASPNETCORE_URLS http://+:80
EXPOSE 80
COPY --from=publish /app/User.Api/out ./
ENTRYPOINT ["dotnet", "User.Api.dll"]

```
# 发布docker-compose
```yml
version: '3'

services:
  user.api:
    image: gitlab.luna.cn/lunaselene/findbook.first/master:release
    container_name: user.api
    ports:
      - '8080:80'
    links:
      - mysql-bate
  mysql-bate:
    image: 'mysql/mysql-server:5.7'
    restart: always
    container_name: mysql-bate
    volumes:
      - ./mysql/data:/var/lib/mysql
      - ./mysql/config/my.cnf:/etc/my.cnf
      - ./mysql/init:/docker-entrypoint-initdb.d/
    ports:
       - '3306'
```
# gitlab-ci.yml的编写
![gitlab环境变量的设定](https://www.github.com/loveshullf/Notes/raw/img/小书匠/使用gitlab构建基于docker的持续集成（三）-2018-4/1524212842718.jpg)
安装构建思路，对应的yml文件如下
```yml
image: tico/docker
variables:
  CONTAINER_TEST_IMAGE: $CI_REGISTRY_IMAGE/$CI_COMMIT_REF_NAME:test
  CONTAINER_RELEASE_IMAGE: $CI_REGISTRY_IMAGE/$CI_COMMIT_REF_NAME:release
  DOCKER_DRIVER: overlay2

before_script:
  - docker login -u gitlab-ci-token -p $CI_JOB_TOKEN $CI_REGISTRY
  #下面是登录远程服务器的设置sshkey操作
   # 其中私有变量$SSH_PRIVATE_KEY 是远程服务器的ssh私匙
   # 我使用的部署gitlab的服务器所以url是gitlab.luna.cn
  - eval $(ssh-agent -s)
  - ssh-add <(echo "$SSH_PRIVATE_KEY")
  - mkdir -p ~/.ssh
  - chmod 700 ~/.ssh
  - ssh-keyscan gitlab.luna.cn > ~/.ssh/known_hosts
  - chmod 644 ~/.ssh/known_hosts
  - ssh -T root@gitlab.luna.cn
stages:
  - build
  - test
  - release
  - deploy

build:
  stage: build
  script:
   - docker build --target testrunner --pull -t $CONTAINER_TEST_IMAGE .
   - docker push $CONTAINER_TEST_IMAGE

test:
  stage: test
  script:
    - docker pull $CONTAINER_TEST_IMAGE
    - docker run $CONTAINER_TEST_IMAGE
    #clert container
    - docker rm $(docker ps -q -l)

release:
  stage: release
  script:
    - docker build --target runtime  --pull -t $CONTAINER_RELEASE_IMAGE .
    - docker push $CONTAINER_RELEASE_IMAGE
  only:
    - master

deploy:
  stage: deploy
  script:
    - scp -r mysql root@gitlab.luna.cn:/root/deploy
    - scp docker-compose.yml root@gitlab.luna.cn:/root/deploy
    - ssh root@gitlab.luna.cn "docker-compose --f /root/deploy/docker-compose.yml up -d --force-recreate "
    - echo "Deploy to staging server"
  only:
    - master
```
## 一些参数的解释
### image 
代表要使用的基础docker镜像，一切构建都是在这个镜像下完成，也就是一个大环境，后面的命令必须要能在这个环境下能执行。
我指定这个是一个自带docker环境的镜像，它的信息可以到docker hub上查看。
### variables 
自定义变量，后面跟的是自定义的变量，$CI_REGISTRY_IMAGE等变量是gitlab-runner自带的变量。
### before_script
前置脚本，也就是每个stages运行之前都会执行的命令。
### stages
定义CI每个步骤，这个步骤是一个执行完才执行下一个的，上一个失败，全部失败。
## 关于远程发布ssh的补充

由于要全自动化发布，必须实现runner所在服务器和远程发布主机的无密登录。
ssh无密码登录，分为单向和双向，这里用单向的就可以了。
因为使用的是docker in docker 的方式运行runner，所以秘钥通过gitlab的环境变量进行传递。

1.远程主机生成ssh秘钥。
```bash
ssh-keygen -t rsa 
```
命令输完一直回车到完成为止。
2.远程主机中把公钥设置到authorized_keys。
```bash
cd /root/.ssh
cat id_rsa.pub > authorized_keys
```
3.把私钥设置成环境变量。
![私钥设置成环境变量](https://www.github.com/loveshullf/Notes/raw/img/小书匠/使用gitlab构建基于docker的持续集成（三）-2018-4/1524230300100.jpg)
4.gitlab-ci.yml中给运行的容器设置runner私钥。
```yml
  #下面是登录远程服务器的设置sshkey操作
   # 其中私有变量$SSH_PRIVATE_KEY 是远程服务器的ssh私匙
   # 我使用的部署gitlab的服务器所以url是gitlab.luna.cn
  - eval $(ssh-agent -s)
  - ssh-add <(echo "$SSH_PRIVATE_KEY")
  - mkdir -p ~/.ssh
  - chmod 700 ~/.ssh
  - ssh-keyscan gitlab.luna.cn > ~/.ssh/known_hosts
  - chmod 644 ~/.ssh/known_hosts
  - ssh -T root@gitlab.luna.cn
```
这样远程服务器的连接就设置完成了。
# 提交测试

![build](https://www.github.com/loveshullf/Notes/raw/img/小书匠/使用gitlab构建基于docker的持续集成（三）-2018-4/1524263725631.jpg)

![test](https://www.github.com/loveshullf/Notes/raw/img/小书匠/使用gitlab构建基于docker的持续集成（三）-2018-4/1524263793401.jpg)

![release](https://www.github.com/loveshullf/Notes/raw/img/小书匠/使用gitlab构建基于docker的持续集成（三）-2018-4/1524263833964.jpg)

![deploy](https://www.github.com/loveshullf/Notes/raw/img/小书匠/使用gitlab构建基于docker的持续集成（三）-2018-4/1524263870594.jpg)

![全部通过](https://www.github.com/loveshullf/Notes/raw/img/小书匠/使用gitlab构建基于docker的持续集成（三）-2018-4/1524264252825.jpg)

![查看发布情况](https://www.github.com/loveshullf/Notes/raw/img/小书匠/使用gitlab构建基于docker的持续集成（三）-2018-4/1524264237045.jpg)

![本地访问查看](https://www.github.com/loveshullf/Notes/raw/img/小书匠/使用gitlab构建基于docker的持续集成（三）-2018-4/1524264290939.jpg)