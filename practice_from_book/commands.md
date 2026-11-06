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

## Services
### Commands
```bash
kubectl expose deployment svc-test --type=LoadBalancer   #create svc from existing Deployment
kubectl get svc -o wide  # to see the external ip and nodeport
kubectl describe svc svc-test # shows endpoints,type,ip address etc

kubectl get endpointslices   # to see the endpoint of the service
kubectl describe endpointslices <name> # to see the endpoint of the service in details
```

## Ingress
### GCP setup
- One-time setup: 
    - install the gcloud CLI and run
    - `gcloud components install gke-gcloud-auth-plugin kubectl`
    - `gcloud auth login`
    - `gcloud config set project <PROJECT_ID>` 
    - `gcloud services enable container.googleapis.com`
- Create the cluster name **tkb-new**:`gcloud container clusters create-auto tkb-new --region asia-southeast1`
- Connect kubectl: `gcloud container clusters get-credentials tkb-new --zone asia-southeast1` merges the cluster into **~/.kube/config** and switches to the context `gke_<project>_<zone>_tkb`. You don't download or rename a kubeconfig file the way you do for LKE.
- `kubectl get nodes` shows three `gke-tkb-default-pool-...` nodes with ROLES `<none>`. They're all workers, because Google runs the control plane for you.
- Delete the cluster when done: `gcloud container clusters delete tkb-new --zone asia-southeast1`. Then check Compute Engine → Disks for retained volumes, which keep costing money

### Running the hands-on on GKE
The apps (app.yml), the hosts-file step and the tests stay the same. Choose one of two controllers.

- GKE needs one extra step first, because the installer creates cluster-wide RBAC and your Google account must be cluster-admin to do that: `kubectl create clusterrolebinding cluster-admin-binding --clusterrole cluster-admin --user $(gcloud config get-value account)`.
- Then apply the same controller URL (`kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0/deploy/static/provider/cloud/deploy.yaml`) and the same `ig-all.yml`, unchanged. The controller's LoadBalancer Service gets a Google Cloud network load balancer, and ADDRESS shows its public IP. Put that IP in your hosts file.
- Remember that ingress-nginx is retired (see Deeper dive). That's fine for this lab, but not for real clusters.

### Commands
```bash
kubectl get pods -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx # to see the ingress-nginx pods
kubectl get svc -n ingress-nginx # to see the ingress-nginx service
kubectl get ingressclass # to see the ingress class

## Test ingress with curl (Bypassing DNS & Web Filters)
# Test shield host
curl -H "Host: shield.mcu.com" http://<INGRESS_IP>/
# Test mcu.com/shield
curl -H "Host: mcu.com" http://<INGRESS_IP>/shield



## Test with port forwarding
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80

# Test host-based routing
curl -H "Host: shield.mcu.com" http://localhost:8080/

# Test path-based routing
curl -H "Host: mcu.com" http://localhost:8080/shield
```

## Service Discovery

```bash
# check cluster DNS info
kubectl get pods   -n kube-system -l k8s-app=kube-dns   
kubectl get deploy -n kube-system -l k8s-app=kube-dns
kubectl get svc    -n kube-system -l k8s-app=kube-dns

kubectl get all -n <namespace>  # check all objects (pods,services,deployments,replicasets etc) from specific namespace

curl ent:8080                           # Resolves short names to the local Namespace
curl ent.prod.svc.cluster.local:8080    # Specifying FQDNs to connect across Namespaces

```

## Storage
```bash
kubectl get sc
kubectl describe sc premium-rwo

kubectl get pv
kubectl get pvc

```

## ConfigMap
```bash
## Creating ConfigMaps imperatively
# using --from-literal
kubectl create configmap testmap1 \
  --from-literal shortname=SAFC \
  --from-literal longname="Sunderland Association Football Club"

# using --from-file
kubectl create configmap testmap2 --from-file cmfile.txt

## inspecting
kubectl get cm
kubectl describe cm <configmap_name>
kubectl get cm <configmap_name> -o yaml

kubectl exec <podname> -- env | grep NAME   # inspecting specific env variable from a pod

kubectl logs <pod_name> -c <container_name> # check logs of specific container of a pod

```

## Secrets
```bash
## Creating Secrets imperatively
# using --from-literal
kubectl create secret generic creds \
  --from-literal user=nigelpoulton \
  --from-literal pwd=Password123

kubectl get secrets creds -o yaml # inspect the secrets
echo <base64_encoded_secret> | base64 -d    # decoding secret value
```

