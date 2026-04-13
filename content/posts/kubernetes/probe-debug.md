---
title: "K8s 折腾记录：Startup Probe 权限排障与 YAML 字段陷阱"
date: 2026-04-13T09:22:00+08:00
draft: false
categories: ["Kubernetes"]
tags: ["k8s", "cka", "troubleshooting"]
description: "记录 K8s Startup Probe 权限排障过程，以及 restartPolicy 字段层级陷阱的发现"
---


## 现象：Pod 在"假 Running"和重启之间循环

早上部署 Nginx Pod，Startup Probe 一直报权限错误，同时发现重启策略好像没生效。

```bash
kubectl get pods -o wide -w
```


### 日志输出部分

```text
NAME         READY   STATUS    RESTARTS   AGE   IP              NODE         NOMINATED NODE   READINESS GATES
nginx-demo   0/1     Running   0          1s    10.244.76.157   k8s-node-3   <none>           <none>
nginx-demo   0/1     Running   1 (1s ago)   34s   10.244.76.157   k8s-node-3   <none>           <none>
nginx-demo   0/1     Running   2 (1s ago)   64s   10.244.76.157   k8s-node-3   <none>           <none>
nginx-demo   0/1     Running   3 (1s ago)   94s   10.244.76.157   k8s-node-3   <none>           <none>
nginx-demo   0/1     Running   4 (1s ago)   2m4s   10.244.76.157   k8s-node-3   <none>           <none>
nginx-demo   0/1     Completed   4 (31s ago)   2m34s   10.244.76.157   k8s-node-3   <none>           <none>
nginx-demo   0/1     Completed   4             2m35s   10.244.76.157   k8s-node-3   <none>           <none>
nginx-demo   0/1     Completed   4             2m35s   10.244.76.157   k8s-node-3   <none>           <none>
nginx-demo   0/1     Completed   4             2m35s   10.244.76.157   k8s-node-3   <none>           <none>

1. 0/1 Running：容器在跑（STATUS=Running），但 Ready 状态是 0/1（因为 Startup Probe 一直失败， deemed 为 Not Ready）
2. RESTARTS 递增：0→1→2→3→4，每 30 秒左右重启一次（对应 Startup Probe 失败重试 5 次后的重启策略）
3. 最后变成 Completed：探针失败次数耗尽，容器主进程退出，Pod 进入 Completed 状态（不再重启）
```



### 现象排查

```text
Events:
  Type     Reason     Age                    From               Message
  ----     ------     ----                   ----               -------
  Normal   Scheduled  8m41s                  default-scheduler  Successfully assigned default/nginx-demo to k8s-node-3
  Normal   Pulled     6m38s (x5 over 8m41s)  kubelet            spec.containers{nginx}: Container image "quay.io/redhattraining/hello-world-nginx:latest" already present on machine and can be accessed by the pod
  Normal   Created    6m38s (x5 over 8m41s)  kubelet            spec.containers{nginx}: Container created
  Normal   Started    6m38s (x5 over 8m41s)  kubelet            spec.containers{nginx}: Container started
  Warning  Unhealthy  6m8s (x15 over 8m28s)  kubelet            spec.containers{nginx}: Startup probe failed: sh: /inited: Permission denied
  Normal   Killing    6m8s (x5 over 8m8s)    kubelet            spec.containers{nginx}: Container nginx failed startup probe, will be restarted
  Warning  BackOff    6m8s                   kubelet            spec.containers{nginx}: Back-off restarting failed container nginx in pod nginx-demo_default(eba59371-c6de-449c-94ec-cf3f4bf582e4)
```



### 为什么是 `0/1 Running` 而不是 `CrashLoopBackOff`？

- **`CrashLoopBackOff`**：容器进程崩溃了（Exit Code 非 0），kubelet 在指数退避重试
- **`0/1 Running`**：容器进程还在（nginx 没崩）,**但 Startup Probe 认为它还没准备好**，所以 Ready 状态是 False

本例中 nginx 进程本身没问题，是探针命令（`sh /inited`）权限不足返回错误，kubelet 认为"容器还没启动完成"，于是不断重启容器尝试，直到 `restartPolicy` 次数耗尽变成 Completed。



### 错误配置部分

一开始把 `restartPolicy` 放容器里了
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-demo
  labels:
    type: app
    version: v0.1.0
  namespace: "default"
spec:
  containers:
  - name: nginx
    image: quay.io/redhattraining/hello-world-nginx:latest
    imagePullPolicy: IfNotPresent
    startupProbe:
      exec:
        command:
        - sh
        - -c
        - "sleep 3;echo 'success' > /inited;"
      failureThreshold: 3
      successThreshold: 1
      periodSeconds: 10
      timeoutSeconds: 5
    command:
    - nginx
    - -g
    - "daemon off;"
    workingDir: /usr/share/nginx/html
    ports:
    - name: http
      containerPort: 8080
      protocol: TCP
    restartPolicy: OnFailure # 错误位置！K8s 会静默忽略
