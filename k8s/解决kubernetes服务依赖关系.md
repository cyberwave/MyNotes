原文: [Kubernetes Demystified: Solving Service Dependencies](https://dzone.com/articles/kubernetes-demystified-solving-service-dependencie)
由于容器与自身的外部服务没有联系，因此解决服务依赖性会是一个挑战
本系列文章探讨了企业客户在使用 `Kubernetes` 时遇到的一些常见问题。[容器服务](https://www.alibabacloud.com/product/container-service)客户经常问的一个问题是：*“如何处理服务之间的依赖关系？”*
在应用程序中，组件依赖项指的是中间件服务和业务服务。在传统的软件部署方法中，必须按特定顺序完成应用程序的启动和停止任务。
使用Kubernetes，Docker Swarm和其他容器编排技术在分布式环境中部署应用程序时，不同的组件会同时启动，因此无法确保一定的启动顺序。此外，在运行应用程序时，它们依赖的服务可能会失败或迁移。因此，解决容器之间的服务依赖性是客户经常提出的问题。
# 方法1：检查应用程序中的依赖项
我们可以在应用程序启动逻辑中添加服务依赖检查逻辑。如果无法访问应用程序所需的服务，则将重试该服务。如果在重试一定次数后仍无法访问该服务，则应用程序将自动放弃。根据容器的重启策略，Kubernetes和Docker会等待一段时间，然后自动放弃。

下面，我们以一个简单的Golang应用程序为例，检查MySQL服务依赖项是否准备就绪。
```go
 ...
    // Connect to database.
    hostPort := net.JoinHostPort(config.Host, config.Port)
    log.Println("Connecting to database at", hostPort)
    dsn := fmt.Sprintf("%s:%s@tcp(%s)/%s?timeout=30s",
        config.Username, config.Password, hostPort, config.Database)
    db, err = sql.Open("mysql", dsn)
    if err != nil {
        log.Println(err)
    }
    var dbError error
    maxAttempts := 20
    for attempts := 1; attempts <= maxAttempts; attempts++ {
        dbError = db.Ping()
        if dbError == nil {
            break
        }
        log.Println(dbError)
        time.Sleep(time.Duration(attempts) * time.Second)
    }
    if dbError != nil {
        log.Fatal(dbError)
    }
    log.Println("Application started successfully.")
    ...
```
“快速失败”是按合同设计的重要原则，有助于确保系统的鲁棒性和可预测性。在前面的代码中，如果重试机制失败，则将 `log.Fatal(dbError)`报告该错误并结束该过程。此外，K8S和Docker容器重新启动回滚功能可确保不会因反复访问应用程序依赖项的失败尝试而耗尽系统资源。
# 方法2：独立的服务依赖性检查逻辑
在现实世界中，某些遗留应用程序和框架无法调整。因此，我们希望将它们的检查策略和应用程序逻辑分离。

一种常见的方法是在容器的Dockerfile启动脚本中添加相关的服务依赖关系检查逻辑。有关此方法的更多信息，请参阅此[Docker文档](https://docs.docker.com/compose/startup-order/)。另一种方法是使用Kubernetes pod机制本身来添加依赖项检查逻辑。

在开始之前，我们必须了解Pod的生命周期。 

首先，窗格包含三种类型的容器：

1. 基础结构容器：这是著名的暂停容器。
2. 初始化容器：这是一个[初始化容器](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)，通常用于初始化和准备应用程序。仅在等待所有初始化容器完成运行之后，应用程序容器才能启动。
3. 主容器：这是一个应用程序容器。
Kubernetes的最佳实践通常依赖于初始化容器来检查服务依赖性。我们使用以下WordPress示例说明如何完成此操作。
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  ports:
  - name: mysql
    port: 3306
  selector:
    app: mysql
---
apiVersion: v1
kind: Service
metadata:
  name: wordpress
spec:
  ports:
  - name: wordpress
    port: 80
    targetPort: 80
  selector:
    app: wordpress
  type: NodePort
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql 
  replicas: 1
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ALLOW_EMPTY_PASSWORD
          value: "true"
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            # Check we can execute queries over TCP (skip-networking is off).
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 2
          timeoutSeconds: 1
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress:4
        ports:
        - containerPort: 80
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql
        - name: WORDPRESS_DB_PASSWORD
          value: ""
      initContainers:
      - name: init-mysql
        image: busybox
        command: ['sh', '-c', 'until nslookup mysql; do echo waiting for mysql; sleep 2; done;']
```
在WordPress部署窗格定义中，我们添加了 `initContainers`。这将检查 `MySQL` 域名是否可以解析以确定MySQL服务依赖项是否准备就绪。

同时，我们在MySQL StatefulSet中引入了 `readinessProbe` 和 `livenessProbe`，以确定MySQL流程是否已准备就绪。在K8S中，只要Pod运行状况良好，它就可以执行ClusterIP访问或DNS解析。
```shell
$ kubectl create -f wordpress.yaml
service "mysql" created
service "wordpress" created
statefulset "mysql" created
deployment "wordpress" created
$ kubectl get pods
NAME                         READY     STATUS     RESTARTS   AGE
mysql-0                      0/1       Running    0          5s
wordpress-797655cf44-w4p87   0/1       Init:0/1   0          5s
$ kubectl get pods
NAME                         READY     STATUS     RESTARTS   AGE
mysql-0                      1/1       Running    0          11s
wordpress-797655cf44-w4p87   0/1       Init:0/1   0          11s
$ kubectl get pods
NAME                         READY     STATUS            RESTARTS   AGE
mysql-0                      1/1       Running           0          14s
wordpress-797655cf44-w4p87   0/1       PodInitializing   0          14s
$ kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
mysql-0                      1/1       Running   0          17s
wordpress-797655cf44-w4p87   1/1       Running   0          17s
$ kubectl describe pods wordpress-797655cf44-w4p87
...
```
注意：

1. 活动性探针：该探针主要用于确定容器是否处于“运行”状态。例如，它可以检测服务死锁，响应缓慢和其他情况。
2. 就绪探针：此探针主要用于确定服务是否已经正常运行。
3. 准备探针不能在初始化容器中使用。
4. 如果pod重新启动，则必须再次运行其所有初始化容器。
# 结论
本文讨论了用于检查服务依赖项的常见解决方案，并提供了一个示例来演示如何使用初始化容器，活跃性和就绪性探针以及其他服务运行状况检查和依赖项检查功能。

Kubernetes提供了灵活的Pod生命周期管理功能。由于篇幅所限，我们没有讨论postStart，preStop和其他生命周期挂钩。