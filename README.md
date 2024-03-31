- [Kubernetes RBAC(Role-based access control)](#kubernetes-rbacrole-based-access-control)
    - [Components of RBAC:](#components-of-rbac)
      - [Users vs Service Accounts](#users-vs-service-accounts)
    - [Provide Access at Namespace Level](#provide-access-at-namespace-level)
      - [Create a Service Account](#create-a-service-account)
      - [Configure kubectl with your Service Account](#configure-kubectl-with-your-service-account)
      - [Create a Role](#create-a-role)
      - [Create a RoleBinding](#create-a-rolebinding)
      - [Verify your service account has been granted the Role’s Permissions](#verify-your-service-account-has-been-granted-the-roles-permissions)
    - [Provide access at Cluster Level](#provide-access-at-cluster-level)
      - [Create ClusterRole](#create-clusterrole)
      - [Create ClusterRoleBinding](#create-clusterrolebinding)
    - [Provide Access to KeyCloak and Ingress Namespaces](#provide-access-to-keycloak-and-ingress-namespaces)

# Kubernetes RBAC(Role-based access control)

Kubernetes RBAC is a way to regulating access to Kubernetes Resources. It enables you to Provide fine-grained control over who can access and perform actions within a Kubernetes cluster.
With RBAC, administrators can grant different levels of access to users or groups based on their roles, ensuring that only authorized entities can perform specific actions.

### Components of RBAC:

1. **Roles:** A Role always sets permissions within a particular namespace; when you create a Role, you have to specify the namespace it belongs in.
2. **RoleBindings**: A role binding grants the permissions defined in a role to a user or set of users. It holds a list of subjects (users, groups, or service accounts), and a reference to the role being granted.
3. **ClusterRoles**: ClusterRole, by contrast, is a non-namespaced resource. The resources have different names (Role and ClusterRole)
4. **ClusterRoleBindings**: you want to bind a ClusterRole to all the namespaces in your cluster, you use a ClusterRoleBinding

#### Users vs Service Accounts

1. **Users**: In Kubernetes, a User represents a human who authenticates to the cluster using an external service.
2. **Service Accounts**: Service Accounts are token values that can be used to grant access to namespaces in your cluster. They’re designed for use by applications and system components.

### Provide Access at Namespace Level

Make sure your kubernetes context set to right to access the cluster

```bash
kubectl config current-context
default
```

If the context is not set to right please update the context using

```bash
kubectl config use-context $CONTEXT_NAME
```

#### Create a Service Account

```bash
kubectl create serviceaccount demo-user
serviceaccount/demo-user created
```

Next, run the following command to create an authorization token for your Service Account:

```bash
$ TOKEN=$(kubectl create token demo-user)
```

#### Configure kubectl with your Service Account

Now add a new kubectl context that lets you authenticate as your Service Account. First, add your Service Account as a credential in your Kubeconfig file:

```bash
kubectl config set-credentials demo-user --token=$TOKEN
User "demo-user" set.
```

Next, add your new context—we’re calling it demo-user-context.

```bash
kubectl config set-context demo-user-context --cluster=default --user=demo-user
```

Use the context that is just created

```bash
kubectl config use-context demo-user-context
Switched to context "demo-user-context".
```

to validate the context

```bash
kubectl config current-context
demo-user-context
```

Try to list the Pods in the namespace:

```bash
$ kubectl get pods

Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:default:demo-user" cannot list resource "pods" in API group "" in the namespace "default"
```

This service account does not have permissions to list the pods in `default` namespace

#### Create a Role

witch back to the Kubectl context that authenticates as Admin User:

```bash
kubectl config use-context $CONTEXT_NAME
```

A Role (or ClusterRole) object is required for each of the roles you want to use with Kubernetes.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: demo-role
  namespace: default
rules:
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - get
      - list
      - create
      - update
```

Save the Role manifest to `role.yaml`, then use Kubectl to add it to your cluster:

```bash
kubectl apply -f role.yaml
role.rbac.authorization.k8s.io/demo-role created
```

#### Create a RoleBinding

The Role has been created but it’s not yet assigned to your Service Account. A RoleBinding is required to make this connection.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: demo-role-binding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: demo-role
subjects:
  - namespace: default
    kind: ServiceAccount
    name: demo-user
```

Save the RoleBinding manifest as `rolebinding.yaml`, then add it to your cluster to grant your role to your user:

```bash
$ kubectl apply -f rolebinding.yaml
rolebinding.rbac.authorization.k8s.io/demo-role-binding created
```

#### Verify your service account has been granted the Role’s Permissions

witch back to the Kubectl context that authenticates as the Service Account user:

```bash
$ kubectl config use-context demo-user-context
Switched to context "demo-user-context".
```

Verify that the get pods command now runs successfully:

```bash
kubectl get pods
No resources found in default namespace.
```

You can also try creating a Pod as your Service Account as Role have the permissions on resources types `pods`

```bash
$ kubectl run nginx --image=nginx:latest
pod/nginx created
```

The Pod is successfully created:

```bash
$ kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          15s
```

However, it’s not possible for the Service Account user to delete Pods because the Role you’ve assigned doesn’t include the required delete action verb:

```bash
$ kubectl delete pod nginx
Error from server (Forbidden): pods "nginx" is forbidden: User "system:serviceaccount:default:demo-user" cannot delete resource "pods" in API group "" in the namespace "default"
```

These Permissions at Namespace level. In this example it is at `default` namespace.

### Provide access at Cluster Level

If we want ot provide Same permissions at Cluster Level

#### Create ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: demo-role
rules:
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - get
      - list
      - create
      - update
```

Save the Role manifest to `cluster-role.yaml`, then use Kubectl to add it to your cluster:

```bash
kubectl apply -f cluster-role.yaml
```

#### Create ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: demo-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: demo-role
subjects:
  - namespace: default
    kind: ServiceAccount
    name: demo-user
```

Save the RoleBinding manifest as `clusterrolebinding.yaml`, then add it to your cluster to grant your role to your user:

```bash
$ kubectl apply -f clusterrolebinding.yaml
rolebinding.rbac.authorization.k8s.io/demo-role-binding created
```

Now you able to list pods in All Namespaces

```bash
kubectl get pods -A
```

Kubernetes provides default ClusterRoles such as `cluster-admin`, `view`, and `edit`. The `cluster-admin` role grants full access to perform any action on any resource within the cluster. Other default roles like `view` provide `read-only` access to cluster resources, and `edit` allows users to create, update, and delete resources, but not access sensitive information like secrets.

### Provide Access to KeyCloak and Ingress Namespaces

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: access-logs
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch", "logs"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["*"]
    resourceNames: [""] # Deny access to all secrets
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: keycloak-namespace-access-binding
  namespace: keycloak
subjects:
  - kind: ServiceAccount
    name: $SERVICE_ACCOUNT_NAME
    namespace: keycloak
roleRef:
  kind: ClusterRole
  name: access-logs
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ingress-namespace-access-binding
  namespace: ingress-nginx
subjects:
  - kind: ServiceAccount
    name: $SERVICE_ACCOUNT_NAME
    namespace: ingress-nginx
roleRef:
  kind: ClusterRole
  name: access-logs
  apiGroup: rbac.authorization.k8s.io
```

Save the Role manifest to `access-log.yaml`, then use Kubectl to add it to your cluster:

```bash
kubectl apply -f access-log.yaml
```
