---
title: kubernetes-Informer详解
date: 2020-07-11 13:11:03
tags: kubernetes
---

# 一、导语
> 本文将对kubernetes informer机制进行详细剖析，从全局出发，从举例说明、接口使用方式到源码剖析，力争讲清楚informer都干了些什么、为什么要这样干。从informer的实现方式总结出使用kubernetes client可能遇到的情况。

> 《详解系列》是笔者从喜马拉雅中《猫哥详说红楼梦》得到的启发，在这篇音频节目中，猫哥一字一句的解释了红楼梦的故事，处处都透出一个"详"字，让初读者能很清楚、很容易、毫无障碍的了解红楼梦到底讲述了什么。《详解系列》也将以此作为立足点，由粗到细，并能讲多细，就讲多细；所举示例，尽量都能保证独立运行。

> 本文是笔者个人理解，若有纰漏之处，还请指正(^.^)。

# 二、Informer是什么（1）。
首先，Informer是给客户端访问kubernetes中资源的一种工具（我把informer称为通知器），注意，它不属于服务器端，通常用于客户端来访问、观察apiserver的对象。在kubernetes生态中，如果作为客户端想要访问k8s资源，几乎就离不开informer。
<!-- more -->

我们先从客户端角度出发，试想一下，如果没有informer，我们想要获取当前kubernetes中的资源情况，并实时得到资源对象的变化（如增删改），就得自己发起一个http短连接来获取当前资源对象（apiserver 的GET接口），以及一个http长连接来实时watch资源对象的变换（apiserver的watch接口）。这样手动实现会遇到以下几个不好的地方，一个是每次要获取对象的时候，都要发起一次http链接，这对apiserver会带来压力；另外一个是，当我们watch对象变化时，由于网络的不稳定性，可能出现了连接断开、超时等异常情况，这样我们得重新维护并且尽量不遗失这些对象的变化事件。那么总结一下，如果我们想要稳定的观察一个kubernetes 资源对象的一切变化，需要实现哪些功能：
1. 发起一个短连接，访问apiserver的GET接口来获取当前集群中所有存在的对象。
2. 维持一个长连接，访问apiserver的WATCH接口实时watch对象的变化。
3. 维护一个缓存池，当我们要访问某个对象时，不需要重新发起http连接来获取对象的情况，而是直接从缓存池中拿到这个对象。
4. 处理各种异常情况，来维持这个缓冲池中对象的状态与apiserver中对象的状态保持一致。

而informer机制，大致来说就是实现了以上4点。informer底层实现上也是用一个短连接来LIST资源对象，用一个长连接来WATCH对象的变化，并且维护了一个对象缓存池，informer屏蔽了很多访问kubernetes资源的细节，并将其总结了成了几个简单接口以及3个钩子函数。下面我们首先来实现一个使用informer接口的简单客户端，看看informer是怎么玩的。

