# Always free k3s
This file provides some commands for you to copy and paste

Values you need to change will be in < > brackets, e.g.:
```
ssh ubuntu@<MASTER_NODE_IP_ADDRESS> -i <PATH_TO_KEY_FILE>
```
translates to
```
ssh ubuntu@1.2.3.4 -i ssh-key-2023-09-01.key
```
# Cluster setup
### SSHing into VMs
```bash
ssh ubuntu@<MASTER_NODE_IP_ADDRESS> -i <PATH_TO_KEY_FILE>
```
### Clearing iptables
```bash
## save existing rules
sudo iptables-save > ~/iptables-rules
## modify rules, remove drop and reject lines
grep -v "DROP" iptables-rules > tmpfile && mv tmpfile iptables-rules-mod
grep -v "REJECT" iptables-rules-mod > tmpfile && mv tmpfile iptables-rules-mod
## apply the modifications
sudo iptables-restore < ~/iptables-rules-mod
## check
sudo iptables -L
## save the changes
sudo netfilter-persistent save
sudo systemctl restart iptables
```
### Installing k3s on your machine
```bash
curl -sLS https://get.k3sup.dev | sh
sudo install k3sup /usr/local/bin/

## Verify installation
k3sup --help
```

### Setting up the master
If you want to merge it with your existing kubeconfig
```bash
k3sup install \
--host <MASTER_NODE_IP_ADDRESS> \
--user ubuntu \
--ssh-key <PATH_TO_KEY_FILE> \
--context k3s-on-oracle \
--local-path $HOME/.kube/config \
--merge
```
If you have no existing kubeconfig:
```bash
k3sup install \
--host <MASTER_NODE_IP_ADDRESS> \
--user ubuntu \
--ssh-key <PATH_TO_KEY_FILE> \
--context k3s-on-oracle \
```

### Joining agents
```bash
k3sup join \
  --ip <AGENT_NODE_IP_ADDRESS> \
  --server-ip <MASTER_NODE_IP_ADDRESS> \
  --user ubuntu \
  --ssh-key <PATH_TO_KEY_FILE>
```

# Configuring the cluster
### Installing ArgoCD
Install to cluster
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
Get admin password
```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 --decode
```
Port forward
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:80
```

## Deploying the first app
### SSH-key generation
```bash
ssh-keygen -t rsa -b 4096 -f github-deploy-key
```
### Helm chart creation
```bash
helm create whoami
```
### Annotations to add to ingress
```yaml
annotations:
  kubernetes.io/ingress.class: traefik
  traefik.ingress.kubernetes.io/router.middlewares: whoami-strip-prefix@kubernetescrd
```
### Strip-prefix middleware
Simply paste this into the `ingress.yaml` at the bottom. IMPORTANT: Keep the `---`!
```yaml
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: strip-prefix
spec:
  stripPrefix:
    prefixes:
      - /whoami
```
### Installing cert-manager
```bash
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.12.0 \
  --set installCRDs=true
  ```
### Installing the ionos-webhook for cert manager
```bash
helm repo add cert-manager-webhook-ionos https://fabmade.github.io/cert-manager-webhook-ionos

helm install cert-manager-webhook-ionos cert-manager-webhook-ionos/cert-manager-webhook-ionos -ncert-manager
```
### Set Argo to insecure
Edit the configmap
```bash
kubectl edit configmap argocd-cmd-params-cm -n argocd
```
Add the following lines at the same level as kind and metadata
```yaml
apiVersion: v1
data:
  server.insecure: "true"
kind: ConfigMap
```
Restart the deployment:
```bash
kubectl -n argocd rollout restart deploy
```

# Let's get stuff deployed
### Installing postgres
You can choose any <RELEASE_NAME> you like, e.g. `my-postgres`
```bash
helm install <RELEASE_NAME> \
  oci://registry-1.docker.io/bitnamicharts/postgresql \
  -n bitnami-postgres \
  --create-namespace \
  --set auth.postgresPassword=<PASSWORD_FOR_POSTGRES_USER>
```
### Posting to url shortener service
```bash
curl --location 'https://<YOUR_DOMAIN>/shortme/shorten' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'original_url=https://www.google.com'
```