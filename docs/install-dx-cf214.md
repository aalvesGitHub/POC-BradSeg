# INSTALAÇÃO DO PORTAL

Antes de iniciar a instalação, tem que ter concluído os [Pré-Requisitos](docs/pre-reqs-portal.md).

Tabela de conteúdo
==================

- [Downloads](#downloads)
  - [Baixar o arquivo](#baixar-o-arquivo)
  - [Extrair o binário](#extrair-o-binário)
- [Conectar no cluster](#conectar-no-cluster)
- [Gitlab portalbradseg](#gitlab-portalbradseg)
- [Images](#images)
  - [Autenticação no docker.io](#autenticação-no-dockerio)
  - [Carrega as imagens](#carrega-as-imagens)
  - [Cria tags](#cria-tags)
  - [Publica imagens](#publica-imagens)
- [Custom-values.yaml](#custom-valuesyaml)
  - [Gerar custom-values.yaml](#gerar-custom-valuesyaml)
  - [Editar custom-values.yaml](#editar-custom-valuesyaml)
- [Helm install](#helm-install)
- [Backup do profile](#backup-do-profile)
- [Próxima etapa: Configuração com DB2](#próxima-etapa-configuração-com-db2)

## Downloads

### Baixar o arquivo
```bash
# HCL Kubernetes for Helm chart
gdown --id 1giTGvjd9Lp9hndjKVuAPLHE_-ZJeMTWG -O ~/Downloads/hcl-dx-kubernetes-v95-CF214.zip
```

>**NOTA:**
>
> Caso não possua os binários, deve realizar os downloads pelo site do fabricante [HCL SoftwareLicensing and Download](https://hclsoftware.flexnetoperations.com/flexnet/operationsportal/logon.do?logoff=true)

### Extrair o binário
```bash
# Criar o diretório
mkdir -p /tmp/install/images

# HCL Kubernetes for Helm chart
unzip -q ~/Downloads/hcl-dx-kubernetes-v95-CF214.zip -d /tmp/install/images
```
>**NOTA:**
>
> O diretório /tmp precisa ter no mínimo 10GB disponível.

## Conectar no cluster
```bash
scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null support@192.168.160.1:/home/support/.kube/config $HOME/.kube
```

## Gitlab portalbradseg
```bash
export USER_GITLAB="usuario_gitlab"
export PASSWORD_GITLAB="senha_gitlab"
git clone http://$USER_GITLAB:$PASSWORD_GITLAB@git.bradseg.com.br/devops/portalbradseg.git
cd portalbradseg
git checkout dev
```

## Images

### Autenticação no docker.io

```bash
export USER_DOCKER="usuario_docker"
docker login docker.io -u $USER_DOCKER
```

### Carrega as imagens
```bash
cd /tmp/install/images
ls -f | grep image | xargs -L 1 docker load -i
```

### Cria tags
```bash
export REMOTE_REPO_PREFIX="dockerbradseg"
docker images | cut -d\/ -f2 | grep -E -v REPOSITORY | awk -F ' ' '{system("docker tag " "dx/"$1 ":" $2 " $REMOTE_REPO_PREFIX/" $1 ":" $2) }'
```

### Publica imagens
```bash
docker images | grep $REMOTE_REPO_PREFIX | awk -F ' ' '{system("docker push " $1 ":" $2)}'
```

## Custom-values.yaml

### Gerar custom-values.yaml
```bash
helm show values /tmp/install/images/hcl-dx-deployment-v2.22.0_20230815-1742.tgz | grep -v \# > custom-values-cf214.yaml
```

### Editar custom-values.yaml
```bash
# Atualizar o repositório
sed -i 's/repository: \"\"/repository: \"dockerbradseg"/g' custom-values-cf214.yaml

export USER_WPS="usuario_wps"
export PASSWORD_WPS="senha_wps"

vim custom-values-cf214.yaml

# Images
names:
  contentComposer: "content-composer"
  core: "core"
  damPluginGoogleVision: "dam-plugin-google-vision"
  digitalAssetManagement: "digital-asset-manager"
  imageProcessor: "image-processor"
  openLdap: "openldap"
  persistenceConnectionPool: "persistence-connection-pool"
  persistenceNode: "persistence-node"
  persistenceMetricsExporter: "persistence-metrics-exporter"
  remoteSearch: "remote-search"
  ringApi: "ringapi"
  runtimeController: "runtime-controller"
  loggingSidecar: "logging-sidecar"
  haproxy: "haproxy"
  damPluginKaltura: "dam-plugin-kaltura"
  licenseManager: "license-manager"
  prereqsChecker: "prereqs-checker"

# Resources
resources:
  contentComposer:
    limits:
      cpu: "100m"
      memory: 192Mi
    requests:
      cpu: 100m
      memory: 192Mi
  core:
    requests:
      cpu: "2000m"
      memory: "4096Mi"
    limits:
      cpu: "4000m"
      memory: "6144Mi"
  digitalAssetManagement:
    limits:
      cpu: "500m"
      memory: 1536Mi
    requests:
      cpu: "500m"
      memory: 1536Mi
  imageProcessor:
    limits:
      cpu: "200m"
      memory: 2048Mi
    requests:
      cpu: "200m"
      memory: 2048Mi
  persistenceConnectionPool:
    limits:
      cpu: "500m"
      memory: 512Mi
    requests:
      cpu: "500m"
      memory: 512Mi
  remoteSearch:
    limits:
      cpu: "500m"
      memory: 2048Mi
    requests:
      cpu: "500m"
      memory: 2048Mi
  runtimeController:
    limits:
      cpu: 100m
      memory: 256Mi
    requests:
      cpu: 100m
      memory: 256Mi
  haproxy:
    requests:
      cpu: "200m"
      memory: "300Mi"
    limits:
      cpu: "200m"
      memory: "300Mi"
  ringApi:
    requests:
      cpu: "300m"
      memory: "256Mi"
    limits:
      cpu: "300m"
      memory: "256Mi"

# Storage
volumes:
  core:
    profile:
      storageClassName: "infra-sc"
      requests:
        storage: "50Gi"

    tranlog:
      storageClassName: "infra-sc"
      requests:
        storage: "50Mi"

    log:
      storageClassName: "infra-sc"
      requests:
        storage: "250Mi"

  digitalAssetManagement:
    binaries:
      storageClassName: "infra-sc"
      requests:
        storage: "2Gi"

  persistenceNode:
    database:
      storageClassName: "infra-sc"
      requests:
        storage: "2Gi"

  licenseManager:
    data:
      storageClassName: "infra-sc"
      requests:
        storage: "2Gi"

  openLdap:
    slapd:
      storageClassName: "infra-sc"
      requests:
        storage: "100Mi"

    certificate:
      storageClassName: "infra-sc"
      requests:
        storage: "10Gi"

    ldap:
      storageClassName: "infra-sc"
      requests:
        storage: "10Gi"

  remoteSearch:
    prsprofile:
      storageClassName: "infra-sc"
      requests:
        storage: "10Gi"

# Applications
applications:
  openLdap: false

# Networking
networking:
  core:
    host: "www-dev.bradsegsolutions.com.br"
    port: "80" 
    ssl: false

# Haproxy
haproxy:
    ssl: false
    serviceType: "ClusterIP"
    tlsCertSecret: "domain-bradseg"

# Security
security:
  core:
    wasUser: $USER_WPS
    wasPassword: $PASSWORD_WPS
    wpsUser: $USER_WPS
    wpsPassword: $PASSWORD_WPS
    configWizardUser: $USER_WPS
    configWizardPassword: $PASSWORD_WPS
  digitalAssetManagement:
    dbUser: "dxuser"
    dbPassword: "d1gitalExperience"
    replicationUser: "repdxuser"
    replicationPassword: "d1gitalExperience"
    damUser: "damuser"
    damPassword: "1234"
  remoteSearch:
      wasUser: $USER_WPS
      wasPassword: $PASSWORD_WPS

 # Configuration:
   containerTimezone: "America/Sao_Paulo"

```
## Helm install
```bash
helm install -n dxdev -f custom-values-cf214.yaml dx /tmp/install/images/hcl-dx-deployment-v2.22.0_20230815-1742.tgz
```

## Backup do profile
```bash
kubectl exec -it dx-core-0 -c core -- bash -c 'mkdir /opt/HCL/profiles/BACKUP'
kubectl exec -it dx-core-0 -c core -- bash -c '/opt/HCL/wp_profile/bin/backupConfig.sh /opt/HCL/profiles/BACKUP/backup-wp-profile-cf214-$(date +%Y%m%d).zip \
  -profileName wp_profile \
  -username $USER_WPS \
  -password $PASSWORD_WPS \
  -nostop'
```

## Próxima etapa: Configuração com DB2
Ir para [Configuração com DB2](setup-db2.md).
