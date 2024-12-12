# DaemonSet
- A DaemonSet defines pods that provides node-local facilities. These might be fundamental for the operation of your cluster, such as networking helper tool or be part of an add-on.
- <u>A DaemonSet ensures that all (or some) Nodes run a copy of a Pod. As nodes are added to the cluster, Pods are added to them. As nodes are removed from the cluster, the pods are garbage collected. Deleting a DaemonSet will clean upthe Pods it created.</u>
- Some typical uses of DaemonSets are
  - running a cluster storage daemon on every node
  - running a logs collection daemon on every node
  - running a node monitoring daemon on every node
- Writing a DaemonSet Spec:

    ```
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: fluentd-elasticcsearch
      namespace: kube-system
      labels:
        k8s-app: fluentd-logging
    spec:
      selector:
        matchLabels:
          name: fluentd-elasticsearch

      template:
        metadata:
          labels:
            name: fluentd-elasticsearch
        spec:
          # these toleratios are to have the daemonset runnable on controlplane nodes
          # remove them if your control plane nodes should not run pods
          tolerations:
            - key: node-role.kubernetes.io/control-plane
              operator: Exists
              effect: NoSchedule
            - key: node-role.kubernetes.io/master
              operator: Exists
              effect: NoSchedule
              
          containers:
            - name: fluentd-elasticsearch
              image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
              resources:
                limits:
                  memory: 300Mi
                requests:
                  cpu: 100m
                  memrory: 200Mi
              volumeMounts:
                - name: varlog
                  mountPath: /var/log
          # it may be desirable to set a high priority class to the Daemon POd to ensure that it preempts running pods
          # priorityClassName: important
          terminationGracePeriodSeconds: 30
          volumes:
            - name: varlog
              hostPath:
                path: /var/log
                
    ```
## How Daemon Pods are Scheduled
- A DaemonSet can be used to ensure that al eligible nodes run a copy of a Pod. <u>The DaemonController creates a Pod for each eligible node and adds the `spec.affinity.nodeAffinity` filed of the pod to match the target host.</u>
- <u>After the Pod is created the default schedular typically takes over and then binds the Pod to the target host by setting the .spec.noadeName filed.</u> If the new Pod cannot fit on the node, the default schedular may preempt (evict) some of the existing Pods based on the priority of the new Pod.
- The user can specify a different schedular for the Pods of the DaemonSet, by setting the `.spec.template.spec.schedularName` filed of the DaemonSet.
- The original node affinity specified at the `.spec.template.spec.affinity.nodeAffinity` filed is taken into consideration by the DaemonSet Controller when evaluating the eligible nodes, but is replaced on the created Pod with the node affinity that macthes the name of the eligibe node.

## Taints and Tolerations
- The DaemonSet Controller automatically adds a set of tolerations to DaemonSet Pods.

  | Toleration Key | Effect | Details |
  | -------------- | ------ | ------- |
  | node.kubernestes.io/not-ready | NoExecute | DaemonSet Pods can be scheduled onto nodes that are not healthy or ready to accept Pods. Any DaemonSet Pods running on such nodes will not be evicted. |
  | node.kubernetes.io/unreachable | NoExecute | DaemonSet Pods can be scheduled onto noades that are unreachable from the node controller. Any DaemonSet Pods running on such nodes will be evicted |
  | node.kubernetes.io/disk-pressure | NoSchedule | DaemonSet Pods can be scheduled onto nodes with disk pressure issues. |
  | node.kubernetes.io/memory-pressure | NoSchedule | DaemonSet Pods can be scheduled onto nodes with memory pressure issues. |
  | node.kubernetes.io/pid-pressure | NoSchedule | DaemonSet Pods can be scheduled onto nodes with prrocess pressure issues. |
  | node.kubernetes.io/unschedulable | NoSchedule | DaemonSet Pods can be scheduled onto nodes that are unschedulable | 
  | node.kubernetes.io/network-unavailable | NoSchedule | **Only added for DaemonSet Pods that request host networking**, i.e, Pods thaving `spec.hostNetwork: true`. Such DaemonSet Pods can be scheduled onto nodes with unavailable network. |

- You can add your own tolerations to the Pods of a DaemonSet as well, by defining these in the Pod template of the DaemonSet
- The DaemonSet Controller sets the node.kubernetes.io/unschedulable:NoSchedule toleration automatically, kubernetes can run DaemonSet Pods on the nodes that are marked as *unschedulable*

## Communicating with Daemon Pods
Some possible patterns for communicating with Pods in a DaemonSet are:
- **Push** : Pods in the DaemonSet are configured to send updates to another service, such as a stats database. They do not have clients.
- **NodeIp** and **Known Port** : Pods in the DaemonSet can use a `hostPort`, so that the pods are reachable via the node IPs. Clients know that the list of node IPs somehow, and know the port by convention.
- **DNS** : Creates a *headless service* with the same pod selector, and then discover DaemonSets using the `endpoints` resource or retrieve multiple A records from DNS.
- **Service** : Create a service with the same Pod selector, and use the service to reach a daemon on a random node. (No way to reach specific node)

