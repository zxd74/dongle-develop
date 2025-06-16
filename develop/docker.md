# 基础
## 安装
### linux
1. 卸载旧版
```shell
sudo yum remove -y docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine
```
2. 安装仓库源
```shell
sudo yum install -y yum-utils

sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# https://download.docker.com/linux/centos/docker-ce.repo 官网仓库，国内访问可能失败
# http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo 阿里云docker源
```
3.  安装docker相关包
```shell
# 安装最新版本
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

## 指定版本
# yum list docker-ce --showduplicates | sort -r
# sudo yum install -y docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io docker-buildx-plugin docker-compose-plugin
```
4. 使用docker
```shell
# 启动docker服务
sudo systemctl start docker

# 运行docker容器，有默认hello-world镜像
sudo docker run hello-world

# 开机启动
systemctl enable docker
```

### windows
1. 下载`Docker Desktop Installer.exe`执行安装即可
2. 可更改安装目录`"Docker Desktop Installer.exe" install --installation-dir="DIR PATH"`

# 私有仓库
1. 拉取registry镜像并启动(服务端)
   1. 可考虑补充镜像源，方便拉取其它镜像
```bash
docker pull registry
[root@localhost] docker run -p 5000:5000 \
--restart=always \
--name registry \
-v /data/docker/registry:/var/lib/registry \
registry
```
2. 客户端绑定仓库地址 `/etc/docker/daemon.json`
   1. 域名解析`/etc/hosts`
   2. 可以是ip，也可以是域名
```json
{
  "insecure-registries":["docker.dongle.com:5000"]  
}
```
1. 刷新docker配置（docker 客户端）
```bash
# 重启docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```
1. 从仓库推送拉取镜像
   1. 注意1：若仓库需要认证，请先登录，参考下方认证仓库
   2. 注意2：需要将镜像名前增加私有仓库信息，若不修改，则直接上传到docker.io仓库
```bash
docker tag centos docker.dongle.com:5000/centos:v1 # 修改镜像标签
docker push docker.dongle.com:5000/centos:v1  # 自动寻找私有仓库docker.dongle.com:5000，若有认证，需先认证
docker pull docker.dongle.com:5000/centos:v1  # 拉取时也会自动根据前缀判断仓库
```
## 查看仓库
**注意**：
* 在V2.x版本中无法使用`docker search`搜索私有仓库
* 使用`http(s)://<your-server-host:port>/v2/_catalog`来代替docker search查看镜像列表
* 想要拉取或推送镜像到私有仓库，需要时刻关联私有仓库地址
  * 镜像名：`<仓库地址>/<镜像名>:<版本号>`

## 认证仓库
1. 拉去仓库步骤同上
2. 使用htpasswd创建加密认证
```bash
# yum install -y httpd-tools  # 如果未安装htpasswd工具，需先安装
[root@localhost] htpasswd -Bbn admin admin > /data/docker/auth/htpasswd
```
3. 启动仓库，携带认证命令
* `REGISTRY_AUTH=htpasswd` # 以 htpasswd 的方式认证
* `REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm` # 注册认证
* `REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd` # 认证的用户密码
```bash
[root@localhost] docker run -p 5000:5000 \
--restart=always \
--name registry \
-v /data/docker/registry:/var/lib/registry \
-v /data/docker/auth/:/auth/ \
-e "REGISTRY_AUTH=htpasswd" \
-e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
-e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
-d registry
```
4. 认证
```bash
[root@localhost] docker login ip:port # 登录
Username: admin
Password: admin
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
[root@localhost] docker logout ip:port # 退出
```

# 镜像管理
* 镜像源：镜像库的代理，只是加速访问docker hub，并不实际同步镜像
* 镜像仓库：镜像的存放库，有docker hub(官方)、gcr(google)、quay(Red Hat)、ghcr(github)，ecr等
* 在拉取或推送时可以指定镜像仓库地址，否则默认docker hub官方地址(docker.io)

## 更换镜像源
* 修改`/etc/docker/daemon.json`文件，配置**registry-mirrors**内容
  * 中国区官方镜像： https://registry.docker-cn.com【无用】
  * 科大源： https://docker.mirrors.ustc.edu.cn
  * 清华源：https://docker.mirrors.tuna.tsinghua.edu.cn
  * 阿里源： https://cr.console.aliyun.com 
    * 已不再提供公共地址，可进入阿里云创建个人镜像加速器【容器镜像服务ACR】
    * https://ustc-edu-cn.mirror.aliyuncs.com/
  * 腾讯源： https://mirror.ccs.tencentyun.com
  * 网易源： http://hub-mirror.c.163.com
  * 百度源：https://mirror.baidubce.com
  * 道客：https://docker.m.daocloud.io
  * 其它：
    * https://noohub.ru
    * https://huecker.io
    * https://dockerhub.timeweb.cloud
    * https://docker.unsee.tech
  * 镜像仓库：
    * docker.io：Docker Hub 官方镜像仓库，也是 Docker 默认的仓库
    * gcr.io、k8s.gcr.io：谷歌镜像仓库
    * quay.io：Red Hat 镜像仓库
    * ghcr.io：GitHub 镜像仓库
