# EFk Stack on Kubernetes
EFK Stands for ElasticSearch, Fluentd, and Kibana. EFK is the popular and open-source choice for the Kubernetes log aggregation and analysis.
1. **Elasticsearch** is a distributed ans scalable search engine commonly used to shift through large volumes of data log. It is a NoSQL database based on the Lucene search engine (search library from Apache). It's primary work is to store logs and retrive logs from fluentd.
2. **Fluentd** is a log shipper. It is an open source log collection agent which support multiple data sources and output formats. Also it can forward logs to solutions like Stackdriver, CLoudwatch, elasticsearch, Splunk, Bigquery and much more. TO be short, it's an unifying layer between systems that generate log data and systems that store log data.
3. **Kibana** is UI tool for querying, data visualization and dashboards. It is a query engine which allows you to explore your data through a web interface, build visualizations for events log, query-specific to filter information for detecting issues. We can visually build any type of dashboards using kibana. **Kibana Query Language (KQL)** is used for querying elasticsearch data. Here we use kibana to query **indexed data in elasticsearch.**

## EFK Architecture
![alt EFK Architecture](images/efk.png)

EFK Components gets deployed as folows
1. **Fluentd** :- Deployed as DaemonSet as it need to collect te container logs from all the nodes. It connects to the ElasticSearch service endpoint to forward the logs
2. **ElasticSearch** :- Deployed as StatefulSet as it holds the log data. We also expose the service endpoint for Fluentd and Kibana to connect to it.
3. **Kibana** :- Deployed as deployment and connects to elasticsearch service endoint.

## Create a `kube-logging` Namespace
- We can define the `kube-logging` namespace using the below manifest
  
    ```
    apiVersion: v1
    kind: Namespace
    metadata:
        name: kube-logging
    ```
- To create the Namespace that we defined, we can use te below command
  
    ```
    $ kubectl apply -f namespace.yaml
    namespace/kube-logging created
    ```

