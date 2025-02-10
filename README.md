# 项目部署

```sql
#个人vx：WYF12129898		-- 欢迎交流

遇到问题可参考文章提到的网址。
```

## 1. k8s集群搭建

链接：[多个公网服务器搭建k8s集群_ip adress,comma separated-CSDN博客](https://blog.csdn.net/weixin_43988498/article/details/122639595?spm=1001.2014.3001.5506)

```
	我使用的是3个云服务器搭建，使用虚拟机是一样的。上面是我参考的博客。（建议使用内网搭建，也就是网段一致。公网搭建比较复杂不建议）。
	搭建过程有问题欢迎交流。
```

## 2. 镜像仓库搭建

网站搜索：阿里云免费镜像仓库

（或直接进入阿里云找到产品）链接：[容器镜像服务](https://cr.console.aliyun.com/cn-shenzhen/instance/repositories)

```shell
# 推荐使用阿里云镜像仓库

# 这里要注意，使用阿里云服务器推送可使用私网，最好使用公网。
内网地址：***out***
公网地址：***int***
```

## 3. 博客项目

### 3.1 拉取博客代码：

```shell
# 1.第一部肯定是git别人的代码啦
git pull https://github.com/mao888/bluebell-plus
（该项目起源于七米老师（李文周），本人学习拉取的b友的代码。感谢七米老师和胡毛毛开源）

（直接把bluebell-backend文件单独拿出来就行，作者已经弄成了前后端一体了。）

# 2.拿到代码后打开修改conf/config.yaml文件，添加自己的云服务上的mysql和redis服务地址和端口。init.sql文件里创建mysql表。

# 3.加载依赖。建议删除先go.sum和go.mod（提一嘴我的go环境是go1.23.2）
go mod init bluebell_backend
go mod tidy
(我记得会现过问题，使用gpt解决)

# 4.访问博客，验证是否代码有效。(端口可以自己设置)
127.0.0.1:8081 
```

### 3.2 构建镜像并推送至自己的镜像仓库

```shell
# 直接复制
docker build . -t bluebell_app
```

```shell
# 推送镜像到阿里云长裤

# 1.登录
docker login --username=****

# 2.给本地镜像打标签
docker tag bluebell_app:latest crpi-ykp5mqwgolf9e106.cn-shenzhen.personal.cr.aliyuncs.com/wuyifeng/feng-repository:latest

# 3.推送至远程仓库
docker push crpi-ykp5mqwgolf9e106.cn-shenzhen.personal.cr.aliyuncs.com/wuyifeng/feng-repository:latest
```

### 3.3 k8s集群构建deployment无状态服务

```shell
# 创建deploy资源文件：bluebell_app.yaml
vim bluebell_app.yaml
```

```shell
#复制进去（记得更改镜像地址：这里使用service做服务发现和负载均衡）
---
apiVersion: v1
kind: Service
metadata:
  name: bluebell-service
  labels:
    app: bluebell
spec:
  type: NodePort  # 使用NodePort
  ports:
  - port: 81  # 服务暴露的端口
    targetPort: 8081  # 映射到容器的端口
  selector:
    app: bluebell  # 选择与标签匹配的 Pod

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bluebell-app
  labels:
    app: bluebell
spec:
  replicas: 3  # 指定副本数量
  selector:
    matchLabels:
      app: bluebell
  template:
    metadata:
      labels:
        app: bluebell
    spec:
      containers:
      - name: bluebell-app
        image: （#镜像地址：标签）  
        ports:
        - containerPort: 8081  # 指定容器暴露的端口 
```

```shell
# 部署deploy
kubectl apply -f bluebell_app.yaml

# 访问服务
  ！#1.找具体的nodePort,不是service的端口，很重要！（nodeport分配的是30000+的端口）
kubectl get svc bluebell-service -o yaml
	# 2. <k8s节点ip>:nodePort
http://ip:nodePort端口

（能成功访问到博客则成功）
```

​	根据以上内容我们就完成了k8s部署博客了，同时这是一个高可用（有三个deploy副本），负债均衡（配置了service流量分发给三个deploy副本）的博客服务。

------到这里，没必要就不需要往下面看了。

## 4. Prometheus

​	接下来我打算用Prometheus+Grafana来监控这个web服务。

### 4.1 下载promethues

安装peomethues链接：可以先看promethues部分

[二进制安装Prometheus+grafana_grafana 二进制安装-CSDN博客](https://blog.csdn.net/2302_78152953/article/details/134546904?spm=1001.2014.3001.5506)

（我是用的是直接安装。网上很多容器安装，我建议不太懂的话就跟着我就行）

### 4.2 代码中添加promethues服务

也就是为程序添加一个go语言的exporter（promethues的组件）。

```go
// 在博客代码中添加Prometheus路由。也就是routers.go文件下
r.GET("/metrics", gin.WrapH(promhttp.Handler()))

//同时在代码开头记得import包。
"github.com/prometheus/client_golang/prometheus/promhttp"

go mod tidy

// 运行试试
go run main.go
// 如果可以运行，则去浏览器。有一堆输出就是说明成功了。
http://服务器ip:8081/metrics


// 最后一步你懂的，打包成镜像。推送到阿里云仓库。(标签设置成latest哦，挤掉之前的镜像)
...
```

### 4.3 部署博客服务

```shell
# 最好先删了以前的deploy和svc

# 用之前的bluebell_app.yaml
kubectl apply -f bluebell_app.yaml
```

### 4.4 让promethues发现服务

找到svc的nodeport端口

```shell
# 修改prometheus/prometheus.yml配置文件
# 追加这个job，方括号里面的内容就行。（也就是告诉prometheus让他监控bluebell，但是我们是配置了service的，所以只要监控service就行啦，直接找到service的NodePort）

{
# bluebell-service监控
  - job_name: 'bluebell_app'
    static_configs:
      - targets: ['master节点ip:服务nodePort端口'] 
    scheme: http  # 指定协议为 HTTP
    tls_config:
      insecure_skip_verify: false  # 不跳过 TLS 证书验证
}

# 重启Prometheus
sudo systemctl restart prometheus

# 访问Prometheus（确定）
k8s公网ip:服务nodePort端口

# 查看一些指标如：协程数量
go_goroutines
```

## 5. Grafana 可视化 promethues

Grafana 可视化 promethues。安装Granafa链接：

[二进制安装Prometheus+grafana_grafana 二进制安装-CSDN博客](https://blog.csdn.net/2302_78152953/article/details/134546904?spm=1001.2014.3001.5506)



需要自己去学习Grafana了。

推荐b站up：麦兜搞IT
【Grafana入门系列(1)——介绍】https://www.bilibili.com/video/BV1DA411s7L8?vd_source=42a25a7b4713a50ef22a53e2a03a904e

```shell
# 启动Grafana
sudo systemctl start grafana-server.service
# 创建一个dashboard监控自己想监控的指标。
```

## 6. 服务压测

```shell
# 使用压测工具对网站进行测试
# 1.go-wrk安装
go install github.com/adjust/go-wrk@latest
ls $GOPATH/bin	#检查是否安装


#2.常用参数
-H="User-Agent: go-wrk 0.1 bechmark\nContent-Type: text/html;": 由'\n'分隔的请求头
-c=100: 使用的最大连接数
-k=true: 是否禁用keep-alives
-i=false: if TLS security checks are disabled
-m="GET": HTTP请求方法
-n=1000: 请求总数
-t=2: 使用的线程数
-b="" HTTP请求体
-s="" 如果指定，它将计算响应中包含搜索到的字符串s的频率

# 3.案例---格式：go-wrk [flags] url
go-wrk -t=2 -c=100 -n=1000 "http://ip/"
```

## 7. AlertManage报警

参考博客：

[Prometheus 监控报警系统 AlertManager 之邮件告警_prometheus邮件告警配置-CSDN博客](https://blog.csdn.net/aixiaoyang168/article/details/98474494?spm=1001.2014.3001.5506)

```shell
# 报警
# 使用AlertManager
https://blog.csdn.net/aixiaoyang168/article/details/98474494
```

```shell
# 设置网站服务器自动扩缩容
kubectl edit deploy bluebell-app

在resource字段：
        resources:
          limits:#期待最大使用资源
            cpu: 200m
            memory: 128Mi
          requests:#扩容所需资源
            cpu: 100m
            memory: 128Mi
      
# 创建HPA，设置超过20%期待cpu就扩容，最小副本数为2个，最大10个
kubectl autoscale deploy bluebell-app --cpu-percent=20 --min=2 --max=10

# 在使用服务压测
go-wrk -t=2 -c=100 -n=10000 "http://ip/"

# 在Grafana上观察每个node上的pod数变化
# 扩容时会快速发生变化，缩容的话没有那么快速
```

### 1000000. 更多玩法一起交流

```shell
# 使用istio服务网格代替service和ingress
```

```shell
# 使用数据卷存储配置信息
# 进阶：使用etcd存储配置信息
```

```shell
# 搭建远程的grpc大模型服务，集成一个小助手功能
后端代码已经实现，但不想写前端哈哈
# 进阶：prompt微调大模型。
```

```shell
# CICD
构建完整的devops工具链
```

```shell
# 自定义指标上传promethues
```

