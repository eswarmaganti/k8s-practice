# Taints and Tolerations
- `taints` are opposite to node affinity, they allow a node t repel a set of pods.
- `tolerations` are applied to pods. They allow the schedular to schedule pods with matching taints. <u>Tolerations allow scheduling but don't gurantee scheduling: the schedulat also evaluates other parameters as part of its function.</u>
- <u>tains and tolerations work together to ensure that pods are not scheduled onto inappropriate nodes. One or more tains are applied to a node; this marks that the node should not accept any pods that do not tolerate the tains.</u>

# Taint Effects
- `NoExecute` : This affects pods that are already running on the node as follows
  - Pods that do not tolerate the taint are evicted immediately
  - Pods that tolerate the taint without specifying `tolerationSeconds` in their toleration specification remain bound forever
  - Pods that tolerate the taint with a specified `tolerationSeconds` remain bound for specified amount of time. After that time elapses, the node lifecycle controller evicts the pods from the node
- `NoSchedule` : No New pods will be scheduled on the taint node unless they have a matching toleration. Pods currently running on the node are not evicted.
- `PreferNoSchedule` : This is a soft version of NoSchedule
  - The control plane will try t avoid placing a pod that does not tolerate the taint on the node, but it id not guranteed.

# The Process

## Adding a taint
- Add a taint to the node using the below command
  
  - Syntax: `kubectl taint nodes node1 key1=value1:NoSchedule`
   
    ```
    $ kubectl taint nodes minikube tier=web:NoSchedule
    node/minikube tainted
    ```
  - This command will place a taint on the node1. The taint has key `key1` and value `value1` and taint effect `NoSchedule`. This means that no pod will be able to schedule onto this `node1` unless it has a matching toleration
## Removing a taint
- To remove a taint, we can use the below command
  - syntax: `kubectl taint nodes node1 key1=value1:NoSchedule-`
    
    ```
    $ kubectl taint nodes minikube tier=web:NoSchedule-
      node/minikube untainted
    ```
  
## Get all the taints applied
- To view all the taints applied to a nodes, we can use the below command
  ```
  $ kubectl get nodes -o json | jq '.items[].spec.taints'
  [
    {
      "effect": "NoSchedule",
      "key": "tier",
      "value": "web"
    }
  ]
  ```

# Applying the toleration
- We have to specify the tolerations in the pod spec under spec.tolerations
    ```
    tolerations:
      - key: tier
        value: web
        operator: Equal
        effect: NoSchedule
    ```
    ```
    tolerations:
        - key: tier
          operator: Exists
          effect: NoSchedule
    ```
- By default the value for the operator is `Equal`.
- If a toleration matches a taint if the keys are the same or the effects are the same, and
  - the operator is `Equal` and the value should be specified
  - the `operator` is `Exists` in this case no value should be specified
- There are two special cases
  - if the `key` is empty, then the operator must be `Exists`, which matches all the keys and values. Note that the  `Effect` still needs to be matched at the same time
  - An empty `Effect` matches all effects with key `key1`

## Pods without tolerations
- If we create a deployment in k8s cluster without any matching tolerations to the applied taint and no other nodes available, The pods will not be scheduled and stay in pending state.
  
  ```
  $ kubectl create deployment --replicas 3 --image nginx:latest nginx-deployment
  deployment.apps/nginx-deployment created
  
  $kubectl get pods
  NAME                               READY   STATUS    RESTARTS   AGE
  nginx-deployment-b554c4487-cx78p   0/1     Pending   0          5s
  nginx-deployment-b554c4487-kn7cm   0/1     Pending   0          5s
  nginx-deployment-b554c4487-w2q4p   0/1     Pending   0          5s
  ```

## Note
- By default the kubernetes schedular takes taints and tolerations into account when selecting a pod to run on a node.
- However <u>if we manually specify the `.spec.nodeName` for a pod, that action bypasses the schedular, the pod is then bound to the node that where we have assigned it, even if there is `NoSchedule` taint is applied on the selected node.</u>
- <u>If the node has a  `NoExecute` taint set, the kubelet will eject the pod from the node unless there is an appropriate toleration set. </u>