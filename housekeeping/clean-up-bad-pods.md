1. Get all pods that are not in a Running state

```Bash
kubectl -n $SERVE_NS get po --field-selector=status.phase!=Running
#NAME                                                             READY   #STATUS                   RESTARTS   AGE
#pod-1-error                                          0/1     #Error                    0          18d
#pod-1-healthy   0/1     Completed                0          19d
```

2. Describe the bad pod and check its logs

```Bash
export BAD_POD=pod-1-error
kubectl -n $SERVE_NS describe po $BAD_POD
kubectl -n $SERVE_NS logs $BAD_POD
```

3. Get the pod's owner

```Bash
kubectl -n SERVE_NS get po $BAD_POD -o jsonpath='{.metadata.ownerReferences}'
#[{"apiVersion":"apps/v1","blockOwnerDeletion":true,"controller":true,"kind":"ReplicaSet","name":"replicaset-1-error","uid":"a76a6615c085-35ce-4227-af6c-e0ee6609"}]
```

The pod belongs to a ReplicaSet, deleting the pod will not resolve this since it will get recreated.

4. Get the deployment of the bad pod

```Bash
export BAD_REPLICA_SET=replicaset-1-error
kubectl -n $SERVE_NS get replicaset $BAD_REPLICA_SET -o jsonpath='{.metadata.ownerReferences}'
#[{"apiVersion":"apps/v1","blockOwnerDeletion":true,"controller":true,"kind":"Deployment","name":"deployment-1-target","uid":"fd20f95041a2-3181-4de8-99c3-d047f49f"}]
```

5. Check the deployment status

```Bash
export BAD_DEPLOYMENT=deployment-1-target
kubectl -n $SERVE_NS get deployment $BAD_DEPLOYMENT
#NAME     READY   UP-TO-DATE   AVAILABLE   AGE
#deployment-1-target   1/1     1            1           18d
```

The deployment is running with a healthy pod, we need to delete the failed pod.

6. Get pod labels of this release

```Bash
kubectl -n $SERVE_NS get po $BAD_POD --show-labels
#NAME                      READY   STATUS   RESTARTS   AGE   LABELS
#pod-1-error   0/1     Error    0          18d   access=project,app=bad-app,networking/allow-egress-to-studio-web=true,networking/allow-internet-egress=true,pod-template-hash=851234f6f6,pod=pod-1-error,project=some-project,release=some-release,type=app
```

List pods of this release

```Bash
export POD_RELEASE=some-release
kubectl -n $SERVE_NS get po -l release=$POD_RELEASE
#NAME                      READY   STATUS    RESTARTS   AGE
#pod-1-healthy   1/1     Running   0          17d
#pod-1-error   0/1     Error     0          18d
```

7. Delete unhealthy pod

```Bash
kubectl -n $SERVE_NS delete po $BAD_POD
```

The pod will not get recreated because there is already a healthy pod running in the deployment satisfying the number of replicas (1/1).
