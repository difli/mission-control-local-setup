# 📡 Remote Access to a kind Kubernetes Cluster Using kubectl

This guide documents how to expose a `kind` (Kubernetes in Docker) cluster's API securely and access it from a remote machine using `kubectl`.

---

## 🧱 Prerequisites

- A remote machine with Docker, `kind`, and `kubectl` installed
- A local development machine with `kubectl` installed
- Public access (e.g. cloud IP like `18.192.51.197`)
- Open TCP port `45451` (or your chosen API port) on the remote machine's firewall

---

## 1. 🚀 kind Cluster Configuration (Remote Host)

Save the following as `kind.yaml` on your **remote machine**:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  apiServerAddress: 0.0.0.0
  apiServerPort: 45451
nodes:
- role: control-plane
  image: kindest/node:v1.32.2
  labels:
    mission-control.datastax.com/role: platform
  extraPortMappings:
    - containerPort: 30880
      hostPort: 30880
      listenAddress: "0.0.0.0"
      protocol: tcp
    - containerPort: 30081
      hostPort: 30081
      listenAddress: "0.0.0.0"
      protocol: tcp
    - containerPort: 30001
      hostPort: 30001
      listenAddress: "0.0.0.0"
      protocol: tcp
- role: worker
  image: kindest/node:v1.32.2
  labels:
    mission-control.datastax.com/role: platform
- role: worker
  image: kindest/node:v1.32.2
  labels:
    mission-control.datastax.com/role: platform
- role: worker
  image: kindest/node:v1.32.2
  labels:
    mission-control.datastax.com/role: platform
```

Start the cluster:

```bash
kind create cluster --config kind.yaml --name kind
```

---

## 2. 🔐 Create a Remote-Accessible ServiceAccount

On the remote host:

```bash
kubectl create serviceaccount remote-admin -n kube-system

kubectl create clusterrolebinding remote-admin-binding \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:remote-admin
```

---

## 3. 🔑 Generate Access Token

Use the following command to generate a token for the service account (Kubernetes v1.24+):

```bash
kubectl -n kube-system create token remote-admin
```

This will return a JWT token. Copy it securely.

---

## 4. 🧾 Create kubeconfig on Local Machine

Create a file: `~/.kube/config-kind`

```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    server: https://18.192.51.197:45451
    insecure-skip-tls-verify: true
  name: kind-kind
contexts:
- context:
    cluster: kind-kind
    user: kind-user
  name: kind-kind
current-context: kind-kind
users:
- name: kind-user
  user:
    token: <INSERT_REMOTE_ADMIN_TOKEN_HERE>
```

Replace `<INSERT_REMOTE_ADMIN_TOKEN_HERE>` with the JWT token you copied earlier.

---

## 5. ✅ Test the Setup

From your local machine:

```bash
KUBECONFIG=~/.kube/config-kind kubectl get nodes
```

You should see:

```bash
NAME                 STATUS   ROLES           AGE     VERSION
kind-control-plane   Ready    control-plane   4m37s   v1.32.2
kind-worker          Ready    <none>          4m27s   v1.32.2
kind-worker2         Ready    <none>          4m27s   v1.32.2
kind-worker3         Ready    <none>          4m27s   v1.32.2
```

---

## 🧪 Debugging Tips

- Confirm API port is listening on all interfaces:
  ```bash
  ss -tuln | grep 45451
  ```
- If using a cloud VM, check your **security group / firewall** allows inbound TCP 45451.
- Get verbose kubectl logs:
  ```bash
  kubectl get nodes -v=6
  ```
- Check service account token:
  ```bash
  kubectl -n kube-system create token remote-admin
  ```

---

## 🧹 Cleanup

```bash
kind delete cluster --name kind
```

---

## 📘 References

- [kind](https://kind.sigs.k8s.io/)
- [Kubernetes ServiceAccounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