```bash
vi /etc/docker/daemon.json
{
    "registry-mirrors": ["https://docker.unsee.tech"]
}

# 重启docker
sudo systemctl daemon-reload # 重点!!!!
sudo systemctl restart docker
```

### 查看实际生效的镜像源
* 方式一：监听pull事件`docker events --filter 'event=pull' --format '{{json .}}'`,但依赖docker服务的输出内容
* 方式二：`docker info | grep Registry`，同样依赖docker的输出内容
* 方式三：docker debug模式，开启docker debug模式，通过日志查看docker命令执行过程
```bash
vi /etc/docker/daemon.json
"debug":true

# 重启docker
sudo systemctl restart docker

# 查看服务日志，以下路径找存在的文件查看
/var/log/docker.log
/var/log/syslog
/var/log/messages
```

## Dockerfile配置构建
1. 创建配置文件java8.Dockerfile
```shell
# 基础镜像来源，必需，可从hub.docker.com查看
FROM docker.io/centos:7         
# 镜像作者相关信息，不可再首行，否则报错
MAINTAINER  dongle "dongle@xxx.com"  
# 工作目录，没有会提前创建
WORKDIR /usr/local/src/java/    
# 运行命令，参与镜像构建，如下载必需软件或依赖
RUN yum install wget -y 
# 复制文件到镜像中，也可以使用ADD
COPY jdk1.8.0_311/* /usr/local/src/java/    
# 设置环境变量
ENV JAVA_HOME /usr/local/src/java
ENV PATH $PATH:$JAVA_HOME/bin
# 暴露端口，运行时可以将该容器端口与外部关联
EXPOSE 3306                     
# CMD是启动容器时执行的任务，如果是项目建议使用启动命令，这样启动容器是直接运行的，否则会中断
# ENTRYPOINT 也是启动命令，但不受RUN影响，CMD受RUN影响
CMD ["java","-version"]
```
2. 构建镜像
   ```shell
   docker build -t dongle/java:1.8 . -f java8.Dockerfile  # -t(--tag)定义镜像名和标签，-f 指定Dockerfile文件
   ```

## 镜像导入导出
* 容器生成镜像：不保存镜像层级，配置信息需调整，无法直接启动
```shell
docker save -O image-file iamge-name # 保存镜像为文件（默认生成tar文件）
docker load -i image-file  # 从文件中加载镜像
# docker save image-name |gzip > image-name.tar.gz # 保存镜像为文件
# gunzip  -c *.tar.gz | docker load
```
* 镜像导入导出：直接保存镜像
```bash
docker save -o image-file iamge-name
docker load -i image-file
```
## 查询镜像
```bash
docker search image-name # 根据名称搜索镜像

# 搜索指定镜像分支信息
# 如果接口v1不好用，换成v2
curl https://registry.hub.docker.com/v1/repositories/image-name/tags | python -m json.tool | grep 2.4
```

# 容器管理
## 生成新镜像
* **更新**方式：`docker commit <contain-id|contain-name> <new-image-name>:<tag>`
* **存储**方式：
```shell
docker export -o <filename.tar> <contain-id|contain-name>
docker import <filename.tar> <new-image-name>:<tag> # 当不配置image信息时，默认为<none>镜像
```

## 文件管理
1. 目录挂载
   1. 匿名挂载 `-v containerpath `
   2. 具名挂载：卷存在名称，需要先创建卷，再于容器绑定 `-v volumename:containerpath`
   3. 指定路径挂载：`-v hostpath:containerpath`
2. 文件传送`docker cp`

## 容器通信
* 通过容器ip访问：每次从其容器ID会变化
  * 通过宿主机的ip:port访问
* 通过`--link`访问(官方不推荐)：
  * 需要确保连接容器必需提前运行，并且只能单向通信，不能双向通信
  * 会自动在`/etc/hosts`中加入连接容器IP
* 通过自定义桥接网络访问
  * 需要先创建桥接网络
  * 然后个容器加入桥接网络中，并对自认网络定义别名

## 运行容器配置更改 
1. 停止容器
2. 找到容器目录`docker info |grep Root`
3. 修改`config.v2.json`和`hostconfig.json`配置文件
4. 重启docker服务(**必需**)和容器：否则配置会被历史覆盖

### 新增挂载
```text
# config.v2.json
"MountPoints": {
   ....,
   "/data/mysql":{"Source":"/data/docker/mysql/","Destination":"/data/mysql","RW":true,"Name":"","Driver":"","Type":"bind","Propagation":"rprivate","Spec":{"Type":"bind","Source":"/data/docker/mysql/","Target":"/data/mysql"},"SkipMountpointCreation":false}
}
```
```text
# hostconfig.json
"Binds": [
   ...,
   "/data/docker/mysql:/data/mysql"
]
```
## 命令执行
1. 容器内执行命令：`docker exec -it containerid command`
   1. `/bin/bash`
   2. `/bin/sh`
2. 容器外执行命令:`docker exec -it containerid bash -c "command"`
