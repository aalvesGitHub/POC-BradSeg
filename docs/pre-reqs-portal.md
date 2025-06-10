# PRÉ-REQUISITOS

Antes de executar os **Pré-requisitos** do Portal, tem que ter realizar à [Criação do cluster](docs/create-k8s.md).

Tabela de conteúdo
==================

- [DNS](#dns)
  - [DNS record](#dns-record)
  - [Certificado Wildcard](#certificado-wildcard)
- [Conecta no cluster K8S](#conecta-no-cluster-k8s)
- [Servidor NFS](#servidor-nfs)
- [Storage](#storage)
- [Loadbalance](#loadbalance)
  - [METALLB](#metallb)
- [Certificado](#certificado)
  - [Criar secret com certificado](#criar-secret-com-certificado)
  - [Configuração do nginx ingress controller](#configuração-do-nginx-ingress-controller)
  - [Criação do ingress DX](#criação-do-ingress-dx)
- [Deploy Dashboard do Cluster K8S](#deploy-dashboard-do-cluster-k8s)
  - [Secret for Token from admin-user](#secret-for-token-from-admin-user)
  - [Criação do ingress dashboard](#criação-do-ingress-dashboard)
  - [Próxima etapa: Instalação do Portal](#próxima-etapa-instalação-do-portal)

## DNS

### DNS record
Solicitar as inclusões dos nomes no DNS do domínio bradseg.com.br, conforme tabela abaixo:

| Nome            | IP address              |
|-----------------|-------------------------|
| www-dev         | 192.168.160.1           |
| intranet-dev    | 192.168.160.1           |
| form-dev        | 192.168.160.1           |
| console-k8s-dev | 192.168.160.1           |

### Certificado Wildcard

* Solicitar um certificado SSL Wildcard para o domínio bradseg.com.br


## Conecta no cluster K8S
```bash
scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null user@192.168.160.1:/home/user/.kube/config $HOME/.kube
```

## Servidor NFS
```bash
ssh user@192.168.160.8
sudo apt install nfs-kernel-server -y
sudo mkdir /nfs
sudo chmod 777 /nfs
sudo vim /etc/exports
/nfs            192.168.160.0/28(rw,sync,no_subtree_check)
---
sudo systemctl restart nfs-server
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server
```

## Storage
```bash
# NFS PROVISIONER
cd /tmp
git clone https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner.git
cd nfs-subdir-external-provisioner/deploy
kubectl create ns infra-services
setns infra-services'
NS=$(kubectl config get-contexts|grep -e "^\*" |awk '{print $5}')
NAMESPACE=${NS:-default}
cd ..
sed -i'' "s/namespace:.*/namespace: $NAMESPACE/g" ./deploy/rbac.yaml ./deploy/deployment.yaml
kubectl create -f deploy/rbac.yaml
kubectl apply -f -<<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  namespace: infra-services
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: k8s.gcr.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 192.168.160.8
            - name: NFS_PATH
              value: /nfs
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.160.8
            path: /nfs
EOF
kubectl apply -f -<<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: infra-sc
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false"
EOF
kubectl get sc
```

## Loadbalance
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

### METALLB
```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update
helm install metallb metallb/metallb -nmetallb-system --create-namespace
kubectl apply -f -<<EOF
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: dx-dev
  namespace: metallb-system
spec:
  addresses:
  - 192.168.160.1/32
  autoAssign: true
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: dx-dev
  namespace: metallb-system
spec:
  ipAddressPools:
  - dx-dev
  nodeSelectors:
  - matchLabels:
      kubernetes.io/hostname: mgsrva0423.magnasistemas.com.br
EOF
```

## Certificado

### Criar secret com certificado
```bash
kubectl create ns dxdev
kubectl -ndxdev create secret tls domain-bradseg --key=ssl/bradseg.key --cert=ssl/bradseg.crt
```

### Configuração do nginx ingress controller
```bash
cat > ingress-nginx.yaml<<EOF
controller:
  autoscaling:
    enabled: true
    maxReplicas: 3
    minReplicas: 1
  extraArgs:
    default-ssl-certificate: dxdev/domain-bradseg
  resources:
    requests:
      cpu: 50m
      memory: 200Mi
EOF
helm upgrade ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace -f ingress-nginx.yaml
```

### Criação do ingress DX
```bash
kubectl apply -f -<<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dx-internet
  namespace: "dxdev"
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: 100m
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "120"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - www-dev.bradseg.com.br
  rules:
  - host: www-dev.bradseg.com.br
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: dx-haproxy
            port:
              number: 80
EOF
```

## Deploy Dashboard do Cluster K8S
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
# Service account
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

### Secret for Token from admin-user
```bash
kubectl apply -f -<<EOF
apiVersion: v1
kind: Secret
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/service-account.name: "admin-user"   
type: kubernetes.io/service-account-token
EOF
kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath={".data.token"} | base64 -d
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

### Criação do ingress dashboard
```bash
kubectl apply -f -<<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: k8s-console
  namespace: kubernetes-dashboard
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: 100m
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "120"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - console-k8s-dev.bradseg.com.br
  rules:
  - host: console-k8s-dev.bradseg.com.br
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kubernetes-dashboard
            port:
              number: 443
EOF
```

## Próxima etapa: Instalação do Portal
Ir para [Instalação do Portal](install-dx-cf214.md).