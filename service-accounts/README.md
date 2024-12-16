# Service Accounts
- A Service account is a type of non-human account that, in kubernetes, provides a distinct identity in a kubernetes cluster.
- Application Pods, system components, and entities inside and outside the cluster can use a specific ServiceAccount's credenticals to identify as that ServiceAccount.
- This identity is useful in various situations, including authenticating to the API server or implementing identity-based security policies.
- Service Accounts exists as ServiceAccount Objects in the API Server. Service accounts have the following properties
  - `Namespaced`: Each service account is bound to a kubernetes namespace. Every namespace gets a default ServiceAccount upon creation.
  - `Lightweight`: Servie accounts exists in the cluster and are defined in the Kubernetes API. You can quicky create service accounts to enable specific tasks.
  - `Portable`: A configuration bundle for a complex containerized workload might include service account definitions for the system's components. The lightweight nature of service accounts and the namespaced identities make the configurations portable.
- Service Accounts are different from user accounts, which are authenticated human users in the cluster.
- By default, user accounts don't exists in the kubernetes API server; instead, the API server treats user identities as opaque data. You can authenticate as a user account using multiple methods. Some kubernetes distributions might add custom extension API's to represent usr accounts in the API server.

    | Description | ServiceAccount | User or Group |
    | ------------| ----------------| -------------------- |
    | Location | Kubernetes API (ServiceAccount object) | External |
    | Access Control | Kubernetes RBAC or other authorization mechanisms | Kubernetes RBAC ot other identify and access management mechanisms |
    | Intended Use | Workloads, automation | People | 

## Default Service Accounts
- When you create a cluster, Kubernetes automatically creates a ServiceAccount object named `default` for every namespace in your cluster.
- The `default` service accounts in each namespace get no permissions by default otherthan the defult API discovery permissions that kubernetes grants to all authenticated principals if role-based access control (RBAC) is enabled.
- If you delete default servcie account in a namespace, the control plane replaces it with a new one.
- If you deploy a Pod in a namespace, and you don't manually assign a ServiceAccount to the Pod, kubernetes assigns the `deafult` ServiceAccount for that namespace to the Pod.

## Use cases for Kubernetes ServiceAccounts
- As a general guideline, you can use service accounts to provide identities in the dollowing scenarios:
  - Your Pod needs to communicate with the Kubernetes API server, for example in situations such as the following:
    - Providing read-only access to sensitive information stored in Secrets.
    - Granting cross-namespace access, such as allowing a Pod in namespace `example` to read, list and watch for Lease objects in the `kube-node-lease` namespace.
  - Your Pods need to communicate with an external service. For example, a workload Pod requires an identity for a commercially available cloud API, and the commercial provider allows configuring a suitable trust relationship.
  - Authenticating to a private image registry using an imagePullSecret.
  - An External service needs to cmmunicate with the Kubernetes API server. For example, authenticating to the cluster as part of a CI/CD pipeline.
  - You use third-party security software in your cluster that relies on the ServiceAccount identity of different Pods to group those Pods into different contexts.
  
## How to use Kubernetes Service Accounts
- To use a Kubernetes service account, you do the following
  - Create a ServiceAccount object using a Kubernete client like `kubectl` or manifest that defines the object.
  - Grant permissions to the ServiceAcount object using an authorization mechanism such as RBAC.
  - Assign the ServiceAccount object to Pods during Pod creation.
    ### Grant permissions to a ServiceAccount
    - You can use the built-in Kubernetes role-based access control (RBAC) mechanism to grant the minimum permissions required by each service account.
    - You create a role, which grants access, and then *bind* the role to your ServiceAccount.
    - RBAC lets you define a minimum set of permissions so that the service account permissions follow the principle of least privilege.

    ### Cross-Namespace access using a ServiceAccount
    - You can use RBAC to allow service accounts in one namespace to perform actions on resources in a different namespace in the cluster.
    - For example, consider a scenario where you have a service account and a Pod in the `dev` namespace and you want your Pod to see Jobs running in the `maintenance` namespace.
    - You should create a Role object that grants permissions to the list job objects.
    - Then, you'd create a RoleBinding object in the `maintanence` namespace to bind the Role to the ServiceAccount object.
    - Now, Pods in the `dev` namespace can list objects in the `maintenance` namespace using the service account.
    
    ### Assign a ServiceAccount to a Pod
    - To assign a ServiceAccount to a Pod, you set the `spec.serviceAccountName` filed in the Pod specification. Kubernetes then automatically provides the credentials for that ServiceAccount to the Pod.
    - In v1.22 and later, Kubernetes gets a short-lived, automatically rotating token using the `TokenRequest` API and mounts the token as a projected volume.
    - By default, Kubernetes provides the Pod with the credentials for an assigned ServiceAccount, whether that is the `default` ServiceAccount or a custom ServiceAccount that you specify.
    - To prevent Kubernetes from automatically injecting credentials for a specified ServiceAccount or the `default` ServiceAccount, set the `automountServiceAccountToken` field in your Pod specification to `false`.