### Events with `FailedToRetrieveImagePullSecret`

1. Print latest events and some info about each:

```Bash
kubectl -n $SERVE_NS get events --field-selector=type=Warning -o jsonpath='{range .items[0:10]}{"\n"}{.lastTimestamp}{">>>"}{.involvedObject.kind}{"  /// "}{.involvedObject.name}{"  ---  "}{.reason}{"--------"}{.message}{"\n\n"}'

#2026-03-20T13:50:49Z>>>Pod  /// adult-lung-iss-1  ---  FailedToRetrieveImagePullSecret--------Unable to retrieve some image pull secrets (regcred); attempting to pull the image may not succeed.


#2026-03-20T13:52:31Z>>>Pod  /// allium-1  ---  FailedToRetrieveImagePullSecret--------Unable to retrieve some image pull secrets (regcred); attempting to pull the image may not succeed.


#2026-03-20T13:52:39Z>>>Pod  /// alloy-1  ---  FailedToRetrieveImagePullSecret--------Unable to retrieve some image pull secrets (regcred); attempting to pull the image may not succeed.


#2026-03-20T13:51:15Z>>>Pod  /// ames-1  ---  FailedToRetrieveImagePullSecret--------Unable to retrieve some image pull secrets (regcred); attempting to pull the image may not succeed.


#2026-03-20T13:52:24Z>>>Pod  /// atlas-scrinshot-1  ---  FailedToRetrieveImagePullSecret--------Unable to retrieve some image pull secrets (regcred); attempting to pull the image may not succeed.
```

The `FailedToRetrieveImagePullSecret` warning appears.

2. The object in question is a pod. Check what image the pod is using:

```Bash
kubectl -n $SERVE_NS get po $BAD_POD -o yaml
#...
#  containers:
#  - image: docker.io/repo/tissuumaps:3
#...
```

3. The pod is running and the app is reachable from the browser. Try to pull the image locally:

```Bash
docker pull docker.io/repo/tissuumaps:3
#Error response from daemon: no matching manifest for linux/arm64/v8 in the manifest list entries: no match for platform in manifest: not found
```

Try to inspect the image and pull using the right architecture:

```Bash
docker manifest inspect docker.io/repo/tissuumaps:3
#...
#    "platform": {
#    "architecture": "amd64",
#    "os": "linux"
#    }
#...
docker pull --platform linux/amd64 docker.io/repo/tissuumaps:3
```

The image is pulled.

```Bash
kubectl get deployments -n $SERVE_NS -o yaml | grep -B5 -A5 "imagePullSecrets"
#...
```

Image pull secrets are being injected into the deployment of every pod.
