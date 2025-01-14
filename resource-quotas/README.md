# Resource Quotas
- When several users or teams share a cluster with a fixed number of nodes, there is a concern that one team could use more than its fair share of resources.
- Resource Quotas are a tool for administrators to address this concern.
- A resource quota defined by a `ResourceQuota` object, *provides constraints that limit aggregate resource consumption per namespace*. *It can limit the quantity of objects that can be created in a namespace by type, as well as the total amount of compute resources that may be consumed by resources in that namespace*
- Resource Quotas work like this
  - Different teams work in different namespaces. This can be enforced with RBAC.
  - The administrator creates one ResourceQuota for each namespace.
  - Users create resources in the namespace, and the quota system tracks usage to ensure it does not exceed hard resource limits defined in a ResourceQuota.
  - If creating or updating resources violates a quota constraint, the request will fail with HTTP status code `403 FORBIDDEN` with a message explaining the constraint that would have been violated.
    - If quota is enabled in a namespace for compute resources like `cpu` and `memory`, users must specify requests or limits for those values; otherwise, the quota system may reject pod creation.
- The name of a ResourceQuota object must be a valid DNS subdomain name.
- Examples of policies that could be created using namespaces and quotas are:
  - In a cluster with a capacity of 32 GiB RAM, and 16 cores, let team A use 20 GiB and 10 cores, let B use 10GiB and 4 cores, and hold 2GiB and 2 cores in reserve for future allocation.
  - Limit the `testing` namespace to using 1 core and 1GiB RAM. Let the `production` namespace use any amount.
  - In the case where the total capacity of the cluster is less than the sum of the quotas of the namespaces, there may be contention for resources. This is handled on a first-com-first-served basis.
  - Neither contention nor changes to quota will affect already created resources.

## Enabling Resource Quota
- ResourceQuota support is enabled by default for many Kubernetes distributions. It is enabled when the API server `` `--enable-admission-plugins=` has `ResourceQuota` as one of its arguments.
- A resource quota is enforced in a particular namespace when there is a ResourceQuota in that namespace.

## Compute Resource Quota
- You can limit the total sum of compute resources that can be requested in a given namespace.
- The following resource types supports

| Resource Name      | Description                                                                                                              |
|--------------------|--------------------------------------------------------------------------------------------------------------------------|
| `limits.cpu`       | Across all pods in a non-terminal state, the sum of CPU limits cannot exceed this value.                                 |
| `limits.memory`    | Across all pods in a non-terminal state, the sum of memory limits cannot exceed this value.                              |
| `requests.cpu`     | Across all pods in a non-terminal state, the sum of CPU requests cannot exceed this value.                               |
| `requests.memory`  | Across all pods in a non-terminal state, the sum of memory requests can not exceed this value.                           |
| `hugepages-<size>` | Across all pods in a non-terminal state, the number of huge page requests of the specified size cannot exceed this value |
| `cpu`              | Same as *requests.cpu*                                                                                                   |
| `memory`           | Same as requests.memory                                                                                                  |

### Resource Quota for Extended Resources
- In addition to the resources mentioned above, in release 1.10, quota support for extended resources is added.
- As overcommit is not allowed for extended resources, it make no sense to specify both `requests` and `limits` for the same extended resource in a quota. So fat extended resources, only quota items with prefix `requests.` are allowed.
- Take the GPU resource as example, If the resource name is nvidia.com/gpu, and you want to limit the total number of GPUs requested in a namespace to 4, you can define a quota as follows
  - `requests.nvidia.com/gpu: 4`

## Storage Resource Quota
- You can limit the total sum of storage resources that can be requested in a given namespace.
- In addition, you can limit consumption of storage resources based on associated storage-class.

| Resource Name                                                             | Description                                                                                                                     |
|---------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| `requests.storage`                                                        | Across all persistent volume claims, the sum of storage requests cannot exceed this value                                       |
| `persistentVolumeClaims`                                                  | The total number of PersistentVolumeClaims that can exist in the namespace                                                      |
| `<storage-class-name>.storageclass.storage.k8s.io/requests.storage`       | Across all persistent volume claims associated with <storage-class-name>, the sum of storage requests cannot exceed this value. |
| `<storage-class-name>.storageclass.storage.k8s.io/persistentvolumeclaims` | Across all persistent volume claims associated with the <storage-class-name>, the total number of persistent volume claims that can exist in the namespace. |

- For example, if you want to quota storage with `gold` StorageClass separate from `bronze` StorageClass, you can define a quota as follows:
  - `gold.storageclass.storage.k8s.io/requests.storage: 500GiB`
  - `bronze.storageclass.storage.k8s.io/requests.storage: 100Gi`
- In release 1.8, quota support for local ephemeral storage is added as an alpha feature:

| ResourceName                 | Description                                                                                             |
|------------------------------|---------------------------------------------------------------------------------------------------------|
| `requests.ephemeral-storage` | Across all pods in the namespace, the sum of local ephemeral storage requests cannot exceed this value. |
| `limits.ephemeral-storage`   | Across all pods in the namespace, the sum of local ephemeral storage limits cannot exceed this value    |
| `ephemeral-storage`           | Same as **requests.ephemeral-storage**                                                                  |

## Object Count Quota
- You can set quota for the total number of one particular resource *kind* in kubernetes API, using the following syntax:
  - `count:/<resource>.<group>`  for resources from non-core groups
  - `count/<resource>` for resources from the core group.
- Here is an example set of resource users may want to put under object count quota.
  - `count/persistentvolumeclaims`
  - `count/services`
  - `count/secrets`
  - `count/configmaps`
  - `count/replicationcontrollers`
  - `count/deployment.apps`
  - `count/statefulsets.apps`
  - `count/jobs.batch`
  - `count/cronjobs.batch`
- If you define a quota this way, it apply to Kubernetes APIs that are part of the API server, and to any custom resources backed by a CustomResourceDefinition.
- If you use API aggregation to add additional, custom APIs that are not defined as CustomResourceDefinitions, the core Kubernetes control plane does not enforce quota for the aggregated API.
- The extension API server is expected to provide quota enforcement if that's appropriate for the custom API. for example, to create a quota on a `widgets` custom resource in `example.com` API group, use `count/widgets.example.com`.

## Quota Scopes
- Each quota can have an associated set of `scopes`. A quota will only measure usage for a resource if it matches the intersection of enumerated scopes.
- When a scope is added to the quota, it limits the number of resources it supports to those that pertain to the scope.
- Resources specified on the quota outside the allowed set results in a validation error.

| Scope                      | Description                                                   |
|----------------------------|---------------------------------------------------------------|
| `Terminating`              | Match pods where `.spec.activateDeadlineSeconds >= 0`         | 
| `NotTerminating`           | Match pods where `.spec.activateDeadlineSeconds` is nil       |
| `BestEffort`               | Match pods that have best effort quality of service           |
| `NotBestEffort`            | Match pods that do not have best effort quality of service    |
| `PriorityClass`            | Match pods that references the specified priority class       |
| `CrossNamespacePodAffinity` | Match pods that have cross-namespace pod (anti)affinity terms |

- The `BestEffort` scope restricts a quota to tracking the following resources:
  - `pods`
- The `Terminating`, `NotTerminating`, `NotBestEffort` and `PriorityClass` scopes restrict a quota to tracking the following resources
  - `pods`
  - `cpu`
  - `memory`
  - `requests.cpu`
  - `requests.memory`
  - `limits.cpu`
  - `limits.memory`
- Note that you cannot specify both the `Terminating` and the `NotTerminating` scopes in the same quota, and you cannot specify both the `BestEffort` and `NotBestEffort` scopes in the same quota either.
- The `scopeSelector` supports the folllowing values in the `operator` filed:
    - `In`
    - `NotIn`
    - `Exists`
    - `DoesNotExists`
- When using one of the following values as the `scopeName` when defining the `scopeSelector`, the `operator` must be `Exists`.
  - `Terminating`
  - `NotTerminating`
  - `BestEffort`
  - `NotBestEffort`
- If the `operator` is `In` or `NotIn`, the `values` field must have at least one value. If the `operator` is `Exists` or `DoesNotExist`, the `values` field must not be specified.

## Resource Quota per PriorityClass
- Pods can be created at a specific priority. You can control a pod's consumption of system resources based on a pod's priority, by using the `scopeSelector` field in the quota spec.
- A quota is matched ans consumes only if `scopeSelector` in the quota spec selects the pod.
- When quota is scoped for priority class using `scopeSelector` filed, quota object is restricted to track only the following resources.
  - `pods`
  - `cpu`
  - `memory`
  - `ephemeral-storage`
  - `limits.cpu`
  - `limits.memory`
  - `limits.ephemeral-storage`
  - `requests.cpu`
  - `requests.memory`
  - `requests.ephemeral-storage`
