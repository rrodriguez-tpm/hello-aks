# Hello AKS - Deploying a secured website

### Create an AKS cluster using Terraform
```
❯ terraform init

❯ terraform plan

❯ terraform apply

```

### Inspect the cluster nodes
```
❯ kubectl get node --kubeconfig kubeconfig
NAME                              STATUS   ROLES   AGE   VERSION
aks-default-30253914-vmss000000   Ready    agent   21m   v1.24.6
aks-default-30253914-vmss000001   Ready    agent   21m   v1.24.6
```

### Export the KUBECONFIG variable to avoid using --kubeconfig on every command. The export is only valid for the current session
```
❯ export KUBECONFIG="${PWD}/kubeconfig"

❯ kubectl get node
NAME                              STATUS   ROLES   AGE   VERSION
aks-default-30253914-vmss000000   Ready    agent   24m   v1.24.6
aks-default-30253914-vmss000001   Ready    agent   24m   v1.24.6
```

### Install Cert-Manager using helm
```
❯ helm repo add jetstack https://charts.jetstack.io

❯ helm repo update

❯ helm upgrade cert-manager jetstack/cert-manager \
    --install \
    --create-namespace \
    --wait \
    --namespace cert-manager \
    --set installCRDs=true

❯ kubectl -n cert-manager get all
NAME                                           READY   STATUS    RESTARTS   AGE
pod/cert-manager-85945b75d4-gpgbt              1/1     Running   0          71s
pod/cert-manager-cainjector-7f694c4c58-vdb65   1/1     Running   0          71s
pod/cert-manager-webhook-7cd8c769bb-zfghh      1/1     Running   0          71s

NAME                           TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/cert-manager           ClusterIP   10.0.200.81   <none>        9402/TCP   72s
service/cert-manager-webhook   ClusterIP   10.0.22.252   <none>        443/TCP    72s

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cert-manager              1/1     1            1           72s
deployment.apps/cert-manager-cainjector   1/1     1            1           72s
deployment.apps/cert-manager-webhook      1/1     1            1           72s

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/cert-manager-85945b75d4              1         1         1       72s
replicaset.apps/cert-manager-cainjector-7f694c4c58   1         1         1       72s
replicaset.apps/cert-manager-webhook-7cd8c769bb      1         1         1       72s
```

### Create an ingress controller
```
❯ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

❯ helm repo update

❯ helm install ingress-nginx ingress-nginx/ingress-nginx \
    --create-namespace \
    --namespace ingress-nginx \
    --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz

❯ kubectl -n ingress-nginx get all
NAME                                            READY   STATUS    RESTARTS   AGE
pod/ingress-nginx-controller-6f7bd4bcfb-whkdh   1/1     Running   0          2m5s

NAME                                         TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
service/ingress-nginx-controller             LoadBalancer   10.0.255.242   0.0.0.0   80:32424/TCP,443:32166/TCP   2m5s
service/ingress-nginx-controller-admission   ClusterIP      10.0.29.204    <none>          443/TCP                      2m5s

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           2m5s

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-6f7bd4bcfb   1         1         1       2m5s
```

### Setup DNS using the EXTERNAL-IP shown by the load balancer

### Setup Let's Encrypt issuer:
```
❯ kubectl apply -f cert-issuer-nginx-ingress.yaml
clusterissuer.cert-manager.io/letsencrypt-cluster-issuer created

❯ kubectl describe clusterissuer letsencrypt-cluster-issuer
```

### Deploy the application
```
❯ kubectl apply -f deployment-app.yaml
deployment.apps/example-deploy created

❯ kubectl apply -f service-app.yaml
service/example-service created

❯ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
example-deploy-747dcf9779-tfqmd   1/1     Running   0          68s
example-deploy-747dcf9779-zx5xw   1/1     Running   0          68s
```

### Point the ingress controller to the app. Edit ingress.yaml and replace "<your.dns.com>" with your hostname
```
❯ kubectl apply -f ingress.yaml
ingress.networking.k8s.io/example-app created

❯ kubectl get services
NAME              TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
example-service   LoadBalancer   10.0.225.247   20.106.106.234   80:32478/TCP   14m
kubernetes        ClusterIP      10.0.0.1       <none>           443/TCP        13h
```

### Set up certificate
### Edit the certificate.yaml by updating the dns name
```
❯ kubectl apply -f certificate.yaml
certificate.cert-manager.io/example-app created
```

### Verify that the certificate is created successfully
```
❯ kubectl describe certificate example-app
```

### Check the secret, you should see the type kubernetes.io/tls
```
❯ kubectl get secret
NAME              TYPE                DATA   AGE
example-app-tls   kubernetes.io/tls   2      131m
```

### Test browsing to the DNS name