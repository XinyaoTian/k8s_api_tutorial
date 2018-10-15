# Kubernetes RESTful Api 研究成果 & 说明文档

利用k8s的apiserver实现一系列pod常用操作。
* 发起HTTP请求的工具为curl。若用nodejs改写，请区分GET、POST等请求。

### Part1 准备工作
首先，需要确保k8s集群的正常工作。可以登陆master节点，使用kubectl命令确认。

	kubectl get nodes

若至少包括两个节点，分别为一主一从；且所有节点的status全部为Ready，则表明集群处于良好的运行状态。

在确保集群处于正常运作状态后，使用ssh登陆master节点，并使用如下命令启动k8s的api-server的代理，并输入相应参数的特定值。

	 # 参数解析:
	 # --address='0.0.0.0' 监听IP为全网IP，在生产中可以改为发送消息的节点IP
	 # --port=8001 启动的代理端口为8001
	 # --accept-hosts='^*$' 表示所有来源都准入
	 # 注意，本条命令未持久化启动。实际生产请持久化启动。
	 kubectl proxy --address='0.0.0.0' --port=8001 --accept-hosts='^*$'

* ps :关于这条命令的详解，具体请参考如下链接及k8s英文官方文档: https://blog.csdn.net/kozazyh/article/details/79421666

至此，k8s的api-server已经启动完毕。可以在master节点使用如下命令确认api-server的开启情况。

	# 将以下URL中的localhost改为IP亦可进行更为可靠的测试
	# 查看当前k8s集群启动的所有pods的情况
	curl localhost:8001/api/v1/pods
	
	# 列出指定节点内物理资源的统计信息
	curl localhost:8001/api/v1/stats
	
	# 列出指定节点的概要信息
	curl localhost:8001/api/v1/spec


### Part2 利用api操作Pods
保证准备工作完成的情况下，可以在任何一台联网的主机上向上文指定的IP+port发起HTTP请求，来查看k8s集群的各种信息情况。

由于k8s的Api是基于REST的设计思想，因此，不同种类的HTTP请求也就对应了不同的操作。比较常用的对应关系是：
* GET（SELECT）：从服务器取出资源（一项或多项）。GET请求对应k8s api的获取信息功能。因此，如果是获取信息的命令都要使用GET方式发起HTTP请求。
* POST（CREATE）：在服务器新建一个资源。POST请求对应k8s api的创建功能。因此，需要创建Pods、ReplicaSet或者service的时候请使用这种方式发起请求。
* PUT（UPDATE）：在服务器更新资源（客户端提供改变后的完整资源）。对应更新nodes或Pods的状态、ReplicaSet的自动备份数量等等。
* PATCH（UPDATE）：在服务器更新资源（客户端提供改变的属性）。
* DELETE（DELETE）：从服务器删除资源。在稀牛学院的学员使用完毕环境后，可以使用这种方式将Pod删除，释放资源。

具体关于RESTful Api的设计思想，详见阮老师的博客:http://www.ruanyifeng.com/blog/2014/05/restful_api.html

	# 默认Get方式对相应master节点的api-server发起请求
	curl 59.110.220.63:8001/api/v1/namespaces/default/
	
	# 列出所有k8s集群中的pods
	curl 59.110.220.63:8001/api/v1/pods
	
	# 创建一个nginx pods (将这条命令最后的参数改为tensenflow就可以启动我们的GPU环境了。当然tensenflow的启动命令要写对，包括镜像的版本等。)
	curl http://59.110.220.63:8001/api/v1/namespaces/default/pods    -XPOST -H'Content-Type: application/json' -d@nginx-pod.json
	# 创建完毕后可以在master节点使用 kubectl get pods 查看
	
	# 查看k8s集群的所有service
	curl 59.110.220.63:8001/api/v1/services
	
	# 删除Pods
	curl 59.110.220.63:8001/api/v1/namespaces/default/pods/nginx -XDELETE

以上，便是利用HTTP请求对k8s中pods进行的基本操作。利用上述的这些命令，即可基本实现上课功能的需求。这些命令的出处和更详细的解析，见：https://github.com/kubernetes/kubernetes/issues/17404

### Part3 业务流程的思路
根据目前稀牛学院与网易云课堂的合作，可以发现目前k8s集群的主要应用还是在于教学。故本人理解的业务流程如下。如无出入，实际的编码和测试也可以按照如下流程进行：
* 由前端页面的学生发生点击行为，点击“启动实验环境”按钮
* 网站服务向k8s的api-server发出一个Post属性的HTTP请求，创建一个Gpu环境的Pod
* 在网站服务中向k8s发起Get属性的HTTP请求向，结合正则表达式，将刚创建好的Pod的相应IP地址、端口号提取出来。
* 根据提取出来的IP和端口号，将学员的当前页面实行跳转，跳转到GPU环境的URL，学员即可开始进行实验
* 在学员单击“结束实验”或实验时间结束，由网站服务向k8s发起DELETE的HTTP请求，delete相应的Pod。实验结束。至此，整个实验流程结束。


