# K8S Home

The repository for the configuration of my home kubernetes cluster. The system utilises the following products:

- [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) - To enable automated deployment of changes in this repository via a gitops process
- [Reloader](https://github.com/stakater/Reloader) - To enable automated redeployment of deployments when related configuration or secrets are changed
- [Cloudflared](https://github.com/cloudflare/cloudflared) - To enable ingress from Cloudflare's CDN to externalise services hosted within the cluster
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) - To allow secrets to be securely stored in a public repo using public/private key cryptography
- [CSI Storage Driver - NFS](https://github.com/kubernetes-csi/csi-driver-nfs) - To enable services to utilise NFS backed persistent storage across multiple nodes.

## Getting Started

### Bootstrap cluster on Raspberry PIs

1. Login to your Raspberry PI

2. Enable memory/cpu settings 
```bash
sudo sed -i '$ s/$/ cgroup_enable=memory cgroup_memory=1/' /boot/firmware/cmdline.txt
```

3. Enable automated updates
```bash
sudo dpkg-reconfigure --priority=low unattended-upgrades
sudo tee -a /etc/apt/apt.conf.d/20auto-upgrades > /dev/null << EOF
> APT::Periodic::Verbose "1";
> EOF
```

5. Install Microk8s
```bash
sudo snap install microk8s --classic
```

5. Update permissions to be able to execute commands without sudo
```bash
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
su - $USER
```

6. Wait until the cluster is up
```bash
microk8s status --wait-ready
```

7. Enable useful services 
```bash
microk8s enable ingress 
microk8s enable metrics-server
microk8s enable dashboard
```

8. Install kubectl cli
```bash 
sudo snap install kubectl --classic
```

9. Install helm cli
```bash 
sudo snap install helm --classic
```

10. Download kubectl config --classic
```bash
microk8s config > ~/.kube/config ; chmod 600 ~/.kube/config
```

### _Optional:_ Adding additional nodes

1. Login to your first Raspberry PI

2. Run command to obtain join token
```bash
microk8s add-node 
```

3. Login to second Raspberry PI
   
4. Run command collected from output step 2 

5. Fin

### Add sealed secrets operator
_These steps assume you have already configured your .kube/config file to allow access to the cluster via the kubectl CLI._

0. Add sealed secrets repo
```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
```
1. Install sealed secrets 
```bash
helm install sealed-secrets -n shared-services sealed-secrets/sealed-secrets --create-namespace
```

2. Install useful tools
```bash
sudo snap install jq
KUBESEAL_VERSION=$(curl -s https://api.github.com/repos/bitnami-labs/sealed-secrets/tags | jq -r '.[0].name' | cut -c 2-)
wget "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-arm64.tar.gz"
tar -xvzf kubeseal-${KUBESEAL_VERSION}-linux-arm64.tar.gz kubeseal
rm kubeseal-${KUBESEAL_VERSION}-linux-arm64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

3. Retrieve the public key to encrypt secrets
```bash
kubeseal --controller-name=sealed-secrets --controller-namespace=shared-services --fetch-cert > sealed-secret-public-key.pem
```

### Enable gitops integration via argocd
_These steps assume you have already configured your .kube/config file to allow access to the cluster via the kubectl CLI._

1. Install argocd
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```

2. Expose UI via a service
```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```
Or alternatively use port-forward on your machine to temporarily provide access:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

3. Obtain the admin password to login to Argo CD portal
```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 --decode ; echo
```

4. Login to portal. If you've forwarded the port it should be available at https://localhost:8080.

## How to

### How to obtain the token to login to the Kubernetes Dashboard

```bash
kubectl get secret microk8s-dashboard-token -n kube-system -o jsonpath={".data.token"} | base64 -d ; echo
```

### How to generate a new sealed secret

1. Create the insecure secret file or using the template provided:
```bash
kubectl -n default create secret generic example-secret \
--from-literal=key1=value1 \
--from-literal=key2=value2 \
--dry-run=client \
-o yaml > example-secret.yaml
```
2. Secure it using public key
```bash
kubeseal --format=yaml --cert=sealed-secret-public-cert.pem < example-secret.yaml > example-secret-sealed.yaml
```

You can obtain the public key with the following command:
```bash
kubeseal --fetch-cert \
--controller-name=sealed-secrets \
--controller-namespace=shared-services \
> sealed-secret-public-cert.pem
```
### How to generate a new TLS sealed secret

1. Grab the cert.pem and key.pem and ensure they're in your current folder
2. Create the insecure secret file:
```bash
kubectl -n default create secret tls cloudflare-origin-cert-secret --key key.pem --cert cert.pem --dry-run=client -o yaml > cloudflare-origin-cert-secret.yaml
```
3. Secure it using public key
```bash
kubeseal --format=yaml --cert=sealed-secret-public-cert.pem < cloudflare-origin-cert-secret.yaml > cloudflare-origin-cert-secret-sealed.yaml
```

### How to bootstrap initial cluster app

1. Login to ArgoCD UI
2. Click on New App
3. Click Edit As YAML
4. Paste content below
```yaml 
project: default
source:
  repoURL: 'git@github.com:jamesgawn/k8s-home.git'
  path: cluster
  targetRevision: HEAD
  directory:
    recurse: true
    jsonnet: {}
destination:
  server: 'https://kubernetes.default.svc'
  namespace: default
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```
5. Click Create