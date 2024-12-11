# Autoscaling Workloads
- In kubernetes we can scale workloads depending on the current demand of resources. This allows the cluster to react to the changes in the resource demand more elastically and efficiently
- When we scale a workload we can either increse or decrease the number of replicas managed by the workload, or adjust the resources available to the replicas in place.
- The first approach is referred as *Horizontal Scaling* and the second one is referrred as *Vertical Scaling*. There are manual and automatic ways to scale the workload based on the usecases.

## Scaling Workloads Automatically
- The concept of autoscaling in kubernetes refers to the ability to automatically update an object that manages a set of pods (Deployment, StatefulSet)
    ### Scaling Workloads Vertically
    - We can automatically scale workloads vertically using *VerticalPodAutoscaler (VPA)*
    - Unlike HPA, the VPA doesn't come with kubernetes by default, but is a separate project that can be found on github
    - Once installed, it allows us to create a custom resource definition (CRD) for your workloads which define how and when to scale the resources for the managed replicas.
    - At the moment, the VPA can be operate in four different modes
      - `Auto` : Currently, `Recreate` might change to in-place in future
      - `Recreate` : The VPA assigns resource requests on pod creation as well as updates them on existing pods by evicting them when the requested resources differ signiicantly from the new recommendation.
      - `Initial` : The VPA oly assigns resource requests on pod creation and never changes them later.
      - `Off` : The VPA doesn't change the resource requirements of the pods. The recommendations are calculated and can be inspected in the VPA object.
    - **Autoscaling Based on Cluster Size**
    - **Event Driven Autoscaling**
    - **Autoscaling Based on Schedules**


    ### Scaling workloads Horizontally
    - <u>A HorizontalPodAutoscalaer automatically  updates a workload resource such as Deployment or StatefulSet, with the aim of automatically scaling the workload to match the demand.</u>
    - Horizontal scaling means that the response to increased load is to deploy more pods. This is different from vertical scaling, which for kubernetes would mean assigning more resources to the pods that are already running for the workload.
    - <u>If the load decreases, and the number of pods in above the configured minimum, the HorizontalPodAutoscalaer instructs the worklod resource to scale back down.</u>

      #### Example
      - create a deployment for apache webserver using the below manifest
      
        ```
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: apache
        spec:
          strategy:
            rollingUpdate:
              maxUnavailable: 1
          selector:
            matchLabels:
              tier: web
          template:
            metadata:
              labels:
                tier: web
            spec:
              containers:
                - name: httpd
                  image: httpd
                  ports:
                    - containerPort: 8080
                  livenessProbe:
                    httpGet:
                      path: /
                      port: 80
                  resources:
                    requests:
                      cpu: "100m"
                    limits:
                      cpu: "200m"

        ``` 
      - Expose the deployment as a service using kubernetes service resource
        
        ```
        apiVersion: v1
        kind: Service
        metadata:
          name: apache-server
        spec:
          selector:
            tier: web
          type: NodePort
          ports:
            - port: 80
              targetPort: 80
        ``` 
      - Now we can create a manifest for our HPA to scale the deployment based on incoming load
      
        ```
        apiVersion: autoscaling/v2
        kind: HorizontalPodAutoscaler
        metadata:
          name: apache-hpa
        spec:
          scaleTargetRef:
            apiVersion: apps/v1
            kind: Deployment
            name: apache
          minReplicas: 3
          maxReplicas: 10
          metrics:
            - type: Resource
              resource:
                name: cpu
                target:
                  type: Utilization
                  averageUtilization: 50
        
        ``` 
      - This HPA will monitor the incoming load to our Deployment workload and scale the number of replicas when the average CPU utilization of workload is greater than 50%
      - We can also use `kubectl` command to define and scale workload as shown belo
        - `kubectl autoscale deployment <deployment_name> --cpu-percent=50 --min=3 --max=10`
      - To test the HPA we can increase the workload on our Deployment using the pod which runs a busybox image as follows
        
        ```
        apiVersion: v1
        kind: Pod
        metadata:
          name: load-test-pod
          labels:
            app: load-test
        spec:
          containers:
            - name: busybox
              image: busybox
              command: ["/bin/sh","-c"]
              args: ['while true; do wget -q -O- http://apache-server:80/; sleep 0.01; done']
        ```
      - This `load-test-pod` will continuously send requests to our service, which will increase the load on our workload. the HPA will monitor the load on our resource and autoscale the replicas.

      #### Scaling workload based on custom metrics
      - The CPU utilization metric is a resource metric, since it's represented as a percentage of resource specified on pod containers. By default the only supported resource metrics is `memory` and `CPU`. These resources don't change names from cluster to cluster and should be available, as long as metrics.k8s.io API is available.
      - We can also specify metrics in terms of direct value, instead of percentage of requested value by using `target.type` as `AverageValue` instead of `Utilization`, and setting the `target.averageValue` field instead of `target.averageUtilization`
        
        ```
        metrics:
          - type: Resource
            resource:
              name: memory
              target:
                type: AverageValue
                averageValue: 500Mi
        ```
      - There are two two other types of metrics, both of them considered **custom metrics**: **pod metrics** and **object metrics**. These metrics may have names which are cluster specific, and require a more advanced cluster monitoring setup.  
        ##### Pod Metrics:
        - These metrics describe Pods, and are avaraged together across Pods and compared with a target value to determine the replica count. THey work like resource metrics, except that they only support a `target` type of `AverageValue`.
        - Pod metrics are specified in metric block in the following way
          ```
          metrics:
            - type: Pods
              pods:
                metric:
                  name: packets-per-second
                target:
                  type: AverageValue
                  averageValue: 1k
          ```
        ##### Object Metrics
        - The second alternative metric type is **object metrics**. These metrics describe a different object in the same namespace, instead of describing Pods. The metrics are not necessarily fetched from the object, they only describe it.
        - Object metrics support target types of both `Value` and `AverageValue`.
        - With `Value`, the target is directly compared to the returned metric from the API. With `AverageValue`, the value returned from the custom metrics API is divided by the number of Pods before being compared to the target.
        - The following is an example of objcet metric for `requests-per-second`
          
          ```
          metrics:
            - type: Object
              object: 
                metric: 
                  name: requests-per-second
                describedObject:
                  apiVersion: networking.k8s.io/v1
                  kind: Ingress
                  name: main-route
                type:
                  name: Value
                  value: 2k
          ```
      - If we provide multiple such metric blocks, the HorizontalPodAutoscalar will consider each metric in turn. The HPA will calculate proposed replica counts for each metric, and then choose the one with the highest replica count.