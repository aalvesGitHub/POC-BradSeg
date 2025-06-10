# Reinicialização

Tabela de conteúdo
==================

- [Reinicialização](#reinicialização)
  - [Portal](#portal)
    - [Parar Portal](#parar-portal)
    - [Iniciar Portal](#iniciar-portal)
  - [DB2](#db2)
    - [Parar DB2](#parar-db2)
    - [Iniciar DB2](#iniciar-db2)
  - [LDAP](#ldap)
    - [ Parar LDAP](#parar-ldap)
    - [ Iniciar LDAP](#iniciar-ldap)


## Reinicialização

### Portal

**time de virtualização** quando tiver que realizar uma manutenção na virtualização e todas as VMs do portal serão afetadas, devem parar todos os serviços antes de realizar essa manutenção.

>**NOTA:**
> Não envolve os servidores de BANCO e LDAP.

#### Parar Portal

1. Baixe o kubeconfig do cluster K8S
```bash
mkdir $HOME/.kube
scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null support@192.168.160.1:/home/support/.kube/config $HOME/.kube
kubectl get nodes
```

2. Parar os workloads
```bash
kubectl -ndxdev scale --replicas 0 sts,deploy
```

#### Iniciar Portal

1. Baixe o kubeconfig do cluster K8S
```bash
mkdir $HOME/.kube
scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null support@192.168.160.1:/home/support/.kube/config $HOME/.kube
kubectl get nodes
```

2. Iniciar os workloads
```bash
kubectl -ndxdev scale --replicas 1 sts,deploy
```

### DB2

**time de virtualização** quando tiver que realizar uma manutenção na virtualização e afetar o servidor de banco, deve parar o serviço do DB2 antes da manutenção.

#### Parar DB2

1. Conectar no servidor com usuário support
```bash
ssh support@192.168.160.9
sudo systemctl stop db2.service
```

#### Iniciar DB2

2. Inicialização do DB2 com usuário support
```bash
ssh support@192.168.160.9
sudo systemctl start db2.service
```

### LDAP

**time de virtualização** quando tiver que realizar uma manutenção na virtualização e afetar o servidor de LDAP, deve parar o serviço do LDAP antes da manutenção.

#### Parar LDAP

1. Conectar no servidor com usuário support
```bash
ssh support@192.168.160.11
sudo systemctl stop ldap.service
```

#### Iniciar LDAP

2. Inicialização do DB2 com usuário support
```bash
ssh support@192.168.160.11
sudo systemctl start ldap.service
```
