# PORTAL bradseg

Projeto responsável por proviosionar toda a documentação técnica referente a instalação e configuração da solução HCL DX Portal 9.5.

Tabela de conteúdo
==================

- [Criação do cluster](docs/create-k8s.md)
- [Instalação do Portal](docs/install-dx-cf214.md)
  - [Pré-Requisitos](docs/pre-reqs-portal.md) 
- [Configurações](docs/configurações.md)
- [Procedimentos](docs/procedimentos.md)
- [Arquitetura de Rede](#arquitetura-de-rede)


## INFORMAÇÕES DO PORTAL
| Nome               | Descrição                                                |
|--------------------|----------------------------------------------------------|
| Portal Versão      | HCL DX Portal 9.5                                        |
| URL Portal Internet| https://www-dev.bradseg.com.br                 |
| URL Portal Intranet| https://intranet-dev.bradseg.com.br            |
| URL Leap           | https://form-dev.bradseg.com.br                |
| URL API K8S        | https://k8.bradseg.com.br:6443             |
| URL Console WAS    | https://www-dev.bradseg.com.br/ibm/console     |
| URL Console WIZARD | https://www-dev.bradseg.com.br/hcl/wizard      |
| URL Console K8S    | https://console-k8s-dev.bradseg.com.br         |

## SERVIDORES

  | Hostname   | Descrição | OS | CPU | MEM | DISCO | IP                                  |
  |------------|-----------|----|-----|-----|-------|-------------------------------------|
  | bradseg3 | K8S MASTER | UBUNTU 22.04 LTS | 4  | 16 | 1x100 GB       | 192.168.160.1 |
  | bradseg4 | K8S MASTER | UBUNTU 22.04 LTS | 4  | 16 | 1x100 GB       | 192.168.160.2 |
  | bradseg5 | K8S MASTER | UBUNTU 22.04 LTS | 4  | 16 | 1x100 GB       | 192.168.160.3 |
  | bradseg6 | K8S WORKER | UBUNTU 22.04 LTS | 16 | 32 | 1x100 GB       | 192.168.160.4 |
  | bradseg7 | K8S WORKER | UBUNTU 22.04 LTS | 16 | 32 | 1x100 GB       | 192.168.160.5 |
  | bradseg8 | K8S WORKER | UBUNTU 22.04 LTS | 16 | 32 | 1x100 GB       | 192.168.160.6 |
  | bradsegldap | LDAP       | CentOS Stream 9  | 4  | 8  | 1x100 GB       | 192.168.160.11|
  | bradseg2 | NFS        | UBUNTU 22.04 LTS | 2  | 4  | 1x120 GB       | 192.168.160.8 |
  | bradseg1 | DB2        | CentOS Stream 8  | 4  | 16 | 1x100/1x150 GB | 192.168.160.9 |
