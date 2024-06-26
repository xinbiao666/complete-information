- [Kubernetes客户端Java库fabric8io，快来自定义你的操作](https://www.pkslow.com/archives/kubernetes-client-fabric8io)



# 一、Kubernetes Java客户端

对于`Kubernetes`集群的操作，官方提供了命令行工具`kubectl`，这也是我们最常用且必须掌握的方式。通过`kubectl`来实现增删改查操作，方便而直接。但总有一些更复杂的场景难以满足，比如我希望在数据库的某个值达到10万后就触发一个`Kubernetes Job`去处理某项任务。

[![img](https://pkslow.oss-cn-shenzhen.aliyuncs.com/images/2021/01/kubernetes-client-fabric8io.k8s-overview.png)](https://pkslow.oss-cn-shenzhen.aliyuncs.com/images/2021/01/kubernetes-client-fabric8io.k8s-overview.png)

从`Kubernetes`的架构图可以看出，我们只要和`API server`做好交互就可以了，实际上`kubectl`也是如此的。那我们就可以使用任何语言来操作`Kubernetes`。

本文将介绍`Java`方面最好用的客户端库`fabric8io/kubernetes-client`，它支持`Kubernetes`和`OpenShift`，并被许多项目引用，如`Spring Cloud`、`Spark`、`Istio Java API`等。



# 二、使用

## 2.1 引入依赖

目前的最新版本为`5.0.0`，通过`maven`引入最新依赖如下：

```xml
<dependency>
  <groupId>io.fabric8</groupId>
  <artifactId>kubernetes-client</artifactId>
  <version>5.0.0</version>
</dependency>
```

这个依赖包含了相关的核心类、模型类、`Json`和`okhttp3`等。

[![img](https://pkslow.oss-cn-shenzhen.aliyuncs.com/images/2021/01/kubernetes-client-fabric8io.maven-dependencies.png)](https://pkslow.oss-cn-shenzhen.aliyuncs.com/images/2021/01/kubernetes-client-fabric8io.maven-dependencies.png)

看它的依赖就可以学习优秀的项目是如何组织和管理的。

## 2.2 创建客户端

创建客户端最简单的方式是使用默认配置：

```java
KubernetesClient client = new DefaultKubernetesClient();
```

它会从目录`~/.kube/config`中读取配置文件。如果想要修改配置，可以通过配置以下设置：

- 系统属性（System Properties）
- 环境变量（Enviroment Variables）
- Kube配置文件
- ServiceAccount的`Token`和加载的CA证书

系统属性和环境变量列表可查看官网。

当然，还可以通过`Java`来自定义配置：

```java
Config config = new ConfigBuilder()
  .withMasterUrl("https://localhost:6443")
  .build();
KubernetesClient client = new DefaultKubernetesClient(config);
```

## 2.3 创建资源

这个`Java`库使用了大量的`Builder`模式来创建对象，创建命令空间如下：

```java
Namespace namespace = new NamespaceBuilder()
  .withNewMetadata()
  .withName("pkslow")
  .addToLabels("reason", "pkslow-sample")
  .endMetadata()
  .build();
client.namespaces().createOrReplace(namespace);
```

非常灵活，上面例子添加了名字和标签，最后通过`createOrReplace`方法可新建，如果存在可替换。

对于`Pod`也是类似的：

```java
Pod pod = new PodBuilder()
  .withNewMetadata()
  .withName("nginx")
  .addToLabels("app", "nginx")
  .endMetadata()
  .withNewSpec()
  .addNewContainer()
  .withName("nginx")
  .withImage("nginx:1.19.5")
  .endContainer()
  .endSpec()
  .build();
client.pods().inNamespace("pkslow").createOrReplace(pod);
```

指定名字、标签和镜像后就可以创建了。

## 2.4 查看资源

查看资源可以查询所有，或者通过条件**options**来过滤，具体代码如下：

```java
// 查看命名空间
NamespaceList namespaceList = client.namespaces().list();
namespaceList.getItems()
  .forEach(namespace ->
           System.out.println(namespace.getMetadata().getName() + ":" + namespace.getStatus().getPhase()));

// 查看Pod
ListOptions options = new ListOptions();
options.setLabelSelector("app=nginx");
Pod nginx = client.pods().inNamespace("pkslow")
  .list(options)
  .getItems()
  .get(0);
System.out.println(nginx);
```

## 2.5 修改资源

修改资源是通过`edit`方法来实现的，可通过命名空间和名字来定位到资源，然后进行修改，示例代码如下：

```java
// 修改命名空间
client.namespaces().withName("pkslow")
  .edit(n -> new NamespaceBuilder(n)
        .editMetadata()
        .addToLabels("project", "pkslow")
        .endMetadata()
        .build()
       );

// 修改Pod
client.pods().inNamespace("pkslow").withName("nginx")
  .edit(p -> new PodBuilder(p)
        .editMetadata()
        .addToLabels("app-version", "1.0.1")
        .endMetadata()
        .build()
       );
```

## 2.6 删除资源

删除资源也是类似的，先定位再操作：

```java
client.pods().inNamespace("pkslow")
  .withName("nginx")
  .delete();
```

## 2.7 通过yaml文件操作

我们还可以直接通过`yaml`文件来描述资源，而不用`Java`来定义，这样可以更直观和方便。完成`yaml`文件的编写后，`Load`成对应的对象，再进行各种增删改查操作，示例如下：

`yaml`文件定义了一个`Deployment`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    myapp: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      myapp: nginx
  template:
    metadata:
      labels:
        myapp: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.7.9
          ports:
            - containerPort: 80
```

`Java`代码如下：

```java
Deployment deployment = client.apps().deployments()
  .load(Fabric8KubernetesClientSamples.class.getResourceAsStream("/deployment.yaml"))
  .get();
client.apps().deployments().inNamespace("pkslow")
  .createOrReplace(deployment);
```

## 2.8 监听事件

我们还可以通过监听资源的事件，来进行对应的反应，比如有人删除了`Pod`就记录日志到数据库等，这个功能还是非常有用的。示例代码如下：

```java
client.pods().inAnyNamespace().watch(new Watcher<Pod>() {
  @Override
  public void eventReceived(Action action, Pod pod) {
    System.out.println("event " + action.name() + " " + pod.toString());
  }

  @Override
  public void onClose(WatcherException e) {
    System.out.println("Watcher close due to " + e);

  }
});
```

通过一个`Watcher`监听了`Pod`的所有动作事件，然后打印动作名和对应的`Pod`。输出后的日志如下：

```bash
event ADDED Pod(apiVersion=v1, kind=Pod, metadata=ObjectMeta(
event MODIFIED Pod(apiVersion=v1, kind=Pod, metadata=ObjectMeta(
event DELETED Pod(apiVersion=v1, kind=Pod, metadata=ObjectMeta(
event MODIFIED Pod(apiVersion=v1, kind=Pod, metadata=ObjectMeta(
```

日志太长，就不完全显示。

# 3 总结

这个`Kubernetes`的Java客户端实在是好用，`API`简单易用，即使不用文档也能通过方法名判断。最让人惊喜的是，官方还提供了许多绝佳的[示例](https://github.com/fabric8io/kubernetes-client/tree/master/kubernetes-examples/src/main/java/io/fabric8/kubernetes/examples)，简直不要太友好。

使用这个`API`，在项目中可以更灵活地管理和使用`Kubernetes`应用了。