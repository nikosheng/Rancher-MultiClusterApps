# EKS Rancher 多集群微服务部署实践

### 目录

- 部署`Rancher`
- 准备`EKS`测试环境（分别在两个区域或多个区域）
- 注册`EKS`集群
- 开发`Helm`微服务示例程序 (Optional)
- 部署`Rancher Multi Cluster Apps`

### 部署`Rancher`

`Rancher`部署可以通过容器的方式来进行快速部署, 可以在有`Docker`的环境中通过如下命令快速拉起`Rancher`服务

```
docker run -d --restart=unless-stopped --name rancher --hostname rancher -p 80:80 -p 443:443 rancher/rancher:latest
```

具体的安装部署可以参考博客 [Managing Amazon EKS Clusters with Rancher](https://aws.amazon.com/blogs/opensource/managing-eks-clusters-rancher/)


### 准备`EKS`测试环境（分别在两个区域或多个区域）

本次实验我们通过`eksctl`工具在`us-east-1`区域和`us-east-2`区域分别启动`EKS`集群

- us-east-1 区域

```
eksctl create cluster --name=eks-virginia --nodes-min=1 --nodes-max=2 --version 1.16 --ssh-access --ssh-public-key=virginia --auto-kubeconfig --region us-east-1
```

- us-east-2 区域

```
eksctl create cluster --name=eks-ohio --nodes-min=1 --nodes-max=2 --version 1.16 --ssh-access --ssh-public-key=ohio --auto-kubeconfig --region us-east-2
```

### 注册`EKS`集群

等待步骤2中的集群都创建完毕后，我们可以在Rancher界面中分别注册上面的EKS集群。

```
[ec2-user@ip-192-168-35-59 ~]$ curl --insecure -sfL https://13.229.95.214/v3/import/j89cflmxzkjctbc5dxp7fsfshvdxbpt6lwp9x46klk8zdg975pp4hs.yaml | kubectl apply -f -
clusterrole.rbac.authorization.k8s.io/proxy-clusterrole-kubeapiserver created
clusterrolebinding.rbac.authorization.k8s.io/proxy-role-binding-kubernetes-master created
namespace/cattle-system created
serviceaccount/cattle created
clusterrolebinding.rbac.authorization.k8s.io/cattle-admin-binding created
secret/cattle-credentials-40ce56d created
clusterrole.rbac.authorization.k8s.io/cattle-admin created
deployment.apps/cattle-cluster-agent created
daemonset.apps/cattle-node-agent created
```

### 开发`Helm`微服务示例程序

本次实验当中我们使用Helm进行微服务应用程序的开发，如果我们自己没有Helm Charts, 可以使用测试的[Helm Charts](https://github.com/nikosheng/charts-repo/tree/master/charts/eksdemo) 来进行后续的实验

具体的Helm应用程序的开发和部署，我们可以参考[EKS Workshop](https://www.eksworkshop.com/beginner/060_helm/)

### 部署`Rancher Multi Cluster Apps`

- 进入`Rancher`主界面，点击导航栏的`Global`，进入`Apps`选项
- 选择`Manage Catalogs` 加入私有的`catalog`部署`helm` 微服务程序
- 点击`Add Catalog`
- 在弹框中输入对应的信息
	- `Name`: `private-catalog`
	- `Catalog`：在本次实验中，可以使用实验已经准备好的`Github Catalog`地址`https://github.com/nikosheng/charts-repo`
	- `Branch`: `master`
	- `Scope`: `global`
	- `Helm Version`: `Helm v3`
	- 创建成功后等待状态更改为`Active`
- 创建私有的Catalog后，我们可以回到Apps界面，然后点击Launch选择我们需要部署的程序
- 进入界面后，可以看到我们刚才添加的catalog栏目，同时里面已经有搜索出我们在catalog中上传的helm应用程序。点击应用程序进入配置页。
- 在配置页中填入对应的信息
	- `Name`
	- `Target Projects`: 分别选择两个EKS集群中的Namespace，默认可以选择Default，这样程序会部署在Default Namespace中
	- 其他选项保持默认
	- 点击Launch启动程序的部署
- 这时候可以看到helm程序会正在部署的页面，等待一段时间后可以看到程序变为Active的界面。
- 此时我们可以分别进入到两个EKS集群查看程序的部署情况

	```
	[ec2-user@ip-192-168-35-59 ~]$ kubectl get po,svc,deploy
	NAME                                    READY   STATUS    RESTARTS   AGE
	pod/ecsdemo-crystal-f675bddbd-77xd2     1/1     Running   0          30s
	pod/ecsdemo-crystal-f675bddbd-fg66r     1/1     Running   0          30s
	pod/ecsdemo-crystal-f675bddbd-flph5     1/1     Running   0          30s
	pod/ecsdemo-frontend-5c8d795d5c-hvd4p   1/1     Running   0          30s
	pod/ecsdemo-frontend-5c8d795d5c-qnbv8   1/1     Running   0          30s
	pod/ecsdemo-frontend-5c8d795d5c-sdvkl   1/1     Running   0          30s
	pod/ecsdemo-nodejs-6b568585fc-57cwk     1/1     Running   0          30s
	pod/ecsdemo-nodejs-6b568585fc-85hfd     1/1     Running   0          30s
	pod/ecsdemo-nodejs-6b568585fc-tdff7     1/1     Running   0          30s
	
	NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
	service/kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   7h11m
	
	NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
	deployment.apps/ecsdemo-crystal    3/3     3            3           30s
	deployment.apps/ecsdemo-frontend   3/3     3            3           30s
	deployment.apps/ecsdemo-nodejs     3/3     3            3           30s
	[ec2-user@ip-192-168-35-59 ~]$ kubectl get po,svc,deploy
	NAME                                    READY   STATUS    RESTARTS   AGE
	pod/ecsdemo-crystal-f675bddbd-77xd2     1/1     Running   0          31m
	pod/ecsdemo-crystal-f675bddbd-fg66r     1/1     Running   0          31m
	pod/ecsdemo-crystal-f675bddbd-flph5     1/1     Running   0          31m
	pod/ecsdemo-frontend-5c8d795d5c-hvd4p   1/1     Running   0          31m
	pod/ecsdemo-frontend-5c8d795d5c-qnbv8   1/1     Running   0          31m
	pod/ecsdemo-frontend-5c8d795d5c-sdvkl   1/1     Running   0          31m
	pod/ecsdemo-nodejs-6b568585fc-57cwk     1/1     Running   0          31m
	pod/ecsdemo-nodejs-6b568585fc-85hfd     1/1     Running   0          31m
	pod/ecsdemo-nodejs-6b568585fc-tdff7     1/1     Running   0          31m
	
	NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
	service/kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   7h43m
	
	NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
	deployment.apps/ecsdemo-crystal    3/3     3            3           31m
	deployment.apps/ecsdemo-frontend   3/3     3            3           31m
	deployment.apps/ecsdemo-nodejs     3/3     3            3           31m
	```
 
	可以看到两个集群中微服务程序都已经可以成功部署并运行起来