## Updating a DaemonSet
- If node labels are changed, the DaemonSet will promptly add Pods to newly matching nodes and delete Pods from newly not-matching nodes.
- You can modify the Pods that a DaemonSet creates. However, Pods do not allow all fields to be updated. Also, <u>the DaemonSet Controller will use the original template the next time a node (even with the same name) is created.</u>
- <u>You can delete a DaemonSet. If you specify --cascade=orphan with kubectl, then the Pods will be left on the nodes</u>. If you subsequently create a new DaemonSet with the same selector, the new DaemonSet adopts to existing Pods. If any Pods need replacing the DaemonSet replaces them according to its `updateStrategy`.

# Performing Rolling Update on a DaemonSet
DaemonSet has two update strategy types:
- `OnDelete` : with `OnDelete` update strategy, after you update a DaemonSet template, new DaemonSet pods will only be created when you manually delete old DaemonSet Pods. This is the same behaviour of DaemonSet in kubernetes version 1.5 or before.
- `RollingUpdate` : This is the default update strategy. with `RollingUpdate` update strategy, after you update a DaemonSet template, old DaemonSet Pods will be killed, and new DaemonSet Pods will be created automatically, in a controlled fashion. At most one pods of the DaemonSet will be running on each node during the whole update process.

## Performing a Rolling Update
- TO enable the rolling update feature of a DaemonSet, you must set its `.spec.updateStrategy.type` to `RollingUpdate`.
- You may want to set `.spec.updateStrategy.rollingUpdate.maxUnavailable` (default to 1), `.spec.minReadySeconds` (default to 0) and `.spec.updateStrategy.rollingUpdate.maxSurge` (default to 0) as well
- To deploy a DaemonSet we can use `kubectl apply -f path/to/manifest.yaml`
  
    ```
    $ kubectl apply -f daemon-set/daemon-set.yaml 
      daemonset.apps/fluentd-elasticsearch created
      daemonset.apps/fluentd-elasticsearch configured
    ```
- To view the pods created by DaemonSet
    
    ```
    $ kubectl get pods -n kube-system

    NAME                               READY   STATUS              RESTARTS       AGE
    coredns-7db6d8ff4d-n87ql           1/1     Running             27 (23m ago)   208d
    etcd-minikube                      1/1     Running             27 (23m ago)   208d
    fluentd-elasticsearch-mgxhk        0/1     ContainerCreating   0              5s
    kube-apiserver-minikube            1/1     Running             27 (23m ago)   208d
    kube-controller-manager-minikube   1/1     Running             27 (23m ago)   208d
    kube-proxy-stbb8                   1/1     Running             27 (23m ago)   208d
    kube-scheduler-minikube            1/1     Running             27 (23m ago)   208d
    metrics-server-c59844bb4-n9msf     1/1     Running             12 (23m ago)   22d
    storage-provisioner                1/1     Running             61 (22m ago)   208d
    ```
- To update the DaemonSet we can change the configuration in `spec.template` and use the below command to apply new revision
    
    ```
    $ kubectl apply -f daemon-set/daemon-set.yaml 
      daemonset.apps/fluentd-elasticsearch configured
    ```
- To view the rollout status of current changes to the workload, we can use the below command
    ```
    $ kubectl rollout status ds/fluentd-elasticsearch -n kube-system
      daemon set "fluentd-elasticsearch" successfully rolled out
    ```
- To view all the revisions of updates on the DaemonSet, we can use the below command
    
    ```
    $ kubectl rollout history ds/fluentd-elasticsearch -n kube-system
      daemonset.apps/fluentd-elasticsearch 
      REVISION  CHANGE-CAUSE
      1         <none>
      2         <none>
    ```
- To view the rollout history of a particular revision of DaemonSet, we can use the below command
  
  ```
  $ kubectl rollout history ds fluentd-elasticsearch --revision=1 -n kube-system
    daemonset.apps/fluentd-elasticsearch with revision #1
    Pod Template:
      Labels:	name=fluentd-elasticsearch
      Containers:
      fluentd-elasticsearch:
        Image:	quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        Port:	<none>
        Host Port:	<none>
        Limits:
          memory:	300Mi
        Requests:
          cpu:	100m
          memory:	200Mi
        Environment:	<none>
        Mounts:
          /var/lib/docker/containers from varlibdockercontainers (ro)
          /var/log from varlog (rw)
      Volumes:
      varlog:
        Type:	HostPath (bare host directory volume)
        Path:	/var/log
        HostPathType:	
      varlibdockercontainers:
        Type:	HostPath (bare host directory volume)
        Path:	/var/lib/docker/containers
        HostPathType:	
      Node-Selectors:	<none>
      Tolerations:	node-role.kubernetes.io/control-plane:NoSchedule op=Exists
      node-role.kubernetes.io/master:NoSchedule op=Exists
  ```
- To rollback the DaemonSet to a particular revision, we can use the below command
  
  ```
  $ kubectl rollout undo ds fluentd-elasticsearch --to-revision=1 -n kube-system
   daemonset.apps/fluentd-elasticsearch rolled back
  ```