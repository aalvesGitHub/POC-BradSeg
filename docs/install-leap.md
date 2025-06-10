# Instalação do Leap

HCL Leap é uma ferramenta ágil e fácil de usar que permite aos usuários a criação e implantação de aplicações de engajamento com o cliente e formulários. 

Tabela de conteúdo
==================

- [Conecta no servidor](#conecta-no-servidor)
- [Database LEAPDB](#database-leapdb)
- [Downloads](#downloads)
  - [Baixar o arquivo](#baixar-o-arquivo)
  - [Extrair o binário](#extrair-o-binário)
- [Images](#images)
  - [Autenticação no docker.io](#autenticação-no-dockerio)
  - [Carrega as imagens](#carrega-as-imagens)
  - [Cria tags](#cria-tags)
  - [Publica imagens](#publica-imagens)
- [Secrets](#secrets)
- [Custom-values-leap.yaml](#custom-values-leapyaml)
  - [Gerar custom-values-leap.yaml](#gerar-custom-values-leapyaml)
  - [Editar custom-values-leap.yaml](#editar-custom-values-leapyaml)
- [Helm install](#helm-install)
- [Próxima etapa: Configurações do LEAP](#próxima-etapa-configurações-do-leap)

## Conecta no Servidor
```bash
ssh -l support 192.168.160.9

# Tornar usuário dbdev
sudo su - 
su - dbdev
```

## Database LEAPDB
```bash
db2 "CREATE DB LEAPDB using codeset UTF-8 territory us PAGESIZE 32768"
db2 connect to LEAPDB user dbdev using <senha>
db2 "CREATE BUFFERPOOL leappool IMMEDIATE SIZE 250 PAGESIZE 32K"
db2 "CREATE USER TEMPORARY TABLESPACE LARGE_USERTEMP PAGESIZE 32k MANAGED BY AUTOMATIC STORAGE EXTENTSIZE 16 PREFETCHSIZE 16 BUFFERPOOL leappool"
```

## Downloads

### Baixar o arquivo
```bash
# HCL Kubernetes for Helm chart
gdown --id 1WvzZoOXHmSgi7diuJWQNrfl8FeFPmYYB -O ~/Downloads/hcl-leap-pvu-kubernetes-9.3.7.18.zip
```

>**NOTA:**
>
> Caso não possua os binários, deve realizar os downloads pelo site do fabricante [HCL SoftwareLicensing and Download](https://hclsoftware.flexnetoperations.com/flexnet/operationsportal/logon.do?logoff=true)

### Extrair o binário
```bash
# Criar o diretório
mkdir -p /tmp/install/images

# HCL Kubernetes for Helm chart
unzip -q ~/Downloads/hcl-leap-pvu-kubernetes-9.3.7.18.zip -d /tmp/install/images
```
>**NOTA:**
>
> O diretório /tmp precisa ter no mínimo 20GB disponível.

## Images

### Autenticação no docker.io

```bash
export USER_DOCKER="usuario_docker"
docker login docker.io -u $USER_DOCKER
```

### Carrega as imagens
```bash
cd /tmp/install/images
ls -f | grep -E 'leap'| xargs -L 1 docker load -i
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

## Secrets
```bash
# Criar namespace
kubectl create ns leap
# Secret para o administrador do Leap
kubectl create secret generic leap-admin-secret --from-literal=username=leapadmin --from-literal=password=<senha> --namespace=leap 

# Secret para o DB2
kubectl create secret generic leap-db-secret --from-literal=DB_USERNAME=dbdev --from-literal=DB_PASSWORD=<senha> --namespace=leap
```

## Custom-values-leap.yaml

### Gerar custom-values-leap.yaml
```bash
helm show values images/hcl-leap-deployment-v1.1.0_20240717-1648-9.3.7.18.tgz | grep -v \# > custom-values-leap.yaml
```

### Editar custom-values-leap.yaml
```bash
vi custom-values-leap.yaml

images:
  pullPolicy: "IfNotPresent"
  imagePullSecrets:
  repository: "dockerbradseg"
  tags:
    leap: "v1.0.0_20240719-1904"
  names:
    leap: "hcl-leap"

volumes:
  leap:
    rwx:
      storageClassName: "infra-sc"
      requests:
        storage: "10Gi"
      selector:
      volumeName:
    rwo:
      storageClassName: "infra-sc"
      requests:
        storage: "10Gi"
      selector:
      volumeName:
    log:
      storageClassName: "infra-sc"
      requests:
        storage: "5Gi"
      selector:
      volumeName:

security:
  leap:
    adminUser: ""
    adminPassword: ""
    customAdminSecret: "leap-admin-secret"
    saml:
      idpMetadata: ""

configuration:
  leap:
    customSecrets: 
      db-credentials: leap-db-secret
    customCertificateSecrets: {}

    roleMapping:
      AdministrativeUsers:
        MappedUsers:
          - uid=leapadmin,ou=users,DC=BRADSEG,DC=COM,DC=BR
        MappedGroups: []
        MappedUsersAccessIDs: []
        MappedGroupsAccessIDs: []
      SuperAdminUsers:
        MappedUsers:
          - uid=leapadmin,ou=users,DC=BRADSEG,DC=COM,DC=BR
        MappedGroups: 
          - cn=wpadmins,ou=groups,DC=BRADSEG,DC=COM,DC=BR
        MappedUsersAccessIDs: []
        MappedGroupsAccessIDs: []
      EditApplicationsUsers:
        AllAuthenticated: false
        MappedUsers:
          - uid=leapadmin,ou=users,DC=BRADSEG,DC=COM,DC=BR
        MappedGroups: 
          - cn=wpadmins,ou=groups,DC=BRADSEG,DC=COM,DC=BR
        MappedUsersAccessIDs: []
        MappedGroupsAccessIDs: []
      UseApplicationsUsers:
        AllAuthenticated: true
        MappedUsers: []
        MappedGroups: []
        MappedUsersAccessIDs: []
        MappedGroupsAccessIDs: []

    configOverrideFiles: 
      db2Override: |  
        <server description="leapServer"> 
          <!-- Adds the jdbc library to the Leap application classpath -->
          <application autoStart="true" type="ear" id="leap" name="HCL Leap" location="leap.ear">
            <classloader id="leapClassloader" delegation="parentLast" commonLibraryRef="jdbcDB2"/>
          </application>          
          <!-- Disable the hard-coded derby datasource -->
          <dataSource id="leapDerbyDatasource" jndiName="disabled" statementCacheSize="10" />
          <authData id="db2AuthAlias" user="${DB_USERNAME}" password="${DB_PASSWORD}" /> 
          <library id="jdbcDB2" > 
            <fileset dir ="${server.config.dir}/lib" includes="db2jcc4.jar db2jcc_license_cu.jar" /> 
          </library> 
          <dataSource id="febDataSource" jndiName="jdbc/BuilderDataSource" statementCacheSize="30" containerAuthDataRef="db2AuthAlias"> 
            <properties.db2.jcc  
                databaseName="LEAPDB"  
                driverType="4" 
                serverName="192.168.160.9"  
                portNumber="25010" 
                fullyMaterializeLobData="false"  
                progressiveStreaming="2" 
                sslConnection="false" 
                streamBufferSize="2097152"
                isolationLevel="2"
            /> 
            <jdbcDriver libraryRef="jdbcDB2"/> 
            <connectionManager connectionTimeout="180" maxPoolSize="10" minPoolSize="1" reapTime="180" maxIdleTime="1800" agedTimeout="7200" purgePolicy="EntirePool"/> 
          </dataSource> 
        </server>


    configOverrideFiles:
      . . .
      sslOverride: |
         <ssl id="defaultSSLConfig" trustDefaultCerts="true" />

environment:
  pod:
    leap: 
      - name:  JVM_MAX
        value: "-Xmx2048m"

```

## Helm install
```bash
helm install -n leap -f custom-values-leap.yaml hcl images/hcl-leap-deployment-v1.1.0_20240717-1648-9.3.7.18.tgz
```

## Próxima etapa: Configurações do LEAP
Ir para [Próxima etapa: Configurações do LEAP](setup-leap.md)
