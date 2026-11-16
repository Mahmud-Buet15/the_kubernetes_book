# Further Notes

Extra explanations that came up while reading, kept out of the chapter notes.

## High ports vs low ports (Chapter 7, NodePort)

A NodePort Service is reached on a high port such as `node-ip:31755`, while people and browsers expect the well-known low ports. That gap is why a LoadBalancer or Ingress sits in front on 80/443.

|  | Low (well-known) ports | NodePort range (high ports) |
| --- | --- | --- |
| Range | 0–1023 | 30000–32767 |
| Examples | 80 (HTTP), 443 (HTTPS), 22 (SSH), 53 (DNS) | 30080, 31755 (random if you don't pick one) |
| How clients use them | by default: `https://shop.example.com` means port 443 | must be typed: `http://node-ip:31755` |
| Firewalls | commonly open for web traffic | often blocked |
| Who listens | binding them on a Linux host normally needs root or the NET\_BIND\_SERVICE capability | any process; kube-proxy opens them on every node |

In practice: use NodePort for debugging or as plumbing behind a load balancer, and expose real services on 80/443 through a LoadBalancer or an Ingress.

## DNS A and SRV records (Chapter 10, service registration)

When a Service is created, the cluster DNS registers DNS **A and SRV records** for it. A DNS record is a row in the DNS server's table: a **name**, a **type**, and the **answer** for that name. The type says what kind of answer it is. For every Service, CoreDNS adds two kinds of row:

| Record | Question it answers | Name it's stored under | Answer for `valkyrie-web` in `default` |
| --- | --- | --- | --- |
| **A** (AAAA for IPv6) | "What IP is this name?" | `valkyrie-web.default.svc.cluster.local` | the ClusterIP, e.g. `10.96.88.12` |
| **SRV** (service record) | "Where does this *service* run: which host and which port?" | `_http._tcp.valkyrie-web.default.svc.cluster.local` (`_<port name>._<protocol>.<service FQDN>`) | priority, weight, port `8080`, target `valkyrie-web.default.svc.cluster.local` |

- **Think of it like an office directory.** The A record gives the building's address (the IP). The SRV record also gives the door number (the port) and how to knock (TCP or UDP).
- **A records are what apps use almost all the time.** An app calls `http://valkyrie-web:8080`. The port is already in the app's config, so it only asks DNS for the IP. This is the lookup in step 1 of Chapter 10's Service discovery diagram.
- **SRV records let a client find the port too**, so the port doesn't have to be written into the app. Some protocols and libraries (LDAP, Kerberos, SIP, some gRPC and database clients) look up SRV records. Kubernetes creates one only for **named** ports (`ports: - name: http`), because the port name is part of the record's name.
- **Headless Services (Ch 13) are different.** Their A record returns **each ready Pod's IP** instead of one ClusterIP, and their SRV records point to each Pod's own DNS name. That's how StatefulSet Pods get stable names.
- **Why it's automatic:** CoreDNS watches the API server. When a Service is created, changed or deleted, CoreDNS updates these records at once, and no one edits DNS by hand.

See them yourself from a tools Pod (the same `dnsutils` Pod as in Chapter 10's troubleshooting checklist):

```bash
kubectl run -it --rm dnsutils --image registry.k8s.io/e2e-test-images/jessie-dnsutils:1.7 -- bash
# A record: name -> ClusterIP
dig +short valkyrie-web.default.svc.cluster.local
# SRV record: priority weight port target
dig +short SRV _http._tcp.valkyrie-web.default.svc.cluster.local
```

## IPVS rules: how kube-proxy routes ClusterIP traffic (Chapter 10)

In the Chapter 10 flow, the last step is kube-proxy creating **local routing rules (IPVS)** on every node. Those rules are what turn a packet sent to a Service's ClusterIP into a packet sent to a real Pod.

**What IPVS is.** IPVS (IP Virtual Server) is a **layer-4 load balancer built into the Linux kernel**. It keeps a table of **virtual services**: an IP, a port and a protocol, each with a list of **real servers** (the backends) and a **scheduling algorithm** that picks one backend per new connection. kube-proxy doesn't forward any traffic itself. It only writes this table, and the kernel does the forwarding.

**What kube-proxy puts in the table:**

- **One virtual service per Service port.** For example, `10.96.88.12:8080/TCP` for `valkyrie-web`. NodePorts, external IPs and LoadBalancer IPs each get their own entry too.
- **One real server per ready endpoint** in the Service's EndpointSlices, such as `10.244.1.5:9000` (Pod IP and targetPort).
- **Every ClusterIP is bound to a dummy interface, `kube-ipvs0`, on each node**, so the kernel treats packets for it as local and hands them to IPVS. The IP is still unreachable from outside the cluster, because nothing routes to it.
- **Small helper sets of iptables rules and ipsets** handle what IPVS can't do alone: masquerading (SNAT) where needed, and NodePort and LoadBalancer filtering.
- kube-proxy **watches Services and EndpointSlices** through the API server. When a Pod becomes unready or is deleted, its real server is removed (first set to weight 0 so open connections can finish). New Pods are added within moments.

**One request, step by step:**

1. A Pod sends a packet to `10.96.88.12:8080`.
2. The packet reaches the node's kernel. Because `10.96.88.12` is on `kube-ipvs0`, IPVS matches it to the virtual service.
3. The scheduler picks a real server (round-robin by default), for example `10.244.2.7:9000`.
4. IPVS rewrites the destination (**DNAT**, shown as `Masq`). The packet then travels over the pod network to that Pod, even if the Pod is on another node.
5. **conntrack** remembers the connection. The rest of the connection's packets go to the same Pod, and replies have their source rewritten back to the ClusterIP, so the client never sees the Pod IP.

**What the table looks like on a node** (`ipvsadm -Ln`):

```text
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port      Forward Weight ActiveConn InActConn
TCP  10.96.88.12:8080 rr
  -> 10.244.1.5:9000         Masq    1      0          0
  -> 10.244.2.7:9000         Masq    1      2          0
  -> 10.244.3.4:9000         Masq    1      1          0
```

Read it as: the virtual service `10.96.88.12:8080` uses round-robin (`rr`) across three Pods. `Masq` means NAT forwarding. `Weight 1` means the Pods share traffic equally, and `ActiveConn` counts open connections per Pod.

**Scheduling and stickiness:**

- The default scheduler is `rr` (round-robin). kube-proxy's config can pick others, such as `lc` (least connections) or `sh` (source hashing), through `ipvs.scheduler`. This setting is cluster-wide, not per Service.
- `sessionAffinity: ClientIP` on a Service becomes IPVS **persistence**: the same client IP keeps going to the same Pod until the timeout (default 3 hours, set with `sessionAffinityConfig`).

**kube-proxy's three Linux modes compared:**

| Mode | How it finds the backend | Load balancing | At scale (thousands of Services) |
| --- | --- | --- | --- |
| **iptables** (the default) | walks a chain of NAT rules, one per Service and endpoint | random, using rule probabilities | slower: lookups and rule rewrites grow with the rule count |
| **IPVS** | hash table lookup in the kernel | real algorithms: rr, lc, sh and others | fast lookups and updates; this is why it was popular for big clusters |
| **nftables** (GA in Kubernetes 1.33) | nftables maps and sets | random | fast, without IPVS's extra iptables helpers; the recommended future mode |

**Important update since the book:** **IPVS mode was deprecated in Kubernetes 1.35** (KEP-5495). kube-proxy now warns when it starts in IPVS mode. The plan is to turn it off by default in 1.40 and remove it in 1.43. The recommended move is to the **nftables** mode, or iptables where nftables isn't available. The concepts above (virtual IP, backend list, DNAT, conntrack) apply to every mode; only the kernel machinery differs.

**Which mode does your cluster use?**

- Many clusters, including Docker Desktop's kind clusters, run the **iptables** default, not IPVS. kubeadm-style clusters show the mode in the kube-proxy ConfigMap: `kubectl -n kube-system get configmap kube-proxy -o yaml | grep mode` (an empty `mode: ""` means iptables).
- On a node, `curl -s localhost:10249/proxyMode` asks kube-proxy directly.
- **GKE** uses iptables, or, with GKE Dataplane V2, **eBPF (Cilium) with no kube-proxy at all**. The same job is done by eBPF programs instead of IPVS or iptables rules.

**Seeing the rules yourself** (run on a node; on Docker Desktop, open a node with `docker exec -it desktop-worker bash`):

```bash
# IPVS mode
ipvsadm -Ln                         # virtual services and their Pods (if ipvsadm is installed)
ip addr show kube-ipvs0             # every ClusterIP bound to the dummy interface

# iptables mode
iptables -t nat -L KUBE-SERVICES -n | grep valkyrie-web

# nftables mode
nft list table ip kube-proxy
```

## RESTful APIs, with Kubernetes examples (Chapter 15)

Chapter 15 says every create, read, update and delete goes through the API server as a **RESTful HTTPS request**. This section explains what that means, first in general and then for Kubernetes.

### What REST is

**REST** (Representational State Transfer) is a style for designing web APIs, described by Roy Fielding in 2000. It isn't a protocol or a library, just a set of rules that most HTTP APIs follow:

- **Everything is a resource with its own URL.** A book is `/books/42`; the list of all books is `/books`. A Pod is `/api/v1/namespaces/default/pods/web`.
- **A small, fixed set of verbs acts on any resource:** the HTTP methods GET, POST, PUT, PATCH and DELETE. You don't invent endpoints like `/createBook`; you send `POST /books`.
- **You exchange representations, not the thing itself.** The server sends a JSON (or YAML) document describing the resource's state, and you send one back to change it.
- **Each request stands alone (stateless).** It carries everything needed to handle it, including the credentials. The server keeps no session, so any replica of the server can answer any request. This is why Kubernetes can run several API server replicas behind a load balancer.
- **Status codes report the outcome**, using standard HTTP numbers (200, 201, 404 and so on), so every client understands them.

### HTTP methods

| Method | Meaning | Safe? (changes nothing) | Idempotent? (repeating it has the same effect as once) | Typical success code |
| --- | --- | --- | --- | --- |
| **GET** | read one resource or a list | yes | yes | 200 OK |
| **POST** | create a new resource in a collection | no | no: two POSTs make two resources | 201 Created |
| **PUT** | replace a resource with the full new version you send | no | yes | 200 OK |
| **PATCH** | change only the fields you send | no | usually | 200 OK |
| **DELETE** | remove a resource | no | yes | 200 OK or 204 No Content |

### Status codes you'll meet

| Code | Meaning | Kubernetes example |
| --- | --- | --- |
| 200 OK | it worked; here is the result | `kubectl get pod web` |
| 201 Created | the new resource exists | `kubectl create -f pod.yml` |
| 400 Bad Request | the request is malformed | broken JSON in the body |
| 401 Unauthorized | who are you? (authentication failed) | expired or missing token |
| 403 Forbidden | I know who you are, but you may not (authorization failed) | RBAC denies the verb (Ch 14) |
| 404 Not Found | no such resource | a Pod name with a typo |
| 409 Conflict | clashes with the current state | creating a name that exists (AlreadyExists), or updating a stale copy |
| 422 Unprocessable Entity | well-formed but invalid | a Pod with no containers (Invalid) |
| 500 Internal Server Error | the server failed | rare; check the API server logs |

### A plain REST example: a bookstore API

```bash
curl https://shop.example.com/books                 # GET: list all books -> 200 + JSON array
curl https://shop.example.com/books/42              # GET: one book -> 200, or 404 if missing

curl -X POST https://shop.example.com/books \
  -H 'Content-Type: application/json' \
  -d '{"title": "The Kubernetes Book", "price": 30}' # POST: create -> 201 Created, Location: /books/43

curl -X PUT https://shop.example.com/books/43 \
  -H 'Content-Type: application/json' \
  -d '{"title": "The Kubernetes Book", "price": 25}' # PUT: replace the whole book -> 200

curl -X PATCH https://shop.example.com/books/43 \
  -H 'Content-Type: application/merge-patch+json' \
  -d '{"price": 20}'                                 # PATCH: change one field -> 200

curl -X DELETE https://shop.example.com/books/43    # DELETE: remove it -> 200 or 204
```

### How Kubernetes maps onto REST

The URL tells the API server **which group and version, which Namespace, which kind, and which object**:

| What | URL path |
| --- | --- |
| all Pods in `default` (core group) | `/api/v1/namespaces/default/pods` |
| one Pod | `/api/v1/namespaces/default/pods/web` |
| a Deployment (named group `apps`) | `/apis/apps/v1/namespaces/default/deployments/web-deploy` |
| a cluster-scoped object (no Namespace) | `/api/v1/nodes/desktop-worker` |
| a **subresource** of an object | `.../deployments/web-deploy/scale`, `.../pods/web/log`, `.../pods/web/status` |

Every kubectl command becomes one or more of these requests:

| kubectl command | HTTP request |
| --- | --- |
| `kubectl get pods` | `GET /api/v1/namespaces/default/pods` |
| `kubectl get pod web` | `GET /api/v1/namespaces/default/pods/web` |
| `kubectl get pods -w` | `GET .../pods?watch=true`, a long-lived stream of change events |
| `kubectl create -f pod.yml` | `POST /api/v1/namespaces/default/pods` with the Pod as the body |
| `kubectl replace -f pod.yml` | `PUT /api/v1/namespaces/default/pods/web` |
| `kubectl apply`, `edit`, `label`, `scale` | `PATCH` on the object (or `/scale`); `apply` sends a `POST` when the object doesn't exist yet |
| `kubectl delete pod web` | `DELETE /api/v1/namespaces/default/pods/web` |

**Watch kubectl make the calls.** Raise the log level and kubectl prints each request and its status:

```bash
kubectl get pods -v=6
# ... GET https://127.0.0.1:6443/api/v1/namespaces/default/pods?limit=500 200 OK in 12 milliseconds
```

### Hands-on: drive the API with curl

`kubectl proxy` handles authentication for you and opens the API on localhost, so plain curl works:

```bash
kubectl proxy --port 9000 &

# READ: list Pods (returns a PodList with an items array)
curl -s localhost:9000/api/v1/namespaces/default/pods

# CREATE: POST a Pod -> 201 Created
curl -s -X POST localhost:9000/api/v1/namespaces/default/pods \
  -H 'Content-Type: application/json' \
  -d '{"apiVersion":"v1","kind":"Pod","metadata":{"name":"rest-demo"},
       "spec":{"containers":[{"name":"web","image":"nginx:1.27"}]}}'

# READ one object
curl -s localhost:9000/api/v1/namespaces/default/pods/rest-demo

# UPDATE: PATCH just a label (merge patch)
curl -s -X PATCH localhost:9000/api/v1/namespaces/default/pods/rest-demo \
  -H 'Content-Type: application/merge-patch+json' \
  -d '{"metadata":{"labels":{"tier":"web"}}}'

# Filter with query parameters (a label selector)
curl -s 'localhost:9000/api/v1/namespaces/default/pods?labelSelector=tier%3Dweb'

# DELETE
curl -s -X DELETE localhost:9000/api/v1/namespaces/default/pods/rest-demo

# Try an error: a second POST of the same Pod returns 409 AlreadyExists
```

When something fails, the body is a `Status` object you can read, for example:

```json
{"kind": "Status", "apiVersion": "v1", "status": "Failure",
 "message": "pods \"rest-demo\" not found", "reason": "NotFound", "code": 404}
```

### What Kubernetes adds on top of plain REST

- **Watch:** `?watch=true` keeps the connection open and streams ADDED, MODIFIED and DELETED events. Every controller, the scheduler and the kubelet use this instead of polling.
- **Optimistic concurrency:** every object has a `metadata.resourceVersion`. If you `PUT` a copy that someone else changed after you read it, the API server answers **409 Conflict** instead of silently overwriting their change.
- **Dry runs:** `?dryRun=All` (what `kubectl apply --dry-run=server` sends) runs authentication, admission and validation without saving anything.
- **Server-side apply:** `kubectl apply --server-side` sends a `PATCH` with content type `application/apply-patch+yaml`, and the API server tracks which tool owns which field.
- **Discovery:** `GET /api` and `GET /apis` list every group, version and resource the server offers, and that's how kubectl learns about CRDs. Try `kubectl get --raw /apis | head` or `kubectl api-resources`.

## Certificate Authorities (CA) and mTLS in Kubernetes (Chapter 16)

Chapter 16 says Kubernetes uses mTLS to stop spoofing, and that *"mTLS is only as secure as the CA: protect the CA. Approve certificate signing requests carefully and never trust the cluster CA outside the cluster."* This section explains what a CA is and then each part of that sentence.

### The problem a CA solves

When the kubelet talks to the API server, each side needs to know **who is really on the other end**. Anyone can claim to be `kube-apiserver`. Encryption alone doesn't help if you're encrypting to an impostor.

The answer uses **public-key cryptography**:

- Each component has a **key pair**: a **private key** it keeps secret and a **public key** it can share. Something signed with the private key can be checked with the public key, and only the holder of the private key can prove they own it.
- A **certificate** is a small file that says *"this public key belongs to this name"*. It also holds a validity period and what the key may be used for, and it is **signed by a CA**.
- A **Certificate Authority (CA)** is the party everyone agrees to trust to vouch for those statements. It has its own key pair, and its signature on a certificate means *"I checked, and this key really belongs to this name"*.

**Analogy:** a passport. The passport (certificate) states your name and has your photo (public key). It is issued and stamped by a government (CA). A border officer doesn't know you, but trusts the government's stamp and checks that the photo matches the person standing there (proof that you hold the private key).

### How it works, step by step

1. **The CA is created.** It generates its key pair and a **self-signed root certificate** (it vouches for itself). Its certificate, `ca.crt`, is copied to everyone who should trust it; its private key, `ca.key`, stays secret.
2. **A component asks for a certificate.** It generates its own key pair and sends the CA a **certificate signing request (CSR)**: its public key plus the name it wants, for example `CN=system:node:worker-1`.
3. **The CA checks the request and signs it.** The result is the component's certificate. The private key never leaves the component.
4. **TLS handshake.** When a client connects, the server presents its certificate. The client checks that the CA's signature is valid (using `ca.crt`), that the name matches who it meant to reach, and that the certificate hasn't expired. The server then proves it holds the matching private key.
5. **mTLS (mutual TLS):** the client also presents a certificate and the server checks it the same way. **Both sides prove who they are**, not just the server, as in a normal HTTPS website.

**How the server proves it holds the private key (step 4).** A certificate is public: anyone who has connected to the server has a copy, so just *showing* it proves nothing. An impostor could replay it. The proof is a **signature over fresh data that only the private key can make**:

1. The client opens the handshake with a **fresh random value** (the "client random") and its half of a one-time key exchange. Both are new for every connection.
2. The server replies with its certificate and then a **CertificateVerify** message. This is a digital signature, made **with the server's private key**, over a hash of the whole handshake so far, including the client's fresh random value.
3. The client checks that signature with the **public key inside the certificate**. If it verifies, whoever is on the other end holds the matching private key. Because the signed data includes the client's new random value, the signature can't be one recorded from an earlier connection.
4. Both sides then send **Finished** messages, keyed from the one-time key exchange. These confirm they derived the same session keys and that nobody changed any handshake message in transit. From here on, traffic is encrypted with those session keys.

It's like a border officer who, instead of only looking at your passport, says *"sign this random number I just made up"* and checks your signature against the one in the passport. Someone who stole a photocopy of your passport can't do that.

**In mTLS** the server also sends a *CertificateRequest*. The client then sends its own certificate and its own CertificateVerify signature over the same handshake, and the server checks it the same way. That's how the API server knows a kubelet really holds the private key for `system:node:worker-1`, not just a copy of its certificate.

(Details are for TLS 1.3, which Kubernetes uses by default. In older TLS 1.2 with RSA key exchange, the proof is indirect. The client encrypts a secret with the server's public key, and only the private key can decrypt it, so only the real server can produce correct Finished messages.)

Try the core idea with openssl: sign a random challenge with a private key, then verify it with only the public key:

```bash
openssl genpkey -algorithm ed25519 -out key.pem          # private key (keep secret)
openssl pkey -in key.pem -pubout -out pub.pem            # public key (what a certificate carries)
echo "challenge-7f3a9c" > challenge.txt                 # the verifier's fresh random value
openssl pkeyutl -sign -inkey key.pem -rawin -in challenge.txt -out sig.bin
openssl pkeyutl -verify -pubin -inkey pub.pem -rawin -in challenge.txt -sigfile sig.bin
# Signature Verified Successfully
# change challenge.txt, or sign with a different key, and it prints: Signature Verification Failure
```

On the public web, browsers and operating systems ship with a list of trusted public CAs (Let's Encrypt, DigiCert and others). That's why `https://` sites work without you installing anything. Inside a cluster, Kubernetes uses **its own private CA** instead.

### The CA inside a Kubernetes cluster

- On a kubeadm-built cluster, the cluster CA lives in `/etc/kubernetes/pki/ca.crt` and `ca.key` on the control plane nodes. It is **self-signed** and created automatically when the cluster is built. This is the "auto-generated self-signed CA" in the bullet above the one you selected.
- It signs certificates for the **API server, every kubelet, the scheduler, the controller manager and admin users**. etcd and the front proxy usually have their own separate CAs in the same folder.
- **A client certificate is also an identity for RBAC (Ch 14).** The certificate's `CN` becomes the **user name** and each `O` becomes a **group**. Your admin kubeconfig holds a certificate in an admin group; that's how the API server knows you're allowed everything.
- `ca.crt` (public, safe to share) is placed in every kubeconfig as `certificate-authority-data`, and in every Pod at `/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`, so everything can verify the API server.
- On **managed clusters (GKE, LKE, EKS)**, the provider holds the CA key and you never see it. That's one of the security benefits of a managed control plane.

Look at them yourself:

```bash
# The cluster CA: subject and issuer are the same (self-signed), plus its validity dates
openssl x509 -in /etc/kubernetes/pki/ca.crt -noout -subject -issuer -dates

# Your own client certificate from kubeconfig: CN = user name, O = groups
kubectl config view --raw -o jsonpath='{.users[0].user.client-certificate-data}' \
  | base64 -d | openssl x509 -noout -subject -issuer -enddate
```

### The selected text, explained

**1. "mTLS is only as secure as the CA: protect the CA."**

mTLS trusts *any* certificate the CA signed. So whoever holds `ca.key` can make a certificate for **any name in the cluster**: the API server, any node, or a user in the admin group `system:masters`, which **RBAC can't restrict**. With that key, an attacker becomes the cluster, and all the encryption still looks perfectly valid. `ca.key` is the master key to the whole cluster.

- Keep it only on control plane nodes, readable by root only, and never in Git, a Secret or a container image.
- Back it up encrypted and offline, and know how to **rotate** it.
- For stronger setups, keep the root CA offline and let the cluster use an **intermediate CA**, or use a managed control plane where you never hold the key.

**2. "Approve certificate signing requests carefully."**

Kubernetes has a **CertificateSigningRequest** API object. Anyone with permission to create one can ask for a certificate. Someone with approve rights runs `kubectl certificate approve`, and the controller manager signs it with the cluster CA. That's how Chapter 14 creates new users.

- **Approving a CSR grants an identity.** If the request says `O=system:masters`, approving it hands that person full cluster admin.
- **Kubernetes can't revoke a certificate.** It stays valid until it expires, unless you rotate the whole CA. A bad approval is hard to undo, so keep `expirationSeconds` short.
- Before approving, decode the request and check the **name, groups, signer and usages**:

```bash
kubectl get csr
kubectl get csr <name> -o jsonpath='{.spec.request}' | base64 -d \
  | openssl req -noout -subject        # who it claims to be: CN=user, O=groups
kubectl describe csr <name>             # signerName, usages, requested duration, requester
kubectl certificate approve <name>      # only when all of that is what you expect
```

Kubelets' own routine certificate renewals are approved automatically by a built-in controller. Requests from people are the ones to check by hand.

**3. "Never trust the cluster CA outside the cluster."**

The cluster CA is meant to vouch only for things **inside this cluster**. If you add its `ca.crt` to your laptop's, browser's or company's trusted list, it can vouch for **any website**: Kubernetes CAs have no limit on which names they may sign. Anyone who gets `ca.key`, or gets a CSR approved, could then make a certificate for `yourbank.com` or your company's login page. Your machine would accept it, allowing a man-in-the-middle attack. Its key also lives on cluster nodes, which aren't guarded like a public CA's.

- Use the cluster CA only for cluster components talking to each other.
- Give anything users or browsers reach (Ingress, public APIs) certificates from a **trusted public CA**, for example Let's Encrypt through cert-manager. That's the next bullet's advice: **two trusted key pairs**, the cluster CA for internal systems and third-party CAs for external ones.