## Deploy ElsticSearch StatefulSet
- ElasticSearch is deployed as **StatefulSet** and the multiple replicas connect with each other using a headless service. The headless service helps in the DNS domain of the pods.

    ### ElasticSearch Headless Service
    - The service manifest for ElasticSearch StatefulSet will be as below.
    
        ![alt elasticsearch service](images/elasticsearch-service.png)

    - Create the Headless service for ElasticSearch using the below command
    
        ```
        $ kubectl apply -f elasticsearch/service.yaml 
          service/elasticsearch created
        ```
    - TO view the details of the service created, we can use the below command
  
        ```
        $ kubectl get svc/elasticsearch -n kube-logging -o wide
          NAME            TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)             AGE   SELECTOR
          elasticsearch   ClusterIP   None         <none>        9200/TCP,9300/TCP   46s   app=elasticsearch
        ```

    - We define a `Service` called `elasticsearch` in the `kube-logging` Namespace, and five it the `app: elasticsearch` label.
    - We then set the `.spec.selector` to `app: elasticsearch` so that the Service selects Pods with the `app: elasticsearch` label.
    - When we assign our ElasticSearch StatefulSet with this Service, the Service will return DNS A records that point to ElasticSearch Pods with the `app: ealsticsearch` label
    - We then set `clusterIP: None`, which renders the service headless. Finally we define ports 9200 and 9300 which are used to interact with REST API and for inter-node communication.
    
    ### ELasticSearch StatefulSet
    - A Kubernetes StatefulSet allows you to assign a stable identity to Pods and grant them stable, persistent storage.
    - ElasticSearch requires stable storage to persist data across Pod rescheduling and restarts.
        
        ![alt elasticsearch statefulset](images/elasticsearch-statefulset.png)
    - To create the statefulset we can use the below command
        ```
        $ kubectl apply -f elasticsearch/statefulset.yaml
        statefulset.apps/elasticsearch-cluster created
        ```
    - 
    - We define a StatefulSet called `es-cluster` in the `kube-logging` namespace We then associate it with previously created `elasticsearch` Service using  `serviceName` field.
    - This ensures that each Pod in the StatefulSet will be accessible using the following DNS addresses: `es-cluster-[0,1,2].elasticsearch.kube-logging.svc.cluster.local`, where `[0,1,2]` corresponds to the Pod's assigned integer ordinal.
    - We Specify 3 `replicas` and set the `matchLabels` selector to `app: elasticsearch`, which we then mirror in the `.spec.template.metadata` section.
    - We name the containers `elasticsearch` and choose the `docker.elastic.co/elasticsearch/elasticsearch:7.2.0` Docker image.
    - We then use `resources` field to specify that the container nees at leas 0.1 vCPU guranteed to it. and can burst up to 1 vCPU.
    - We then open and name ports `9200` and `9300` for REST and inter-node communication, respectively. We specify a `volumeMount` called `data` that will mount the PersistentVolume names `data` to the container at the path `/usr/share/elasticsearch/data`.
    - Finally we set some environment variables in the container
      - `cluster.name`: The ElasticSearch cluster's name, which in this guide is `k8s-logs`.
      - `node.name` : The node's name, ehich set to the `.metadata.name` field using `valueFrom`. This will resolve to `es-cluster-[0,1,2]`, depending on the node's assigned ordinal.
      - `discovery.seed_hosts` : This field sets a list of master-eligible nodes in the cluster that will seed the node discovery process. In this guide, we use headless service to configure. our Pods have domains of the form `es-cluster-[0,1,2].elasticsearch.kube-logging.svc.cluster.local`, so we set this environment variable accordingly. Using local namespace K8S DNS resolution, we csn shorten this to `es-cluster-[0,1,2].elasticsearch`
      - `cluster.initial_master_nodes` : This field also specifies a list of master-eligible nodes that will participate in the master election process. Note that for this field you should identify nodes by their `node.name` and not their hostnames.
      - `ES_JAVA_OPTS` : Here we set this to `-Xms512m -Xms512m` which tells the JVM ti use a minimum and maximum heapsize of 512 MB. You should tune these parameters depending on your cluster's resource availability and needs.
    - We define several init containers that run before the main elasticsearch app container. These init containers each run to completion in the order they are defined.
    - The first, named fix-permissions, runs a `chown` command to change the owner and group of the ElasticSearch data direcotory to `1000:1000`, the Elasticsearch user's UID. By defult kubernetes mounts the data direcotry as root, which renders it inaccessible to ElasticSearch.
    - The next init Container to run is `increates-fd-limit`, which runs the `ulimit` command to increase the maximun number of open file descriptors.
    - Next, we define the StatefulSet `VolumeClaimTemplates`. Kubernetes will use this to create PersistentVolumes for the Pods. In the block above, we name it `data`. We gave it the same `app: elasticsearch` label as our StatefulSet.
    - we then specify its access mode as `ReadWriteOnce`, which means that it can only be mounted as read-write by a single node.
    - Finally we specify that we'd like each PeristentVilume to be 3GiB size. You should adjust this value depending on your production needs.
  
## Creating Kibana Deployment and Service
- To Launch Kibana on Kubernetes, we'll create a Service called `kibana`, and a Deployment consisting of one Pod replica. You can scale the number of replicas based on production needs, and optionally specify a `LoadBalencer` type for the Service to load balence the requests across the Deployment Pods.
  
  ### Kibana Service
  - We can create the Kibana Service using the below configuration
      ![alt kibana service](images/kibana-service.png)
  - To create the kibana Deployment, we can use the below configuration
    ![alt kibana deployment](images/kibana-deployment.png)
  - In the above spec we have defined a Service called `kibana` in the `kube-logging` namespace, and give it the `app: kibana` label
  - We have specified that it should be accessible on port `5601` and use the `app: kibana` label to seect the Service's target Pods.
  - In the `Deployment` spec, we have define a Deployment called `kibana` and specified one Pod replica.
  - we use the `docker.elasticsearch.co/kibana/kibana:7.2.0` image.
  - We specify that we'd like at the very least 01.vCPU guranteed to the Pod, bursting up to a limit of 1 vCPU. 
  - We use the `ELASTICSEARCH_URL` environment variable to set the endpoint and port for the Elasticsearch cluster. Using kubernetes DNS, this endpoint corresponds to its Service name `elasticsearch`. 
  - We can rollout the creation of kibana Service and Deployment using the below commands
    ```
    $ kubectl apply -f kibana/service.yaml

    $ kubectl apply -f kibana/deployment.yaml
    ```
  - To check the rollout status of kibana deployment, we can use the below command
    ```
    $ kubectl rollout status deployment/kibana -n kube-logging

    ```
  - To view the running pods in our kubernetes cluster we can run the below command
    ```
    $ kubectl get pods -n kube-logging
    ```
  - To access the kibana service in our local machine, we can forward the localport to kibana service port using the below command  
  
    ```
    $ kubectl port-forward <kibana-pod-name> 5601:5601 -n kube-logging

    ```
  - Now we can access the kibana dashboard in our local browser using the url endpoint `http://localhost:5601`

