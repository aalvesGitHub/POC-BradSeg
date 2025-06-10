# Upgrade com CF221

Antes de executar o upgrade, deve realizar algumas configurações antes para evitar erros inesperados na versão CF221.

* [Instalação do Portal](docs/install-dx-cf214.md), com versão mínima CF214.
* [Configuração do DB2](docs/setup-db2.md)
* [Configuração do LDAP](docs/setup-ldap.md)


Tabela de conteúdo
==================

- [Downloads](#downloads)
  - [Baixar o arquivo](#baixar-o-arquivo)
  - [Extrair o binário](#extrair-o-binário)
- [Conecta no cluster](#conecta-no-cluster)
- [Backup](#backup)
  - [DAM/Postgresql](#dampostgresql)
    - [Export DAM collection](#export-dam-collection)
    - [DAM Persistence](#dam-persistence)
    - [DAM Binaries](#dam-binaries)
  - [DB2](#db2)
    - [Procimento de backup](#procimento-de-backup)
      - [Carregar as variáveis](#carregar-as-variáveis)
      - [Executar backup](#executar-backup)
- [Cumulative Fix Health Checker](#cumulative-fix-health-checker)
- [Gitlab portalbradseg](#gitlab-portalbradseg)
- [Images](#images)
  - [Autenticação no docker.io](#autenticação-no-dockerio)
  - [Carrega as imagens](#carrega-as-imagens)
  - [Cria tags](#cria-tags)
  - [Publica imagens](#publica-imagens)
- [Custom-values.yaml](#custom-valuesyaml)
  - [Gerar custom-values.yaml](#gerar-custom-valuesyaml)
  - [Editar custom-values.yaml](#editar-custom-valuesyaml)
- [Helm upgrade](#helm-upgrade)
- [Backup do profile](#backup-do-profile)
- [Próxima etapa: Configuração com URLs amigáveis](#próxima-etapa-configuração-com-urls-amigáveis)


## Downloads

### Baixar o arquivo
```bash
# HCL Kubernetes for Helm chart
gdown --id 1uh5NTP45W_LxjLtYVVspb4DAFroMFMdY -O ~/Downloads/hcl-dx-kubernetes-v95-CF221.zip
```

>**NOTA:**
>
> Caso não possua os binários, deve realizar os downloads pelo site do fabricante [HCL SoftwareLicensing and Download](https://hclsoftware.flexnetoperations.com/flexnet/operationsportal/logon.do?logoff=true)

### Extrair o binário
```bash
# Criar o diretório
mkdir -p /tmp/install/images

# HCL Kubernetes for Helm chart
unzip -q ~/Downloads/hcl-dx-kubernetes-v95-CF221.zip -d /tmp/install/images
```
>**NOTA:**
>
> O diretório /tmp precisa ter no mínimo 20GB disponível.

## Conecta no cluster
```bash
scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null support@192.168.160.1:/home/support/.kube/config $HOME/.kube
```

## Backup

### DAM/Postgresql

#### Export DAM collection
```bash
export USER_WPS="usuario_wps"
export PASSWORD_WPS="senha_wps"

kubectl -n dxdev exec pod/dx-core-0 -c core -- /bin/bash -c "/opt/HCL/PortalServer/bin/xmlaccess.sh -user $USER_WPS -password $PASSWORD_WPS -url http://localhost:10039/wps/config -in /opt/HCL/PortalServer/doc/xml-samples/ExportAllDAMCollections.xml -out /tmp/damExport.xml"
kubectl -ndxdev cp -c core dx-core-0:/tmp/damExport.xml /tmp/damExport.xml
```

#### DAM Persistence
```bash
kubectl -n dxdev exec pod/dx-persistence-node-0 -c persistence-node -- repmgr cluster show --compact --terse 2>/dev/null | grep "primary" | awk '{split($0,a,"|"); print a[2]}' | xargs
kubectl -n dxdev exec pod/dx-persistence-node-0 -c persistence-node -- /bin/bash -c "pg_dump dxmediadb > /tmp/dxmediadb.dmp"
kubectl -ndxdev cp -c persistence-node dx-persistence-node-0:/tmp/dxmediadb.dmp /tmp/dxmediadb.dmp
```

#### DAM Binaries
```bash
kubectl -n dxdev exec pod/dx-digital-asset-management-0 -- /bin/bash -c "tar -cvpzf /tmp/backupml.tar.gz --one-file-system --directory /opt/app/upload ."
kubectl -n dxdev cp dx-digital-asset-management-0:/tmp/backupml.tar.gz /tmp/backupml.tar.gz
```

## DB2

### Procimento de backup

#### Carregar as variáveis
```bash
export USER_DB="usuario_banco"
export PASSWORD_DB="senha_banco"
export NAME_DB="nome_banco"
```

#### Executar backup
```bash
# Tornar usuário do banco
su - $USER_DB

# Carrega o db2profile
source sqllib/db2profile

# Cria o diretório armazena o backup
mkdir /db2/dbdev/BACKUP

# Conectar no database NAME_DB
db2 connect to $NAME_DB user USER_DB using $PASSWORD_DB

# Lista os processos com database NAME_DB
db2 list applications

# Força a parada das conexões
db2 force application all

# Para o database NAME_DB
db2stop force

# Limpa todos os processos da instância NAME_DB
ipclean -a

# Desabilita o protocolo TCPIP na comunicação do database NAME_DB
db2set -null DB2COMM

# Inicializa o database NAME_DB com acesso restrito
db2start admin mode restricted access

# Gera o backup do database NAME_DB
db2 backup db $NAME_DB on all dbpartitionnums to /db2/$USER_DB/BACKUP

# Para o database NAME_DB
db2stop force

# Limpa todos os processos da instância NAME_DB
ipclean -a

# Configura o protocolo TCPIP na comunicação do database NAME_DB
db2set DB2COMM=TCPIP

# InInicializa o database NAME_DB
db2start

# Conectar no database NAME_DB
db2 connect to $NAME_DB user $USER_DB using $PASSWORD_DB

# Lista os processos com database NAME_DB
db2 list applications
```

## Cumulative Fix Health Checker
```bash
kubectl exec -it dx-core-0 -- bash

cd /opt/HCL/wp_profile/ConfigEngine/properties
vi wkplc_dbdomain.properties
---
feedback.DbPassword=$PASSWORD_DB
feedback.DBA.DbPassword=$PASSWORD_DB
likeminds.DbPassword=$PASSWORD_DB
likeminds.DBA.DbPassword=$PASSWORD_DB
release.DbPassword=$PASSWORD_DB
release.DBA.DbPassword=$PASSWORD_DB
community.DbPassword=$PASSWORD_DB
community.DBA.DbPassword=$PASSWORD_DB
customization.DbPassword=$PASSWORD_DB
customization.DBA.DbPassword=$PASSWORD_DB
jcr.DbPassword=$PASSWORD_DB
jcr.DBA.DbPassword=$PASSWORD_DB
----

vi wkplc.properties
---
WasPassword=$PASSWORD_WPS
PortalAdminPwd=$PASSWORD_WPS
PWordDelete=false
---

/opt/HCL/wp_profile/ConfigEngine/ConfigEngine.sh health-check-update
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
helm show values /tmp/install/images/hcl-dx-deployment-v2.30.0_20240709-2027.tgz | grep -v \# > custom-values-cf221.yaml
```

### Editar custom-values.yaml
```bash
# Atualizar o repositório
cd portalbradseg
sed -i 's/repository: \"\"/repository: \"dockerbradseg"/g' custom-values-cf221.yaml

export USER_WPS="usuario_wps"
export PASSWORD_WPS="senha_wps"

vim custom-values-cf221.yaml

# Images:
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

#replicas:
  contentComposer: 1
  core: 1
  damPluginGoogleVision: 1
  damPluginKaltura: 1
  digitalAssetManagement: 1
  haproxy: 1
  imageProcessor: 1
  licenseManager: 1
  persistenceConnectionPool: 1
  persistenceNode: 1
  ringApi: 1

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

# Security
security:
  core:
    wasUser: $USER_WPS
    wasPassword: $PASSWORD_WPS
    wpsUser: $USER_WPS
    wpsPassword: $PASSWORD_WPS
    configWizardUser: "wpsadmin"
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

## Helm upgrade
```bash
helm upgrade dx -n dxdev -f custom-values-cf221.yaml /tmp/install/images/hcl-dx-deployment-v2.30.0_20240709-2027.tgz
```

## Backup do profile
```bash
kubectl exec -it dx-core-0 -c core -- bash -c '/opt/HCL/wp_profile/bin/backupConfig.sh /opt/HCL/profiles/BACKUP/backup-wp-profile-db2-ldap-cf221-$(date +%Y%m%d).zip \
  -profileName wp_profile \
  -username $USER_WPS \
  -password $PASSWORD_WPS \
  -nostop'
```

## Próxima etapa: Configuração com URLs amigáveis
Ir para [Configuração com URLs amigáveis](docs/setup-furl.md).