```



### 诡异现象：apply 成功，但配置不生效

```bash
[root@k8s-node-1 pods]# kubectl apply --dry-run=server -f nginx-demo.yaml
pod/nginx-demo created (server dry run)
```



### 没有报错！ 但查看实际配置：

```text
[root@k8s-node-1 pods]# kubectl apply -f nginx-demo.yaml
[root@k8s-node-1 pods]# kubectl get pod nginx-demo -o yaml | grep restartPolicy
      {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"labels":{"type":"app","version":"v0.1.0"},"name":"nginx-demo","namespace":"default"},"spec":{"containers":[{"command":["nginx","-g","daemon off;"],"image":"quay.io/redhattraining/hello-world-nginx:latest","imagePullPolicy":"IfNotPresent","name":"nginx","ports":[{"containerPort":8080,"name":"http","protocol":"TCP"}],"restartPolic":"OnFailure","startupProbe":{"exec":{"command":["sh","-c","sleep 3;echo 'success' \u003e /inited;"]},"failureThreshold":3,"periodSeconds":10,"successThreshold":1,"timeoutSeconds":5},"workingDir":"/usr/share/nginx/html"}],"securityContext":{"fsGroup":101,"runAsUser":101}}}
    restartPolicy: OnFailure
  restartPolicy: Always
```



### 发现华点:

-   容器级别显示 restartPolicy: OnFailure（原始输入被保留）
-   但 Pod 级别显示 restartPolicy: Always（实际生效的是默认值！）



### 根因：K8s API 的"静默丢弃"机制

K8s API 是宽松解析（lenient parsing）：

    不认识容器里的 restartPolicy 字段 → 不报错，直接丢弃
    用默认值 Always 填充 Pod 级别的 spec.restartPolicy
    所以 kubectl get 能看到两个字段：一个是保留的"尸体"，一个是实际生效的默认值

这就是为什么 Docker Compose 用户容易踩坑：

    Docker：服务级别配置 restart: always
    K8s：Pod 是原子单位，重启策略必须是 Pod 级别（spec 下）



### 解决方案

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-demo
  labels:
    type: app
    version: v0.1.0
  namespace: "default"
spec:
  securityContext:
    fsGroup: 101
    runAsUser: 101
  restartPolicy: OnFailure #restartPolicy 必须是 Pod 级别（spec 下），放在 containers 里会被静默忽略，实际使用默认值 Always。
  containers:
  - name: nginx
    image: quay.io/redhattraining/hello-world-nginx:latest
    imagePullPolicy: IfNotPresent
    startupProbe:
      exec:
        command:
        - sh
        - -c
        - "sleep 3;echo 'success' > /tmp/inited;"
      failureThreshold: 3
      successThreshold: 1
      periodSeconds: 10
      timeoutSeconds: 5
    command:
    - nginx
    - -g
    - "daemon off;"
    workingDir: /usr/share/nginx/html
    ports:
    - name: http
      containerPort: 8080
      protocol: TCP
 ```



### 验证

```text
[root@k8s-node-1 pods]# kubectl apply -f nginx.yaml
pod/nginx-demo created
[root@k8s-node-1 pods]# kubectl get pod nginx-demo -o yaml | grep restartPolicy
      {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"labels":{"type":"app","version":"v0.1.0"},"name":"nginx-demo","namespace":"default"},"spec":{"containers":[{"command":["nginx","-g","daemon off;"],"image":"quay.io/redhattraining/hello-world-nginx:latest","imagePullPolicy":"IfNotPresent","name":"nginx","ports":[{"containerPort":8080,"name":"http","protocol":"TCP"}],"startupProbe":{"exec":{"command":["sh","-c","sleep 3;echo 'success' \u003e /tmp/inited;"]},"failureThreshold":3,"periodSeconds":10,"successThreshold":1,"timeoutSeconds":5},"workingDir":"/usr/share/nginx/html"}],"restartPolicy":"OnFailure","securityContext":{"fsGroup":101,"runAsUser":101}}}
  restartPolicy: OnFailure # 输出OnFailure而不是Always 
  ```



### 总结

    Startup Probe 权限错误：非 root 容器别写根目录，用 /tmp 或 emptyDir 卷
    restartPolicy 层级陷阱：必须是 Pod 级别（spec 下），放容器里会被静默忽略
    K8s API 行为：--dry-run=server 不校验字段位置合理性，只会看语法对不对



### 参考与延伸阅读

- [Pod Restart Policy - Kubernetes Documentation](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy) - 官方对 restartPolicy 层级的明确说明
- [Lenient decoding of API requests #82292](https://github.com/kubernetes/kubernetes/issues/82292) - K8s API 宽松解析机制的技术讨论

---

*最后更新：2026-04-13 10:41*