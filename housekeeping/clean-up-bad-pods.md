0. Get all pods that are not in a Running state

```Bash
kubectl -n $SERVE_NS get po --field-selector=status.phase!=Running
#OR
kubectl -n $SERVE_NS get po | grep -v Running
#NAME                                                             READY   #STATUS                   RESTARTS   AGE
#pod-1-error                                          0/1     #Error                    0          18d
#pod-1-healthy   0/1     Completed                0          19d
#sp-pod-12345                   0/1     ImagePullBackOff        0          22s
```

1. Describe the bad pod and check its logs

```Bash
export BAD_POD=pod-1-error
kubectl -n $SERVE_NS describe po $BAD_POD
kubectl -n $SERVE_NS logs $BAD_POD
```

The pod can have several types of parent objects or it can be orphaned. Follow the procedure based on the parent type:

### Bad pod with `Deployment` as parent

2. Get the pod's owner

```Bash
kubectl -n SERVE_NS get po $BAD_POD -o jsonpath='{.metadata.ownerReferences}'
#[{"apiVersion":"apps/v1","blockOwnerDeletion":true,"controller":true,"kind":"ReplicaSet","name":"replicaset-1-error","uid":"a76a6615c085-35ce-4227-af6c-e0ee6609"}]
```

The pod belongs to a ReplicaSet, deleting the pod will not resolve this since it will get recreated.

3. Get the deployment of the bad pod

```Bash
export BAD_REPLICA_SET=replicaset-1-error
kubectl -n $SERVE_NS get replicaset $BAD_REPLICA_SET -o jsonpath='{.metadata.ownerReferences}'
#[{"apiVersion":"apps/v1","blockOwnerDeletion":true,"controller":true,"kind":"Deployment","name":"deployment-1-target","uid":"fd20f95041a2-3181-4de8-99c3-d047f49f"}]
```

4. Check the deployment status

```Bash
export BAD_DEPLOYMENT=deployment-1-target
kubectl -n $SERVE_NS get deployment $BAD_DEPLOYMENT
#NAME     READY   UP-TO-DATE   AVAILABLE   AGE
#deployment-1-target   1/1     1            1           18d
```

The deployment is running with a healthy pod, we need to delete the failed pod.

5. Get pod labels of this release

```Bash
kubectl -n $SERVE_NS get po $BAD_POD --show-labels
#NAME                      READY   STATUS   RESTARTS   AGE   LABELS
#pod-1-error   0/1     Error    0          18d   access=project,app=bad-app,networking/allow-egress-to-studio-web=true,networking/allow-internet-egress=true,pod-template-hash=851234f6f6,pod=pod-1-error,project=some-project,release=some-release,type=app
```

List all pods of this release

```Bash
export POD_RELEASE=some-release
kubectl -n $SERVE_NS get po -l release=$POD_RELEASE
#NAME                      READY   STATUS    RESTARTS   AGE
#pod-1-healthy   1/1     Running   0          17d
#pod-1-error   0/1     Error     0          18d
```

6. Delete unhealthy pod

```Bash
kubectl -n $SERVE_NS delete po $BAD_POD
```

The pod will not get recreated because there is already a healthy pod running in the deployment satisfying the number of replicas (1/1).


### Bad pod with `Job` as parent

Note that the jobs can be a pre-hook or a post-hook job.

2. Get the pod's owner

```Bash
kubectl -n $SERVE_NS get po $BAD_POD -o jsonpath='{.metadata.ownerReferences}'
#[{"apiVersion":"batch/v1","blockOwnerDeletion":true,"controller":true,"kind":"Job","name":"delete-user-job","uid":"3f18-4593-b7cd-49daa365"}]% 
```

This pod will probably have an `ImagePullBackOff` status because it is using a Bitnami image that no longer exists. Or there could be another reason the pod is failing.

```Bash
kubectl -n $SERVE_NS describe po $BAD_POD
#Events:
#  Type     Reason   Age                     From     Message
#  ----     ------   ----                    ----     -------
#  Normal   BackOff  54s (x121167 over 19d)  kubelet  Back-off pulling image "bitnami/kubectl:1.28"
#  Warning  Failed   54s (x121167 over 19d)  kubelet  Error: ImagePullBackOff
```

3. List the pod's parent i.e. the job

```Bash
export BAD_JOB=delete-user-job
kubectl -n $SERVE_NS get job $BAD_JOB
#NAME                                          STATUS    COMPLETIONS   DURATION   AGE
#delete-user-job   Running   0/1           19d        19d
```

The job has been running for 19 days.

4. Delete the job

```Bash
kubectl -n $SERVE_NS delete job $BAD_JOB
```

The job can also be provisioning a MinIO bucket in which case the job status would still be `Running` and the pods would be erroring out.

```Bash
kubectl -n $SERVE_NS get jobs
NAME                                 STATUS     COMPLETIONS   DURATION   AGE
job-ok   Complete   1/1           81s        377d
job-not-ok-1-minio-provisioning         Running    0/1           22h        22h
job-not-ok-2-minio-provisioning         Running    0/1           24h        24h
```

You can check if the release label has any resources left but most likely the user has moved on and deployed another app, which means you can delete the hung jobs which will clear out the bad pods.


### Bad pod with no parent object

`sp-` pods are a common example of orphaned ShinyProxy pods and are created when a user tries to access a ShinyProxy app. A problematic `sp-` pods would have failed to get properly cleaned.

If the problematic `sp-` pod has a `CrashLoopBackOff` status, you can simply delete the pod:

```Bash
kubectl -n $SERVE_NS delete po $BAD_POD
```
