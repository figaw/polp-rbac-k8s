# Minikube RBAC Demo

By default our "minikube context/user" is cluster-admin.
We'd like to limit our permissions a bit.

## Creating a new User

Generate a key and certificate signing request:

```shell
$ openssl genrsa -out nicolaj.key 2048
$ openssl req -new -key nicolaj.key -out nicolaj.csr
```

Set your username as the `CN` and the groups you belong to as the
`O` (organization attribute) of the CSR.

> NB: In the example below, I'm using the username `nicolaj` and
> the organization `frontend-developer`.

Verify the information using,

```shell
$ openssl req -text -noout -verify -in nicolaj.csr
verify OK
Certificate Request:
    Data:
        Version: 1 (0x0)
        Subject: O = frontend-developer, CN = nicolaj
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                ...
```

Get the Base64 encoding of the `.csr` and add that to a
CertificateSigningRequest resource:

```shell
$ cat nicolaj.csr | base64
LS0tLS...
```

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: nicolaj
spec:
  request: <base64>
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - client auth
```

## Approving the CSR

As an administrator we can `get` certificateSigningRequests,
    describe them, approve them and finally see that the condition is
    `Approved,Issued`.

```shell
$ kubectl get certificatesigningrequests
NAME      AGE   SIGNERNAME                            REQUESTOR       REQUESTEDDURATION   CONDITION
nicolaj   6s    kubernetes.io/kube-apiserver-client   minikube-user   24h                 Pending

$ kubectl describe csr      # shortname for certificatesigningrequests
Name:         myuser
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration={...}

CreationTimestamp:   Mon, 20 Jun 2022 15:28:43 +0200
Requesting User:     minikube-user
Signer:              kubernetes.io/kube-apiserver-client
Requested Duration:  24h
Status:              Pending
Subject:
         Common Name:    nicolaj
         Serial Number:
         Organization:   frontend-developer
Events:  <none>

$ kubectl certificate approve nicolaj
certificatesigningrequest.certificates.k8s.io/nicolaj approved

$ kubectl get csr
NAME     AGE   SIGNERNAME                            REQUESTOR       REQUESTEDDURATION   CONDITION
nicolaj   56s   kubernetes.io/kube-apiserver-client   minikube-user   24h                 Approved,Issued
```

## Get the certificate

Either use `kubectl get csr/nicolaj -o yaml` and decode the certificate or,
the handy oneliner..

```shell
$ kubectl get csr nicolaj -o jsonpath='{.status.certificate}'| base64 -d > nicolaj.crt
```

## Create a Role for the User

```shell
$ kubectl create role developer --verb=create --verb=get --verb=list --verb=update --verb=delete --resource=pods
$ kubectl create rolebinding developer-binding-nicolaj --role=developer --user=nicolaj
```

We can verify that `user:nicolaj` can `get pods` by using impersonation,

```shell
$ kubectl get pods --as=nicolaj
```

## Create the user in our Kubeconfig and map it to a context

```shell
$ kubectl config set-credentials nicolaj@minikube --client-key=nicolaj.key --client-certificate=nicolaj.crt --embed-certs=true
$ kubectl config set-context nicolaj@minikube --cluster=minikube --user=nicolaj@minikube
```

Switch context and try out our access

```shell
$ kubectl config use-context nicolaj@minikube
$ kubectl get pods
No resources found in default namespace.
$ kubectl get nodes
Error from server (Forbidden): nodes is forbidden: User "nicolaj" cannot list resource "nodes" in API group "" at the cluster scope
```

## Using groups instead of users

By switching back to our cluster-admin minikube context,
    we can edit the `rolebinding` and map it to the `group: frontend-developer`
    instead of the `user: nicolaj`, and see that it still works,

```shell
$ kubectl edit rolebindings developer-binding-nicolaj
```

Rename the user and add the group, like below:

```diff
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding-nicolaj
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: frontend-developer
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
+   name: nicolajX
+ - apiGroup: rbac.authorization.k8s.io
+   kind: Group
+   name: frontend-developer
```

Now verify with impersonation that the change was successful:

```shell
# as minikube cluster admins, it works
$ kubectl get pods
No resources found in default namespace.

# as nicolaj, it fails because the user is nicolajX now
$ kubectl get pods --as=nicolaj
Error from server (Forbidden): pods is forbidden: User "nicolaj" cannot list resource "pods" in API group "" in the namespace "default"

# as nicolajX it works.
$ kubectl get pods --as=nicolajX
No resources found in default namespace.

# as "any-user" in group "frontend-developer", it works
$ kubectl get pods --as=some-user --as-group=frontend-developer
No resources found in default namespace.
```
