## Minikube Config and Helm Deployment
```bash
minikube start --driver=docker

# If using Minkube in WSL2, you will to provide some fixed node ports to access from local
# I'm going to use 30080 for my app and 30081/2/3 for argocd ui and other things!
# Now if you deploy your app using one of these node ports you can access from local :)
minikube start \
    --driver=docker \
    --ports=127.0.0.1:30080:30080 \
    --ports=127.0.0.1:30081:30081 \
    --ports=127.0.0.1:30082:30082 \
    --ports=127.0.0.1:30083:30083

kubectl config get-contexts
kubectl config set-context minikube

# Helm install app
kubectl create namespace apps
helm upgrade --install argocd-python-fastapi charts/apps \
    --namespace apps \
    --values apps/fastapi/helm-config/minikube.yaml

# Run a simple curl to check
curl http://localhost:30080/items/5\?q\=somequery
```



### Minikube Enabling NGINX
```bash
# If you need, enable nginx for minikube
minikube addons enable ingress

# Get minikube ip
minikube ip

# Change ip below to your minikube ip
echo "192.168.49.2 argocd-python-fastapi.local argocd-python-fastapi" >> /etc/hosts

# NOTE: I had to do this on WSL
sudo -- sh -c "echo \"192.168.49.2 argocd-python-fastapi.local\" >> /etc/hosts"

# Test using nginx host
curl http://argocd-python-fastapi.local/items/5\?q\=somequery
```


## Deploy ArgoCD on Minikube and Login

Links: 
- https://argo-cd.readthedocs.io/en/stable/getting_started/

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

The cli let's you login to the server and configure various things. You can read more about this here: https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli



### Accessing Argo CD Api Server

As I'm using WSL2 and Minkube in this example, I'm just going to use a node port as mentioned above.
```bash
# So we can access from local outside of WSL2
kubectl apply -n argocd -f  minikube-setup/argocd-wsl-custom-nodeport-svc.yaml

# Login at https://localhost:30082 with username admin and auto generated secret which you can get running:
kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

When deploying to the cloud, you will need to expose ArgoCd using a Load Balancer, Ingress or port forwarding. See here for details: https://argo-cd.readthedocs.io/en/stable/getting_started/#3-access-the-argo-cd-api-server



### Deploying our ArgoCD Application using Github Repository

```bash
# So I can use ssh, I created an ssh key pair and uploaded it to my github account. I created a k8s secret so we can reference this in our argocd application manifest when defining repositories

# NOTE: github ssh key issue discussed here https://github.com/argoproj/argo-cd/issues/7600
ssh-keygen -t ed25519 -a 100

# Add secret to argocd namespace
kubectl create secret generic github-private-key \
    --from-file=privateKey=/home/josh/.ssh/argocd_id_ed25519 \
    --namespace argocd

# Update ArgoCD config map with repo ssh creds
kubectl apply -n argocd -f argocd/argocd-repos-configmap.yaml

# Deploy argo cd application
kubectl apply -n argocd -f argocd/application.yaml
```


### ArgoCD CLI
There is also fancy argocd cli you can use to setup apps in your cluster. Commands below show you how I deploy a nodejs express app using it.

```bash
# Install argocd
# NOTE: I just used linuxbrew for this, there is probably a much better way to install this on WSL2 Ubuntu, I was just being lazy!
brew install argocd

# login to argocd instance
# NOTE: if you need to fetch auto generate admin password again you can  run this
kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
argocd login localhost:30082

# Setup an application using the cli
argocd app create argocd-nodejs-express \
    --project default \
    --revision HEAD \
    --repo git@github.com:jhawthorn22/azure-argocd-gitops.git \
    --path charts/apps \
    --release-name argocd-nodejs-express \
    --values "../../apps/express/helm-config/minikube.yaml" \
    --dest-server https://kubernetes.default.svc \
    --dest-namespace apps \
    --sync-policy auto \
    --sync-option CreateNamespace=true \
    --self-heal \
    --auto-prune \
    --sync-retry-limit 5 \
    --label "depoyed-via-argocd-cli=true"

```



#### ArgoCD CLI Update Admin Password
```bash
# assuming you've logged in...
argocd account update-password
```



#### ArgoCD CLI Deploying to External Clusters (Multicluster Architecture)
If deploying to an external cluster (a cluster where ArgoCD isn't installed), you will need to define the cluster. Only required for multi cluster architectires. For this minikube example, you can skip this!
```bash
kubectl config get-contexts -o {name}
argocd cluster add {name}
```



## Links
- https://github.com/argoproj/argocd-example-apps