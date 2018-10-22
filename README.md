# K8s Rest API 研究成果&基本功能 说明文档

## 开发流程
* 在学员点击开启实验时，向K8s-apiserver发起两个Post请求，分别创建pod和service
* 使用Get请求循环查询pod和service的状态、及service的端口号。
* 待pod的状态变为running后，将学员的页面跳转至 主节点的IP+端口号/?token=...
*  待学员实验时间结束后，向K8s-apiserver发起两个Delete请求，按service和pod的顺序先后删除。

## K8s 实用 api 速查

首先，需要在K8s集群获取一串用于认证的token。这里省略这个步骤。
以下命令中的所有如下格式的命令，均替换为完整的token才能够执行。

	-H "Authorization: Bearer ey...Xg" 


### 查询Pods的基本信息
通过使用k8s-api接口，查询Pods的基本信息。返回一个json格式的长文件。

	curl -k -v -X GET -H "Authorization: Bearer ey...Xg" -H "Content-Type: application/json" https://59.110.220.63:6443/api/v1/namespaces/default/pods/

### 查询指定Pod的信息
在上一条命令中的最后，加入指定的Pod名，即可获得指定Pod的信息。

	curl -k -v -X GET -H "Authorization: Bearer ey..Xg" -H "Content-Type: application/json" https://59.110.220.63:6443/api/v1/namespaces/default/pods/pod-jupyter-233

### 创建指定的Pod
Pod中的各种信息由命令中的json所决定

	curl -k -v -X POST -H "Authorization: Bearer ey..Xg" -H 'Content-Type: application/json' --data '
	{
		"apiVersion": "v1",
		"kind": "Pod",
		"metadata": {
			"name": "pod-jupyter-233",
			"namespace": "default",
			"labels": {
				"name": "jupyter"
			}
		},
		"spec": {
			"restartPolicy": "Always",
			"containers": [
				{
					"name": "pod-jupyter-233",
					"image": "tensorflow/tensorflow:latest",
					"imagePullPolicy": "Never",
					"ports": [
						{
							"containerPort": 8888
						}
					],
					"command": [
						"/run_jupyter.sh"
					],
					"args": [
						"--allow-root",
						"--NotebookApp.token=abcdef"
					],
					"volumeMounts": [
						{
							"mountPath": "/notebooks",
							"name": "test-volume"
						}
					]
				}
			],
			"volumes": [
				{
					"name": "test-volume",
					"hostPath": {
						"path": "/mnt/userid",
						"type": "Directory"
					}
				}
			]
		}
	}
	'  https://59.110.220.63:6443/api/v1/namespaces/default/pods

### 创建Pod的相应Service
Service是K8s中，用于自动分配范围从30000至32767的端口号的一个服务类型。每一个Pod如果想获取一个公网IP和相应的port，都必须创建一个相应的Service。
* 注意! 在k8s中，外网访问Pod的IP地址可以为任意一台连入集群的主机IP地址。


	curl -k -v -X POST -H "Authorization: Bearer ey..Xg" -H 'Content-Type: application/json' --data '
	{
		"kind": "Service",
		"apiVersion": "v1",
		"metadata": {
			"name": "pod-jupyter-233",
			"namespace": "default",
			"labels": {
				"name": "jupyter"
			}
		},
		"spec": {
			"type":"NodePort",
			"ports": [{
					"protocol":"TCP",
					"port": 8888
					}],
			"selector": {"name": "jupyter"}
		}
	}
	' https://59.110.220.63:6443/api/v1/namespaces/default/services

### 查询全部的Services的信息
通过查询Services的信息，可以获取K8s分配给Pod的外网端口号等信息。

	curl -k -v -X GET -H "Authorization: Bearer ey..Xg" -H "Content-Type: application/json" https://59.110.220.63:6443/api/v1/namespaces/default/services

### 指定一个service名称，查询指定名称的service信息
用于获取端口号的常用操作

	curl -k -v -X GET -H "Authorization: Bearer ey..Xg" -H "Content-Type: application/json" https://59.110.220.63:6443/api/v1/namespaces/default/services/pod-jupyter-233

### 删除已创建的Pod
用于删除已经存在的Pod。请注意，发起这个HTTP请求后，相应的Pod还需要经历一段时间的Terminating后，才会真正被删除。

	curl -k -v -X DELETE -H "Authorization: Bearer ey..Xg" -H "Content-Type: application/json" https://59.110.220.63:6443/api/v1/namespaces/default/pods/pod-jupyter-233

## 删除已创建的service
删除service与删除pod的不同之处在于：service可以立刻被删除。因此，在结束学员实验时，请先删除service，再删除Pod。

	curl -k -v -X DELETE -H "Authorization: Bearer ey..Xg" -H "Content-Type: application/json" https://59.110.220.63:6443/api/v1/namespaces/default/services/pod-jupyter-233

以上，即为K8s的常用API操作。感谢您的阅读。








