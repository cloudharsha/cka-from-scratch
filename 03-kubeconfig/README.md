# kubeconfig — A CKA-oriented README

A **kubeconfig** file tells `kubectl` **which cluster to talk to, who you are, and what default settings to use**. In the CKA, you’ll frequently switch contexts, set default namespaces, or fix broken auth. This guide focuses on exactly what you need in the exam.

---

## Where kubeconfig comes from (and priority)

`kubectl` loads config in this order (highest priority first):

1. `--kubeconfig /path/to/file` (flag on the command)
2. `$KUBECONFIG` (colon-separated list of files)
3. Default: `~/.kube/config`

If multiple files are provided (flag or `$KUBECONFIG=a:b:c`), they are **merged left-to-right**. Later entries can override earlier ones.

> CKA tip: Exam questions often say “use the context for cluster **k8s-1**.” Don’t invent paths—**use the context names given** and switch with `kubectl config use-context k8s-1`.

---

## kubeconfig structure (3 pillars)

A kubeconfig is YAML with three main lists plus some metadata:

```yaml
apiVersion: v1
kind: Config
clusters:        # (WHERE) API server endpoints + TLS info
- name: cluster-1
  cluster:
    server: https://10.0.0.1:6443
    certificate-authority: /etc/kubernetes/pki/ca.crt   # or certificate-authority-data: <base64>

users:           # (WHO) client auth (cert/key, token, exec, etc.)
- name: admin
  user:
    client-certificate: /etc/kubernetes/pki/admin.crt   # or client-certificate-data: <base64>
    client-key: /etc/kubernetes/pki/admin.key

contexts:        # (WHAT) a pairing of cluster + user (+ default namespace)
- name: admin@cluster-1
  context:
    cluster: cluster-1
    user: admin
    namespace: kube-system

current-context: admin@cluster-1  # default context used by kubectl
preferences: {}                   # rarely used
```

- **clusters**: API endpoint (`server`), TLS validation (CA file/data, or `insecure-skip-tls-verify: true`—avoid in real life).
- **users**: Auth methods:
  - Client cert & key (`client-certificate`, `client-key`)
  - Static `token`
  - `username`/`password` (rare)
  - `exec` plugin (e.g., cloud auth)
  - `auth-provider` (legacy OIDC)
- **contexts**: Glue that picks a **cluster**, a **user**, and an optional default **namespace**.
- **current-context**: What `kubectl` uses if you don’t pass `--context`.

> CKA tip: You will **modify contexts and namespaces** far more often than editing clusters/users.

---

## High-frequency `kubectl config` commands (memorize these)

```bash
# Show the active context and full view (merged)
kubectl config current-context
kubectl config get-contexts
kubectl config view --minify

# Switch context
kubectl config use-context <context-name>

# Set default namespace for current context
kubectl config set-context --current --namespace=<ns>

# Set default namespace for a specific context
kubectl config set-context <context-name> --namespace=<ns>

# Create or modify a context (bind cluster+user+namespace)
kubectl config set-context <ctx> --cluster=<cl> --user=<user> --namespace=<ns>

# Define/override a cluster
kubectl config set-cluster <cl> \
  --server=https://<host>:6443 \
  --certificate-authority=/path/ca.crt \
  --embed-certs=true

# Define/override a user (client certs)
kubectl config set-credentials <user> \
  --client-certificate=/path/admin.crt \
  --client-key=/path/admin.key \
  --embed-certs=true

# Define a user with a token
kubectl config set-credentials <user> --token="$(cat /path/token)"

# Point a context at the cluster+user
kubectl config set-context <ctx> --cluster=<cl> --user=<user>

# Delete entries if needed
kubectl config delete-context <ctx>
kubectl config unset users.<user>
kubectl config unset clusters.<cl>
```

> CKA tip: `--embed-certs=true` stores certs inside kubeconfig (base64) so you don’t rely on external file paths—handy in isolated exam pods.

---

## Namespaces & contexts (common exam pitfalls)

- `kubectl get po` uses the context’s **default namespace**.
- If a task says “operate in namespace **alpha**,” either:
  - `-n alpha` on each command, or
  - set context default: `kubectl config set-context --current --namespace=alpha`
- Verify with `kubectl config view --minify | grep namespace` and `kubectl get ns`.

---

## File locations you’ll see

