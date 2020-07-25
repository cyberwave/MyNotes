在开发的过程中，有的场景需要pod中的应用程序去上报自己的ip地址和端口。当然也可以通过一个第三方程序专门获取各种pod的ip
实现过程有两种方式：使用k8s客户端获取和使用使用环境变量注入

# 使用k8s客户端（sdk）

这引入了一个 `role` 的概念。在获取k8s信息的时候，会由于权限问题而 `forbidden`。这时候需要为之添加一个`role` 

1. 获取 k8s 的配置
```go
import(
  corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
)
func GetK8sConfig()(*kubernetes.Clientset, error){
  config, err := rest.InClusterConfig()
  if err != nil {
  	return nil, err
  }
  
  clientSet, err := kubernetes.NewForConfig(config)
  if err != nil {
    return nil, err
  }
  return clientSet, nil
}

func GetK8sNodeIps() ([]string, error){
  clientSet, err := GetK8sConfig()
  if err != nil {
    return nil, err
  }
  k8sNodes, err := clientSet.CoreV1().Nodes().List(metav1.ListOptions{})
  if err != nil {
    return nil, err
  }
  var nodeIpList []string
  for i, node := range k8sNodes.Items {
    for _, addr := range node.Status.Address{
      // 如果不需要过滤，可以把 if语句删除，只保留if语句块内的 append操作
      if addr.Type == corev1.NodeInternalIP {
        nodeIpList = append(nodeIpList, add.Address)
        return
      }
    }
  }
  return nodeIpList, nil
}

// GetServices 获取服务信息，namespace 可以由环境变量注入
func GetServices(namespace string)error{
  clientSet, err := GetK8sConfig()
  if err != nil {
		return err
	}
  srvList, err := clientSet.CoreV1().Services(namespace).List(metav1.ListOptions{}){
  	return err
  }
  for _, item := range srvList.Items {
    fmt.Printf("service name:%s\n", item.Name)
    fmt.Printf("service spec type:%s\n", item.Spec.Type)
    // 可以打印 item，根据需要取值
  }
}
```

# 使用环境变量注入

参考 [Expose Pod Information to Containers Through Environment Variables](https://kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/)

使用 k8s 的 [Downward API](https://kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/#the-downward-api) 将Pod 和容器字段暴露到运行的容器中。文中提供了两种方式：环境变量和文件挂载

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-envars-fieldref
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "sh", "-c"]
      args:
      - while true; do
          echo -en '\n';
          printenv MY_NODE_NAME MY_POD_NAME MY_POD_NAMESPACE;
          printenv MY_POD_IP MY_POD_SERVICE_ACCOUNT;
          sleep 10;
        done;
      env:
        - name: MY_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: MY_POD_SERVICE_ACCOUNT
          valueFrom:
            fieldRef:
              fieldPath: spec.serviceAccountName
  restartPolicy: Never
```

上述配置文件的 `env` 字段是 [EnvVars](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#envvar-v1-core)的数组。环境变量从Pod字段中获取它的名字。

**注意：**示例中的字段是Pod的字段，而不是Pod中容器的字段。