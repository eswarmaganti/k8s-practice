# Kubernetes Init Containers
- A Pod can have multiple containers running apps within it, but it can also have one or more init containers, which can run before the app containers are started.
- Init Containers are exactly like regular containers, except:
  - <u>Init containers always run to completion</u>
  - <u>Each init container must complete successfully before the next one starts.</u>
- If a Pod's init container fails, the kubelet repeatedly restarts that init container until it succeeds.<u> However, if the Pod has a `restartPolicy` of Never, and an init container fails during startup of that pod, Kubernetes treats the overall pod is failed.</u>
- To specify an init container for a Pod, add the `initContainer` filed into the Pod specification, as an array of container items.

## Differences from Regular Containers
- Init Containers supports all the fields and features of app containers, including resource limits, volumes, and security settings. However, the resource requests and limits for an init container are handled differently.
- Regular init containers (excluding sidecar containers) do not support `lifecycle`, `livenessProbe`, `redinessProbe`, or `startupProbe` fields.
- Init Containers must run to completion before the Pod can be ready; sidecar containers continue running during a Pod's lifetime, and do support some probes.
- If you specify multiple init containers for a Pod, kubelet runs each init container sequentially. Each init container must succeed before the next can run.
- When all the init containers have run to completion, kubelet initializes the application containers for the Pods and runs them as usual.

## Differences from Sidecar Containers
- Init containers run and complete their tasks before the main application container starts. Unlike sidecar containers, init containers are not continuously running alongside the main containers.
- Init containers run to completion sequentially, and the main container does not start until all the init containers have successfully completed.
- Init Containers do not support `lifecycle`, `livenessProbe`, `readinessProbe` or `startupProbe` whereas sidecar containers support all these probes to control their lifecycle.
- Init Containers shares the same resources (CPU, memory, network) with the main application containers but do not interact directly with them. They can, however, use shared volumes for data exchange.

## Using Init Containers
- Init Containers have separate mages from app containers, they have some advantage for start-up related code.
  - Init Containers an contain utilities or custom code for setup that are not present in an app image, For example, there is no need to make an image FROM another image just to use a tool like `sed`, `awk`, `python`, or `dig` during setup.
  - The application builder and deployer roles can work independently without the need to jointly build a single app image.
  - Init Containers can run with a different view of the filesystem than app containers in the same Pod. Consequently, then can be given access to Secrets that app containers cannot access.
  - Because init containers run to completion before any app containers start, init containers offer a mechanism to block or delay app container startup until a set of preconditions are met. Once preconditions are met, all the app containers in a Pod can start in parallel.
  - Init containers can securely run utilities or custom code that would otherwise make an app container image less secure. By keeping unnecessary tools separate you can limit the attack surface of your app container image.

