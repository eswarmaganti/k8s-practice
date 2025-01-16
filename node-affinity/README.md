# Assigning Pods to Nodes
- You can constrain a Pod so that it is restricted to tun on particular node(s), or to prefer to tun on particular nodes.
- There are several ways to do this and the recommended approaches all use label selectors to facilitate the selection.
- Often, you do not need to set any such constraints; the schedular will automatically do a reasonable placement.
- However, there are some circumstances where you may want to <u>control which node the Pods deploys to, for example, to ensure that a Pod ends up on a node with an SSD attached to it, or to co-locate Pods from two different services that communicate a lot into the same availability zone.</u>
- You can use the following methods to choose where kubernetes schedules specific Pods:
  - nodeSelector field matching against node labels
  - Affinity and anti-affinity
  - nodeName field
  - Pod topology spread constraints.

## Node Labels
- Like many other kubernetes objects, nodes have labels. You can attach labels manually. Kubernetes also populates a standard set of labels on all nodes in a cluster.
- To set a label to a node, we can use the below command
- To view all the nodes available for our cluster
  ```
  $ kubectl get nodes
  NAME       STATUS   ROLES           AGE    VERSION
  minikube   Ready    control-plane   244d   v1.30.0
  ```
- TO view the node labels applied to all the nodes in a cluster.

  ```
  $ kubectl get nodes --show-labels
  NAME       STATUS   ROLES           AGE    VERSION   LABELS
  minikube   Ready    control-plane   244d   v1.30.0   beta.kubernetes.io/arch=arm64,beta.kubernetes.io/os=linux,kubernetes.io/arch=arm64,kubernetes.io/hostname=minikube,kubernetes.io/os=linux,minikube.k8s.io/commit=248d1ec5b3f9be5569977749a725f47b018078ff,minikube.k8s.io/name=minikube,minikube.k8s.io/primary=true,minikube.k8s.io/updated_at=2024_05_17T11_21_48_0700,minikube.k8s.io/version=v1.33.1,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
  ```
- To set a new label to a node, we can use the below command
  ```
  $ kubectl label node minikube capacity=medium
  node/minikube labeled
  
  $kubectl get nodes --show-labels
  NAME       STATUS   ROLES           AGE    VERSION   LABELS
  minikube   Ready    control-plane   244d   v1.30.0   beta.kubernetes.io/arch=arm64,beta.kubernetes.io/os=linux,capacity=medium,kubernetes.io/arch=arm64,kubernetes.io/hostname=minikube,kubernetes.io/os=linux,minikube.k8s.io/commit=248d1ec5b3f9be5569977749a725f47b018078ff,minikube.k8s.io/name=minikube,minikube.k8s.io/primary=true,minikube.k8s.io/updated_at=2024_05_17T11_21_48_0700,minikube.k8s.io/version=v1.33.1,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
  ```
- To remove a label applied to a node, we can use the below command
  ```
  $kubectl label node minikube capacity-    
  node/minikube unlabeled
  ```
### Node isolation/restriction
- Adding labels to nodes allows you to target Pods for scheduling on specific nodes or groups of nodes.
- You can use this functionality to ensure that specific pods run only on nodes with certain isolation, security, or regulatory properties.
- If you use labels for node isolation, choose label keys that the kubelet cannot modify. This prevents a compromised node from setting those labels on itself so that the schedular schedules workloads onto the compromised node.
- THe `NodeRestriction` admission plugin prevents the kubelet from setting or modifying labels with a `node-restriction.kubernetes.io/` prefix.
- TO make use of that label preix for node isolation:
  - Ensure you are suing the `Node authorizer` and have enabled the `NodeRestriction` admission plugin.
  - Add labels with the `node-restriction.kubernetes.io/` prefix to your nodes, and use those labels in your node selectors. For example `example.com.node-restriction.kubernetes.io/fips=true` or `example.com.node-restriction.kubernetes.io/pci-dss=true`

