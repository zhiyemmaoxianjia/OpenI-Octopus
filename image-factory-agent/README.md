# IMAGE FACTORY 

> Agents of S.H.I.E.L.D

`go >= 1.12`


镜像工厂，该服务可以帮助平台用户直接将容器打包成docker镜像。

分为 两个部分：

* agent （image-factory-agent）在目标物理节点上运行的代理，负责执行 docker commit  和 docker push 

* shield (image-factory-shield) 接收镜像打包请求，并将任务分发给对应物理节点上的 agent

> commit请求只需要同 shield 交互即可

该项目是 image-factory-agent


## 快速部署

1. 该服务以`Daemonset`的方式部署, 在部署前需要在根目录下的部署文件`deploy.yaml`里面指明`image-factory-shield`服务的访问地址。


2. 部署到k8s集群

`kubectl apply -f ./deploy.yaml`

## 源码编译部署

根目录下已经给出了`Dockerfile`文件，可以直接根据该文件将项目源码编译成docker镜像

修改`deploy.yaml`里的镜像为新编译的镜像，重新发布即可

## 一. Agent 的 API 说明


#### 1.1  查询镜像大小

>查询一个容器打成 docker 镜像后的大小

GET `/v1/commit/size?container=xxx`

参数(query):

* container  容器的id 或者容器的名字

返回（json格式）

```js
{
    "success":bool,// 操作成功或者失败
    "size":number ,// 镜像的大小，单位 kb
    "msg":"",// 附加消息，如果success 为 false,则是对应的失败信息
}
```

#### 1.2 同步commit 

> 直到镜像打包成功 或者 失败，该API 才返回， 如果镜像打包时间过长，请使用另一个异步打包的接口 

POST `/v1/commit/sync`

参数(JSON)：

```js
{
    "author":"",// 作者，镜像作者
    "container":"",// 目标容器的id 或 目标容器的名字
    "image":"",// 新镜像的名字
    "note":"",// 备注,docker commit 的 --message 选项
    "transaction": "",// 该次打包镜像的事务id 
    "hub_user":"",// 镜像仓库用户名，可选参数
    "hub_pwd":"",//镜像仓库密码,可选参数
    "hub_addr":"",// 镜像仓库登录地址，比如 registry.cn-hangzhou.aliyuncs.com ， 可选参数
}
```

返回(json 格式): 


```js
{
    "success": true/false ,
    "msg":""// 
}
```

#### 1.3 异步commit 

> 接收到提交请求，立即返回，镜像工场会根据打包进度更新该次任务的状态

POST `/v1/commit/async`

参数(JSON)：

```js
{
    "author":"",// 作者，镜像作者
    "container":"",// 目标容器的id 或 目标容器的名字
    "image":"",// 新镜像的名字
    "note":"",// 备注,docker commit 的 --message 选项
    "transaction": "",// 该次打包镜像的事务id 
    "hub_user":"",// 镜像仓库用户名，可选参数
    "hub_pwd":"",//镜像仓库密码,可选参数
    "hub_addr":"",// 镜像仓库登录地址，比如 registry.cn-hangzhou.aliyuncs.com ， 可选参数
}
```
返回(json 格式): 


```js
{
    "success": true/false ,
    "msg":""// 
}
```

## 二. commit 任务状态说明

* NOT_FOUND  该任务没有找到
* SUCCEEDED 镜像commit 成功
* FAILED  commit 失败
* INITIALIZED commit 任务已经初始化
* PROCESSING  commit 任务正在处理中
* COMMITTING  正在执行 docker commit 操作
* PUSHING 正在上传镜像


## 三. 部署说明


#### 3.1 注意事项

第一个： agent 在每个物理节点都运行一份实例，agent 负责 执行 `docker commit`，`docker push` 命令。

因此首先agent应该以守护进程的方式部署在物理机上，且需要有执行 `docker` 命令的权限！


第二个： docker 的客户端采用的是https，在与docker 镜像仓库交互时，如果镜像仓库不是采用的https 的方式，那么会报`http: server gave HTTP response to HTTPS client`错误 ， 解决方法是在 docker 的 `daemon.json` 配置文件中加上 `{ "insecure-registries":["192.168.1.100:5000"] }`(假设`192.168.1.100:5000`就是你的镜像仓库地址), 然后重启docker

请在部署 agent 时 确认一下 这个点。


第三个: 在 `commit` 的 `api` 中支持用户指定镜像仓库(即`hub_addr`参数) ，这是可选的, 如果对应的物理机上 docker 有与目标仓库交互的权限，那么`hub_user`,`hub_pwd`,`hub_addr` 可以不填。所以在部署的时候直接 运行  `docker login` 命令 登录目标仓库， 那么之后就不用在 调用 `commit`的api时 填 仓库的用户名密码了


#### 3.2  agent 

agent 运行在集群的物理机上(各个物理机上都需要运行)，需要有执行物理机上 `docker` 命令的权限

配置：

配置文件的形式(json):

```jsxinx
{
    "shield_address":"",// shield 服务的地址 ，如 http://192.168.202.20:9001
    "agent_address":"",// 该agent的访问地址，可以不配置，不配置将使用物理机ip拼接地址 http://ip:port
    "port":"",// 服务 监听的端口，默认是 9002
}
```

以配置文件的形式启动： `agent --config config_file_path`

或者设置环境变量：

* SHIELD_ADDRESS
* AGENT_ADDRESS
* PORT

#### 3.3 shield

shield 只需运行一个实例即可，理论上运行在哪里都可以

配置：

配置文件的形式(json):

```js
{
    "max_commit_exist_time":number,// commit 任务终止后，commit 记录在shield 里面的最大存活时间，默认3小时
    "port":"",// 服务 监听的端口，默认是 9001
}
```
以配置文件的形式启动： `shield --config config_file_path`

或者设置环境变量：

* MAX_COMMIT_EXIST_TIME
* POR