## Example
- In the below configuration, we hava created a nginx pod with two init containers, which waits for  `myservice` and `mydb` Kubernetes Services to be completed.
- The Init Containers will be in pending state until they find the respective services

  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx
    labels:
      app: nginx
  spec:
    containers:
      - name: nginx
        image: nginx
        imagePullPolicy: IfNotPresent
        ports:
          - name: nginx
            containerPort: 80
            protocol: TCP
    initContainers:
      - name: init-myservice
        image: busybox
        command: ["sh","-c","until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice to start; sleep 2; done"]
      - name: init-db
        image: busybox
        command: ["sh","-c","until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo Waiting for mydb to start; sleep 2; done"]
    restartPolicy: Always
    
  ```
- create the pod using the below command
  ```
  $ kubectl apply -f pod.yaml 
  pod/nginx created
  ```
- check the status of the pod, the application nginx container will be pending to be scheduled, until all the init containers are completed.
  ```
  $ kubectl get pods
  NAME    READY   STATUS     RESTARTS   AGE
  nginx   0/1     Init:0/2   0          46s
  ```
- To view the complete information about the pod, we can use the below command.
  ```
  $ kubectl describe pods nginx
  Name:             nginx
  Namespace:        default
  Priority:         0
  Service Account:  default
  Node:             minikube/************
  Start Time:       Tue, 17 Dec 2024 10:25:56 +0530
  Labels:           app=nginx
  Annotations:      <none>
  Status:           Pending
  IP:               ***********
  IPs:
  IP:  ***********
  Init Containers:
  init-myservice:
  Container ID:  docker://97e318b2b54a33de000a82ed673899d5a0a5acaafde5d879893cbf1c585246a2
  Image:         busybox
  Image ID:      docker-pullable://busybox@sha256:2919d0172f7524b2d8df9e50066a682669e6d170ac0f6a49676d54358fe970b5
  Port:          <none>
  Host Port:     <none>
  Command:
  sh
  -c
  until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice to start; sleep 2; done
  State:          Running
  Started:      Tue, 17 Dec 2024 10:25:59 +0530
  Ready:          False
  Restart Count:  0
  Environment:    <none>
  Mounts:
  /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-vd8n8 (ro)
  init-db:
  Container ID:  
  Image:         busybox
  Image ID:      
  Port:          <none>
  Host Port:     <none>
  Command:
  sh
  -c
  until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo Waiting for mydb to start; sleep 2; done
  State:          Waiting
  Reason:       PodInitializing
  Ready:          False
  Restart Count:  0
  Environment:    <none>
  Mounts:
  /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-vd8n8 (ro)
  Containers:
  nginx:
  Container ID:   
  Image:          nginx
  Image ID:       
  Port:           80/TCP
  Host Port:      0/TCP
  State:          Waiting
  Reason:       PodInitializing
  Ready:          False
  Restart Count:  0
  Environment:    <none>
  Mounts:
  /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-vd8n8 (ro)
  Conditions:
  Type                        Status
  PodReadyToStartContainers   True
  Initialized                 False
  Ready                       False
  ContainersReady             False
  PodScheduled                True
  Volumes:
  kube-api-access-vd8n8:
  Type:                    Projected (a volume that contains injected data from multiple sources)
  TokenExpirationSeconds:  3607
  ConfigMapName:           kube-root-ca.crt
  ConfigMapOptional:       <nil>
  DownwardAPI:             true
  QoS Class:                   BestEffort
  Node-Selectors:              <none>
  Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
  node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
  Events:
  Type    Reason     Age    From               Message
    ----    ------     ----   ----               -------
  Normal  Scheduled  2m25s  default-scheduler  Successfully assigned default/nginx to minikube
  Normal  Pulling    2m25s  kubelet            Pulling image "busybox"
  Normal  Pulled     2m23s  kubelet            Successfully pulled image "busybox" in 2.819s (2.819s including waiting). Image size: 4042190 bytes.
  Normal  Created    2m23s  kubelet            Created container init-myservice
  Normal  Started    2m23s  kubelet            Started container init-myservice
  ```
- No, we create the services that our pod is looking for using below configuration
  ```
  ---
  apiVersion: v1
  kind: Service
  metadata:
  name: mydb
  spec:
  selector:
  app: db
  type: ClusterIP
  ports:
  - targetPort: 80
  port: 80
  name: dbport
  
  ---
  apiVersion: v1
  kind: Service
  metadata:
  name: myservice
  spec:
  selector:
  app: service
  type: ClusterIP
  ports:
  - targetPort: 80
  port: 80
  name: service-port

  ```
- One we created the services, the pod now will be in running state, we can check the events using the below comamnd
  ```
  $ kubectl get pods --watch
  NAME    READY   STATUS     RESTARTS   AGE
  nginx   0/1     Init:0/2   0          7m34s
  nginx   0/1     Init:1/2   0          8m6s
  nginx   0/1     PodInitializing   0          8m9s
  nginx   1/1     Running           0          8m10s
  ```