# 三、Informer的使用示例
我们将使用informer来实现一个简单的client。该client观察所有pod的变化事件，并且打印出这些事件。然后基于这个demo，从全局的角度来看待如何使用informer来观察kubernetes中各个资源的变化。
该demo可以独立运行。代码如下:
```go
package main
import (
	"fmt"

	"k8s.io/apimachinery/pkg/labels"
	"k8s.io/client-go/rest"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/kubernetes"
	kubeinformers "k8s.io/client-go/informers"
)
func main() {
	fmt.Println("start informer demo")
	cfg, err := clientcmd.BuildConfigFromFlags("", "/etc/kubernetes/admin.conf")
	if err != nil {
		fmt.Errorf("Error building kubeConfig: %s\n", err.Error())
	}
	kubeClientSet, err := kubernetes.NewForConfig(rest.AddUserAgent(cfg, "informer-demo"))
	if err != nil {
		fmt.Errorf("Error NewForConfig : %s\n", err.Error())
		return
	}
	// new一个Informer工厂
	kubeInformerFactory := kubeinformers.NewSharedInformerFactoryWithOptions(kubeClientSet, 0)
	// 获取Informer工厂中生产出一个pod informer
	podInformer := kubeInformerFactory.Core().V1().Pods()
	// 为pod informer增加事件处理函数。分别处理新增pod、更新pod、删除pod事件。
	// 作为Demo，我们只是简单的打印出这些事件。
	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			pod := obj.(*corev1.Pod)
			fmt.Printf("podInformer addFunc called, a new pod created: %s\n", pod.Namespace + "/" + pod.Name)
			//doSomething
		},
		UpdateFunc: func(old, new interface{}) {
			fmt.Println("podInformer updateFunc called")
			oPod := old.(*corev1.Pod)
			nPod := new.(*corev1.Pod)
			if oPod.Status.Phase != nPod.Status.Phase {
				fmt.Printf("pod update status: %s\n", nPod.Status.Phase)
				//doSomething
			}
		},
		DeleteFunc: func(obj interface{}) {
			pod := obj.(*corev1.Pod)
			fmt.Printf("podInformer deleteFunc called, a pod is deleted: %s\n", pod.Namespace+"/"+pod.Name)
			//doSomething
		},
	})

	// Informer工厂开始工作，即开始真正的List和watch pod。
	stopCh := make(chan struct{})
	kubeInformerFactory.Start(stopCh)

	// 等待informer中cache同步完毕
	kubeInformerFactory.WaitForCacheSync(stopCh)

	// 通过Informer来List出label为"k8s-app": "kube-dns"的pods
	labelSelector := labels.Set(map[string]string{"k8s-app": "kube-dns"}).AsSelector()
	pods, err := podInformer.Lister().List(labelSelector)
	if err != nil {
		fmt.Errorf("podInformer lister error: %s\n", err.Error())
		return
	}
	for _, pod := range pods {
		fmt.Printf("list pod: (%s:%s)\n", pod.Namespace, pod.Name)
	}
	select {
	}
}
```
运行结果如下：
```sh
# master1: ~ ./informer_demo
start informer demo
podInformer addFunc called, a new pod created: kube-system/kube-proxy-prnws
podInformer addFunc called, a new pod created: kube-system/coredns-6955765f44-nhrg5
...
list pod: (kube-system:coredns-6955765f44-h2kds)
list pod: (kube-system:coredns-6955765f44-nhrg5)
podInformer updateFunc called
podInformer addFunc called, a new pod created: default/frontend-5cb785b459-82blj
podInformer deleteFunc called, a pod is deleted: default/frontend-5cb785b459-btfnt
podInformer updateFunc called
pod update status: Running
```

我们可以看到，通过informer我们获取到了kubernetes中所有的pod（通过addFunc打印出来），并且利用informer list函数列出了label为`k8s-app: kube-dns`的pod；并且为了让我们能看到`updateFunc`和`deleteFunc`被调用，我们特意重建了一个pod(namespace/frontend-5cb785b459-82blj)。

我们接下来从这个demo入手，介绍一下一个client使用nformer的典型方法。本小节我们先从全局出发，说明一下这个demo做了哪些事情，调用了informer提供的哪些接口，宏观的看一下informer工作原理；下一小节再深入到各个函数实现细节中，理清楚informer中各个组件之间的关系；最后一小节再回到整体上总结一下informer机制。

上面的demo可以分解为以下3个部分:
## 3.1 准备工作：创建informer
首先，每一个不同的kubernetes api resource都有各自的informer，这些informer都是通过一个工厂对象生产出来的，因此我们要先new一个informer工厂对象，然后再通过这个informer工厂来产生具体某个资源的informer（这个例子中就是pod的informer）:
```
	// new一个Informer工厂
	kubeInformerFactory := kubeinformers.NewSharedInformerFactoryWithOptions(kubeClientSet, 0)
	// 获取Informer工厂中生产出一个pod informer
	podInformer := kubeInformerFactory.Core().V1().Pods()
```
## 3.2 增加事件处理函数
其次，为informer添加对应事件的处理函数(本质是钩子函数或回调函数，相应事件发生时，函数会被调用)，这个可以认为是最重要的步骤，在这些处理函数中来实现自己的业务逻辑:
```
	// 为pod informer增加事件处理函数。分别处理新增pod、更新pod、删除pod事件。
	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			//doSomething...
		},
		UpdateFunc: func(old, new interface{}) {
			//doSomething...
		},
		DeleteFunc: func(obj interface{}) {
			//doSomething...
		},
	})
```
可以看到，informer一共就提供了三种事件，分别是：
- **增加事件**：即有新的对象在缓存中创建（注意是在缓存中创建而不是在kubernetes中）。这个事件可能容易引起误解，让人以为是kubernetes增加了某个对象触发该事件，其实不然，主语应该是informer cache而不是kubernetes，也就是说，是**informer cache**增加了某个对象触发该事件**而不是kubernetes**增加某对象而触发。这个区别很微妙，举个例，当用户在kubernetes中新建了一个对象，informer cache 当然也会及时更新缓存，从而触发这个增加事件，**但是**，如果kubernetes中啥也没做，而informer被重新启动了，对informer cache来说kubernetes中已有的存量对象都是新对象，因为当下informer由于重启了缓存是空的，所有对象都需要在缓存中重新创建，因此每个存量对象都会触发这个新增事件。因此，本质原因是informer cache是kubernetes缓存(或者说子集)，缓存可以被清空，一旦被清空就要重新缓存这些对象。
- **更新事件**：即存量对象有更新的时候，被触发。
- **删除事件**：即存量对象被删除后，被触发。