- This example creates a quota object and matches it with pods at specific priorities. The following works as follows
  - Pods in the cluster have one of the three priority classes, "low", "medium" and "high".
  - One quota object is created for each priority.
  
  ```
  apiVersion: v1
  kind: List
  items:
    - apiVersion: v1
      kind: ResourceQuota
      metadata:
        name: pods-high
      spec:
        hard:
          cpu: "1000"
          memory: 200Gi
          pods: "10"
        scopeSelector:
          matchExpressions:
            - operator: In
              scopeName: PriorityClass
              values:
                - high
  
    - apiVersion: v1
      kind: ResourceQuota
      metadata:
        name: pods-medium
      spec:
        hard:
          cpu: "10"
          memory: 20Gi
          pods: "10"
        scopeSelector:
          - matchExpressions:
              - operator: In
                scopeName: PriorityClass
                values:
                  - medium
    - apiVersion: v1
      kind: ResourceQuota
      metadata:
        name: pods-low
      spec:
        hard:
          cpu: "5"
          memory: 10Gi
          pods: "10"
        scopeSelector:
          - matchExpressions:
              - operator: In
                scopeName: PriorityClass
                values:
                  - low
            
  ```
  
## Requests compared to Limits
- When allocating compute resources, each container may specify a request and a limit value for either CPU or memory. The quota can be configured to quota either value.
- If the quota has a value specified for `requests.cpu` or `requests.memory`, then it required that every incoming container makes an explicit request for those resources.
- If the quota has a value specified for `limits.cpu` or `limits.memory`, then it requires that every incoming container specifies an explicit limit for those resources.

## Viewing and Setting Quotas
- kubectl supports creating, updating, and viewing quotas:
- To create a namespace, we can use the below command
  - `kubectl create namespace myspace`
- The following configuration describes the ResourceQuota object for compute resources and kubernetes object counts.
  
  ```
  apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: compute-resource-quota
    namespace: myspace
  spec:
    hard:
      requests.cpu : "1"
      requests.memory: 1Gi
      limits.cpu: "2"
      limits.memory: 2Gi
      requests.nvidia.com/gpu: "4"
  ```
  
  ```
  apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: resource-quota-count-objects
    namespace: myspace
  spec:
    hard:
      pods: "10"
      deployments.apps: "2"
      configmaps: "10"
      secrets: "10"
      services: "5"
      services.loadbalancers: "2"
      replicationcontrollers: "2"
  ```
- TO create the above resource quota in kubernetes cluster, we can use the below commands
  - `kubectl create -f compute-resources.yaml`
  - `kubectl create -f object-counts.yaml`
- To view and describe the resource quotas we can use the below commands
```
$ kubectl get quota --namespace myspace --output yaml
apiVersion: v1
items:
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    creationTimestamp: "2025-01-14T09:45:15Z"
    name: resource-quota-count-objects
    namespace: myspace
    resourceVersion: "733491"
    uid: 151c9155-083b-4b74-8c9b-1011dde400e2
  spec:
    hard:
      configmaps: "10"
      count/deployments.apps: "2"
      pods: "10"
      replicationcontrollers: "2"
      secrets: "10"
      services: "5"
      services.loadbalancers: "2"
  status:
    hard:
      configmaps: "10"
      count/deployments.apps: "2"
      pods: "10"
      replicationcontrollers: "2"
      secrets: "10"
      services: "5"
      services.loadbalancers: "2"
    used:
      configmaps: "1"
      count/deployments.apps: "1"
      pods: "2"
      replicationcontrollers: "0"
      secrets: "0"
      services: "0"
      services.loadbalancers: "0"
kind: List
metadata:
  resourceVersion: ""
```

```
$ kubectl describe quota -n myspace
Name:                   resource-quota-count-objects
Namespace:              myspace
Resource                Used  Hard
--------                ----  ----
configmaps              1     10
count/deployments.apps  1     2
pods                    2     10
replicationcontrollers  0     2
secrets                 0     10
services                0     5
services.loadbalancers  0     2

$ kubectl describe quota compute-resource-quota -n myspace
Name:                    compute-resource-quota
Namespace:               myspace
Resource                 Used  Hard
--------                 ----  ----
limits.cpu               0     2
limits.memory            0     2Gi
requests.cpu             0     1
requests.memory          0     1Gi
requests.nvidia.com/gpu  0     4

```
- We can also create a ResourceQuota directly using the kubectl utility
  - `kubectl create quota test-quota --hard=count/deployments.apps=2,count/replicasets.apps=4,count/pods=10,count/secrets=4 --namespace myspace`


## Quota and Cluster Capacity
- ResourceQuotas are independent of the cluster capacity. They are expresses in absolute units.
- So, if you add nodes to your cluster, this does not automatically give each namespace the ability to consume more resources.
- Sometimes more complex policies may be desired, such as:
  - Proportionally divide total cluster resources among several teams.
  - Allow each tenet to grow resource usage as needed, but have a generous limit to prevent accidental resource exhaustion.
  - Detect demand from one namespace, add nodes, and increase quota.
- Such policies could be implemented using ResourceQuotas as building blocks, by writing a custom controller that watches the quota usage and adjusts the quota hard limits of each namespace according to other signals.
- Note that resource quota divides up aggregate cluster resources, but it creates no restrictions around nodes: pods from several namespaces may run on the same node.