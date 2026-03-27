# Kubesonde graph bug - only one deployment showing

## What's happening

When you look at the results JSON or the UI graph, every pod shows `deploymentName: java-app`
even when the source is `mysql-0` or `monitoring-probe`. That's wrong.

There are two bugs causing this.

---

## Bug 1 — the main one

In `crd/controllers/inner/continuous_mode_controller.go` around line 128, there's a
copy-paste mistake. The code is trying to fill in the destination pod's deployment name
but it accidentally writes to `output.Source` instead of `output.Destination`:

```go
// broken
if dst_in_state {
    output.Source.ReplicaSetName = dst.Source.ReplicaSetName
    output.Source.DeploymentName = dst.Source.DeploymentName
}
```

Change both `output.Source` to `output.Destination`:

```go
// fixed
if dst_in_state {
    output.Destination.ReplicaSetName = dst.Source.ReplicaSetName
    output.Destination.DeploymentName = dst.Source.DeploymentName
}
```

---

## Bug 2 — StatefulSet and bare pods

In `crd/controllers/utils/utils.go`, `GetReplicaAndDeployment` only knows how to find
the owner if the pod belongs to a `ReplicaSet → Deployment`. It returns empty strings
for StatefulSet pods like `mysql-0` and bare pods like `monitoring-probe` and `db-admin`.

Add a StatefulSet fallback so those pods at least show their StatefulSet name, and for
bare pods just use the pod name itself:

```go
func GetStatefulSet(pod k8sAPI.Pod) (string, error) {
    refs := pod.OwnerReferences
    ssRefs := lo.Filter(refs, func(ref metav1.OwnerReference, idx int) bool {
        return ref.Kind == "StatefulSet"
    })
    names := lo.Map(ssRefs, func(ref metav1.OwnerReference, idx int) string {
        return ref.Name
    })
    if len(names) == 1 {
        return names[0], nil
    }
    return "", errors.New("no statefulset")
}
```

Then update `GetReplicaAndDeployment` to try this fallback:

```go
func GetReplicaAndDeployment(client kubernetes.Interface, pod k8sAPI.Pod) (string, string) {
    if replicaSetName, err := GetReplicaSet(pod); err == nil {
        replicaSet, err := client.AppsV1().ReplicaSets(pod.Namespace).Get(
            context.TODO(), replicaSetName, metav1.GetOptions{},
        )
        if err != nil {
            return replicaSetName, ""
        }
        if deployment, err2 := GetDeployment(*replicaSet); err2 == nil {
            return replicaSetName, deployment
        }
        return replicaSetName, ""
    }

    if statefulSetName, err := GetStatefulSet(pod); err == nil {
        return statefulSetName, statefulSetName
    }

    return "", pod.Name
}
```

---

## Rebuild and redeploy

The running image is `ghcr.io/kubesonde/controller:latest` pulled from the internet.
After fixing the code you need to build your own image and push it to minikube's
internal registry.

> Note: `minikube docker-env` does not work with multi-node clusters so we use the
> registry addon instead.

```bash
# enable the registry addon (only needed once)
minikube addons enable registry

# forward the registry to your machine
kubectl port-forward -n kube-system service/registry 5000:80 &

# go to the crd folder
cd crd/

# build and push to the local registry (sudo needed if docker requires it)
sudo docker build --no-cache -t localhost:5000/kubesonde-controller:fix .
sudo docker push localhost:5000/kubesonde-controller:fix

# switch the deployment to your fixed image
kubectl set image deployment/kubesonde-controller-manager \
  manager=localhost:5000/kubesonde-controller:fix \
  -n kubesonde-system

# verify
kubectl rollout status deployment/kubesonde-controller-manager -n kubesonde-system
```

---

## Re-trigger scans and get results

```bash
# go back to the project
cd /path/to/projects

# delete old scans
kubectl delete kubesonde --all -n kubesonde-system

# reapply
kubectl apply -f path-to.../kubesonde-cr-app.yaml
kubectl apply -f path-to../kubesonde-cr-db.yaml

# wait ~30 seconds then forward the probes API port (runs on 2709)
kubectl port-forward -n kubesonde-system deployment/kubesonde-controller-manager 9001:2709 &

# save results
curl -s http://localhost:9001/probes > /tmp/results-fixed.json
```

Upload `/tmp/results-fixed.json` to the kubesonde UI to see the graph.