## StatefulSets
```bash

kubectl get sts # check statefulsets
kubectl get sts --watch
kubectl describe sts <sts_name> # describe statefulsets

kubectl describe pod <StatefulSets-pod> | grep ClaimName # check PV assigned to statefulset pods

# checking existance of DNS names from jump-pod 
dig SRV dullahan.default.svc.cluster.local

# deleting stateful sets (scaling down to 0 before deleting)
kubectl scale sts tkb-sts --replicas=0
kubectl delete sts tkb-sts

kubectl explain sts.spec.updateStrategy # explain statefulset update strategy

```


## API security and RBAC
```bash
kubectl api-resources --sort-by name -o wide  # check available resources for the cluster (API version, namespaced or not, etc.)

# initiate
kubectl create namespace dev # create a namespace
kubectl get ns # check namespaces
kubectl apply -f read_deployments.yml # apply roles and rolebindings for specific namespace

# get role info
kubectl get role --all-namespaces # check roles from all namespaced
kubectl get rolebindings --all-namespaces # check rolebindings from all namespaced

kubectl get clusterrole # check cluster roles
kubectl get clusterrolebindings  # check cluster rolebindings

# check details
kubectl describe role <role_name> -n <namespace> # describe role of specific namespace
kubectl describe rolebindings <rolebinding_name> -n <namespace> # describe rolebinding of specific namespace
kubectl describe clusterrole <clusterrole_name> # describe cluster role
kubectl describe clusterrolebindings <clusterrolebinding_name> # describe cluster rolebinding

# Examples
kubectl get role -n dev
kubectl describe role read-deployments -n dev

kubectl describe clusterrole cluster-admin
kubectl get clusterrolebindings | grep cluster-admin 
kubectl describe clusterrolebindings cluster-admin   

# Test permissions without guessing
kubectl auth can-i create deployments -n terran
kubectl auth can-i list deployments -n shield --as sky     # impersonate a user (needs rights to impersonate)
kubectl auth can-i list deployments -n dev --as sky        # impersonate a user (needs rights to impersonate)
kubectl auth can-i --list -n shield                         # everything you may do there
```

## The Kubernetes API
```bash
# Hands-on: talking REST directly
kubectl proxy --port 9000 &        # exposes the API on localhost and handles auth
curl -X GET http://localhost:9000/api/v1/namespaces/shield/pods   # PodList, items: []
curl -X GET http://localhost:9000/api/v1/namespaces
curl -v -X GET http://localhost:9000/api/v1/namespaces/shield/pods
#  > lines = sent: GET path, Accept: */*
#  < lines = received: HTTP/1.1 200 OK, Content-Type: application/json,
#    X-Kubernetes-Pf-* (priority and fairness headers)


# POST and DELETE a Namespace
curl -X POST -H "Content-Type: application/json" \
  --data-binary @ns.json http://localhost:9000/api/v1/namespaces
kubectl get ns                                  # shield Active
curl -X DELETE -H "Content-Type: application/json" \
  http://localhost:9000/api/v1/namespaces/shield  # status.phase: Terminating


# Inspecting the API
kubectl api-resources          # names, short names, APIVERSION, NAMESPACED, KIND
kubectl api-versions           # enabled group/versions (spot alpha, multiple versions)
kubectl explain <kind>         # KIND and VERSION of a resource
curl http://localhost:9000/api     # core group versions: ["v1"]
curl http://localhost:9000/apis    # APIGroupList with preferredVersion per group

# Watch kubectl's HTTP calls
kubectl get pods -n shield -v=6     # prints GET https://<server>/api/v1/namespaces/shield/pods ... 200 OK
kubectl get pods -n shield -v=8     # also prints request and response bodies

```


## Threat Modeling
### PSS and PSA
```bash
kubectl create ns psa-test

# enforce PSS
kubectl label --overwrite ns psa-test pod-security.kubernetes.io/enforce=baseline
kubectl apply -f psa-pod.yml      # privileged: true -> Forbidden: violates PodSecurity "baseline:latest"

# set privileged: false
kubectl apply -f psa-pod.yml      # pod/psa-pod created

# checking what might happen if further restricted PSS enforced
kubectl label --dry-run=server --overwrite ns psa-test pod-security.kubernetes.io/enforce=restricted
#  warns: allowPrivilegeEscalation != false, unrestricted capabilities,
#         runAsNonRoot != true, seccompProfile

# implementing further restricted PSS
kubectl label --overwrite ns psa-test pod-security.kubernetes.io/enforce=restricted
kubectl get pods -n psa-test      # psa-pod still Running: PSA only acts on create/modify

# enforce multiple PSS (enforce, warn, audit)
kubectl label --overwrite ns psa-test \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```