- User default: `~/.kube/config`
- Control plane (kubeadm): `/etc/kubernetes/admin.conf` (often used with root)
- On nodes: `/etc/kubernetes/kubelet.conf`, `/etc/kubernetes/controller-manager.conf`, `/etc/kubernetes/scheduler.conf`

> CKA tip: If `~/.kube/config` is missing on a control-plane node, you can:
> ```bash
> # As root:
> kubectl --kubeconfig=/etc/kubernetes/admin.conf get nodes
> # Or copy:
> mkdir -p $HOME/.kube
> sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
> sudo chown "$(id -u)":"$(id -g)" $HOME/.kube/config
> ```

---

## TLS and certificate fields you may edit

Inside `clusters[].cluster`:

- `server`: API URL (`https://IP:6443`)
- `certificate-authority` **or** `certificate-authority-data`
- `insecure-skip-tls-verify: true` (avoid unless the task explicitly says so)

Inside `users[].user`:

- `client-certificate` / `client-certificate-data`
- `client-key` / `client-key-data`
- `token` (bearer token)
- `exec` (external auth command) or `auth-provider` (legacy)

> CKA tip: **x509 errors** usually mean CA mismatch or server name mismatch. If the API uses a hostname, ensure it’s covered by the server cert’s SANs; otherwise target the IP listed in the cert SANs or update the `server` field accordingly.

---

## Merging and inspecting configs quickly

```bash
# Temporarily point to a specific file
export KUBECONFIG=/path/a:/path/b
kubectl config get-contexts

# See what kubectl will actually use (single, current context)
kubectl config view --minify --flatten
```

- `--flatten` resolves `*-data` and removes file path references—handy for debugging or exporting.

---

## Troubleshooting quick hits

- **“Unable to connect to the server”**: check `clusters[].cluster.server` IP/port, network/firewall.
- **TLS x509 errors**: wrong CA, wrong `server` hostname, expired certs. Try:
  ```bash
  kubectl --v=6 get ns   # verbose logs to see which file/field is used
  ```
- **Permissions (RBAC)**: your user may lack verbs. Confirm with:
  ```bash
  kubectl auth can-i list pods -n <ns> --as <user> --context <ctx>
  ```
- **Multiple clusters provided in exam**: Don’t edit blindly—**switch context** first, then verify cluster with `kubectl cluster-info` and `kubectl get nodes`.

---

## Rapid CKA workflow (muscle memory)

```bash
# 0) Identify the target context from the question
kubectl config get-contexts
kubectl config use-context <target>

# 1) Set the target namespace (if specified)
kubectl config set-context --current --namespace=<ns>

# 2) Validate connectivity & scope
kubectl get nodes
kubectl get all -A | head

# 3) Do the task (deployments, services, RBAC, etc.)

# 4) Sanity check
kubectl get <resource> -n <ns>
```

---

## One-page cheatsheet

```bash
# Show & switch
kubectl config current-context
kubectl config get-contexts
kubectl config use-context <ctx>

# Namespace defaults
kubectl config set-context --current --namespace=<ns>
kubectl config set-context <ctx> --namespace=<ns>

# Create/modify building blocks
kubectl config set-cluster <cl> --server=https://<ip>:6443 \
  --certificate-authority=/path/ca.crt --embed-certs=true
kubectl config set-credentials <user> --token=<tok>   # or client certs with --client-certificate/--client-key --embed-certs=true
kubectl config set-context <ctx> --cluster=<cl> --user=<user> --namespace=<ns>

# Inspect
kubectl config view --minify --flatten
kubectl cluster-info

# Cleanup
kubectl config delete-context <ctx>
kubectl config unset users.<user>
kubectl config unset clusters.<cl>
```

---

## FAQ (exam-style)

- **Q: Task says “operate on cluster X”—what first?**  
  **A:** `kubectl config use-context X` → `kubectl get nodes` to confirm.

- **Q: I keep hitting the wrong namespace.**  
  **A:** `kubectl config set-context --current --namespace=<ns>` or use `-n <ns>` explicitly.

- **Q: Cert path vs embedded certs?**  
  **A:** Prefer `--embed-certs=true` in ad-hoc kubeconfigs to avoid path issues.

- **Q: How do I quickly make a throwaway kubeconfig?**  
  **A:** Use `kubectl config set-*` commands as shown in the minimal examples.
