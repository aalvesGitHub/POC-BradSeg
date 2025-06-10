# Criação do cluster

Antes de iniciar a criação do cluster, deve ter acesso ao **usuário support** e o **sudoers com permissão total** em todos os servidores do ambiente. 

Tabela de conteúdo
==================

- [Pré-requisitos](#pré-requisitos)
- [Ansible](#ansible)
  - [Arquivo hosts](#arquivo-hosts)
  - [Arquivo hosts.ini](#arquivo-hostsini)
  - [Scripts Ansible](#scripts-ansible)
- [Acessar o cluster](#acessar-o-cluster)
- [Próxima etapa: Instalação do Portal](#próxima-etapa-instalação-do-portal)

## Pré-requisitos

Precisa ter instalados o [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html), [Vagrant](https://developer.hashicorp.com/vagrant/docs/installation) e criar uma chave ssh `ssh-keygen -t rsa'.

## Ansible

### Arquivo hosts
```bash
echo "## K8S PPROD ##
192.168.160.20 SSRVA0029
192.168.160.21 SSRVA0030
192.168.160.22 SSRVA0031
192.168.160.23 SSRVA0032
192.168.160.24 SSRVA0033
192.168.160.25 SSRVA0034 > | sudo tee -a /etc/hosts > hosts
```

### Arquivo hosts.ini
```bash
cat > hosts.ini<<EOF

[control_plane]
SSRVA0029 ansible_host=192.168.160.20
SSRVA0030 ansible_host=192.168.160.21
SSRVA0031 ansible_host=192.168.160.22

[workers]
SSRVA0032 ansible_host=192.168.160.23
SSRVA0033 ansible_host=192.168.160.24
SSRVA0034 ansible_host=192.168.160.25

[all:vars]
ansible_python_interpreter=/usr/bin/python3
interface=eth1
cp_endpoint_ip=192.168.160.20
cp_endpoint=SSRVA0029
k8s_version=1.28.2
pod_network_cidr=10.244.0.0/16
service_cidr=172.2.0.0/20
cri_socket=unix:///run/containerd/containerd.sock
cni_plugin=flannel
EOF
```

## Scripts Ansible
```bash
ansible-playbook -i hosts.ini --ssh-common-args='-o StrictHostKeyChecking=no' ./kube-dependencies.yml
ansible-playbook -i hosts.ini --ssh-common-args='-o StrictHostKeyChecking=no' ./control-planes.yml
ansible-playbook -i hosts.ini --ssh-common-args='-o StrictHostKeyChecking=no' ./workers.yml
ansible-playbook -i hosts.ini --ssh-common-args='-o StrictHostKeyChecking=no' ./install-tools.yml
```

## Acessar o cluster
```bash
mkdir $HOME/.kube
scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null support@192.168.160.20:/home/k8sadmin/.kube/config $HOME/.kube
kubectl get nodes
```

## Próxima etapa: Instalação do Portal
Ir para [Instalação do Portal](install-dx-cf214.md).
