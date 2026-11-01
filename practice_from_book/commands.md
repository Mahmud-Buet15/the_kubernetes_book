# Kubernetes Commands and notes Summary

## Init
```bash
docker --version    # check docker version
kubectl version --client=true -o yaml   # check client version
kubectl get nodes   # check nodes
kubectl config view   # view kubeconfig file
kubectl config get-contexts   # check contexts
kubectl config use-context <context-name>   # change context
kubectl config rename-context <old> <new>   # rename context
kubectl config delete-context <name>        # delete context
kubectl cluster-info                      # show cluster info
```

## Pods
| Command | What it does |
| --- | --- |
| `kubectl apply -f pod.yml` | Post the manifest to the API server |
| `kubectl get pods [-o wide \| -o yaml]` | Status; `-o yaml` shows everything: `spec` = desired state, `status` = observed state (plus defaults you didn't set) |
| `kubectl get pods --watch` | Stream status changes |
| `kubectl get pods --show-labels` | Show pods with labels |
| `kubectl describe pod <pod>` | Formatted overview incl. conditions and **Events** |
| `kubectl logs <pod> [--container <name>]` | Logs; defaults to the first container |
| `kubectl exec <pod> -- <cmd>` | Run a command in the container (remote execution) |
| `kubectl exec -it <pod> -- sh` | Interactive exec session (like SSH) |
| `kubectl edit pod <pod>` | Edit a live object (most Pod fields are rejected as immutable) |
| `kubectl delete pod <a> <b>` / `kubectl delete -f <file>` | Delete by name or by manifest |


### Connect to the app inside a pod
```bash
kubectl exec git-sync -- curl localhost:80
# Or using external IP (of load balancer): 
kubectl get svc svc-sidecar
curl <EXTERNAL-IP>:80
```

## Namespaces
```bash
kubectl api-resources                   # list all available resources
kubectl api-resources --namespaced        # list all namespaced resources
kubectl get namespaces                    # or: kubectl get ns
kubectl describe ns default               # quotas, LimitRanges
kubectl get svc --namespace kube-system   # -n for short
kubectl get pods --all-namespaces
kubectl create ns hydra                   # imperative
kubectl apply -f shield-ns.yml            # declarative
kubectl delete ns hydra                   # deletes everything inside it
kubectl config get-contexts               # check current contexts
kubectl config set-context --current --namespace shield   # make shield the default for kubectl
kubectl config set-context --current --namespace default  # reset
```

## Deployments
### Notes
| Field | Meaning |
| --- | --- |
| `replicas` | Number of identical Pods to run |
| `selector.matchLabels` | Labels the Deployment/ReplicaSet use to find their Pods; must match `template.metadata.labels`; immutable after creation |
| `revisionHistoryLimit: 5` | Keep the last 5 ReplicaSets for rollback (too many bloats the object) |
| `progressDeadlineSeconds: 300` | Each new replica gets 5 minutes to start before it's reported as stalled |
| `minReadySeconds: 10` | Wait 10 s after each replica before replacing the next: throttles the rollout so you catch bad releases before they replace everything |
| `strategy.type: RollingUpdate` | Replace replicas incrementally |
| `maxUnavailable: 1` | May drop 1 below desired (10 → 9) |
| `maxSurge: 1` | May go 1 above desired (10 → 11) |

With desired = 10, the range 9–11 means **two Pods are replaced at a time**.


### Commands
```bash
kubectl apply -f deploy.yml
kubectl get deploy hello-deploy
kubectl describe deploy hello-deploy
kubectl get rs      # Get replicaset. ReplicaSet is named <deployment>-<hash>
kubectl describe rs <name>

kubectl scale deploy hello-deploy --replicas 5 # manual scaling

kubectl rollout status deploy/hello-deploy  # status of the deployment (progress)
kubectl rollout history deploy/hello-deploy   # revision history

kubectl rollout pause deploy hello-deploy     # pause the rollout
kubectl rollout resume deploy hello-deploy    # resume the rollout

kubectl rollout undo deploy/hello-deploy      # undo last revision
kubectl rollout undo deploy/hello-deploy --to-revision=1 # undo to specific revision

#cleanup
kubectl delete -f deploy.yml
kubectl delete -f lb.yml
```
### Connect to the app inside a pod
```bash
kubectl exec <pod_name> -- curl localhost:8080 # FAILED
# Or using external IP (of load balancer): 
kubectl get svc lb-svc
curl <EXTERNAL-IP>:8080
```