## 3.3 开启informer工作，等待事件处理函数被调用
最后，我们在建立事件处理函数会后，需要开启(enable)informer的工作，这样informer才会正在向apiserver发起请求，list&&watch资源对象并建立cache，相关代码如下:
```
	// Informer工厂开始工作，即开始真正的List和watch pod。
	stopCh := make(chan struct{})
	kubeInformerFactory.Start(stopCh)

	// 等待informer中cache同步完毕
	kubeInformerFactory.WaitForCacheSync(stopCh)
```
## 总结
我们这小节从客户端的使用informer的角度上进行了粗略的介绍，暂未深入细节中，如果不需要了解代码细节的话，照着写应该也能完成一个使用informer访问apiserver resource的client了。最后再总结一下，作为一个client使用informer大概经历了三个流程：
1. 创建informer工厂并生成某个资源的informer
2. 构建informer事件处理函数
3. 开启informer工厂，正式构建informer cache并调用事件处理函数。


# 四、Informer实现机制
本小节主要在上一小节的基础上，着重介绍上一小节出现的informer相关的函数是如何实现的，以及informer内部组件的运行原理及其关系，并且会详细讲解代码实现以及为什么要这样实现。

> 在深入代码细节之前，笔者有一个比较好的经验，就是熟悉各个源文件所在的目录结构，通过这个目录结构可以大概了解组件之间的整体脉络与关系。例如，informer相关的代码都放在`k8s.io/client-go/informers`下面，这再次说明informer就是给client端用的；又例如，与pod相关的informer在`k8s.io/client-go/informers/core/v1/pod.go`中并引用了`SharedIndexInformer`和`PodLister`，其中`SharedIndexInformer`是位于`k8s.io/client-go/tools/cache/`目录下，这说明informer自己并没有实现缓存机制，而是引用了`k8s.io/client-go/tools/cache`下实现的缓存；而`PodLister`位于`k8s.io/client-go/listers/core/v1/pod.go`中，这说明informer中的list功能是由lister来实现，也就是说informer和lister是同级别的概念，informer提供了watch机制，lister提供了lister机制，只不过informer集成了lister而已。

## 4.1 InformerFactory和Informer之间的关系。
我们可以从demo中看到，在创建pod informer的时候，我们调用了两个函数，分别是:
```
	// new一个Informer工厂
	kubeInformerFactory := kubeinformers.NewSharedInformerFactoryWithOptions(kubeClientSet, 0)
	// 获取Informer工厂中生产出一个pod informer
	podInformer := kubeInformerFactory.Core().V1().Pods()
```
很简单，它分别创建informerFactory并利用informerFactory创建了pod informer。

kubeInformerFactory实际是个接口：
```
type SharedInformerFactory interface {
	//这一部分作用于整个informer
	internalinterfaces.SharedInformerFactory
	ForResource(resource schema.GroupVersionResource) (GenericInformer, error)
	WaitForCacheSync(stopCh <-chan struct{}) map[reflect.Type]bool

	//下面一部分是具体资源类型的informer
	Admissionregistration() admissionregistration.Interface
	...
	Core() core.Interface
	Discovery() discovery.Interface
	Events() events.Interface
	...
}
```
它包含两个部分，第一部分就是与informerFactory相关的，与具体某个类型的informer无关；另一部分是创建具体某个资源类型的informer。
我们先看第二部分，在demo中我们用kubeInformerFactory.Core().V1().Pods()来创建了一个podInformer，可以看到core, v1, pod分别是pod的GVR(Group，Version, Resource)，也就是说通过该函数，我们创建了一个`core/v1/pods`这样的api resource informer。Admissionregistration、Core、Discovery、Events这些实际都是资源对象的group，这些接口里面又包含了资源对象的version，最后这些version包含了具体的资源对象。我们用一个目录结构解释下它们之间的关系:
![](/images/image_1.png)

## 4.2 Informer内部各个组件


# 四、Informer是什么（2）