## nodeSelector
- `nodeSelector` is the simplest recommended form of node selection constraint.
- You can add the `nodeSelector` field to your pod specification and specify the node labels you want the target node to have. Kubernetes only schedules the Pod onto nodes that have each of the labels you specify.
- Example Pod Specification using nodeSelector
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-node-selector
  spec:
    containers:
      - name: nginx
        image: nginx
        ports:
          - containerPort: 80
    nodeSelector:
      capacity: low
  ```
## Affinity and anti-affinity
- `nodeSelector` is the simplest way to constrain Pods to nodes with specific labels. Affinity and anti-affinity expands the types of constraint you can define. Some benefits of affinity and anti-affinity include:
  - The affinity/anti-affinity language is more expressive. `nodeSelector` only selects nodes with all the specified labels. Affinity/anti-affinity gives you more control over selection logic.
  - You can indicate that a rule is soft ot preferred, so that the scheduler still schedules the Pod even if it can't find a matching node.
  - You can constrain a pod using labels on other Pods running on the node, instead of just node labels, which allows defining rules for which Pods can be co-located on a node.
- The affinity feature consists of two types of affinity:
  - *Node affinity* functions like the `nodeSelector` field but is more expressive and allows you to specify soft rules.
  - *inter-pod affinity/anti-affinity* allows constraining pods against labels on other Pods.

### Node Affinity
- Node Affinity is a concept simillar to `nodeSelector`, allowing you yo contrain which nodes your Pod cn be scheduled on based on node labels.
- There are two types of node affinity:
  - `requiredDuringSchedulingIgnoredDuringExecution`: The schedular can't schedule the Pod unless the rule is met. This functions like `nodeSelector`, but with a more expressive syntax.
  - `preferredDuringSchedulingIgnoredDuringExecution`: The scheduler tries to find a node that meets the rule. If a matching node is not available, the scheduler still schedules the Pod.
  - In the preceding types, `IgnnoredDuringExecution` means that if the node labels change after kubernetes schedules the Pod, the Pod continues to run
- You can specify the node affinity using `.spec.affinity.nodeAffinity` field in your Pod spec.
- For example, consider the below Pod spec:
    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx-node-affinity
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: topology.kubernetes.io/zone
                    operator: In
                    values:
                      - antarctica-east1
                      - antarctica-west1
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 1
              preference:
                matchExpressions:
                  - key: another-node-label-key
                    operator: In
                    values:
                      - another-node-label-value
    
    ```
- In this example, the following rules apply:
  - The node must have a label with the key `topology.kubernetes.io/zone` and the value of that label must be either `antarctica-east1` or `antarctica-west1`.
  - The node *preferably* has a label with the key `another-node-label-key` and the value `another-node-label-value`.
- You can use the `operator` field to specify a logical operator for kubernetes to use when interpreting the rules. You can use `In`, `NotIn`, `Exists`, `DoesNotExists`, `Gt` and `Lt`.
- If you specify both `nodeSelector` and `nodeAffinity`, both must be satisfied for the POd to be scheduled onto a node.
- If you specify multiple terms in `nodeSelectorTerms` associated with `nodeAffinity` types, the Pod can be scheduled onto a node if one of the specified terms can be satisfied.
- If you specify multiple expressions in a single `matchExpressions` field associated with a terms in `nodeSelectorTerms`, then the Pod can be scheduled onto a node only if all the expressions are satisfied.

### Node Affinity Weight
- You can specify a `weight` between 1 and 100 for each instance of the `preferredDuringSchedulingIgnoredDuringExecution` affinity type.
- When the scheduler finds nodes that meet all the other scheduling requirements of the Pod, the schedular iterates through every preferred rule that the node satisfied and adds the value of the weight for that expression to a sum.
- The final sum is added to the score of other priority functions for the node. Nodes with the highest total score are prioritized when the scheduler makes a scheduling decision for the Pod.

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-node-affinity-weights
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                  - linux
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 1
          preference:
            matchExpressions:
              - key: label-1
                operator: In
                values:
                  - value-1
        - weight: 50
          preference:
            - matchExpressions:
                - key: label-2
                  operator: In
                  values:
                    - value-2
