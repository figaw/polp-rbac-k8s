# Minikube Auditing

The api-server is a control-plane component.

It's often configured by the Kubernetes Addon Manager,
which means the Kubelet is configured to look in the
`/etc/kubernetes/manifests`-folder for manifests that should be always deployed.

This is also the case when using Minikube, except the nodes are containers,
and thus we need to make changes inside these.

In Minikube we can configure auditing using a hack like:
<https://minikube.sigs.k8s.io/docs/tutorials/audit-policy/>

> A magic certs-folder minikube mounts into its "nodes"
> can wrap our audit-policy.

I chose my own "hack;" editing the manifest inside the container. Because

- you wouldn't use Minikube in production, so it doesn't need to be scripted
    (it probably could be 🤷‍♂️)
- in production you'd edit the api-server on the control-plane nodes.


## Adding the audit-policy.yaml to the "node" container

```shell
$ docker exec -it minikube
root@minikube:/# cat /etc/kubernetes/audit-policy.yaml <<EOF
# Replace this with your desired audit-file
# Log all requests at the Metadata level.
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
EOF
```

## Reconfiguring the api-server manifest

1. open the manifest for editing

    ```shell
    $ docker exec -it minikube bash
    root@minikube:/# vi /etc/kubernetes/manifests/kube-apiserver.yaml
    ```

2. Add the audit-log flags:

    ```yaml
    spec:
    containers:
    - command:
        - kube-apiserver
        ...
        - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
        - --audit-log-path=/var/log/kubernetes/audit/audit.log
    ```

3. Configure the volumes.
    From [Kubernetes, Auditing, Log Backend](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/).
    Configure the volumeMounts:

    ```yaml
    volumeMounts:
    ...
    - mountPath: /etc/kubernetes/audit-policy.yaml
      name: audit
      readOnly: true
    - mountPath: /var/log/kubernetes/audit/
      name: audit-log
      readOnly: false
    ```

1. .. and then the hostpath volumes:

    ```yaml
    volumes:
    ...
    - name: audit
      hostPath:
        path: /etc/kubernetes/audit-policy.yaml
        type: File
    - name: audit-log
      hostPath:
        path: /var/log/kubernetes/audit/
        type: DirectoryOrCreate
    ```
