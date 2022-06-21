# Minikube PoLP Impersonation Example

## Prerequisites

A user with a group/organization attached called `backend-developer`.
(See: [minikube-rbac-demo.md](minikube-rbac-demo.md).)

Otherwise the `group` should be updated in the `ClusterRoleBinding` called
`be-developer` in `resources/polp-rbac.yaml`.

## Applying the Manifests

```shell
kubectl apply -f ./resources/polp-rbac.yaml
```

This will create a number of resources.

- Two namespaces `be-database` and `be-application` intended for our backend-team.
- A role `be-application` that gives "all permissions" in the `be-application`
    namespace.
- A role `be-database` that gives "get/list pods" in the `be-database`
    namespace.
- A roleBinding `be-application` mapping the
    `be-application`-role to a "virtual" `be-application` user.
- A roleBinding `be-database` mapping the
    `be-database`-role to a "virtual" `be-database` user. And finally,
- a clusterRole and clusterRoleBinding.
    - The clusterRole `be-developer-impersonator` allows to impersonate the
        `be-database` and `be-application` users.
    - The clusterRoleBinding `be-developer` maps the `be-developer-impersonator` role to the
        `backend-developer` group.

## Using the specific roles

We can now do exactly as defined in the roles, and demonstrate it with impersonate:

```shell
####
## testing the be-application namespace
####
# get pods in the be-application namespace, as the be-application user
$ kubectl get pods --namespace=be-application --as=be-application
No resources found in be-application namespace.

# we can also get the secrets in this namespace:
$ kubectl get secrets --namespace=be-application --as=be-application
NAME                  TYPE                                  DATA   AGE
default-token-7j9vf   kubernetes.io/service-account-token   3      18m

# however, we are not allowed to get pods in namespace=be-application, if we impersonate the be-database user
$ kubectl get pods --namespace=be-application --as=be-database
Error from server (Forbidden): pods is forbidden: User "be-database" cannot list resource "pods" in API group "" in the namespace "be-application"

####
## testing the be-database namespace
####
# we are allowed to get pods in namespace be-database, as the user be-database
$ kubectl get pods --namespace=be-database --as=be-database
No resources found in be-database namespace.

# we are not allowed to get secrets
kubectl get secrets --namespace=be-database --as=be-database
Error from server (Forbidden): secrets is forbidden: User "be-database" cannot list resource "secrets" in API group "" in the namespace "be-database"

# and we are not allowed to get pods in the be-database namespace as user be-application
$ kubectl get pods --namespace=be-database --as=be-application
Error from server (Forbidden): pods is forbidden: User "be-application" cannot list resource "pods" in API group "" in the namespace "be-database"

####
## testing the impersonation
####
# we are not allowed to just impersonate a random not-existing user.
$ kubectl get pods --as=not-existing
Error from server (Forbidden): users "not-existing" is forbidden: User "nicolaj" cannot impersonate resource "users" in API group "" at the cluster scope
```
