

## Fulx CD简介

GitOps的四个原则：

1.以声明式格式编写所有代码（部署代码）2.将基础设施的所需状态存储在git中（基于git的工作模型，为不同的环境创建不同的分支）

3.将更改应用到基础设施环境（通过不同的分支设置审批流程，然后设置分支策略：确保功能分支中编写的任何代码只有在代码审查后才会合并到主分支）

4.在k8s集群内运行软件代理（例如 Flux 控制器），以持续从 Git 存储库中提取更改、检测新提交、比较所需状态 (Git) 与当前状态 (Kubernetes)，然后发出有关偏差的警报，或者通过确保当前状态与所需状态匹配（通过应用来自 git 的清单）来纠正它

Flux 如何工作：

通过控制器及其CRD资源来扩展kubernetes功能

Flux的五个控制器：

控制器监视资源的变动，每当git repo有更新时候，控制器都会将其写入k8s events（用于k8s控制器的通信）

- Source Controller
  - [GitRepository CRD](https://fluxcd.io/flux/components/source/gitrepositories/)（）
  - [OCIRepository CRD](https://fluxcd.io/flux/components/source/ocirepositories/)
  - [HelmRepository CRD](https://fluxcd.io/flux/components/source/helmrepositories/)
  - [HelmChart CRD](https://fluxcd.io/flux/components/source/helmcharts/)
  - [Bucket CRD](https://fluxcd.io/flux/components/source/buckets/)
- Kustomize Controller
  - [Kustomization CRD](https://fluxcd.io/flux/components/kustomize/kustomizations/)
- Helm Controller
  - [HelmRelease CRD](https://fluxcd.io/flux/components/helm/helmreleases/)
- Notification Controller
  - [Provider CRD](https://fluxcd.io/flux/components/notification/providers/)
  - [Alert CRD](https://fluxcd.io/flux/components/notification/alerts/)
  - [Receiver CRD](https://fluxcd.io/flux/components/notification/receivers/)
- Image automation controllers
  - [ImageRepository CRD](https://fluxcd.io/flux/components/image/imagerepositories/)
  - [ImagePolicy CRD](https://fluxcd.io/flux/components/image/imagepolicies/)
  - [ImageUpdateAutomation CRD](https://fluxcd.io/flux/components/image/imageupdateautomations/)

### Features of the GitOps Toolkit

Among the features of the GitOps toolkit are the following:

- Source management (e.g., Git, Helm, Bucket)
- Manifest management (Kustomize and Helm) support
- Event-based (push) and on-a-schedule (pull) reconciliation
- Role-based reconciliation (assume Service Accounts), multi-tenancy
- Health assessment (infrastructure, workloads)
- Dependency management (infrastructure and workloads)
- Outgoing events with webhook senders (e.g., alerting to external systems)
- Incoming events with webhook receivers (e.g., deploy on git commit)
- Update git manifests on new image tag in registry (source writeback/automated patching)
- Policy-driven validation (OPA, admission controllers)
- Integration with git services (GitHub, GitLab, BitBucket)
- Interoperability with CAPI providers (cluster management, fleet management).

### 使用 Flux 和 Flagger 进行渐进式交付

flux是一个持续部署的工具，在协调循环中运行，要想实现渐进式交付和持续交付解决方案，需要一个称为flagger的工具

Flagger要么与Ingress Controller一起工作，要么与Service Mesh一起工作,以实现蓝绿部署，金丝雀发布和A/B测试等。

同时。Flagger公开了一个CRD作为金丝雀发布（可以进行多重的自动化分析，决定是否可以投入生产，是否继续进行发布或者回滚）

## CD to Kubernetes with Flux

### 安装说明

```shell
1.Flux CLI安装
#配置访问gitlab repo的权限
echo  "export GITLAB_TOKEN=glpat-souazJAGugiLBXJHKwvZ"  >> /root/.bashrc
echo  ". <(flux completion bash)"  >> /root/.bashrc
root@master01:~# source ~/.bashrc 

#flux cli安装,用来引导和设置flux CD的组件以及gitops的工具包
curl -s https://fluxcd.io/install.sh | sudo bash

2.引导Flux基础设施
#使用gitlab，并将创建一个FLUX基础设施存储库来安装flux组件，同时也是与k8s集群同步及其部署到集群的任何其他那内容
  都会添加到这个存储库中[有关更多内容请参考https://docs.gitlab.com/ee/user/clusters/agent/gitops/flux_tutorial.html]
  NOTE: 对于引导过程中存在的问题，官方地址都有排查问题的步骤

root@master01:~# flux   check   --pre    #验证k8s集群
► checking prerequisites
✔ Kubernetes 1.27.1 >=1.25.0-0
✔ prerequisites checks passed
  
---------------
#此配置基于access token存在一定的问题，下边的方式基于ssk key是没有问题的
flux bootstrap gitlab --owner=flux-cd \
  --hostname="http://gitlab.x.xinghuihuyu.cn" \
  --repository=flux-infra \
  --branch=main \
  --path=clusters/dev \
  --log-level=debug \
  --network-policy=false 
  
其中大部分的字段都是见名知义的. --branch=main表示与main分支同步，--path=clusters/dev表示并在此存储库的clusters/dev查找部署清单 --path=clusters/dev可以根据不同的环境设置不同的路径
-------------------
#基于ssh  key
flux bootstrap git \
  --url=ssh://git@gitlab.x.xinghuihuyu.cn/flux-cd/flux-infra \
  --branch=main \
  --private-key-file=/root/.ssh/id_rsa \
  --path=clusters/dev \
  --log-level=debug \
  --network-policy=false 
在引导的过程中，部署了GitOps工具包的组件，然后创建secert等信息，以完成与kubernetes的同步

3.对于引导过程的分析
root@master01:~# flux  check     #检查所有组件是否正确运行
► checking prerequisites
✔ Kubernetes 1.27.1 >=1.25.0-0
► checking controllers
✔ helm-controller: deployment ready
► ghcr.io/fluxcd/helm-controller:v0.36.2
✔ kustomize-controller: deployment ready
► ghcr.io/fluxcd/kustomize-controller:v1.1.1
✔ notification-controller: deployment ready
► ghcr.io/fluxcd/notification-controller:v1.1.0
✔ source-controller: deployment ready
► ghcr.io/fluxcd/source-controller:v1.1.2
► checking crds
✔ alerts.notification.toolkit.fluxcd.io/v1beta2
✔ buckets.source.toolkit.fluxcd.io/v1beta2
✔ gitrepositories.source.toolkit.fluxcd.io/v1
✔ helmcharts.source.toolkit.fluxcd.io/v1beta2
✔ helmreleases.helm.toolkit.fluxcd.io/v2beta1
✔ helmrepositories.source.toolkit.fluxcd.io/v1beta2
✔ kustomizations.kustomize.toolkit.fluxcd.io/v1
✔ ocirepositories.source.toolkit.fluxcd.io/v1beta2
✔ providers.notification.toolkit.fluxcd.io/v1beta2
✔ receivers.notification.toolkit.fluxcd.io/v1
✔ all checks passed

一但引导过程完成，就可以将配置仓库作为源，将应用部署到K8s集群中

root@master01:~# kubectl    -n flux-system   get  pods 
NAME                                       READY   STATUS    RESTARTS   AGE
helm-controller-dcf787b5b-flc45            1/1     Running   0          76m
kustomize-controller-75445c59b6-b7x7f      1/1     Running   0          76m
notification-controller-7fb6d8c57f-wjs7f   1/1     Running   0          76m
source-controller-f8655594c-89t2n          1/1     Running   0          76m     

source-controller主要监git repo视源的变化，一旦发生变化，会生成yaml文件或者tar.gz文件,他会使用Kubernetes时间队列进行通知，kustomize-controller会定期检查这些事件（kubernetes events）,拉取对应的文件，将其部署到k8s集群中
```

### CRD分析

```shell
4.source-controller分析
要想跟踪git存储库，要利用CRD定义出来的资源（gitrepositories）
源控制器的目的是跟踪源并与源进行同步（支持多种类型的源，这就是为什么有各种自定义资源的原因）
源控制器会定期同步或者检查源是否有更改，如果有更改，将生成一个atifacts(采用 tar.gz 或者yaml文件的格式),这些文件被kustomize-controller控制器所使用，因此source-controller通知kubernetes events ,kustomize-controller监视这些事件 \
如果有变化，就会生成新版本的atifacts，然后部署到k8s集群
源控制器同时支持对各种源的安全相关的设置和有条件的更新（检查源，但是有特定的条件，例如标签是xxxx的时候才会检查更新，并生成atifacts）

5.GitRepository CRD 规范
flux   create   source  git --help
有关source API的说明请参考Toolkit Components

flux create source git instavote \
  --url=ssh://git@gitlab.x.xinghuihuyu.cn/flux-cd/instavote.git \
  --branch=main \
  --interval 30s \
  --private-key-file=/root/.ssh/id_rsa

root@master01:~# flux   get sources  git 
NAME       	REVISION          	SUSPENDED	READY	MESSAGE                                           
flux-system	main@sha1:447ab7d0	False    	True 	stored artifact for revision 'main@sha1:447ab7d0'	
instavote  	main@sha1:8e230d58	False    	True 	stored artifact for revision 'main@sha1:8e230d58'	

6.kustomize-controller
kustomize-controller将查看更改的来源，如果有任何更改，会将其部署到k8s集群中（这个过程称为reconcile循环），会检查源清单中定义的所需状态，然后与集群中的状态进行比较
需要指出的是1.在生产环境中有时不需要进行自动化部署，kustomize可以对其进行部署验证，然后发出告警 2.kustomize-controller还可以进行垃圾收集（回收git源中删除的应用）3.可以进行排序，以特定的顺序进行部署，以及依赖项4.进行健康检测5.利用K8S的RBAC,根据不同的服务承担不同的角色

7.kustomizations CRD 规范
kustomizations本质上是查看source，然后从中获取atifacts,然后部署到k8s环境
有关更多信息查看Toolkit Components

flux create kustomization  vote-dev --source=app-demo --path="./deploy/vote/base" --prune=true --interval=1m --target-namespace=instavote 


8.Exporting Sync Manifests
我们在引导过程的时候，gitops工具的部署清单是放在了clusters/dev/flux-system下
假设我们现在部署一个应用（需要创建GitRepository和kustomizations资源）这两个资源的yaml文件正确的方式是应该存储在flux-cd/flux-infra.git这个项目的clusters/dev/下(因为这样会跟踪部署清单的版本)，如果这两个文件有修改，会自定同步到集群中（例如在开始的时候kustomizations没有使用健康检测功能，现在需要加上健康检测，一旦仓库的yaml文件具有健康检测功能，就会应用到集群中）
root@master01:~/game/flux-infra/clusters/dev# flux   export  source  git    app-demo    >>instavote-gitrepository.yaml  
root@master01:~/game/flux-infra/clusters/dev# cat instavote-gitrepository.yaml 
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: app-demo
  namespace: flux-system
spec:
  interval: 30s
  ref:
    branch: main
  secretRef:
    name: app-demo
  timeout: 1m0s
  url: ssh://git@gitlab.x.xinghuihuyu.cn/flux-cd/app-demo.git
root@master01:~/game/flux-infra/clusters/dev# ls
flux-system  instavote-gitrepository.yaml
root@master01:~/game/flux-infra/clusters/dev# flux   export    kustomization   vote-dev     
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: vote-dev
  namespace: flux-system
spec:
  interval: 1m0s
  path: ./deploy/vote/base
  prune: true
  sourceRef:
    kind: GitRepository
    name: app-demo
  targetNamespace: instavote
root@master01:~/game/flux-infra/clusters/dev# flux   export    kustomization   vote-dev       >>vote-dev-kustomization.yaml 
root@master01:~/game/flux-infra/clusters/dev# ll
total 8
drwxr-xr-x 3 root root  96 Oct 30 15:26 ./
drwxr-xr-x 4 root root  32 Oct 27 15:51 ../
drwxr-xr-x 2 root root  82 Oct 27 15:51 flux-system/
-rw-r--r-- 1 root root 272 Oct 30 15:24 instavote-gitrepository.yaml
-rw-r--r-- 1 root root 268 Oct 30 15:26 vote-dev-kustomization.yaml
root@master01:~/game/flux-infra/clusters/dev# cat  vote-dev-kustomization.yaml 
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: vote-dev
  namespace: flux-system
spec:
  interval: 1m0s
  path: ./deploy/vote/base
  prune: true
  sourceRef:
    kind: GitRepository
    name: app-demo
  targetNamespace: instavote
  
root@master01:~/game/flux-infra/clusters/dev# git add  .
root@master01:~/game/flux-infra/clusters/dev# git status 
On branch main
Your branch is up to date with 'origin/main'.

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	new file:   instavote-gitrepository.yaml
	new file:   vote-dev-kustomization.yaml

root@master01:~/game/flux-infra/clusters/dev# git commit   -am "add source and  kustomization yaml "
[main f2cf6bf] add source and  kustomization yaml
 2 files changed, 28 insertions(+)
 create mode 100644 clusters/dev/instavote-gitrepository.yaml
 create mode 100644 clusters/dev/vote-dev-kustomization.yaml
root@master01:~/game/flux-infra/clusters/dev# git  push  origin 
HEAD          main          origin/HEAD   origin/main   
root@master01:~/game/flux-infra/clusters/dev# git  push  origin  main   
Username for 'http://gitlab.x.xinghuihuyu.cn': root
Password for 'http://root@gitlab.x.xinghuihuyu.cn': 
Enumerating objects: 9, done.
Counting objects: 100% (9/9), done.
Delta compression using up to 2 threads
Compressing objects: 100% (6/6), done.
Writing objects: 100% (6/6), 886 bytes | 886.00 KiB/s, done.
Total 6 (delta 0), reused 0 (delta 0), pack-reused 0
To http://gitlab.x.xinghuihuyu.cn/flux-cd/flux-infra.git
   e74db2d..f2cf6bf  main -> main
9.健康检测
root@master01:~/game/flux-infra/clusters/dev# flux create kustomization  vote-dev \
> --source=app-demo \
> --path="./deploy/base" \
> --prune=true \
> --interval=1m \
> --target-namespace=instavote \
> --health-check="Deployment/vote.instavote" \
> --export > vote-dev-kustomization.yaml      
root@master01:~/game/flux-infra/clusters/dev# cat  vote-dev-kustomization.yaml 
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: vote-dev
  namespace: flux-system
spec:
  healthChecks:
  - kind: Deployment
    name: vote
    namespace: instavote
  interval: 1m0s
  path: ./deploy/base
  prune: true
  sourceRef:
    kind: GitRepository
    name: app-demo
  targetNamespace: instavote
  timeout: 2m0s
root@master01:~/game/flux-infra/clusters/dev# git  commit -am "health check for   kustomization "
[main c2f30d9] health check for   kustomization
 1 file changed, 6 insertions(+), 1 deletion(-)
root@master01:~/game/flux-infra/clusters/dev# git  push origin  main   
Username for 'http://gitlab.x.xinghuihuyu.cn': root
Password for 'http://root@gitlab.x.xinghuihuyu.cn': 
Enumerating objects: 9, done.
Counting objects: 100% (9/9), done.
Delta compression using up to 2 threads
Compressing objects: 100% (5/5), done.
Writing objects: 100% (5/5), 538 bytes | 538.00 KiB/s, done.
Total 5 (delta 2), reused 0 (delta 0), pack-reused 0
To http://gitlab.x.xinghuihuyu.cn/flux-cd/flux-infra.git
   f2cf6bf..c2f30d9  main -> main


10.Garbage Collection
假设在我们应用程序的source存储库（flux-cd/app-demo.git）删除了某个yaml文件（因为是git存储库，文件的的删除应该遵循git项目流程），反映在集群中后也是被删除的 （kustomization    --prune=true启用）
但是我们可以通过为source中的yaml文件添加这样的注解来排除删除（kustomize.toolkit.fluxcd.io/prune: disabled）,即使存储库删除了文件，集群中的资源也不会删除
11.假设现在一个应用程序（投票）依赖于另一个应用（redis），所以当我们为"投票"应用程序定义kustomizations规范时候，我们可以添加一个配置来定义对redis的依赖
虽然说"投票"和"redis"的kustomizations不一致，但是他们的GitRepository是同一个（同一个大型的应用仓库）
-----
#create the customization for redis:
cd flux-infra/clusters/dev

flux create kustomization redis-dev \
--source=instavote \
--path="./deploy/redis" \
--prune=true \
--interval=1m \
--target-namespace=instavote \
--health-check="Deployment/redis.instavote" \
--export > redis-kustomization.yaml                           将其放在flux-cd/flux-infra的clusters/dev/下

flux create kustomization vote-dev \
--source=instavote \
--path="./deploy/vote" \
--prune=true \
--interval=1m \
--target-namespace=instavote \
--health-check="Deployment/vote.instavote" \
--depends-on=redis-dev \
--export > vote-dev-kustomization.yaml                       将其放在flux-cd/flux-infra的clusters/dev/下

然后这两个应用会自动部署
```

## Kustomize

```shell
为不用的环境创建自定义配置的时候，将应用程序部署到staging production qa dev环境时候，这是一中非常普遍的做法
1）接下来我们创建两个不同的环境并为每个环境提供单独的配置
1.假设我们在GKE上创建了一个staging环境的k8s集群
2.将GKE集群的上下文配置到kubeconfig文件中
3.暂存环境的引导(克隆存储库和分支，生成清单，同步清单，安装gitops工具)
flux bootstrap git \
  --url=ssh://git@gitlab.x.xinghuihuyu.cn/flux-cd/flux-infra \
  --context=<your-staging-context>
  --branch=main \
  --private-key-file=/root/.ssh/id_rsa \
  --path=./clusters/staging\
  --log-level=debug \
  --network-policy=false 
  
  
2）为每个环境创建自定义配置
现在假设在不用的环境部署相同的应用程序，但是每个环境希望使用不同的属性，可以这样实现：一份通用配置，各个环境加载通用配置，然后对其进行覆盖，当我们修改了通用配置之后，也会应用于各个环境（BASE +  Overlays）。这个工具就是Kustomize
  
现在我们的flux-cd 工具存储库是这样的
root@master01:~/game/flux-infra/clusters# tree  
.
├── dev
│   └── flux-system
│       ├── gotk-components.yaml
│       ├── gotk-sync.yaml
│       └── kustomization.yaml
└── staging
    └── flux-system
│       ├── gotk-components.yaml
│       ├── gotk-sync.yaml
│       └── kustomization.yaml        

安装kustomize工具【参见https://kustomize.io/】    

1.按照kustomize管理应用的原理，现在BASE目录是这样的
root@master01:~/game/app-demo/deploy# tree  
.
├── redis
│   └── base
│       ├── deployment.yaml
│       ├── kustomization.yaml
│       └── service.yaml
└── vote
    └── base
        ├── deployment.yaml
        └── service.yaml

4 directories, 5 files

root@master01:~/game/app-demo/deploy/vote# cd base/
root@master01:~/game/app-demo/deploy/vote/base# ls
deployment.yaml  service.yaml
root@master01:~/game/app-demo/deploy/vote/base# kustomize   create  --autodetect     
root@master01:~/game/app-demo/deploy/vote/base# ll
total 12
drwxr-xr-x 2 root root  75 Oct 30 16:40 ./
drwxr-xr-x 3 root root  18 Oct 30 16:11 ../
-rw-r--r-- 1 root root 454 Oct 30 16:11 deployment.yaml
-rw-r--r-- 1 root root 108 Oct 30 16:40 kustomization.yaml
-rw-r--r-- 1 root root 261 Oct 30 16:11 service.yaml
root@master01:~/game/app-demo/deploy/vote/base# cat   kustomization.yaml  
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml


2.为dev环境打补丁
root@master01:~/game/app-demo-main/deploy/vote/dev# tree  
.
├── deployment.yaml
└── kustomization.yaml

0 directories, 2 files
root@master01:~/game/app-demo/deploy/vote/dev# cat deployment.yaml   
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: vote
    tier: front
  name: vote
spec:
  replicas: 3
  template:
    spec:
      containers:
      - image: schoolofdevops/vote:v4
        name: vote
root@master01:~/game/app-demo/deploy/vote/dev# cat kustomization.yaml  
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../base
patchesStrategicMerge:
- deployment.yaml

3.使用自定义配置部署开发环境
#修改gitops存储库的Kustomization yaml文件
root@master01:~/game/flux-infra/clusters/dev# git pull origin  main 
From http://gitlab.x.xinghuihuyu.cn/flux-cd/flux-infra
 * branch            main       -> FETCH_HEAD
Already up to date.
root@master01:~/game/flux-infra/clusters/dev# ll
total 12
drwxr-xr-x 3 root root 128 Oct 30 16:29 ./
drwxr-xr-x 4 root root  32 Oct 30 16:26 ../
drwxr-xr-x 2 root root  82 Oct 30 16:26 flux-system/
-rw-r--r-- 1 root root 272 Oct 30 16:26 instavote-gitrepository.yaml
-rw-r--r-- 1 root root 364 Oct 30 16:29 redis-kustomization.yaml
-rw-r--r-- 1 root root 394 Oct 30 16:26 vote-dev-kustomization.yaml
root@master01:~/game/flux-infra/clusters/dev# vim vote-dev-kustomization.yaml 
root@master01:~/game/flux-infra/clusters/dev# cat vote-dev-kustomization.yaml 
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: vote-dev
  namespace: flux-system
spec:
  dependsOn:
  - name: redis-dev
  healthChecks:
  - kind: Deployment
    name: vote
    namespace: instavote
  interval: 1m0s
  path: ./deploy/vote/dev
  prune: true
  sourceRef:
    kind: GitRepository
    name: app-demo
  targetNamespace: instavote
  timeout: 2m0s
root@master01:~/game/flux-infra/clusters/dev# git commit  -am   "update   vote-devkustomization " 
[main db5bf46] update   vote-devkustomization
 1 file changed, 1 insertion(+), 1 deletion(-)
root@master01:~/game/flux-infra/clusters/dev# git push origin  main  
Username for 'http://gitlab.x.xinghuihuyu.cn': root 
4.配置staging环境
root@master01:~/game/app-demo/deploy/vote# tree  
.
├── base
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   └── service.yaml
├── dev
│   ├── deployment.yaml
│   └── kustomization.yaml
└── staging
    ├── deployment.yaml
    └── kustomization.yaml


#1.修改应用部署存储库
root@master01:~/game/app-demo/deploy/vote# vim staging/kustomization.yaml 
root@master01:~/game/app-demo/deploy/vote# vim  staging/deployment.yaml 
root@master01:~/game/app-demo/deploy/vote# cat staging/deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: vote
    tier: front
  name: vote
spec:
  replicas: 4
  template:
    spec:
      containers:
      - image: schoolofdevops/vote:v3
        name: vote
root@master01:~/game/app-demo/deploy/vote# git add   staging/*  
root@master01:~/game/app-demo/deploy/vote# git commit   -m "commit staging  "
[main ed13df1] commit staging
 2 files changed, 21 insertions(+)
 create mode 100644 deploy/vote/staging/deployment.yaml
 create mode 100644 deploy/vote/staging/kustomization.yaml
root@master01:~/game/app-demo/deploy/vote# git push  origin  main 

root@master01:~/game/app-demo/deploy/vote/staging# cp  ../base/service.yaml    .
root@master01:~/game/app-demo/deploy/vote/staging# vim service.yaml 
root@master01:~/game/app-demo/deploy/vote/staging# ll
total 12
drwxr-xr-x 2 root root  75 Oct 30 17:57 ./
drwxr-xr-x 5 root root  44 Oct 30 17:26 ../
-rw-r--r-- 1 root root 243 Oct 30 17:28 deployment.yaml
-rw-r--r-- 1 root root 126 Oct 30 17:26 kustomization.yaml
-rw-r--r-- 1 root root 173 Oct 30 17:57 service.yaml

root@master01:~/game/app-demo/deploy/vote/staging# cat  service.yaml 
apiVersion: v1
kind: Service
metadata:
  name: vote
spec:
  ports:
  - name: "80"
    nodePort: 30301
    port: 80
    protocol: TCP
    targetPort: 80
  type: LoadBalancer
root@master01:~/game/app-demo/deploy/vote/staging# cat kustomization.yaml  
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../base
patchesStrategicMerge:
- deployment.yaml
- service.yaml

#2.修改gitops存储库（因为我的环境没有第二个ks8环境，所以没有执行git push）
root@master01:~/game/flux-infra# cd clusters/
root@master01:~/game/flux-infra/clusters# ls
dev  staging
root@master01:~/game/flux-infra/clusters# cd dev/
root@master01:~/game/flux-infra/clusters/dev# ls
flux-system  instavote-gitrepository.yaml  redis-kustomization.yaml  vote-dev-kustomization.yaml
root@master01:~/game/flux-infra/clusters/dev# cp *.yaml    ../staging/
root@master01:~/game/flux-infra/clusters/dev# cd ../staging/ 
root@master01:~/game/flux-infra/clusters/staging# ls
flux-system  instavote-gitrepository.yaml  redis-kustomization.yaml  vote-dev-kustomization.yaml
root@master01:~/game/flux-infra/clusters/staging# rm redis-kustomization.yaml 
root@master01:~/game/flux-infra/clusters/staging# mv vote-dev-kustomization.yaml vote-staging-kustomization.yaml 
root@master01:~/game/flux-infra/clusters/staging# vim vote-staging-kustomization.yaml 
root@master01:~/game/flux-infra/clusters/staging# cat vote-staging-kustomization.yaml 
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: vote-staging
  namespace: flux-system
spec:
  healthChecks:
  - kind: Deployment
    name: vote
    namespace: instavote
  interval: 1m0s
  path: ./deploy/vote/staging
  prune: true
  sourceRef:
    kind: GitRepository
    name: app-demo
  targetNamespace: instavote
  timeout: 2m0s
```

### Understanding the Kustomization File

具体的内容请[参考](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/crds/)

下面是一些配置的例子

+  replicas

  ```shell
  root@master01:~/game/app-demo/deploy/vote/dev# tree 
  .
  ├── deployment.yaml
  └── kustomization.yaml
  
  0 directories, 2 files
  root@master01:~/game/app-demo/deploy/vote/dev# cat deployment.yaml 
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    creationTimestamp: null
    labels:
      app: vote
      tier: front
    name: vote
  spec:
    replicas: 5
    template:
      spec:
        containers:
        - image: schoolofdevops/vote:v4
          name: vote
  root@master01:~/game/app-demo/deploy/vote/dev# cat kustomization.yaml   
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
  - ../base
  patches:
  - path: deployment.yaml
    target:
      kind: Deployment
      name: vote
  #此处为示例
  replicas:
  - name: vote
    count: 1
  root@master01:~/game/app-demo/deploy/vote/dev# kustomize  build  
  apiVersion: v1
  kind: Service
  metadata:
    creationTimestamp: null
    labels:
      app: vote
      tier: front
    name: vote
  spec:
    ports:
    - name: "80"
      nodePort: 30300
      port: 80
      protocol: TCP
      targetPort: 80
    selector:
      app: vote
    type: NodePort
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    labels:
      app: vote
      tier: front
    name: vote
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: vote
    strategy: {}
    template:
      metadata:
        creationTimestamp: null
        labels:
          app: vote
          tier: front
      spec:
        containers:
        - image: schoolofdevops/vote:v4
          imagePullPolicy: Always
          name: vote
          resources: {}
  ```

+ image and namespace

  ```shell
  root@master01:~/game/app-demo/deploy/vote/dev# tree 
  .
  ├── deployment.yaml
  └── kustomization.yaml
  
  0 directories, 2 files
  root@master01:~/game/app-demo/deploy/vote/dev# cat kustomization.yaml  
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
  - ../base
  patches:
  - path: deployment.yaml
    target:
      kind: Deployment
      name: vote
  
  replicas:
  - name: vote
    count: 1
  #此处为配置项
  images:
  - name: schoolofdevops/vote:v4
    newTag: v3
  
  namespace: instavote
  root@master01:~/game/app-demo/deploy/vote/dev# kustomize  build  
  apiVersion: v1
  kind: Service
  metadata:
    creationTimestamp: null
    labels:
      app: vote
      tier: front
    name: vote
    namespace: instavote
  spec:
    ports:
    - name: "80"
      nodePort: 30300
      port: 80
      protocol: TCP
      targetPort: 80
    selector:
      app: vote
    type: NodePort
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    labels:
      app: vote
      tier: front
    name: vote
    namespace: instavote
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: vote
    strategy: {}
    template:
      metadata:
        creationTimestamp: null
        labels:
          app: vote
          tier: front
      spec:
        containers:
        - image: schoolofdevops/vote:v3
          imagePullPolicy: Always
          name: vote
          resources: {}
  
  ```

+ [commonAnnotations](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/commonannotations/) and [commonLabels](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/commonlabels/)

  需要注意的是对于已经部署好的应用，需要删除掉，然后重新加载（flux reconcile）

+ configMapGenerator

  在没有kustomize之前，我们对于options.env转化为configmap.yaml 需要维护两份配置，同时还不会触发新的部署动作

  而有了kustomize之后，修改完options.env会自动生成一个具有新的哈希的configmap,并且自动触发新的部署

  ```shell
  root@master01:~/game/app-demo/deploy/vote/dev# tree 
  .
  ├── deployment.yaml
  ├── kustomization.yaml
  └── options.env
  
  0 directories, 3 files
  root@master01:~/game/app-demo/deploy/vote/dev# cat deployment.yaml  
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    creationTimestamp: null
    labels:
      app: vote
      tier: front
    name: vote
  spec:
    replicas: 5
    template:
      spec:
        containers:
        - image: schoolofdevops/vote:v4
          name: vote
          envFrom:
          - configMapRef:
              name: vote-options
              optional: true
  
  root@master01:~/game/app-demo/deploy/vote/dev# cat kustomization.yaml  
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
  - ../base
  patches:
  - path: deployment.yaml
    target:
      kind: Deployment
      name: vote
  
  replicas:
  - name: vote
    count: 1
  
  images:
  - name: schoolofdevops/vote:v4
    newTag: v3
  
  namespace: instavote
  
  configMapGenerator:
  - name: vote-options
    envs:
    - options.env
  root@master01:~/game/app-demo/deploy/vote/dev# cat options.env 
  OPTION_A=DEV
  OPTION_B=STAGI
  ```

## 使用 FLUX 部署 HELM CHART

假设现在编写了一个 Helm 图表来部署其中一个微服务（i.e., the **result** app）。此外，大多数现成的应用程序（如 Redis 和 Postgres）似乎都有 Helm 图表，而且可以从 Artifact Hub 搜索到

在本章中，您将：

- 探索 [Helm Controller](https://fluxcd.io/docs/components/helm/).
- 创建**HelmRepository**源来跟踪 Postgres 的最新图表
- 创建**HelmRelease**以获取和部署 Postgres 的 Helm 图表，并将其设置为数据库服务。
- 为**result**应用程序创建一个简单的 Helm 图表，并将代码添加到 Git 存储库，该存储库已包含 Kustomize 覆盖形式的部署代码。
- 创建**HelmRepository**以从存储 Helm 图表源的 Git 存储库中构建和部署图表。

1.先使用helm部署一个示例chart

```shell
root@master01:~/prepare/18-flux-cd#  helm repo add lfs269 https://lfs269.github.io/helm-charts/    
"lfs269" has been added to your repositories
root@master01:~/prepare/18-flux-cd# helm repo list   
NAME       	URL                                  
lfs269     	https://lfs269.github.io/helm-charts/
root@master01:~/prepare/18-flux-cd# helm repo update   lfs269  
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "lfs269" chart repository
Update Complete. ⎈Happy Helming!⎈

root@master01:~/prepare/18-flux-cd# helm   search repo   postgre
NAME                 	CHART VERSION	APP VERSION	DESCRIPTION                                       
bitnami/postgresql   	13.1.5       	16.0.0     	PostgreSQL (Postgres) is an open source object-...
bitnami/postgresql-ha	12.0.4       	16.0.0     	This PostgreSQL cluster solution includes the P...
lfs269/postgres      	0.2.11       	13.2       	A Helm chart for PostgreSQL on Kubernetes, fork...
bitnami/supabase     	2.0.1        	0.23.9     	Supabase is an open source Firebase alternative...
root@master01:~/prepare/18-flux-cd# helm    -n instavote  list  
NAME	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART          	APP VERSION
db  	instavote	1       	2023-10-31 14:42:27.007854773 +0800 CST	deployed	postgres-0.2.11	13.2       
root@master01:~# helm install db lfs269/postgres   --set   settings.authMethod=trust,service.name=db -n instavote   
root@master01:~# helm  -n instavote   uninstall  db  
release "db" uninstalled
```

2.Flux 如何处理 Helm 的部署
source controller： 1）bucket  2）git  3）helm    4）oci  转换helm存储库或者git存储库为helm chart(建议使用预先创建的helm存储库或者包含helmt图标的git存储库)
helmreleases：是一个CRD,HELM CONTROLLER使用CRD来接收用户的规范（主要包括chart规范（源在哪，如何生成cahrt,使用哪些自定义值）），在此基础上源控制器与helm控制器一起生成chart,同时HELM CONTROLLER监视helm chart的变化
3.helm controller的特点
主要监视helmreleases资源，然后部署到k8s集群，主要有以下几个特点
1）进行自动修复（如果【升级】失败，还能够进行自动修复（默认是回滚，补救措施必须自己定义）
2）排序：假设要部署多个应用，可以使用排序功能来实现
3）垃圾收集
4）收集状态信息，向alter控制器发送通知
5）为先前生成的helm chart进行更新或者打补丁（helm与kustomize的结合）

4.将helm存储库定义为源
首先我们了解helmrepositories  CRD,他由源控制器管理与创建helm chart
示例如下：

```
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: podinfo
  namespace: default
spec:
  interval: 5m0s
  url: oci://ghcr.io/my-user/my-private-repo
  type: "oci"
  secretRef:
    name: oci-creds
---
apiVersion: v1
kind: Secret
metadata:
  name: oci-creds
  namespace: default
stringData:
  username: example
  password: 123456
-------------------
root@master01:~# flux   create   source  helm   lfs269   --url=https://lfs269.github.io/helm-charts  --interval=10m 
✚ generating HelmRepository source
► applying HelmRepository source
✔ source created
◎ waiting for HelmRepository source reconciliation
✔ HelmRepository source reconciliation completed
✔ fetched revision: sha256:66ad59de10728ee4001ac4019f92164c9f9b3e40f2576735634be3829dc5a122
```

5.helmreleases CRD
通过上一部创建的源来发布应用

```shell
root@master01:~/game/app-demo/deploy/vote/dev# tree 
.
├── deployment.yaml
├── kustomization.yaml
├── options.env
└── values.db.yaml

0 directories, 4 files
root@master01:~/game/app-demo/deploy/vote/dev# cat  values.db.yaml 
settings:
  authMethod: trust

service:
  name: db

root@master01:~/game/app-demo/deploy/vote/dev#   flux create helmrelease db \
>     --source=HelmRepository/lfs269 \
>     --chart=postgres \
> --values=./values.db.yaml \
> --target-namespace=instavote  
✚ generating HelmRelease
► applying HelmRelease
✔ HelmRelease updated
◎ waiting for HelmRelease reconciliation
✔ HelmRelease db is ready
✔ applied revision 0.2.11
root@master01:~/game/app-demo/deploy/vote/dev# flux  get sources  all 
NAME                     	REVISION          	SUSPENDED	READY	MESSAGE                                           
gitrepository/app-demo   	main@sha1:9ac9dbea	False    	True 	stored artifact for revision 'main@sha1:9ac9dbea'	
gitrepository/flux-system	main@sha1:3ada5fa0	False    	True 	stored artifact for revision 'main@sha1:3ada5fa0'	

NAME                 	REVISION       	SUSPENDED	READY	MESSAGE                                     
helmrepository/lfs269	sha256:66ad59de	False    	True 	stored artifact: revision 'sha256:66ad59de'	

NAME                    	REVISION	SUSPENDED	READY	MESSAGE                                       
helmchart/flux-system-db	0.2.11  	False    	True 	pulled 'postgres' chart with version '0.2.11'	  #自动创建了新的helm chart对象

root@master01:~/game/app-demo/deploy/vote/dev# flux  get helmreleases  
NAME	REVISION	SUSPENDED	READY	MESSAGE                          
db  	0.2.11  	False    	True 	Release reconciliation succeeded	
root@master01:~/game/app-demo/deploy/vote/dev# kubectl   -n instavote   get  pods  
NAME                      READY   STATUS    RESTARTS   AGE
instavote-db-postgres-0   1/1     Running   0          4m38s
redis-78d4b8b77c-xqss7    1/1     Running   0          24h
vote-7977f468d8-ndtgd     1/1     Running   0          13h
```

6.确认没问题之后生成清单文件，将控制器有关的yaml文件移动到gitops存储库

```shell
root@master01:~/game/app-demo/deploy/vote/dev# ll
total 24
drwxr-xr-x 2 root root 153 Oct 31 17:09 ./
drwxr-xr-x 5 root root  44 Oct 30 17:26 ../
-rw-r--r-- 1 root root 385 Oct 31 17:09 db-helmrelease.yaml
-rw-r--r-- 1 root root 188 Oct 31 17:07 db-helmrepository.yaml
-rw-r--r-- 1 root root 343 Oct 31 02:56 deployment.yaml
-rw-r--r-- 1 root root 340 Oct 31 02:48 kustomization.yaml
-rw-r--r-- 1 root root  30 Oct 31 02:44 options.env
-rw-r--r-- 1 root root  51 Oct 31 16:33 values.db.yaml
root@master01:~/game/app-demo/deploy/vote/dev# cat  db-helmrepository.yaml 
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: lfs269
  namespace: flux-system
spec:
  interval: 10m0s
  url: https://lfs269.github.io/helm-charts
root@master01:~/game/app-demo/deploy/vote/dev# cat  db-helmrelease.yaml 
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: db
  namespace: flux-system
spec:
  chart:
    spec:
      chart: postgres
      reconcileStrategy: ChartVersion
      sourceRef:
        kind: HelmRepository
        name: lfs269
  interval: 1m0s
  targetNamespace: instavote
  values:
    service:
      name: db
    settings:
      authMethod: trust
root@master01:~/game/app-demo/deploy/vote/dev# ll
total 24
drwxr-xr-x 2 root root 153 Oct 31 17:09 ./
drwxr-xr-x 5 root root  44 Oct 30 17:26 ../
-rw-r--r-- 1 root root 385 Oct 31 17:09 db-helmrelease.yaml
-rw-r--r-- 1 root root 188 Oct 31 17:07 db-helmrepository.yaml
-rw-r--r-- 1 root root 343 Oct 31 02:56 deployment.yaml
-rw-r--r-- 1 root root 340 Oct 31 02:48 kustomization.yaml
-rw-r--r-- 1 root root  30 Oct 31 02:44 options.env
-rw-r--r-- 1 root root  51 Oct 31 16:33 values.db.yaml
root@master01:~/game/app-demo/deploy/vote/dev# mv values.db.yaml  db-helmrelease.yaml   db-helmrepository.yaml   /root/game/
app-demo/      app-demo-main/ flux-infra/    
root@master01:~/game/app-demo/deploy/vote/dev# mv values.db.yaml  db-helmrelease.yaml   db-helmrepository.yaml   /root/game/flux-infra/clusters/dev/
root@master01:~/game/app-demo/deploy/vote/dev# cd /root/game/flux-infra/clusters/dev/ 
root@master01:~/game/flux-infra/clusters/dev# ls
db-helmrelease.yaml  db-helmrepository.yaml  flux-system  instavote-gitrepository.yaml  redis-kustomization.yaml  values.db.yaml  vote-dev-kustomization.yaml
root@master01:~/game/flux-infra/clusters/dev# ll
total 28
drwxr-xr-x 3 root root 4096 Oct 31 17:20 ./
drwxr-xr-x 4 root root   32 Oct 31 17:18 ../
-rw-r--r-- 1 root root  385 Oct 31 17:09 db-helmrelease.yaml
-rw-r--r-- 1 root root  188 Oct 31 17:07 db-helmrepository.yaml
drwxr-xr-x 2 root root   82 Oct 31 17:18 flux-system/
-rw-r--r-- 1 root root  272 Oct 31 17:18 instavote-gitrepository.yaml
-rw-r--r-- 1 root root  363 Oct 31 17:18 redis-kustomization.yaml
-rw-r--r-- 1 root root   51 Oct 31 16:33 values.db.yaml
-rw-r--r-- 1 root root  393 Oct 31 17:18 vote-dev-kustomization.yaml
root@master01:~/game/flux-infra/clusters/dev# git  add  *  
root@master01:~/game/flux-infra/clusters/dev# git commit -m  "commit for helm "
[main 57840bd] commit for helm
 3 files changed, 35 insertions(+)
 create mode 100644 clusters/dev/db-helmrelease.yaml
 create mode 100644 clusters/dev/db-helmrepository.yaml
 create mode 100644 clusters/dev/values.db.yaml
root@master01:~/game/flux-infra/clusters/dev# git push origin  main  
Username for 'http://gitlab.x.xinghuihuyu.cn': root
Password for 'http://root@gitlab.x.xinghuihuyu.cn': 
Enumerating objects: 10, done.
Counting objects: 100% (10/10), done.
Delta compression using up to 2 threads
Compressing objects: 100% (7/7), done.
Writing objects: 100% (7/7), 1.04 KiB | 1.04 MiB/s, done.
Total 7 (delta 0), reused 0 (delta 0), pack-reused 0
To http://gitlab.x.xinghuihuyu.cn/flux-cd/flux-infra.git
   3ada5fa..57840bd  main -> main
```

### Using Git Repository as a Chart Source

在之前使用git source + kustomize的示例中，关联的git[存储库为](http://gitlab.x.xinghuihuyu.cn/flux-cd/app-demo.git)

![2](.//img/2.png)

```shell
root@master01:~/game/app-demo/deploy# tree  charts/    #这个目录是我们新建的，也是我们自己维护的helm的本地chart
charts/
└── result
    ├── Chart.yaml
    ├── templates
    │   ├── NOTES.txt
    │   ├── _helpers.tpl
    │   ├── deployment.yaml
    │   ├── hpa.yaml
    │   ├── ingress.yaml
    │   ├── service.yaml
    │   ├── serviceaccount.yaml
    │   └── tests
    │       └── test-connection.yaml
    └── values.yaml

3 directories, 10 files
root@master01:~/game/app-demo/deploy# cat charts/result/Chart.yaml 
apiVersion: v2
name: result
description: A Helm chart for Result NodeJS app

type: application

version: 0.1.2                #每次更新版本是必须要修改的

appVersion: "1.0.0"
root@master01:~/game/app-demo/deploy/charts/result# git add *  
root@master01:~/game/app-demo/deploy/charts/result# git commit  -m "git repo  for   helm "
[main 9767379] git repo  for   helm
 10 files changed, 353 insertions(+)
 create mode 100644 deploy/charts/result/Chart.yaml
 create mode 100644 deploy/charts/result/templates/NOTES.txt
 create mode 100644 deploy/charts/result/templates/_helpers.tpl
 create mode 100644 deploy/charts/result/templates/deployment.yaml
 create mode 100644 deploy/charts/result/templates/hpa.yaml
 create mode 100644 deploy/charts/result/templates/ingress.yaml
 create mode 100644 deploy/charts/result/templates/service.yaml
 create mode 100644 deploy/charts/result/templates/serviceaccount.yaml
 create mode 100644 deploy/charts/result/templates/tests/test-connection.yaml
 create mode 100644 deploy/charts/result/values.yaml
root@master01:~/game/app-demo/deploy/charts/result# git  push origin  main 
Username for 'http://gitlab.x.xinghuihuyu.cn': root
Password for 'http://root@gitlab.x.xinghuihuyu.cn': 
Enumerating objects: 19, done.
Counting objects: 100% (19/19), done.
Delta compression using up to 2 threads
Compressing objects: 100% (14/14), done.
Writing objects: 100% (17/17), 5.27 KiB | 5.27 MiB/s, done.
Total 17 (delta 0), reused 0 (delta 0), pack-reused 0
To http://gitlab.x.xinghuihuyu.cn/flux-cd/app-demo.git
   9ac9dbe..9767379  main -> main

```

![3](.//img/3.png)

我们已经创建了针对某个应用的helm chart,并提交到了我们的git存储库（kustomize也是使用的这个存储库）

现在为了持续部署他，我们必须创建helmrelease，这次我们使用的源是git存储库而不是helm存储库，他需要从这个特定路径**deploy/charts/result**部署

```shell
root@master01:~/game/flux-infra/clusters/dev# flux  get sources  git   
NAME       	REVISION          	SUSPENDED	READY	MESSAGE                                           
app-demo   	main@sha1:97673796	False    	True 	stored artifact for revision 'main@sha1:97673796'	
flux-system	main@sha1:57840bda	False    	True 	stored artifact for revision 'main@sha1:57840bda'	

root@master01:~/game/flux-infra/clusters/dev#   flux create helmrelease result \
>     --interval=10m \
>     --source=GitRepository/app-demo \
>     --chart=./deploy/charts/result \
> --target-namespace=instavote \
> --export >result-helmrelease.yaml 
root@master01:~/game/flux-infra/clusters/dev# cat result-helmrelease.yaml  
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: result
  namespace: flux-system
spec:
  chart:
    spec:
      chart: ./deploy/charts/result
      reconcileStrategy: ChartVersion
      sourceRef:
        kind: GitRepository
        name: app-demo
  interval: 10m0s
  targetNamespace: instavote
root@master01:~/game/flux-infra/clusters/dev#   flux create helmrelease result \
>     --interval=10m \
>     --source=GitRepository/app-demo \
>     --chart=./deploy/charts/result \
> --target-namespace=instavote
✚ generating HelmRelease
► applying HelmRelease
✔ HelmRelease created
◎ waiting for HelmRelease reconciliation
✔ HelmRelease result is ready
✔ applied revision 0.1.2
root@master01:~/game/flux-infra/clusters/dev# flux  get sources  git   
NAME       	REVISION          	SUSPENDED	READY	MESSAGE                                           
app-demo   	main@sha1:97673796	False    	True 	stored artifact for revision 'main@sha1:97673796'	
flux-system	main@sha1:57840bda	False    	True 	stored artifact for revision 'main@sha1:57840bda'	
root@master01:~/game/flux-infra/clusters/dev# flux  get sources  all 
NAME                     	REVISION          	SUSPENDED	READY	MESSAGE                                           
gitrepository/app-demo   	main@sha1:97673796	False    	True 	stored artifact for revision 'main@sha1:97673796'	
gitrepository/flux-system	main@sha1:57840bda	False    	True 	stored artifact for revision 'main@sha1:57840bda'	

NAME                 	REVISION       	SUSPENDED	READY	MESSAGE                                     
helmrepository/lfs269	sha256:66ad59de	False    	True 	stored artifact: revision 'sha256:66ad59de'	

NAME                        	REVISION	SUSPENDED	READY	MESSAGE                                       
helmchart/flux-system-db    	0.2.11  	False    	True 	pulled 'postgres' chart with version '0.2.11'	
helmchart/flux-system-result	0.1.2   	False    	True 	packaged 'result' chart with version '0.1.2' 	

```

然后将其提交到gitops存储库

```shell
root@master01:~/game/flux-infra/clusters/dev# ll
total 32
drwxr-xr-x 3 root root 4096 Oct 31 18:12 ./
drwxr-xr-x 4 root root   32 Oct 31 17:18 ../
-rw-r--r-- 1 root root  385 Oct 31 17:09 db-helmrelease.yaml
-rw-r--r-- 1 root root  188 Oct 31 17:07 db-helmrepository.yaml
drwxr-xr-x 2 root root   82 Oct 31 17:18 flux-system/
-rw-r--r-- 1 root root  272 Oct 31 17:18 instavote-gitrepository.yaml
-rw-r--r-- 1 root root  363 Oct 31 17:18 redis-kustomization.yaml
-rw-r--r-- 1 root root  329 Oct 31 18:12 result-helmrelease.yaml
-rw-r--r-- 1 root root   51 Oct 31 16:33 values.db.yaml
-rw-r--r-- 1 root root  393 Oct 31 17:18 vote-dev-kustomization.yaml
root@master01:~/game/flux-infra/clusters/dev# git  add   result-helmrelease.yaml   
root@master01:~/game/flux-infra/clusters/dev# git commit  -m  "result-helmrelease.yaml"
[main 39f1943] result-helmrelease.yaml
 1 file changed, 16 insertions(+)
 create mode 100644 clusters/dev/result-helmrelease.yaml
root@master01:~/game/flux-infra/clusters/dev# git push origin  main 
Username for 'http://gitlab.x.xinghuihuyu.cn': root
Password for 'http://root@gitlab.x.xinghuihuyu.cn': 
Enumerating objects: 8, done.
Counting objects: 100% (8/8), done.
Delta compression using up to 2 threads
Compressing objects: 100% (5/5), done.
Writing objects: 100% (5/5), 646 bytes | 646.00 KiB/s, done.
Total 5 (delta 1), reused 0 (delta 0), pack-reused 0
To http://gitlab.x.xinghuihuyu.cn/flux-cd/flux-infra.git
   57840bd..39f1943  main -> main

```

## MONITORING AND ALERTING

 Flux 现在本质上是通过 git 提交进行部署的，因此最好在每次提交时更新相应 **Flux reconciliations**的状态。这不仅可以为开发人员提供即时反馈，还可以作为以  后将代码升级到发布分支的标准。

在本章中，您将：

- 探索[通知控制器](https://fluxcd.io/docs/components/notification/)及其功能。
- 为所有事件设置从 Flux 到**Fluxcd Slack 通道的传出通知。**
- 设置 GitHub 的自动状态更新，以进行**投票**、**redis**和**工作人员**自定义。
- 使用 Prometheus 和 Grafana 设置 Flux 监控。

在继续实施之前，您可以首先阅读有关[通知](https://fluxcd.io/docs/guides/notifications/)、 [webhook 接收器](https://fluxcd.io/docs/guides/webhook-receivers/)以及如何设置[监控的内容。](https://fluxcd.io/docs/guides/monitoring/)

### Notification Controller的特点

他充当incoming,outgoing通知和告警服务，和其他控制器的协调是基于k8s evetes 流的。所有控制器连接到K8S事件流，对其进行读取和写入，现在通知控制器向其余的三种控制器提供不同的 服务。如下所示

![通知控制器](.//img/通知控制器.png)

<img src=".//img/通知控制器2.png" alt="通知控制器2" style="zoom:50%;" />



### Configuring an Incoming Webhook

我们使用的是Mattermost,类似于slack

<img src=".//img/slack.png" alt="slack" style="zoom:50%;" />

```shell
root@master01:~#kubectl  create   secret    generic   mattermost-url  --from-literal=address="http://mattermost.x.xinghuihuyu.cn:8065/hooks/ji7a9iec8jyjmgegaoq9xtuhqa"  -n flux-system 
root@master01:~#   flux create alert-provider mattermost \
>   --type slack \
>   --channel="#alerts" \
>   --secret-ref=mattermost-url  
✚ generating Provider
► applying Provider
✔ Provider created
◎ waiting for Provider reconciliation
✔ Provider mattermost is ready
root@master01:~# flux  get alert-providers  
NAME      	READY	MESSAGE     
mattermost	True 	Initialized	

```

### Creating an Alert

创建警报是定义某些事件或者原因与其影响应通知的外部系统之前的映射

<img src=".//img/alter crd.png" alt="alter crd" style="zoom:50%;" />

eventSeverity定义了日志级别，决定将哪些消息发送到外部的系统

```shell
root@master01:~/game/flux-infra/clusters/dev#   flux create alert  mattermost-notification \
>   --event-severity info \
>   --event-source=Kustomization/*  \
>   --event-source=GitRepository/*  \
>   --event-source=HelmRepository/* \
>   --provider-ref mattermost \
>   --export   
---
apiVersion: notification.toolkit.fluxcd.io/v1beta2
kind: Alert
metadata:
  name: mattermost-notification
  namespace: flux-system
spec:
  eventSeverity: info
  eventSources:
  - kind: Kustomization
    name: '*'
  - kind: GitRepository
    name: '*'
  - kind: HelmRepository
    name: '*'
  providerRef:
    name: mattermost
root@master01:~/game/flux-infra/clusters/dev#   flux create alert  mattermost-notification \
>   --event-severity info \
>   --event-source=Kustomization/*  \
>   --event-source=GitRepository/*  \
>   --event-source=HelmRepository/* \
>   --provider-ref mattermost 
✚ generating Alert
► applying Alert
✔ Alert created
◎ waiting for Alert reconciliation
✔ Alert mattermost-notification is ready
root@master01:~/game/flux-infra/clusters/dev# flux  get alert
NAME                   	SUSPENDED	READY	MESSAGE     

```

然后将其同步到gitops工具库中

```shell
root@master01:~/game/flux-infra/clusters/dev# ll
total 32
drwxr-xr-x 3 root root 4096 Oct 31 18:12 ./
drwxr-xr-x 4 root root   32 Oct 31 17:18 ../
-rw-r--r-- 1 root root  385 Oct 31 17:09 db-helmrelease.yaml
-rw-r--r-- 1 root root  188 Oct 31 17:07 db-helmrepository.yaml
drwxr-xr-x 2 root root   82 Oct 31 17:18 flux-system/
-rw-r--r-- 1 root root  272 Oct 31 17:18 instavote-gitrepository.yaml
-rw-r--r-- 1 root root  363 Oct 31 17:18 redis-kustomization.yaml
-rw-r--r-- 1 root root  329 Oct 31 18:12 result-helmrelease.yaml
-rw-r--r-- 1 root root   51 Oct 31 16:33 values.db.yaml
-rw-r--r-- 1 root root  393 Oct 31 17:18 vote-dev-kustomization.yaml
root@master01:~/game/flux-infra/clusters/dev# flux  get alert
NAME                   	SUSPENDED	READY	MESSAGE     
mattermost-notification	False    	True 	Initialized	
root@master01:~/game/flux-infra/clusters/dev# flux  export  alert  mattermost-notification   >mattermost-notification-alert.yaml 
root@master01:~/game/flux-infra/clusters/dev# flux  get alert-providers 
NAME      	READY	MESSAGE     
mattermost	True 	Initialized	
root@master01:~/game/flux-infra/clusters/dev# flux export  alert-provider  mattermost    >mattermost-alert-provider.yaml 
root@master01:~/game/flux-infra/clusters/dev# git add   mattermost-* 
root@master01:~/game/flux-infra/clusters/dev# git commit -m  "mattermost-*"
[main 49c26c9] mattermost-*
 2 files changed, 28 insertions(+)
 create mode 100644 clusters/dev/mattermost-alert-provider.yaml
 create mode 100644 clusters/dev/mattermost-notification-alert.yaml
root@master01:~/game/flux-infra/clusters/dev# git push  origin  main 
Username for 'http://gitlab.x.xinghuihuyu.cn': root
Password for 'http://root@gitlab.x.xinghuihuyu.cn': 
Enumerating objects: 9, done.
Counting objects: 100% (9/9), done.
Delta compression using up to 2 threads
Compressing objects: 100% (6/6), done.
Writing objects: 100% (6/6), 830 bytes | 830.00 KiB/s, done.
Total 6 (delta 1), reused 0 (delta 0), pack-reused 0
To http://gitlab.x.xinghuihuyu.cn/flux-cd/flux-infra.git
   39f1943..49c26c9  main -> main

```

### 自动更新 GIt 提交状态

通知控制器的第二个用例是将提交消息直接更新到git的托管服务（gitlab，github）

![fluxcd-provider](.//img/fluxcd-provider.png)

```shell
root@master01:~# kubectl  create   secret    generic   gitlab-token  --from-literal=token="glpat-1KzP8K3TA7ij7TtsYEQh"   -n flux-system   
root@master01:~# flux   create    alert-provider   gitlab-app-demo   --type gitlab --address="http://gitlab.x.xinghuihuyu.cn/flux-cd/app-demo.git"  --secret-ref=gitlab-token   
✚ generating Provider
► applying Provider
✔ Provider created
◎ waiting for Provider reconciliation
✔ Provider gitlab-app-demo is ready
root@master01:~# flux  create  alert  vote-dev  --provider-ref=gitlab-app-demo --event-severity=info   --event-source=Kustomization/vote-dev     #只能使用Kustomization
✚ generating Alert
► applying Alert
✔ Alert created
◎ waiting for Alert reconciliation
✔ Alert vote-dev is ready

root@master01:~# flux  create  alert  redis-dev  --provider-ref=gitlab-app-demo --event-severity=info   --event-source=Kustomization/redis-dev   
✚ generating Alert
► applying Alert
✔ Alert created
◎ waiting for Alert reconciliation
✔ Alert redis-dev is ready
root@master01:~# flux  get alert
NAME                   	SUSPENDED	READY	MESSAGE     
mattermost-notification	False    	True 	Initialized	
redis-dev              	False    	True 	Initialized	
vote-dev               	False    	True 	Initialized	

```

![alter-gitlab](.//img/alter-gitlab.png)

然后将其导入到gitops存储库

```shell
root@master01:~/game/flux-infra/clusters/dev# flux  export  alert-provider   gitlab-app-demo       >gitlab-app-demo-provider.yaml 
root@master01:~/game/flux-infra/clusters/dev# flux   export   alert   vote-dev     >gitlab-vote-dev-alert.yaml  
root@master01:~/game/flux-infra/clusters/dev# flux   export   alert   redis-dev     >gitlab-redis-dev-alert.yaml  
root@master01:~/game/flux-infra/clusters/dev# ll
total 52
drwxr-xr-x 3 root root 4096 Nov  1 17:02 ./
drwxr-xr-x 4 root root   32 Oct 31 17:18 ../
-rw-r--r-- 1 root root  385 Oct 31 17:09 db-helmrelease.yaml
-rw-r--r-- 1 root root  188 Oct 31 17:07 db-helmrepository.yaml
drwxr-xr-x 2 root root   82 Oct 31 17:18 flux-system/
-rw-r--r-- 1 root root  249 Nov  1 17:00 gitlab-app-demo-provider.yaml
-rw-r--r-- 1 root root  249 Nov  1 17:02 gitlab-redis-dev-alert.yaml
-rw-r--r-- 1 root root  247 Nov  1 17:02 gitlab-vote-dev-alert.yaml
-rw-r--r-- 1 root root  272 Oct 31 17:18 instavote-gitrepository.yaml
-rw-r--r-- 1 root root  203 Nov  1 16:02 mattermost-alert-provider.yaml
-rw-r--r-- 1 root root  329 Nov  1 16:01 mattermost-notification-alert.yaml
-rw-r--r-- 1 root root  363 Oct 31 17:18 redis-kustomization.yaml
-rw-r--r-- 1 root root  329 Oct 31 18:12 result-helmrelease.yaml
-rw-r--r-- 1 root root   51 Oct 31 16:33 values.db.yaml
-rw-r--r-- 1 root root  393 Oct 31 17:18 vote-dev-kustomization.yaml
root@master01:~/game/flux-infra/clusters/dev# git add gitlab-*  
root@master01:~/game/flux-infra/clusters/dev# git commit    -m  "commit for gitlab notification "
[main 0c6291b] commit for gitlab notification
 3 files changed, 37 insertions(+)
 create mode 100644 clusters/dev/gitlab-app-demo-provider.yaml
 create mode 100644 clusters/dev/gitlab-redis-dev-alert.yaml
 create mode 100644 clusters/dev/gitlab-vote-dev-alert.yaml
root@master01:~/game/flux-infra/clusters/dev# git push   origin  main  
Username for 'http://gitlab.x.xinghuihuyu.cn': root
Password for 'http://root@gitlab.x.xinghuihuyu.cn': 
Enumerating objects: 10, done.
Counting objects: 100% (10/10), done.
Delta compression using up to 2 threads
Compressing objects: 100% (7/7), done.
Writing objects: 100% (7/7), 893 bytes | 446.00 KiB/s, done.
Total 7 (delta 2), reused 0 (delta 0), pack-reused 0
To http://gitlab.x.xinghuihuyu.cn/flux-cd/flux-infra.git
   49c26c9..0c6291b  main -> main

```

### Push-Based Reconciliation

<img src=".//img/基于推送的协调.png" alt="基于推送的协调" style="zoom:50%;" />

为了实现上边的流程，通知控制器使用Receiver CRD  

<img src=".//img/基于推送的协调2.png" alt="基于推送的协调2" style="zoom:50%;" />

### Exposing a Webhook Receiver

要想观测到基于推送的reconciliation ，需要将**pull reconciliation **间隔更改为更长的事件

```shell
#将下边的三个文件的时间间隔改为1h,然后推送到gitops存储库
root@master01:~/game/flux-infra/clusters/dev# ll  *kustomiza*
-rw-r--r-- 1 root root 363 Oct 31 17:18 redis-kustomization.yaml
-rw-r--r-- 1 root root 393 Oct 31 17:18 vote-dev-kustomization.yaml
root@master01:~/game/flux-infra/clusters/dev# ll  *gitrepo*
-rw-r--r-- 1 root root 272 Oct 31 17:18 instavote-gitrepository.yaml
root@master01:~/game/flux-infra/clusters/dev# vim instavote-gitrepository.yaml 
root@master01:~/game/flux-infra/clusters/dev# vim redis-kustomization.yaml 
root@master01:~/game/flux-infra/clusters/dev# vim vote-dev-kustomization.yaml 
```

需要暴露webhook-receiver  svc

```shell
root@master01:~/game/flux-infra/clusters/dev# kubectl    -n flux-system   get svc 
NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
notification-controller   ClusterIP   10.96.237.29    <none>        80/TCP    13d
source-controller         ClusterIP   10.109.35.120   <none>        80/TCP    13d
webhook-receiver          ClusterIP   10.111.125.72   <none>        80/TCP    13d
root@master01:~/game/flux-infra/clusters/dev# kubectl   -n flux-system  describe  workloadspreads.apps.kruise.io   
No resources found in flux-system namespace.
root@master01:~/game/flux-infra/clusters/dev# kubectl   -n flux-system  describe  svc  webhook-receiver  
Name:              webhook-receiver
Namespace:         flux-system
Labels:            app.kubernetes.io/component=notification-controller
                   app.kubernetes.io/instance=flux-system
                   app.kubernetes.io/part-of=flux
                   app.kubernetes.io/version=v2.1.2
                   control-plane=controller
                   kustomize.toolkit.fluxcd.io/name=flux-system
                   kustomize.toolkit.fluxcd.io/namespace=flux-system
Annotations:       <none>
Selector:          app=notification-controller
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.111.125.72
IPs:               10.111.125.72
Port:              http  80/TCP
TargetPort:        http-webhook/TCP
Endpoints:         192.168.140.78:9292
Session Affinity:  None
```

利用kustomization controller进行覆盖

```shell
root@master01:~/game/flux-infra/clusters/dev/flux-system# cat  kustomization.yaml 
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- gotk-components.yaml
- gotk-sync.yaml
root@master01:~/game/flux-infra/clusters/dev/flux-system# cat  > expose-webhook-receiver.yaml 
apiVersion: v1
kind: Service
metadata:
  name: webhook-receiver
  namespace: flux-system
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: http-webhook
    nodePort: 31234
  selector:
    app: notification-controller
  type: NodePortroot@master01:~/game/flux-infra/clusters/dev/flux-system# 
root@master01:~/game/flux-infra/clusters/dev/flux-system# 
root@master01:~/game/flux-infra/clusters/dev/flux-system# cat expose-webhook-receiver.yaml 
apiVersion: v1
kind: Service
metadata:
  name: webhook-receiver
  namespace: flux-system
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: http-webhook
    nodePort: 31234
  selector:
    app: notification-controller
  type: NodePortroot@master01:~/game/flux-infra/clusters/dev/flux-system# vim kustomization.yaml 
root@master01:~/game/flux-infra/clusters/dev/flux-system# vim kustomization.yaml 
root@master01:~/game/flux-infra/clusters/dev/flux-system# cat kustomization.yaml 
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
patches:
- path: expose-webhook-receiver.yaml
  target:
    kind: Service
    name: webhook-receiver
resources:
- gotk-components.yaml
- gotk-sync.yaml

```

接下来进行**reconciliation**

```shell
root@master01:~/game/flux-infra/clusters/dev/flux-system# flux    reconcile kustomization flux-system  
► annotating Kustomization flux-system in flux-system namespace
✔ Kustomization annotated
◎ waiting for Kustomization reconciliation
✗ Kustomization reconciliation failed: failed to decode Kubernetes YAML from /tmp/kustomization-4098093471/clusters/dev/values.db.yaml: missing Resource metadata <nil>
#存储gitops存储库的values.db.yaml文件（现在不需要了）
#接下来同步source和kustomization 
root@master01:~/game/flux-infra/clusters/dev/flux-system# flux   reconcile     source git     flux-system   
► annotating GitRepository flux-system in flux-system namespace
✔ GitRepository annotated
◎ waiting for GitRepository reconciliation
✔ fetched revision main@sha1:cd0fdd68a3cebf8ba77dea6cb9d17772ca967e78
root@master01:~/game/flux-infra/clusters/dev/flux-system# flux   reconcile      kustomization      flux-system   
► annotating Kustomization flux-system in flux-system namespace
✔ Kustomization annotated
◎ waiting for Kustomization reconciliation
✔ applied revision main@sha1:cd0fdd68a3cebf8ba77dea6cb9d17772ca967e78
root@master01:~/game/flux-infra/clusters/dev/flux-system# kubectl  -n flux-system   get svc  
NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
notification-controller   ClusterIP   10.96.237.29    <none>        80/TCP         13d
source-controller         ClusterIP   10.109.35.120   <none>        80/TCP         13d
webhook-receiver          NodePort    10.111.125.72   <none>        80:31234/TCP   13d

```

### 使用webhook将Flux 与Gitlab集成

Receiver可以接受来自helm存储库或者gitlab的webhook(一个接收器可以接受多个类型的webhook)



```shell
root@master01:~#  WEBHOOK_TOKEN=`date   |md5sum  |cut -d ' ' -f1`
root@master01:~# echo $WEBHOOK_TOKEN
e16a4756551effe4f240aed9555d34c8
root@master01:~# kubectl  create   secret  generic webhook-token   --from-literal=token=$WEBHOOK_TOKEN  -n flux-system 
secret/webhook-token created
root@master01:~#   flux create receiver gitlab-receiver-app-demo \
> --type gitlab \
> --event "Push Hook" \
> --event "Tag Push Hook" \
> --secret-ref webhook-token \
> --resource GitRepository/app-demo \
>     --export  
---
apiVersion: notification.toolkit.fluxcd.io/v1
kind: Receiver
metadata:
  name: gitlab-receiver-app-demo
  namespace: flux-system
spec:
  events:
  - Push Hook
  - Tag Push Hook
  resources:
  - kind: GitRepository
    name: app-demo
  secretRef:
    name: webhook-token
  type: gitlab
root@master01:~#   flux create receiver gitlab-receiver-app-demo \
> --type gitlab \
> --event "Push Hook" \
> --event "Tag Push Hook" \
> --secret-ref webhook-token \
> --resource GitRepository/app-demo  
✚ generating Receiver
► applying Receiver
✔ Receiver created
◎ waiting for Receiver reconciliation
✔ Receiver gitlab-receiver-app-demo is ready
✔ generated webhook URL /hook/d4314f85d6d9c2ef538a071a619cc4e747a1d984c7fa3a717d1b4adfa54e3f2c
root@master01:~# flux  get  receivers  
NAME                    	SUSPENDED	READY	MESSAGE                                                                                               
gitlab-receiver-app-demo	False    	True 	Receiver initialized for path: /hook/d4314f85d6d9c2ef538a071a619cc4e747a1d984c7fa3a717d1b4adfa54e3f2c	

```

**NOTE:** *有效负载URL http://<NODEIP>:<NODEPORT>/RECEIVER_PATH*

所以真实的URL为http://12.0.0.20:31234/hook/b1173dac6d64936b4ebad49d14098af7d97253545ffdd5402186756aac041573

TOKEN 为 e16a4756551effe4f240aed9555d34c8

接下来在gitlab（app-demo）进行配置webhook,并修改文件的值进行测试

![push](.//img/push.png)

![push2](.//img/push2.png)

现在已经开始了部署

### 使用 Prometheus 和 Grafana 进行监控

prometheus和grafana之前已经是部署了的，所以这里不在叙述，接下来使用flux官方推介的示例进行演示。请[参考](https://github.com/fluxcd/flux2-monitoring-example)

```shell
#首先创建真正的有关部署的yaml文件(具体的内容，官方示例都有)
root@master01:~/coo/app-demo/deploy# tree  prometheus/ 
prometheus/
└── test
    ├── dashboards
    │   ├── cluster.json
    │   ├── control-plane.json
    │   └── logs.json
    ├── kustomization.yaml
    └── podmonitor.yaml

2 directories, 5 files
#接下来创建gitops的kustomization
root@master01:~/coo/flux-infra/clusters/dev# flux create kustomization  monitoring-configs  --source=app-demo --path="./deploy/prometheus/test" --prune=true --interval=1h  --retry-interval=2m   --wait=true  --timeout=5m
#然后导出到一个文件中
root@master01:~/coo/flux-infra/clusters/dev# flux create kustomization  monitoring-configs  --source=app-demo --path="./deploy/prometheus/test" --prune=true --interval=1h  --retry-interval=2m   --wait=true  --timeout=5m  --export    >monitoring-kustomization.yaml
root@master01:~/coo/flux-infra/clusters/dev# cat  monitoring-kustomization.yaml  
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: monitoring-configs
  namespace: flux-system
spec:
  interval: 1h0m0s
  path: ./deploy/prometheus/test
  prune: true
  retryInterval: 2m0s
  sourceRef:
    kind: GitRepository
    name: app-demo
  timeout: 2m0s
  wait: true
#然后使用git push推送到gitops存储库
```

这就完成了对flux组件状态的监控

## Tekton

包括三个组件：pipelines Triggers   Dashboard

Tekton 是一个 Kubernetes 原生 CI/CD 框架，工程师和开发人员使用它来构建构建、测试代码并将其部署到云或本地 Kubernetes 集群的工作流程。

<img src=".//img/tekton简介.png" alt="tekton简介" style="zoom:50%;" />

Tekton 包含五个主要概念：

- **Steps:** 步骤是使用 Tekton 构建工作流程的最小单位。 arguments、images和command等块元素都定义在步骤中.

- **Tasks:** 任务元素是按顺序表达的步骤组合

  如果你想运行一次性作业或者临时作业，主需要创建一个任务，并通过TaskRun实例化他

- **Pipelines:**管道是按顺序排列的任务组合。它可以根据使用情况设置为并发运行。还可以指定输入、输出和工作流参数

  如果你必须按特定的顺序运行多个作业（构建--测试--部署），你可以创建一个pipeline

- **TaskRuns:**顾名思义，任务运行元素就是将特定任务实例化。它还指定了任务运行 Git 仓库所需的详细信息以及container registry信息

- **PipelineRuns:**管道运行是层次结构中的最后一个元素。与 TaskRun 类似，它实例化了特定的管道元素。它还指定了所需的runtime信息，如 Git 仓库和container registry(相当于是在pipiline 模板中插入值，也就是说使用一个模板，常见两个PipelineRuns)

  <img src=".//img/tekton实例化.png" alt="tekton实例化" style="zoom:50%;" />

### Tekton 管道的工作原理

在tokton的管道执行中，每个任务都作为Pod和作业运行

每个作业是有一个工作区的，我们可以使用共享卷的概念为每个pod提供工作区（一个管道的多个任务有多个pod,共享存储很重要）

为作业提供注解信息，这些注解通常以文件的形式作为任务的输入，任务也许会向工作区输入一些内容。任务完成之后，pod被标记为已经完成的，默认情况下是不会删除的。对于第二个任务，运行另一个pod，如果该任务包含多个步骤（会创建多个容器）

<img src=".//img/tekton执行原理.png" alt="tekton执行原理" style="zoom:50%;" />

### Install

有关安装的部分，请参考官方文档

对于CLI 的安装，推介deb包

### CI Pipeline and  Pipelinerun

<img src=".//img/ci pipeline.png" alt="ci pipeline" style="zoom:50%;" />

![pipelinerun](.//img/pipelinerun.png)

## Image Automation Workflow

通过上一步的Tekton完成了CI的流程，那么它如何与Flux CD结合呢：每当Token CI管道运行时，他会生成容器镜像(带有新的标签)，FLUX CD必须有能力监控容器注册表的新的镜像，然后获取他并更新kustomization（必须有一种能力来更新git 存储库的清单）,以便将其部署到k8s环境

### Image reflector and automation controllers

![镜像自动化工作流程](.//img/镜像自动化工作流程.png)

image-reflector-controller 和 image-automation-controller 协同工作，在新容器镜像可用时更新 Git 仓库。

+ image-reflector-controller

  扫描容器注册表，并在该容器注册表中扫描具有特定标签模式的存储库，以获取该特定的存储库

  每当添加新镜像的时候，还会通过flux将其反映或更新到k8s存储的镜像元数据中（这个是image-reflector-controller和image-automation-controller的**协调点**，mage-automation-controller也在监视着镜像的元数据）

+ image-automation-controller

  根据扫描到的最新图像更新 YAML 文件，并将更改提交到指定的 Git 仓库 ，已共kustomizaton读取

  （使用yaml存储库中设置的某些标记来更新某些行，一旦更新进入git 仓库，部署就开始进行（源控制器将找到更新的信息，然后kustomization控制器或者helm控制器（基于你使用的源）将检测该更改，并于k8s环境进行协调））

#### 容器注册表扫描

要使用image-reflector和image-automation必须使用"额外组件"再次运行引导程序

```shell
flux bootstrap git \
  --url=ssh://git@gitlab.x.xinghuihuyu.cn/flux-cd/flux-infra \
  --branch=main \
  --private-key-file=/root/.ssh/id_rsa \
  --path=clusters/dev \
  --log-level=debug \
  --network-policy=false \
  --components-extra=image-reflector-controller,image-automation-controller
  
root@master01:~/prepare/18-flux-cd/05-image-automation-workflow# kubectl apply  -f  01-imagerepository-secret.yaml  
secret/image-repository created
root@master01:~/prepare/18-flux-cd/05-image-automation-workflow# cat  01-imagerepository-secret.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: image-repository
  namespace: flux-system
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: ewoJImF1dGhzIjogewoJCSJoYXJib3IuZm9yY2Vjcy5jb206MzI0MTUiOiB7CgkJCSJhdXRoIjogIllXUnRhVzQ2U0dGeVltOXlNVEl6TkRVPSIKCQl9Cgl9Cn0=
  

flux create secret tls harbor-certs \
  --namespace=flux-system \
  --tls-crt-file=./client.crt \
  --tls-key-file=./client.key \
  --ca-crt-file=./ca.crt \
  --export > harbor-certs.yaml


flux create image repository vote  --secret-ref=image-repository   --cert-ref   harbor-certs  --image=harbor.forcecs.com:32415/vote/vote --interval=1m
```

接下来定义镜像扫描策略

```shell
flux create image policy vote \
--image-ref=vote \
--select-numeric=asc \
--filter-regex='^main-[a-f0-9]+-(?P<ts>[0-9]+)' \
--filter-extract='$ts'

root@master01:~/prepare/18-flux-cd/05-image-automation-workflow# flux   get   images   policy   
NAME	LATEST IMAGE                                               	READY	MESSAGE                                                                                        
vote	harbor.forcecs.com:32415/vote/vote:main-646fa717-1699257173	True 	Latest image tag for 'harbor.forcecs.com:32415/vote/vote' resolved to main-646fa717-1699257173	

#触发生成新的镜像，再次观察（flux   get   images   policy   ）
kubectl apply  -f   ../04-app-demo-tekton-ci/04-instavote-ci-pipeline.yaml 
kubectl create  -f   ../04-app-demo-tekton-ci/05-vote-ci-pipelinerun.yaml 
NAME	LATEST IMAGE                                               	READY	MESSAGE                                                                                                                     
vote	harbor.forcecs.com:32415/vote/vote:main-646fa717-1699328554	True 	Latest image tag for 'harbor.forcecs.com:32415/vote/vote' updated from main-646fa717-1699257173 to main-646fa717-1699328554	
```

#### 设置镜像更新规则

完成flux与Git 存储库通信并更新镜像，这一步分成两个部分

+ 更新部署仓库（app-demo）的清单，并使用新的镜像（因此，必须在图像名称前边加一个占位符，以使flux知道更新哪一行代码）

  ```shell
  root@master01:~/coo/app-demo/deploy/vote/dev# tree 
  .
  ├── deployment.yaml
  ├── kustomization.yaml
  └── options.env
  
  0 directories, 3 files
  root@master01:~/coo/app-demo/deploy/vote/dev# cat  kustomization.yaml 
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
  - ../base
  patches:
  - path: deployment.yaml
    target:
      kind: Deployment
      name: vote
  
  replicas:
  - name: vote
    count: 1
  
  images:
  - name: schoolofdevops/vote   # {"$imagepolicy": "flux-system:vote"}
    newTag: v7    
  
  namespace: instavote
  
  configMapGenerator:
  - name: vote-options
    envs:
    - options.env
  
  #这一步可以忽略，我们之前创建过GitRepository中所使用的app-demo  然后执行 flux   reconcile   kustomization  flux-system    --with-source
  flux create secret git gitlab-auth \
    --url=http://gitlab.x.xinghuihuyu.cn/flux-cd/app-demo.git \
    --username=root \
    --password=basic123   -n flux-system
    
  
  
  flux create image update instavote-all \
  --interval=30m \
  --git-repo-ref=app-demo \
  --git-repo-path="./deploy/vote/dev" \
  --checkout-branch=main \
  --push-branch=main \
  --author-name=wukui \
  --author-email=kuiinative@gmail.com \
  --commit-template="{{range .Updated.Images}}{{println .}}{{end}}" \
  --export > instavote-all-automation.yaml
  
  再次更新一个镜像
  kubectl create  -f   ../04-app-demo-tekton-ci/05-vote-ci-pipelinerun.yaml 
  
  root@master01:~/prepare/18-flux-cd/04-app-demo-tekton-ci# flux get images policy  
  NAME	LATEST IMAGE                                               	READY	MESSAGE                                                                                                                     
  vote	harbor.forcecs.com:32415/vote/vote:main-31357ded-1699343111	True 	Latest image tag for 'harbor.forcecs.com:32415/vote/vote' updated from main-646fa717-1699328554 to main-31357ded-1699343111	
  
  
  
  #然后执行部署（我们在之前步骤中改为了基于push的工作模型，这一步也可以省略）
  root@master01:~/prepare/18-flux-cd/04-app-demo-tekton-ci# flux    reconcile  kustomization   vote-dev    --with-source  
  
  root@master01:~/prepare/18-flux-cd/04-app-demo-tekton-ci# kubectl get pods  -n instavote 
  NAME                                READY   STATUS    RESTARTS   AGE
  instavote-db-postgres-0             1/1     Running   0          6d23h
  instavote-result-7b7ff78678-whn7k   1/1     Running   0          6d21h
  redis-78d4b8b77c-xqss7              1/1     Running   0          7d23h
  vote-6d7d55ff5-284dg                1/1     Running   0          17m
  
  ```

  **说明:** **更新整个镜像image: sofd/vote:v8 # {“$imagepolicy”: “flux-system:vote”}**

  ​          **Tag的更新newTag: v8 # {"$imagepolicy": "flux-system:vote:tag"}**

  ![image policy](.//img/image policy.png)

对于result是同样的步骤（没有进行测试，仅供参考），这里需要指出的是如果使用helm进行发布，必须更改chart.yaml中的**version**

```shell
flux create image repository result  --secret-ref=image-repository   --cert-ref   harbor-certs  --image=harbor.forcecs.com:32415/result/result --interval=1m   --export  >result-image-repository.yaml


flux create image policy vote \
--image-ref=result \
--select-numeric=asc \
--filter-regex='^main-[a-f0-9]+-(?P<ts>[0-9]+)' \
--filter-extract='$ts'   --export >result-image-policy.yaml


flux create image update result \
--interval=30m \
--git-repo-ref=app-demo \
--git-repo-path="./deploy/charts/result" \
--checkout-branch=main \
--push-branch=main \
--author-name=wukui \
--author-email=kuiinative@gmail.com \
--commit-template="{{range .Updated.Images}}{{println .}}{{end}}" \
--export > result-automation.yaml


#然后修改文件
root@master01:~/coo/app-demo/deploy/charts/result# cat values.yaml 
# Default values for result.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1
image:
  repository: dopsdemo/result
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "main-e8b357e3-1619185072" # {"$imagepolicy": "flux-system:result:tag"}
imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""
serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # Annotations to add to the service account
  annotations: {}

```

+ 然后将代码提交到Gitops存储库

  ```shell
  flux  export   image  repository   vote    |tee    vote-image-repository.yaml  
  flux  export   image  policy   vote      |tee    vote-image-policy.yaml  
  flux  export   image  update    instavote-all      |tee    vote-image-update.yaml  
  root@master01:~/coo/flux-infra/clusters/dev# tree 
  .
  ├── db-helmrelease.yaml
  ├── db-helmrepository.yaml
  ├── flux-system
  │   ├── expose-webhook-receiver.yaml
  │   ├── gotk-components.yaml
  │   ├── gotk-sync.yaml
  │   └── kustomization.yaml
  ├── gitlab-app-demo-provider.yaml
  ├── gitlab-redis-dev-alert.yaml
  ├── gitlab-vote-dev-alert.yaml
  ├── instavote-gitrepository.yaml
  ├── mattermost-alert-provider.yaml
  ├── mattermost-notification-alert.yaml
  ├── monitoring-kustomization.yaml
  ├── redis-kustomization.yaml
  ├── result-helmrelease.yaml
  ├── vote-dev-kustomization.yaml
  ├── vote-image-policy.yaml
  ├── vote-image-repository.yaml
  └── vote-image-update.yaml
  
  ```



### CI/CD的遗留问题

<img src=".//img/cicd中的问题.png" alt="cicd中的问题" style="zoom:50%;" />



在我们之前的讲解中，我们的app-demo项目有源代码目录（包括Dcokerfile文件），也有部署代码（基于kustomize），现在遗留的CI部分的问题是，假如源代码发生了变化，如何通过webhook自动触发Tekton CI的执行（现在Tekton能够接受webhook，gitlab能够发送webhook）

现在有这样一个问题：我们的程序源代码和部署代码在同一个存储库中，容器注册表的更新导致了仓库的**新提交**，这就导致了一直处于循环状态（这就是为什么没有实现自动化CI）所以一种简单的解决问题的方式是拆分源代码存储库和部署存储库

## Flux实现多租户

<img src=".//img/flux多租户.png" alt="flux多租户" style="zoom:50%;" />

目前我们必须开发人员，运维工程师，SRE提供对flux-infra存储库的访问权限，因为我们的flux同步清单可能由开发人员贡献，他们将参与其中的更新和维护，然而此存储库中也有集群的配置（在集群上安装哪些控制器，CRD 组件，以及不止有一个集群），并且假设启动一个新的项目，本质上是使用的同一个存储库，那么如何解决这个问题呢？

### Flux的多租户设计

<img src=".//img/多租户设计.png" alt="多租户设计" style="zoom:50%;" />

现在是这样的：

+ 1）我们的应用程序源代码在app存储库 
+ 2）部署代码（kustomization，helm）和flux同步清单（kustomizations Gitrepositories Helmreleases Helmreposities 及其告警通知 镜像自动更新等 ）在deploy存储库 
+ 3）使用fleet存储库设置多租户，projects目录来引入多租户，其中一个子目录（例如instavote）与flux同步清单关联。这个子目录中可以设置在哪个名称空间中运行。现在假设对于staging基础设施，需要加入哪些项目，可以在其下添加一个列表

### 代码组织策略

你将维护三个不同的存储库

- Flux Fleet
- Project Deployment (tenant)
- Project/App Source.

**Flux Fleet Repository **用于使用 FluxCD 管理整个组织向 Kubernetes 的部署。该资源库将包含

- 所有环境（暂存、生产等）的群集配置。您可以通过指向该资源库来引导群集。
- 群集范围内的基础设施组件（e.g., Ingress Controllers, Repository Secrets, etc）
- 项目管理配置，也称为**tenant**规范。该规范将指向每个租户的项目存储库，获取同步清单并进行部署。

flux-fleet 资源库通常由 SRE 管理

The **Project Deployment Repository (Tenant****)** would contain:

- The Flux sync/deployment code. For example, GitRepository, HelmRepository, Kustomization (FluxCD deployments), HelmRelease, Helm charts source code, YAML manifests as either plain YAML or with Kustomize overlays.

该项目资源库是群集上的一个租户

The **Project/App Source Repository **contains the application's source code. It is self-explanatory.

- Contains Source Code + Dockerfile, etc.

### 多租户管理流程

\1. SREs Setup Fleet Management of Kubernetes clusters by:

- Creating the flux-fleet repository with:
  \- Cluster definitions (e.g. staging, QA, production)
  \- Cluster-wide infrastructure deployment code
  \- Projects (Tenants) onboarding scaffold
- Bootstrapping clusters with:
  \- Git source Integration with the flux-fleet repository
  \- Flux controllers
  \- CRDs
  \- Policies: RBAC, Network Policy, etc.
  \- Git source repository integration

\2. Developers/SREs create the deployment repository with:

- Flux Sync/Deployment manifests
- YAML Deployment + Kustomize Overlays
- Helm Charts

\3. Project owners raise an onboarding request:

- Provide the project deployment repository

\4. SREs onboard the project by

- Generating the tenant RBAC
- Project onboarding manifests

\5. FluxCD reconciles the cluster state by

- Syncing the project deployment code
- Generating Flux resources for the project (e.g., GitRepository, Kustomization, HelmRelease)
- Running reconciliation for each of the Flux resources with the actual Kubernetes cluster.

### 导出同步清单

在设置多租户环境之前，必须导出yaml格式的所有可用内容，以便为新租户使用

```shell
root@master01:~/flux-cd/flux-infra/clusters/dev# ll
total 64
drwxr-xr-x 4 root root 4096 Nov  8 10:31 ./
drwxr-xr-x 4 root root   32 Nov  7 16:52 ../
-rw-r--r-- 1 root root  385 Nov  7 16:52 db-helmrelease.yaml
-rw-r--r-- 1 root root  188 Nov  7 16:52 db-helmrepository.yaml
drwxr-xr-x 2 root root  118 Nov  7 16:52 flux-system/
-rw-r--r-- 1 root root  307 Nov  7 19:12 gitlab-receiver-app-demo.yaml
-rw-r--r-- 1 root root  249 Nov  7 16:52 gitlab-redis-dev-alert.yaml
-rw-r--r-- 1 root root  247 Nov  7 16:52 gitlab-vote-dev-alert.yaml
-rw-r--r-- 1 root root  271 Nov  7 16:52 instavote-gitrepository.yaml
-rw-r--r-- 1 root root  203 Nov  7 16:52 mattermost-alert-provider.yaml
-rw-r--r-- 1 root root  329 Nov  7 16:52 mattermost-notification-alert.yaml
-rw-r--r-- 1 root root  308 Nov  7 16:52 monitoring-kustomization.yaml
-rw-r--r-- 1 root root  361 Nov  7 16:52 redis-kustomization.yaml
-rw-r--r-- 1 root root  329 Nov  7 16:52 result-helmrelease.yaml
drwxr-xr-x 2 root root   64 Nov  8 10:31 secrets/
-rw-r--r-- 1 root root  391 Nov  7 16:52 vote-dev-kustomization.yaml
-rw-r--r-- 1 root root  274 Nov  7 16:54 vote-image-policy.yaml
-rw-r--r-- 1 root root  316 Nov  7 16:54 vote-image-repository.yaml
-rw-r--r-- 1 root root  508 Nov  7 16:54 vote-image-update.yaml
root@master01:~/flux-cd/flux-infra/clusters/dev# tree  secrets/
secrets/
├── default-secret.yaml
└── flux-system-secret.yaml

0 directories, 2 files

```

其中需要说明的是secret这是在本地的环境，并没有同步到远程仓库

### 清理环境

```shell
#这是我们之前的环境
root@master01:~/flux-cd/flux-infra/clusters/dev# kubectl get all   -n  instavote 
NAME                                    READY   STATUS    RESTARTS   AGE
pod/instavote-db-postgres-0             1/1     Running   0          7d17h
pod/instavote-result-7b7ff78678-whn7k   1/1     Running   0          7d16h
pod/redis-78d4b8b77c-xqss7              1/1     Running   0          8d
pod/vote-6d7d55ff5-284dg                1/1     Running   0          18h

NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/db                 ClusterIP   10.97.34.2      <none>        5432/TCP       7d17h
service/instavote-result   NodePort    10.97.61.44     <none>        80:31912/TCP   7d16h
service/redis              ClusterIP   10.96.218.132   <none>        6379/TCP       8d
service/vote               NodePort    10.107.91.115   <none>        80:30300/TCP   8d

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/instavote-result   1/1     1            1           7d16h
deployment.apps/redis              1/1     1            1           8d
deployment.apps/vote               1/1     1            1           8d

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/instavote-result-7b7ff78678   1         1         1       7d16h
replicaset.apps/redis-78d4b8b77c              1         1         1       8d
replicaset.apps/vote-6477985488               0         0         0       19h
replicaset.apps/vote-655568bfdb               0         0         0       6d17h
replicaset.apps/vote-66587d7b74               0         0         0       6d18h
replicaset.apps/vote-6769d8c5b7               0         0         0       8d
replicaset.apps/vote-694755cb87               0         0         0       6d
replicaset.apps/vote-6bdb4777b4               0         0         0       6d
replicaset.apps/vote-6d7d55ff5                1         1         1       18h
replicaset.apps/vote-7977f468d8               0         0         0       8d
replicaset.apps/vote-7bcbd8955f               0         0         0       6d18h
replicaset.apps/vote-84c6677b64               0         0         0       6d
replicaset.apps/vote-f4cc984b                 0         0         0       8d

NAME                                     READY   AGE
statefulset.apps/instavote-db-postgres   1/1     7d17h

#卸载与flux有关的所有内容
root@master01:~/flux-cd/flux-infra/clusters/dev# flux   uninstall  --namespace=flux-system   --dry-run

#然后删除通过flux之前部署的应用
root@master01:~/flux-cd/flux-infra/clusters/dev# kubectl delete  ns instavote    
namespace "instavote" deleted

#然后清理tekton pr已经完成的pod
root@master01:~/flux-cd/flux-infra/clusters/dev# tkn  pr delete --all  
Are you sure you want to delete all PipelineRuns in namespace "default" (y/n): y
All PipelineRuns(Completed) deleted in namespace "default"

```

### 多租户示例说明

**facebooc-deploy**：

+ 对于facebooc-deploy存储库来讲，kustomize就是实际的部署代码所在的地方，目录结构根据kustomize的设计模式进行定义，flux目录是为这个名为*faceboocd*特定的应用程序创建的flux同步清单
+ 在kustomize中可以创建另外的目录（production环境），通过kustomize overlay实现对base的覆盖
+ flux目录下的结构与kustomize目录是相识的

**flux-fleet**：

+ Cluster目录结构表示，在staging环境中指定可以关联的租户或者项目(sourceRef:表示具体的flux fleet存储库)

  ```yaml
  root@master01:~/tenant/flux-fleet-main/clusters/staging# cat onboard-projects.yaml 
  ---
  apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
  kind: Kustomization
  metadata:
    name: projects
    namespace: flux-system
  spec:
    interval: 10m0s
    path: ./projects/staging  @1
    prune: true
    sourceRef:
      kind: GitRepository     
      name: flux-system       @2控制着@1，@1又关联到了deploy存储库
    validation: client
  
  ```

  

+ 其中对于staging项目也使用了kustomize overlay的设计模式

  + 1）**Flux CD 管理的flux sync**在哪找（如下）以及一些其他的自己加入的配置，例如RBAC（设置每个租户的权限，例如制定了名称空间后，facebooc-deploy只能部署到这个名称空间）  SVC

    ```yaml
    root@master01:~/flux-cd/flux-infra/clusters/dev/flux-system# cat gotk-sync.yaml 
    # This manifest was generated by flux. DO NOT EDIT.
    ---
    apiVersion: source.toolkit.fluxcd.io/v1
    kind: GitRepository
    metadata:
      name: flux-system
      namespace: flux-system
    spec:
      interval: 1m0s
      ref:
        branch: main
      secretRef:
        name: flux-system
      url: ssh://git@gitlab.x.xinghuihuyu.cn/flux-cd/flux-infra
    ---
    apiVersion: kustomize.toolkit.fluxcd.io/v1
    kind: Kustomization
    metadata:
      name: flux-system
      namespace: flux-system
    spec:
      interval: 10m0s
      path: ./clusters/dev
      prune: true
      sourceRef:
        kind: GitRepository
        name: flux-system
    ```

  + 2）Flux 控制器资源清单

### 示例租户集群引导

```shell
flux bootstrap git \
  --url=ssh://git@gitlab.x.xinghuihuyu.cn/multi-tenant/flux-fleet \
  --branch=main \
  --private-key-file=/root/.ssh/id_rsa \
  --path=./clusters/staging \
  --log-level=debug \
  --network-policy=false \
  --components-extra=image-reflector-controller,image-automation-controller
```

需要注意的是有四个kustomization:

+ flux-system名称空间下有

  + flux-system 用来指定集群上下文

  + projects       用来指定租户的上下文

+ facebooc名称空间下有 
  + facebooc-deploy用来指定flux sync的上下文
  + facebooc 用来执行部署清单的上下文（只有一个是在**facebooc-deploy存储库**定义的）

### 迁移之前的instavote项目

在之前的没有租户概念的讲述下：1）app-demo为部署存储库，同时也有源代码 2）flux-infra存储库有flux sync清单 也有集群的引导配置

我们将用以上的两个仓库进行合并

+ 生成新的instavote-deploy存储库

```shell
1.迁移app-demo中的部署清单
root@master01:~/tenant# tree instavote-deploy/
instavote-deploy/
├── README.md
├── flux
│   ├── base
│   │   └── README.md
│   ├── production
│   │   ├── README.md
│   │   └── kustomization.yaml
│   └── staging
│       ├── README.md
│       └── kustomization.yaml
├── helm
│   └── charts
│       └── README.md
└── kustomize
    └── README.md

7 directories, 8 files
root@master01:~/tenant# cp -r  ./app-demo/
.git/   deploy/ result/ vote/   
root@master01:~/tenant# cp -r  ./app-demo/deploy/charts/*   ./instavote-deploy/helm/charts/

root@master01:~/tenant/app-demo/deploy# cp -r prometheus   ../../instavote-deploy/

root@master01:~/tenant/app-demo/deploy# cp -r redis vote   ../../instavote-deploy/kustomize/
root@master01:~/tenant/app-demo/deploy# tree ../../instavote-deploy/kustomize/ 
../../instavote-deploy/kustomize/
├── README.md
├── redis
│   ├── base
│   │   ├── deployment.yaml
│   │   ├── kustomization.yaml
│   │   └── service.yaml
│   ├── dev
│   │   └── kustomization.yaml
│   └── staging
│       └── kustomization.yaml
└── vote
    ├── base
    │   ├── deployment.yaml
    │   ├── kustomization.yaml
    │   └── service.yaml
    ├── dev
    │   ├── deployment.yaml
    │   ├── kustomization.yaml
    │   └── options.env
    └── staging
        ├── deployment.yaml
        ├── kustomization.yaml
        └── service.yaml
root@master01:~/tenant/instavote-deploy# ll
total 12
drwxr-xr-x 7 root root   94 Nov  8 17:06 ./
drwxr-xr-x 8 root root  121 Nov  8 16:52 ../
drwxr-xr-x 8 root root  185 Nov  8 16:37 .git/
-rw-r--r-- 1 root root 9502 Nov  8 16:34 README.md
drwxr-xr-x 5 root root   51 Nov  8 16:34 flux/
drwxr-xr-x 3 root root   20 Nov  8 16:34 helm/
drwxr-xr-x 4 root root   48 Nov  8 17:07 kustomize/
drwxr-xr-x 3 root root   18 Nov  8 17:06 prometheus/

2.迁移flux sync同步清单
root@master01:~/tenant/instavote-deploy/flux/base# cp /root/tenant/flux-infra/clusters/dev/*.yaml   ./
root@master01:~/tenant/instavote-deploy/flux/base# ll
total 72
drwxr-xr-x 2 root root 4096 Nov  8 17:25 ./
drwxr-xr-x 5 root root   51 Nov  8 16:34 ../
-rw-r--r-- 1 root root  314 Nov  8 16:34 README.md
-rw-r--r-- 1 root root  385 Nov  8 17:25 db-helmrelease.yaml
-rw-r--r-- 1 root root  188 Nov  8 17:25 db-helmrepository.yaml
-rw-r--r-- 1 root root  249 Nov  8 17:25 gitlab-app-demo-provider.yaml
-rw-r--r-- 1 root root  307 Nov  8 17:25 gitlab-receiver-app-demo.yaml
-rw-r--r-- 1 root root  249 Nov  8 17:25 gitlab-redis-dev-alert.yaml
-rw-r--r-- 1 root root  247 Nov  8 17:25 gitlab-vote-dev-alert.yaml
-rw-r--r-- 1 root root  271 Nov  8 17:25 instavote-gitrepository.yaml
-rw-r--r-- 1 root root  203 Nov  8 17:25 mattermost-alert-provider.yaml
-rw-r--r-- 1 root root  329 Nov  8 17:25 mattermost-notification-alert.yaml
-rw-r--r-- 1 root root  308 Nov  8 17:25 monitoring-kustomization.yaml
-rw-r--r-- 1 root root  361 Nov  8 17:25 redis-kustomization.yaml
-rw-r--r-- 1 root root  329 Nov  8 17:25 result-helmrelease.yaml
-rw-r--r-- 1 root root  391 Nov  8 17:25 vote-dev-kustomization.yaml
-rw-r--r-- 1 root root  274 Nov  8 17:25 vote-image-policy.yaml
-rw-r--r-- 1 root root  316 Nov  8 17:25 vote-image-repository.yaml
-rw-r--r-- 1 root root  508 Nov  8 17:25 vote-image-update.yaml

然后执行
root@master01:~/tenant/instavote-deploy/flux/base# kustomize  create  --autodetect

3.执行之前导出的secret(需要替换名称空间，但是有可能存在问题)
4.将之前gitops存储库flux-infra的exposeyam迁移

```

### 实施另一个租户（项目）

```shell
1.创建新租户的RBAC
root@master01:~/tenant/flux-fleet/projects# tree
.
├── base
│   └── facebooc
│       ├── facebooc-deploy-gitrepository.yaml
│       ├── facebooc-deploy-kustomization.yaml
│       ├── kustomization.yaml
│       └── rbac.yaml
└── staging
    ├── facebooc-deploy-kustomization.yaml
    └── kustomization.yaml

3 directories, 6 files
root@master01:~/tenant/flux-fleet/projects# cd base/
root@master01:~/tenant/flux-fleet/projects/base# mkdir  instavote  
root@master01:~/tenant/flux-fleet/projects/base# flux  create   tenant   instavote   --with-namespace=instavote   --export   >instavote/rbac.yaml 

2.添加对部署存储库的引用


root@master01:~/tenant/flux-fleet/projects/base# kubectl create   secret   generic   instavote-deploy-repo     --from-literal=username=root --from-literal=password=basic123    -n instavote 
secret/instavote-deploy-repo created

flux create source git instavote-deploy \
--namespace=instavote \
--url=http://gitlab.x.xinghuihuyu.cn/multi-tenant/instavote-deploy.git \
--branch=main \
--export > instavote/instavote-deploy-gitrepository.yaml


flux create kustomization instavote-deploy \
--namespace=instavote \
--service-account=instavote \
--source=GitRepository/instavote-deploy \
--path="./flux" \
--export > instavote/instavote-deploy-kustomization.yaml

root@master01:~/tenant/flux-fleet/projects/base# cd   instavote/
root@master01:~/tenant/flux-fleet/projects/base/instavote# ll
total 12
drwxr-xr-x 2 root root 109 Nov  9 11:08 ./
drwxr-xr-x 4 root root  39 Nov  9 10:16 ../
-rw-r--r-- 1 root root 286 Nov  9 11:06 instavote-deploy-gitrepository.yaml
-rw-r--r-- 1 root root 274 Nov  9 11:08 instavote-deploy-kustomization.yaml
-rw-r--r-- 1 root root 677 Nov  9 10:18 rbac.yaml
root@master01:~/tenant/flux-fleet/projects/base/instavote# kustomize   create  autodecete    #可能需要手动需改
root@master01:~/tenant/flux-fleet/projects/base/instavote# cat  instavote-deploy-kustomization.yaml 
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: instavote-deploy
  namespace: instavote
spec:
  interval: 1m0s
  path: ./flux                               #@1首先引用instavote-deploy存储库的flux sync清单（以后在project下使用overlay覆盖关于某个应用的清单）
  prune: false
  serviceAccountName: instavote
  sourceRef:
    kind: GitRepository
    name: instavote-deploy

root@master01:~/tenant/flux-fleet/projects/staging# cp   facebooc-deploy-kustomization.yaml   instavote-deploy-kustomization.yaml
root@master01:~/tenant/flux-fleet/projects/staging# vim instavote-deploy-kustomization.yaml 
root@master01:~/tenant/flux-fleet/projects/staging# cat instavote-deploy-kustomization.yaml 
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: instavote-deploy
  namespace: instavote
spec:
  path: ./flux/staging

root@master01:~/tenant/flux-fleet/projects/staging# vim kustomization.yaml 
root@master01:~/tenant/flux-fleet/projects/staging# cat kustomization.yaml 
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../base/facebooc
- ../base/instavote                                                                     
patches:
- path: facebooc-deploy-kustomization.yaml
  target:
    kind: Kustomization
    name: facebooc-deploy
- path: instavote-deploy-kustomization.yaml                                             #覆盖@1 
  target:
    kind: Kustomization
    name: instavote-deploy

 
3.需要注意的是之前的instavote应用在flux-system 名称空间，需要修改
root@master01:~/tenant/instavote-deploy/flux/base# sed  -i  's/flux-system/instavote/g'    *

#同时一部分secrtet也需要部署在instavote名称空间下
```

### 创建接收器

**Note:**其实在instavote namespace下需要一个source就可以了，因为我们一直使用的是app-demo来实现的，所以一下也使用这个源

```shell
root@master01:~# flux   get sources git  -n instavote 
NAME            	REVISION          	SUSPENDED	READY	MESSAGE                                           
app-demo        	main@sha1:8bd744fa	False    	True 	stored artifact for revision 'main@sha1:8bd744fa'	
instavote-deploy	main@sha1:8bd744fa	False    	True 	stored artifact for revision 'main@sha1:8bd744fa'	

```

接下来要完成instavote-deploy有push事件,就发送消息到 flux fleet的webhhok

+ 创建一个接收器，指明各个控制器对接到哪个部署存储库（instavote-deploy）

  ```shell
  root@master01:~# flux -n  instavote   create receiver gitlab-receiver-instavote-deploy  --type gitlab  --event "Push Hook"   --event "Tag Push Hook"  --secret-ref webhook-token  --resource GitRepository/app-demo  
  ✚ generating Receiver
  ► applying Receiver
  ✔ Receiver created
  ◎ waiting for Receiver reconciliation
  ✔ Receiver gitlab-receiver-instavote-deploy is ready
  ✔ generated webhook URL /hook/f4f2f43ed33714e13a126a5c30db0b7235d8f58f57cbfccb8b2051665dc50f96
  
  ```

+ 在instavote-deploy配置要发送到的flux  webhook的地址

  ```shell
  http://12.0.0.20:31234/hook/f4f2f43ed33714e13a126a5c30db0b7235d8f58f57cbfccb8b2051665dc50f96
  ```

  token为

  ```shell
  root@master01:~# echo $WEBHOOK_TOKEN  
  07b4ef32ae349bec1b62da7408595e30
  
  ```


## Flagger

<img src=".//img/Flagger图景.png" alt="Flagger图景" style="zoom:50%;" />

Flagger管理金丝雀部署并进行渐进式基础设施部署，而Nginx将提供基于权重路由流量的功能

利用Prom进行分析，逐步的进行流量的转换

### 安装

我们使用的是flux安装方式，有关更多的内容请参考[官网](https://docs.flagger.app/)

```shell
root@master01:~/tenant/flux-fleet# tree
.
├── clusters
│   ├── production
│   │   └── onboard-projects.yaml
│   └── staging
│       ├── correlation-infra.yaml                  #指明kustomization控制器去哪找部署清单
│       ├── flux-system
│       │   ├── expose-webhook-receiver.yaml
│       │   ├── gotk-components.yaml
│       │   ├── gotk-sync.yaml
│       │   └── kustomization.yaml
│       └── onboard-projects.yaml
├── infra
│   └── staging
│       └── flagger.yaml                            #部署清单在这个位置
└── projects
    ├── base
    │   ├── facebooc
    │   │   ├── facebooc-deploy-gitrepository.yaml
    │   │   ├── facebooc-deploy-kustomization.yaml
    │   │   ├── kustomization.yaml
    │   │   └── rbac.yaml
    │   └── instavote
    │       ├── instavote-deploy-gitrepository.yaml
    │       ├── instavote-deploy-kustomization.yaml
    │       ├── kustomization.yaml
    │       └── rbac.yaml
    └── staging
        ├── facebooc-deploy-kustomization.yaml
        ├── instavote-deploy-kustomization.yaml
        └── kustomization.yaml
```

测试使用Ingess发布应用(Ingress controller我们之前按已经部署了)

```shell
root@master01:~/instavote-deploy/kustomize/vote/base# flux  -n instavote  get kustomizations 
NAME              	REVISION          	SUSPENDED	READY	MESSAGE                              
instavote-deploy  	main@sha1:3cefb57f	False    	True 	Applied revision: main@sha1:3cefb57f	
monitoring-configs	main@sha1:3cefb57f	False    	True 	Applied revision: main@sha1:3cefb57f	
redis-dev         	main@sha1:3cefb57f	False    	True 	Applied revision: main@sha1:3cefb57f	
vote-dev          	main@sha1:3cefb57f	False    	True 	Applied revision: main@sha1:3cefb57f	
root@master01:~/instavote-deploy/kustomize/vote/base# flux  -n instavote   reconcile   kustomization   vote-dev  --with-source 
► annotating GitRepository app-demo in instavote namespace
✔ GitRepository annotated
◎ waiting for GitRepository reconciliation
✔ fetched revision main@sha1:3cefb57f9a20188fb7419de866e2245abd50ef7a
► annotating Kustomization vote-dev in instavote namespace
✔ Kustomization annotated
◎ waiting for Kustomization reconciliation
✔ applied revision main@sha1:3cefb57f9a20188fb7419de866e2245abd50ef7a
No resources found in instavote namespace.
root@master01:~/instavote-deploy/kustomize/vote/base# kubectl  -n instavote  get  ingress
NAME   CLASS   HOSTS              ADDRESS     PORTS   AGE
vote   nginx   vote.forcecs.com   12.0.0.21   80      24s
root@master01:~/instavote-deploy/kustomize/vote/base# #使用vote.forcecs.com:31601进行访问
```

### Changing Flux Code to Work with Flagger

#### Blue/Green部署

<img src=".//img/canary分析.png" alt="canary分析" style="zoom:50%;" />

<img src=".//img/canary分析2.png" alt="canary分析2" style="zoom:50%;" />

<img src=".//img/canaryfen分析3.png" alt="canaryfen分析3" style="zoom:50%;" />

Flux用现有的deployment to decete 镜像的改变和新镜像的拉取，然后完成对现有应用的更新。在更新的过程中会创建一个新的deployment。这就是flagger所关注的。
使用现有的版本检测更新并触发新版本，这就是flagger与现有系统的配合。然后使用现有的部署来运行金丝雀分析.我们还需要一个HPA来控制缩放比例
所以我们需要对flux代码执行三步 1）不管理服务 2）将副本计数设置为0  3）执行部署并创建HPA

Ingress已经存在，使用它与flagger来定义路由规则

执行一下步骤

+ 1）首先去掉instavote-deploy中对service的引用（base  dev）2）在base目录下添加hpa资源 3）修改vote的副本

  ```shell
  root@master01:~/tenant/instavote-deploy/kustomize/vote/base# kubectl get all  -n instavote  
  NAME                                    READY   STATUS    RESTARTS   AGE
  pod/instavote-db-postgres-0             1/1     Running   0          4d4h
  pod/instavote-result-7b7ff78678-sgbz8   1/1     Running   0          4d4h
  pod/redis-78d4b8b77c-ccwbx              1/1     Running   0          4d4h
  
  NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
  service/db                 ClusterIP   10.105.120.18   <none>        5432/TCP       4d4h
  service/instavote-result   NodePort    10.103.79.92    <none>        80:31951/TCP   4d4h
  service/redis              ClusterIP   10.97.116.122   <none>        6379/TCP       4d4h
  
  NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
  deployment.apps/instavote-result   1/1     1            1           4d4h
  deployment.apps/redis              1/1     1            1           4d4h
  deployment.apps/vote               0/0     0            0           4d4h
  
  NAME                                          DESIRED   CURRENT   READY   AGE
  replicaset.apps/instavote-result-7b7ff78678   1         1         1       4d4h
  replicaset.apps/redis-78d4b8b77c              1         1         1       4d4h
  replicaset.apps/vote-6d7d55ff5                0         0         0       4d4h
  replicaset.apps/vote-6fcd97c964               0         0         0       4d2h
  
  NAME                                     READY   AGE
  statefulset.apps/instavote-db-postgres   1/1     4d4h
  
  NAME                                       REFERENCE         TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
  horizontalpodautoscaler.autoscaling/vote   Deployment/vote   <unknown>/80%   5         10        0          2m30s
  
  ```

+ 部署负载测试程序

  ```shell
  root@master01:~/tenant/flux-fleet/infra/staging# cat flagger-loadtester.yaml    #我们先部署一个负载测试【请参考官网】
  ---
  apiVersion: source.toolkit.fluxcd.io/v1beta2
  kind: OCIRepository
  metadata:
    name: flagger-loadtester
    namespace: instavote
  spec:
    interval: 6h # scan for new versions every six hours
    url: oci://ghcr.io/fluxcd/flagger-manifests
    ref:
      semver: 1.x # update to the latest version 
    verify: # verify the artifact signature with Cosign keyless
      provider: cosign
  	
  ---
  apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
  kind: Kustomization
  metadata:
    name: flagger-loadtester
    namespace: instavote
  spec:
    interval: 6h
    wait: true
    timeout: 5m
    prune: true
    sourceRef:
      kind: OCIRepository
      name: flagger-loadtester
    path: ./tester
    targetNamespace: instavote
  ```

+ Blue/Green  with flagger

  1)创建canary资源（执行分析： interval  thresholds interations）

  2)通过更新镜像，cm ,sercert等触发更新

  3)具体的如何实施，请自行搜索官方文档
  
  ```shell
  #kubectl config set-context --current --namespace=instavote
  
  
  #watch kubectl describe ing
  
  
  #watch 'kubectl get all -o wide \
  -l "kustomize.toolkit.fluxcd.io/name=vote-dev"; \
  kubectl describe canary vote | tail -n 20'
  
  
  #kubectl -n instavote set image deploy vote vote=schoolofdevops/vote:v6
  
  
  #while :; do     sleep 3s  ;curl -H 'Host:vote.forcecs.com'  http://12.0.0.20:31601; done
  ```
  
  

#### NGINX Canary Deployments

1）现在的状态

<img src=".//img/渐进式金丝雀1.png" alt="渐进式金丝雀1" style="zoom:50%;" />

2）修改镜像触发

![渐进式金丝雀2](.//img/渐进式金丝雀2.png)

请参考官方文档[here](https://docs.flagger.app/tutorials/nginx-progressive-delivery)

**需要指出的是:为了防止改动flux相关代码导致kustomization的更新，我们不管是使用哪种发布模型，都应该执行  flux   -n instavote   suspend   kustomization   xxxxx**

当达到稳定后全部的流量切换到新版本，而不是部分流量，<vote-primary> deployment提供的是新版本

### 说明

翻译自[here](https://docs.flagger.app/usage/how-it-works)

**Canary target**

Based on the above configuration, Flagger generates the following Kubernetes objects:

- `deployment/<targetRef.name>-primary`
- `hpa/<autoscalerRef.name>-primary`

The primary deployment 被视为应用程序的稳定版本，默认情况下，所有流量都会被路由到该版本，目标部署则被缩放为零。Flagger 会检测目标部署的更改（包括机密和配置映射），并在将新版本提升为primary deployment之前执行金丝雀分析。
使用 .spec.autoscalerRef.primaryScalerReplicas 可覆盖生成的primary HorizontalPodAutoscaler 的副本缩放配置。当你想为primary工作负载设置不同的缩放配置，而不是使用原始工作负载 HorizontalPodAutoscaler 的相同值时，这将非常有用。

**请注意：**target deployment 部署必须有一个单一的标签选择器，格式为 app： <DEPLOYMENT-NAME>：

除 app 外，Flagger Flagger supports name and app.kubernetes.io/name selectors，可以在 Flagger 部署清单的容器参数下使用 -selector-labels=my-app-label 命令标志指定您的标签，或者在使用 Helm 安装 Flagger 时设置 --set selectorLabels=my-app-label。

如果target deployment使用了secrets and/or configmaps，Flagger 将使用 -primary 后缀创建每个对象的副本，并在主部署中引用这些对象。如果用 flagger.app/config-tracking: 禁用注释 ConfigMap 或 Secret，Flagger 将在primary deployment中使用相同的对象，而不会创建主副本。你可以在 Flagger 部署清单的 containers args 下使用 -enable-config-tracking=false 命令标志全局禁用 secrets/configmaps 跟踪，或在使用 Helm 安装 Flagger 时设置 --set configTracking.enabled=false ，但使用每个 Secret/ConfigMap 注释禁用配置跟踪可能更适合你的使用情况

The autoscaler reference 是可选的，如果指定，Flagger 将在target and primary deployments扩大或缩小时暂停流量增加。HPA 可以帮助减少金丝雀分析期间的资源使用。指定了自动分 析器引用后，对自动分 析器所做的任何更改只有在the deployment启动并成功完成后才会在primary autoscaler中激活。您还可以选择创建两个 HPA，一个用于canary，另一个用于 primary，这样就可以在不进行新部署的情况下更新 HPA。由于canary deployment将缩放为 0，canary上的 HPA 将处于非活动状态。

**Canary service**

canary资源决定了目标工作负载在群集内的暴露方式。The canary target应暴露一个 TCP 端口，Flagger 将使用该端口创建 ClusterIP Services.

```yaml
spec:
  service:
    name: podinfo
    port: 9898
    portName: http
    appProtocol: http
    targetPort: 9898
    portDiscovery: true
```

The container port from the target workload should match the `service.port` or `service.targetPort`. The `service.name` is optional, defaults to `spec.targetRef.name`. The `service.targetPort` can be a container port number or name. The `service.portName` is optional (defaults to `http`), if your workload uses gRPC then set the port name to `grpc`. The `service.appProtocol` is optional, more details can be found [here](https://kubernetes.io/docs/concepts/services-networking/service/#application-protocol).

如果 port discovery is enabled，Flagger 会扫描目标工作负载并提取容器端口，但不包括canary service and service mesh sidecar ports。这些端口将在生成 ClusterIP 服务时使用。

基于 canary 规范服务，Flagger 创建以下 Kubernetes ClusterIP 服务：

```
<service.name>.<namespace>.svc.cluster.local
```

selector `app=<name>-primary`

```
<service.name>-primary.<namespace>.svc.cluster.local
```

selector `app=<name>-primary`

```
<service.name>-canary.<namespace>.svc.cluster.local
```

selector `app=<name>`

这可确保指向 podinfo.test:9898 的流量将被路由到应用程序的最新稳定版本。podinfo-canary.test:9898 地址仅在  canary analysis期间可用，可用于conformance testing or load testing.

**Note: **将之前的deployment缩放至0,创建primary deployment,同时创建三个service,现在是两个service都可以路由到新的deployment(旧版本)

​             这个<service.name>-canary service路由到旧的deployment（由于此时副本为0，所以不提供服务）

​            当**部署新版本**时候（由镜像，cm, secret的触发），将更新原始的deployment，它将扩大Green/Canary环境以开始提供canary版本,蓝色版本不受影响

​            The webhook 响应 HTTP 500 是阻止 Flagger 将绿色升级为蓝色的唯一原,测试完金丝雀后，将 webhook 改为 HTTP 200 响应。现在的实时版本为新版本



您可以配置 Flagger，为生成的服务设置注释和标签：

```yaml
spec:
  service:
    port: 9898
    apex:
      annotations:
        test: "test"
      labels:
        test: "test"
    canary:
      annotations:
        test: "test"
      labels:
        test: "test"
    primary:
      annotations:
        test: "test"
      labels:
        test: "test"
```

请注意，apex 注释会被添加到生成的 Kubernetes Service和生成的服务mesh/ingress object.中。这样就可以在 Istio VirtualServices 和 TraefikServices 中使用 external-dns。请注意这里的配置冲突。
除了端口映射和元数据，服务规范还可以包含 URI 匹配和重写规则、超时和重试策略

```yaml
spec:
  service:
    port: 9898
    match:
      - uri:
          prefix: /
    rewrite:
      uri: /
    retries:
      attempts: 3
      perTryTimeout: 1s
    timeout: 5s
```

When using **Istio** as the mesh provider, you can also specify HTTP header operations, CORS and traffic policies, Istio gateways and hosts. The Istio routing configuration can be found [here]().

### 补充

要想实现Tekton CI与Flagger的结合，请参考

```shell
1.创建镜像仓库用来指明使用关联到的私有存储库<flux create image repository vote xxxxx>
2.定义镜像扫描策略<flux create image policy vote xxxx>
3.更新部署仓库<instavote-deploy>的清单,并使用新的镜像（因此，必须在图像名称前边加一个占位符，以使flux知道更新哪一行代码）
4.创建镜像更新，指明第三步标记的位置具体在哪 <flux create image update instavote-all xxx>
5.tekton触发CI



root@master01:~/prepare/18-flux-cd/04-app-demo-tekton-ci# flux  get   images   repository  -A  
NAMESPACE	NAME	LAST SCAN                	SUSPENDED	READY	MESSAGE                       
instavote	vote	2023-11-17T16:40:25+08:00	False    	True 	successful scan: found 4 tags	
root@master01:~/prepare/18-flux-cd/04-app-demo-tekton-ci# flux  get   images   policy  -A  
NAMESPACE	NAME	LATEST IMAGE                                               	READY	MESSAGE                                                                                        
instavote	vote	harbor.forcecs.com:32415/vote/vote:main-31357ded-1699343111	True 	Latest image tag for 'harbor.forcecs.com:32415/vote/vote' resolved to main-31357ded-1699343111	
root@master01:~/prepare/18-flux-cd/04-app-demo-tekton-ci# flux  get   images   update  -A  
NAMESPACE	NAME         	LAST RUN	SUSPENDED	READY	MESSAGE                                                                                                    
instavote	instavote-all	        	False    	False	walking path for files: lstat /tmp/instavote-app-demo2867013450/deploy/vote/dev: no such file or directory	       #需要进行修改


root@master01:~/prepare/18-flux-cd/04-app-demo-tekton-ci# flux get  sources git  -A  
NAMESPACE  	NAME            	REVISION          	SUSPENDED	READY	MESSAGE                                           
facebooc   	facebooc        	main@sha1:46f0faea	False    	True 	stored artifact for revision 'main@sha1:46f0faea'	
facebooc   	facebooc-deploy 	main@sha1:e2136587	False    	True 	stored artifact for revision 'main@sha1:e2136587'	
flux-system	flux-system     	main@sha1:6881a15e	False    	True 	stored artifact for revision 'main@sha1:6881a15e'	
instavote  	app-demo        	main@sha1:fd1f5c3b	False    	True 	stored artifact for revision 'main@sha1:fd1f5c3b'	
instavote  	instavote-deploy	main@sha1:fd1f5c3b	False    	True 	stored artifact for revision 'main@sha1:fd1f5c3b'	
```

