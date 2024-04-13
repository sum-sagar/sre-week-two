![Screenshot 2024-04-13 234832](https://github.com/sum-sagar/sre-week-two/assets/166273480/688bda4b-af01-43ed-8b0f-8a10ef20e16c)
Noticed above errors on the Alerts Manager wrt deployment failing

Followed the below steps for troubleshooting



1: Following command gives us the entire status of the minikube cluster within our namespace sre, As we can see the upcommerce-app-two is not in a ready state
```
@sum-sagar ➜ /workspaces/sre-week-two (main) $ kubectl get deploy -n sre
NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
grafana                             1/1     1            1           32m
prometheus-kube-state-metrics       1/1     1            1           32m
prometheus-prometheus-pushgateway   1/1     1            1           32m
prometheus-server                   1/1     1            1           32m
upcommerce-app-two                  0/1     1            0           30m
```
2: Following command lists all the pods in the namespace sre, Observe the pod not ready: upcommerce-app-two-56cff9c64d-srs5l 
```
@sum-sagar ➜ /workspaces/sre-week-two (main) $ kubectl get pods -n sre
NAME                                                READY   STATUS    RESTARTS   AGE
grafana-557d966c8c-rqx25                            1/1     Running   0          22m
prometheus-alertmanager-0                           1/1     Running   0          23m
prometheus-kube-state-metrics-65468947fb-6bq2j      1/1     Running   0          23m
prometheus-prometheus-node-exporter-8l6xh           1/1     Running   0          23m
prometheus-prometheus-pushgateway-76976dc66-2hbjl   1/1     Running   0          23m
prometheus-server-8444b5b7f7-t7w7m                  2/2     Running   0          23m
upcommerce-app-two-56cff9c64d-srs5l                 0/1     Pending   0          20m
```

3: Following command gives the description of the failing pod. Notice the issue "1 Insufficient cpu. preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod"
```
@sum-sagar ➜ /workspaces/sre-week-two (main) $ kubectl describe pods upcommerce-app-two-56cff9c64d-srs5l -n sre
Name:             upcommerce-app-two-56cff9c64d-srs5l
Namespace:        sre
Priority:         0
Service Account:  default
Node:             <none>
Labels:           app=upcommerce-app-two
                  pod-template-hash=56cff9c64d
Annotations:      <none>
Status:           Pending
IP:               
IPs:              <none>
Controlled By:    ReplicaSet/upcommerce-app-two-56cff9c64d
Containers:
  upcommerce:
    Image:      uonyeka/upcommerce:v3
    Port:       5000/TCP
    Host Port:  0/TCP
    Limits:
      cpu:     10
      memory:  4Gi
    Requests:
      cpu:        10
      memory:     4Gi
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-f2vnl (ro)
Conditions:
  Type           Status
  PodScheduled   False 
Volumes:
  kube-api-access-f2vnl:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Guaranteed
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason            Age                 From               Message
  ----     ------            ----                ----               -------
  Warning  FailedScheduling  118s (x5 over 22m)  default-scheduler  0/1 nodes are available: 1 Insufficient cpu. preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod..
```
4:Following command gives the number of available CPU's that can be allocated within the cluster,As seen max is 2
```
@sum-sagar ➜ /workspaces/sre-week-two (main) $ kubectl get nodes minikube -o yaml | grep -A2 alloc
  allocatable:
    cpu: "2"
    ephemeral-storage: 32847680Ki
```
5: Follwing code gets us the current deployment docker file to get the status of deployment config. Notice the CPU limit as 10
```
@sum-sagar ➜ /workspaces/sre-week-two (main) $ kubectl describe deploy upcommerce-app-two -n sre
Name:                   upcommerce-app-two
Namespace:              sre
CreationTimestamp:      Sat, 13 Apr 2024 17:49:29 +0000
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=upcommerce-app-two
Replicas:               1 desired | 1 updated | 1 total | 0 available | 1 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=upcommerce-app-two
  Containers:
   upcommerce:
    Image:      uonyeka/upcommerce:v3
    Port:       5000/TCP
    Host Port:  0/TCP
    Limits:
      cpu:        10
      memory:     4Gi
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      False   MinimumReplicasUnavailable
  Progressing    False   ProgressDeadlineExceeded
OldReplicaSets:  <none>
NewReplicaSet:   upcommerce-app-two-56cff9c64d (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  24m   deployment-controller  Scaled up replica set upcommerce-app-two-56cff9c64d to 1
```

6: We need to change the following in deployment.yaml file
limits:
~~cpu: "10"~~             
  cpu: "1"

7: After changing and committing changes to the master branch, we repeat the steps and check for the current pod status of minikube cluster within namespace sre
```
@sum-sagar ➜ /workspaces/sre
-week-two (main) $ kubectl get deployment -n sre
NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
grafana                             1/1     1            1           65m
prometheus-kube-state-metrics       1/1     1            1           66m
prometheus-prometheus-pushgateway   1/1     1            1           66m
prometheus-server                   1/1     1            1           66m
upcommerce-app-two                  1/1     1            1           63m
@sum-sagar ➜ /workspaces/sre-week-two (main) $ kubectl get pods -n sre
NAME                                                READY   STATUS    RESTARTS        AGE
grafana-557d966c8c-rqx25                            1/1     Running   2 (4m37s ago)   67m
prometheus-alertmanager-0                           1/1     Running   2 (4m37s ago)   67m
prometheus-kube-state-metrics-65468947fb-6bq2j      1/1     Running   2 (4m38s ago)   67m
prometheus-prometheus-node-exporter-8l6xh           1/1     Running   2 (4m37s ago)   67m
prometheus-prometheus-pushgateway-76976dc66-2hbjl   1/1     Running   2 (4m37s ago)   67m
prometheus-server-8444b5b7f7-t7w7m                  2/2     Running   4 (4m37s ago)   67m
upcommerce-app-two-846fd54f5d-w62lz                 1/1     Running   1 (4m38s ago)   5m32s
```
We can see that the upcommerce-app-two-846fd54f5d-w62lz pod is now running.
