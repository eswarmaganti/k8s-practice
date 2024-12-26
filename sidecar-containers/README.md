# SideCar Containers
- Kubernetes implements sidecar containers as a special case of init containers; sidecar containers remains running after the Pod startup.
- Sideccar containers are the secondary containers that tun along with the main application container within the same Pod. These containers are used to enhance or to extend the functionality of the primary app container by providing the additional services, or functionality such as logging, monitoring, security, or data synchronization with out directly altering the primary application code.
- Provided that your cluster has the `SidecarContainers` feature gate enabled, you can specify a `restartPolicy` for containers listed in a Pod's `initContainers` filed.
- These restartable sidecar containers are independent from other init containers and from the main application container within the same pod. These can be started, stopped, restarted without effecting the main application container and other init containers.

## Sidecar Containers and Pod lifecycle
- <u>If an init container is created with its `restartPolicy` set to `Always`, it will start and remain running during the entire life of the Pod. This can be helpful for running supporting services seperated from the amin application containers.</u>
- If a `readinessProbe` is specified for this init container, its result will be used to determine the ready state of the Pod.
- Since these containers are defined as init containers, they benefit from the same ordering and sequential guarantees as regular init containers, allowing you to mix sidecar containers with regular containers fro complex Pod initialization flows.
- Compared to regular init containers, sidecars defined within `initContainers` continue to run after they have started. This is important when there is more than one entry inside `.spec.initContainers` for a Pod.
- After a sidecar style init container is running the kubelet then startes the next init container from the ordered `.spec.initContainers` list.
- That status either becomes true because there is a process running in the container and no startup probe defines, or as a result of its `startupProbe` succeeding.
- <u>Upon the Pod termination, the kubelet postpones terminating sidecar containers until the main application container has fully stopped. The sidecar containers are then shutdown in the opposite order of their appearance in the pos specification.</u>
- This approach ensures that the sidecar containers remains operational, supporting other containers within the Pod, until their service is no longer required.

### Using Sidecar Containers in Deployment
    ```
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: sidecar-demo
      labels:
        app: sidecar
    spec:
      selector:
        matchLabels:
          app: sidecar
      replicas: 1
      strategy:
        rollingUpdate:
          maxUnavailable: 1
      template:
        metadata:
          labels:
            app: sidecar
        spec:
          containers:
            - name: app-container
              image: busybox
              command: ["sh","-c","while true; do echo $(date) Writing the logs >> /var/log/app.log; sleep 1; done "]
              volumeMounts:
                - mountPath: /var/log
                  name: varlog
          initContainers:
            - name: sidecar
              image: busybox
              command: ["sh","-c", "tail -F /var/log/app.log"]
              restartPolicy: Always
              volumeMounts:
                - mountPath: /var/log
                  name: varlog
          volumes:
            - name: varlog
              emptyDir: {}
    ```

- We can apply the above configuration and check the containers working using the below commands
    ```
    $ kubectl apply -f examples/deployment/deployment.yaml
    deployment.apps/sidecar-demo created
  
    $ kubectl get pods --watch
    NAME                            READY   STATUS            RESTARTS   AGE
    sidecar-demo-55675cd949-g79sg   1/2     PodInitializing   0          6s
    sidecar-demo-55675cd949-g79sg   2/2     Running           0          7s
    
    $ kubectl exec -it sidecar-demo-55675cd949-g79sg -- tail /var/log/app.log
    Defaulted container "app-container" out of: app-container, sidecar (init)
    Wed Dec 18 02:23:34 UTC 2024 Writing the logs
    Wed Dec 18 02:23:35 UTC 2024 Writing the logs
    Wed Dec 18 02:23:36 UTC 2024 Writing the logs
    Wed Dec 18 02:23:37 UTC 2024 Writing the logs
    Wed Dec 18 02:23:38 UTC 2024 Writing the logs
    Wed Dec 18 02:23:39 UTC 2024 Writing the logs
    Wed Dec 18 02:23:40 UTC 2024 Writing the logs
    Wed Dec 18 02:23:41 UTC 2024 Writing the logs
    Wed Dec 18 02:23:42 UTC 2024 Writing the logs
    Wed Dec 18 02:23:43 UTC 2024 Writing the logs                                                                        
           
    $ kubectl logs sidecar-demo-55675cd949-g79sg -c sidecar 
    tail: can't open '/var/log/app.log': No such file or directory
    tail: /var/log/app.log has appeared; following end of new file
    Wed Dec 18 02:22:50 UTC 2024 Writing the logs
    Wed Dec 18 02:22:51 UTC 2024 Writing the logs
    Wed Dec 18 02:22:52 UTC 2024 Writing the logs
    Wed Dec 18 02:22:53 UTC 2024 Writing the logs
    Wed Dec 18 02:22:54 UTC 2024 Writing the logs
    Wed Dec 18 02:22:55 UTC 2024 Writing the logs
    Wed Dec 18 02:22:56 UTC 2024 Writing the logs
    Wed Dec 18 02:22:57 UTC 2024 Writing the logs
    Wed Dec 18 02:22:58 UTC 2024 Writing the logs
    Wed Dec 18 02:22:59 UTC 2024 Writing the logs
    Wed Dec 18 02:23:00 UTC 2024 Writing the logs
    Wed Dec 18 02:23:01 UTC 2024 Writing the logs
    Wed Dec 18 02:23:02 UTC 2024 Writing the logs
    Wed Dec 18 02:23:03 UTC 2024 Writing the logs
    Wed Dec 18 02:23:04 UTC 2024 Writing the logs
    Wed Dec 18 02:23:05 UTC 2024 Writing the logs    
    ```


### Using Sidecar Containers in Jobs
- The below manifest shows how to use the sidecar containers in a job
    ```
    apiVersion: batch/v1
    kind: Job
    metadata:
      name: sidecar-job-demo
      labels:
        app: sidecar
    spec:
      template:
        spec:
          containers:
            - name: job
              image: busybox
              command: ["sh", "-c", "echo $(date) Job is running successfully, logging the status >> /var/log/job-status.log"]
              volumeMounts:
                - mountPath: /var/log
                  name: varlog
          initContainers:
            - name: sidecar
              image: busybox
              command: ["sh","-c","tail -F /var/log/job-status.log"]
              restartPolicy: Always
              volumeMounts:
                - mountPath: /var/log
                  name: varlog
          restartPolicy: Never
          volumes:
            - name: varlog
              emptyDir: {}
    
    ```
- We can apply the above configuration using the commands shown below
    ```
    $ kubectl apply -f examples/job/job.yaml
    job.batch/sidecar-job-demo created
    
    $ kubectl get jobs
    NAME               STATUS    COMPLETIONS   DURATION   AGE
    sidecar-job-demo   Running   0/1           9s         9s
    
    $ kubectl get pods
    NAME                     READY   STATUS      RESTARTS   AGE
    sidecar-job-demo-skxvm   1/2     Completed   0          13s
    
    $ kubectl logs sidecar-job-demo-skxvm -c sidecar
    tail: can't open '/var/log/job-status.log': No such file or directory
    tail: /var/log/job-status.log has appeared; following end of new file
    Wed Dec 18 02:33:59 UTC 2024 Job is running successfully, logging the status
    ```

