# Kubernetes Commands Summary

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
| `kubectl describe pod <pod>` | Formatted overview incl. conditions and **Events** |
| `kubectl logs <pod> [--container <name>]` | Logs; defaults to the first container |
| `kubectl exec <pod> -- <cmd>` | Run a command in the container (remote execution) |
| `kubectl exec -it <pod> -- sh` | Interactive exec session (like SSH) |
| `kubectl edit pod <pod>` | Edit a live object (most Pod fields are rejected as immutable) |
| `kubectl delete pod <a> <b>` / `kubectl delete -f <file>` | Delete by name or by manifest |


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