```

- If there are two possible nodes that match the `preferredDuringSchedulingIgnoredDuringExecution` rule, one with the `label-1:key-1` label and another with the `label-2:key2` label, the schedular considers the weight of each node and adds the weight to the other scores for that node, and schedules the Pod onto the node with the highest final score.

### Inter-pod affinity and anti-affinity
- Inter-pod affinity and anti-affinity allow you to constrain which nodes your pods can be scheduled on based on the labels of Pods already running on that node, instead of the node labels.
- Inter-pod affinity and anti-affinity rules take the form "this Pod should run in an X if that X is already running one or more Pods that meet rule Y", Where X is a topology domain like node, rack, cloud provider zone or region, or similar and Y is the rule Kubernetes tried to satisfy.
- You can express these rules (Y) as label selectors with an optional associated list of namespaces. Pods are namespaced objects in kubernetes, so Pod labels also implicitly have namespaces. Any label selectors for Pod labels should specify the namespaces in which kubernetes should look for those labels.
- You express the topology domain (X) using a `topologyKey`, which is the key for the node label that the system used to donate the domain.
- `Inter-pod affinity and anti-affinity required substantial amounts of processing which can slow down scheduling in large clusters significantly. We do not recommend using them in clusters larger than several hundred nodes.`
- `Pod anti-affinity requires nodes to be consistently labeled, in other words, every node in the cluster must have an appropriate label matching topologyKey, If some or all nodes are missing the specified topologyKey label, it can lead to unintended behaviour`.

### Types of inter-pod affinity and anti-affinity
- Similar to node affinity, there are two types of Pod affinity and anti-affinity as follows.
  - `requiredDuringSchedulingIgnoredDuringExecution`
  - `preferredDuringSchedulingIgnoredDuringExecution`
- For example, you could use `requiredDuringSchedulingIgnoredDuringExecution` affinity to tell the schedular to co-locate Pods of two services in the same cloud provider zone because they communicate with each other a lot. Similarly, you could use `preferredDuringSchedulingIgnoredDuringExecution` anti-affinity to spread Pods from a service across multiple cloud provider zones.
- TO use inter-pod affinity, use the `affinity.podAffinity` field in the Pod spec. For inter-pod anti-affinity, use the `affinity.podAntiAffinity` field in the Pod spec.
- Example

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-pod-affinity
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - topologyKey: topology.kubernetes.io/zone
          labelSelector:
            matchExpressions:
              - key: security
                operator: In
                values:
                  - S1
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - podAffinityTerm:
            topologyKey: topology.kubernetes.io/zone
            labelSelector:
              matchExpressions:
                - key: security
                  operator: In
                  values:
                    - S2
          weight: 100
```
- This example defines one Pod affinity rule and one Pod anti-affinity rule. The pod affinity rule uses the "hard" `requiredDuringSchedulingIgnoredDuringExecution`, while the anti-affinity rule uses the "soft" `preferredDuringSchedulingIgnoredDuringExecution`.
- THe affinity rule specifies that the schedular is allowed to place the example Pod on a node oly if that node belongs to a specific zone where the other Pods have been labeled with `security=S1`.
- The anti-affinity rule specifies that the scheduler should try to avoid scheduling the Pod on a node if that node belongs to a specific zone where other Pods have been labeled with `security=S2`.

## nodeName
- `nodeName` is more direct form if node selection than affinity or `nodeSelector`. `nodeName` is a field in the Pod spec. If the `nodeName` field is not empty, the scheduler ignores the Pod and the kubelet on the named node tries to place the Pod on that node.
- Using `nodeName` overrules using `nodeSelector` or affinity and anti-affinity rules.
- SOme of the limitations of using `nodeName` to select nodes are:
  - If the named node doesn't exists, the Pod will not run, and in some cases may be automatically deleted.
  - If the named node doesn't have the resources to accommodate the Pod, the Pod will fail and its reason will indicate why, for example, OutOfmemory or OutOfcpu.
  - Node names in cloud environments are not always predictable or stable.
  
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-with-nodename
  spec:
    containers:
      - name: nginx
        image: nginx
        ports:
          - containerPort: 80
    nodeName: kube-01
  ```