## Creating the Fluentd DaemonSet
- Here, we will setup Fluentd as a DaemonSet, which is kubernetes workload type that runs a copy f given Pod on each Node in kubernetes cluster. Using this DaemonSet controller we'll rollout a fluentd logging agent Pod on every node in our cluster.
- In Kubernetes, containerized applications that llog to `stdout` and `stderr` have their log streams captured and redirected to JSON files on the nodes. The Fluentd Pod will tail these log files, filter log events, transform the log data, and ship it off to the ElasticSearch logging backend.
- In addition to container logs, the Fluentd agent will tail Kubernetes system component logs like kubelet, kube-proxy, and Doker logs.
  ### Fluentd ServiceAccount
  - Here, we create a ServiceAccount called `fluentd` that the Fluentd Pods will use to access the Kubernetes API. We will create it in the `kube-logging` namespace and once again give it the label `app: fluentd`.
    
    ![alt fluentd service account](images/fluentd-service-account.png)
  ### Fluentd ClusterRole
  - Here we define a ClusterRole called `fluentd` to which we grant the get, list, and watch permissions on the `pods` and `namespaces` objects.
  - ClusterRoles allow you to grant access to cluster-scoped kubernetes resources like Nodes.
    
    ![alt fluentd cluster role](images/fluentd-cluster-role.png)

  ### Fluentd ClusterRoleBinding
  - Here, we define a `ClusterRoleBinding` called `fluentd` which binds the fluentd ClusterRole to the fluentd ServiceAccount. This grants the `fluentd` ServiceAccount the permissions listed in the `fluentd` ClusterRole.
    
    ![alt fluentd cluster role binding](images/fluentd-cluster-role-binding.png)

  ### Fluentd DaemonSet
  - Here, we define a DaemonSet called `fluentd` in the `kube-logging` namespace and give it the `app: fluentd` label.
  - We match the `app: fluentd` label defined in `.metadata.labels` and then assign the DaemonSet the `fluentd` ServiceAccount. We also select the `app: fluentd` as the Pods managed by this DaemonSet.
  - Next, we define a `NoSchedule` toleration to match the equivalent taint on Kubernetes master nodes. This will ensure that the DaemonSet also get rolled out to the Kubernetes masters.
  - If you dont want to run a Fluentd Pod on your master nodes, remove this toleration.
  - Next, we begin defining the Pod container, which we call `fluentd`
  - Next, we configure Fluentd using some environment variables
    - `FLUENT_ELASTICSEARCH_HOST`: We set this to the ElasticSearch headless Service address defined earlier: `elasticsearch.kube-logging.svc.cluster.local`. This will resolve to a list of IP addresses for the 3 ElasticSearch Pods.
    - `FLUENTD_ELASTICSEARCH_PORT`: We set this to the ElasticSearch port we configured earlier, `9200`.
    - `FLUENTD_ELASTICSEARCH_SCHEME`: We set this to `http`.
    - `FLUENTD_SYSTEMD_CONF`: We set this to disable to suppress output related to `systemd` not being set up in the container.
    - Here, we specify 512MiB memory limit on thr FluentD Pod, and gurantee it 0.1 vCPU and 200MiB of memory. 
    - Next, we mount the `/var/log` and `/var/lib/docker/containers` host paths into container using volume mounts. These volumes are defined at the end of the block under `volumes`
    - The final parameter we define in the block is `terminationGracePeriodSeconds`, which gives Fluentd 30 seconds to shutdown gracefully upon receiving a `SIGTERM` signal. After 30 seconds, the containers are sent a `SIGKILL` signal.
    
        ![alt fluentd daemonset](images/fluentd-daemonset.png)