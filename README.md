# Etcd-Cluster on Amazon EKS

Step by step tutorial for running etcd-cluster on Amazon EKS cluster

### Disclaimer

Please note this tutorial is for demonstration purpose only, please **_DO NOT_** blindly apply it to your production environments.

## Prerequisites

- [kubectl](https://kubernetes.io/docs/tasks/tools/) - The Kubernetes command-line tool
- [helm](https://helm.sh/) - The Kubernetes Package Manage

### Assumptions

- All the tools required were setup properly
- The cluster have at least 3 nodes (2 vCPU + 2G RAM per node)

## Preparation

For demo purpose, here I will use minimized [values.yaml](values.yaml).

and the instance type for the EKS nodes would be `t3.small (2 vCPU + 2G RAM)`.

    $ kubectl get nodes # ...
    NAME                              STATUS   INSTANCE-TYPE
    ip-192-168-101-101.ec2.internal   Ready    t3.small
    ip-192-168-101-102.ec2.internal   Ready    t3.small
    ip-192-168-101-103.ec2.internal   Ready    t3.small

## Let's get started

First, we need to setup bitnami charts,

    $ helm repo add bitnami https://charts.bitnami.com/bitnami
    "bitnami" has been added to your repositories

Then, make sure the "bitnami" chart is update-to-date,

    $ helm repo update bitnami
    Hang tight while we grab the latest from your chart repositories...
    ...Successfully got an update from the "bitnami" chart repository
    Update Complete. ⎈Happy Helming!⎈

Now, we are ready to provision the etcd-cluster, it is suggested to review the content of [values.yaml](values.yaml) once again.

    $ helm install my-etcd-cluster bitnami/etcd --values ./values.yaml

Then, you should get the following output,

```plaintext
NAME: my-etcd-cluster
LAST DEPLOYED: Sat Oct  1 23:55:07 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: etcd
CHART VERSION: 8.5.5
APP VERSION: 3.5.5

// ...
```

That's all !!!

## Frequently Asked Questions

### Why my pods can't be started?

:warning: If you are running with Amazon EKS 1.23 (or later), you will need to setup [Amazon EBS CSI driver](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html) or it will failed like this :point_down:.

    % kubectl version --client=false --output=json | jq .serverVersion
    {
        // ...
        "gitVersion": "v1.23.10-eks-15b7512", # <-------------
        // ...
    }

    % kubectl get pods
    NAME          READY   STATUS    RESTARTS      AGE
    etcd-0        0/1     Running   1 (72s ago)   4m54s
    etcd-1        0/1     Running   1 (67s ago)   4m54s
    etcd-2        0/1     Running   1 (75s ago)   4m54s

### How to access to the etcd-cluster?

    $ export ETCD_ROOT_PASSWORD=$(kubectl get secret --namespace default etcd -o jsonpath="{.data.etcd-root-password}" | base64 -d)

    $ kubectl run etcd-client \
        --restart='Never' \
        --image docker.io/bitnami/etcd:3.5.5-debian-11-r1 \
        --env ROOT_PASSWORD=${ETCD_ROOT_PASSWORD} \
        --env ETCDCTL_ENDPOINTS="etcd.default.svc.cluster.local:2379" \
        --namespace default \
        --command -- sleep infinity

    $ kubectl exec --namespace default -it etcd-client -- /bin/bash -c "etcdctl --user root:\$ROOT_PASSWORD put /message Hello"
    OK

    $ kubectl exec --namespace default -it etcd-client -- /bin/bash -c "etcdctl --user root:\$ROOT_PASSWORD get /message"
    /message
    Hello

### Is it safe to decrease pods count for maintenance?

Yes, if you scale down Pods properly.

    $ kubectl scale statefulsets etcd --replicas 2
    statefulset.apps/etcd scaled

    $ kubectl get pods
    NAME          READY   STATUS    RESTARTS   AGE
    etcd-0        1/1     Running   0          8m3s
    etcd-1        1/1     Running   0          8m3s # <------------- etcd-2 is gone.
    etcd-client   1/1     Running   0          5m44s

    $ kubectl exec --namespace default -it etcd-client -- /bin/bash -c "etcdctl --user root:\$ROOT_PASSWORD get /message"
    /message
    Hello # <------------- cluster still alive!

    $ kubectl scale statefulsets etcd --replicas 3
    statefulset.apps/etcd scaled

    $ kubectl get pods
    NAME          READY   STATUS    RESTARTS   AGE
    etcd-0        1/1     Running   0          10m
    etcd-1        1/1     Running   0          10m
    etcd-2        1/1     Running   0          80s # <------------- etcd-2 back online after liveness probe passed.
    etcd-client   1/1     Running   0          8m10s

    $ kubectl exec --namespace default -it etcd-client -- /bin/bash -c "etcdctl --user root:\$ROOT_PASSWORD get /message"
    /message
    Hello # <------------- cluster still alive!

## Cleanup

:warning: **_WARNING: all data will be removed permanently, and there's no way back, unrecoverable_** :warning:

    $ helm -n default uninstall my-etcd-cluster
    $ kubectl -n default delete pods etcd-client
    $ kubectl -n default delete persistentvolumeclaim/data-etcd-0
    $ kubectl -n default delete persistentvolumeclaim/data-etcd-1
    $ kubectl -n default delete persistentvolumeclaim/data-etcd-2
