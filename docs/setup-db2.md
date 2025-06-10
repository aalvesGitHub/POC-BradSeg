# Configuração com DB2

Antes de iniciar a configuração, tem que ter concluído à [Instalação do DB2](install-db2.md).

Tabela de conteúdo
==================

- [Conectar no cluster](#conectar-no-cluster)
- [Configurar o Semaphore](#configurar-o-semaphore)
- [DB2 Driver](#db2-driver)
- [Wizard](#wizard)
  - [Acessar o Wizard](#acessar-o-wizard)
  - [Configuração do Wizard](#configuração-do-wizard)
- [Backup do profile](#backup-do-profile)
- [Próxima etapa: Configuração com LDAP](#próxima-etapa-configuração-com-ldap)

 
## Conectar no cluster
```bash
scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null support@192.168.160.1:/home/support/.kube/config $HOME/.kube
```

## Configurar o Semaphore
```bash
# Conecta no pod dx-core
kubectl exec -it dx-core-0 -- bash

# Edita o readiness
vi /opt/app/entrypoints/readiness.sh

CONFIG_SEMAPHORE_FILE="/opt/app/configInProgress"
probeURL=$1

# Check for existence of the semaphore file
if [ -f $CONFIG_SEMAPHORE_FILE ]
then
    # If the file exists, the pod is doing a config task and therefore not ready
    exit 0 ## Altera o primeiro exit 1 para 0
else
    # If the pod is not doing a config task, test whether the server is up
    /opt/app/checkPortalUp.sh $probeURL
    if [ "$?" != "0" ]; then
        exit 1
    fi
    exit 0
fi

# Cria o arquivo 
touch /opt/app/configInProgress
```

## DB2 Driver
```bash
# Criar o diretório
kubectl -ndxdev exec -it dx-core-0 -c core -- bash -c 'mkdir -p /opt/HCL/wp_profile/PortalServer/lib/ext'

# Copia o arquivo db2jcc4.jar
kubectl -ndxdev cp lib/db2jcc4.jar dx-core-0:/opt/HCL/wp_profile/PortalServer/lib/ext/db2jcc4.jar -c core

# Copia o arquivo db2jcc_license_cu.jar
kubectl -ndxdev cp lib/db2jcc_license_cu.jar dx-core-0:/opt/HCL/wp_profile/PortalServer/lib/ext/db2jcc_license_cu.jar -c core
```

## Wizard

### Acessar o Wizard
```text
https://www-dev.bradesg.com.br/hcl/wizard
```

### Configuração do Wizard
```text
export USER_DB="usuario_banco"
export PASSWORD_DB="senha_banco"
export NAME_DB="nome_banco"
export HOST_DB="host_banco"
export PORT_DB="port_banco"
export USER_WPS="usuario_wps"
export PASSWORD_WPS="senha_wps"

SET UP A STAND-ALONE SERVER > DATABASE TRANSFER

Database management software: DB2
Do you want to transfer to one database or multiple databases or schemas: One Database
Do you want the wizard to create your databases: No
Is the database hosted on the same server as the portal: No
Do you want the wizard to create users and assign them permission: Yes, create my users and assign them appropriate permission
Do you need advanced database collation support: No
For DB2 PureScale only): Do you want to enable workload balancing for DB2 pureScale: No
Connect to database server to validate settings: Yes

NEXT

Do you need runtime database user ID for day-to-day operations: No

NEXT

INFO DATABASE

Release database name: $NAME_DB
Host name: $HOST_DB
Port number: $PORT_DB

NEXT

WebSphere Application Server administrator ID: $USER_WPS
WebSphere Application Server administrator password: $PASSWORD_WPS
Re-enter the password: $PASSWORD_WPS

NEXT

Data source: wpdbDS
Database URL: jdbc:db2://$HOST_DB:$PORT_DB/$NAME_DB
Configuration user ID: $USER_DB
Configuration password: $PASSWORD_DB
Re-enter the password: $PASSWORD_DB
Database administrator ID: $USER_DB
Database administrator password: $PASSWORD_DB
Re-enter the password: $PASSWORD_DB
IBM DB2 Library: /opt/HCL/wp_profile/PortalServer/lib/ext/db2jcc4.jar:/opt/HCL/wp_profile/PortalServer/lib/ext/db2jcc_license_cu.jar

NEXT 

Step 1 - Manual Step: Create the database users and groups 
Mark Step Complete

Step 2 - Back up the properties files that the wizard uses during the configuration
Run Step

Step 3 - Manual Step: Download the script and run it on the database server to create your database
Clique Download Script

# Copiar scripts para o DB2
rsync -avl WorkflowInstanceScriptsStep3.zip support@$HOST_DB:/tmp
su - dbdev
unzip -q /install/WorkflowInstanceScriptsStep3.zip -d /tmp/scripts
cd /tmp/scripts/scritps
./CreateDB2Database

Mark Step Complete

Step 4 - Set up your database.
Run Step

Step 5 - Stop the portal server
Run Step

Step 6 - Manual Step: Restart the db2 server
db2stop
db2start

Mark Step Complete

Step 7 - Validate the database connection and environment
Run Step

Step 8 - Transfer the database
Run Step

Step 9 - Configure the JCR domain to support large files
Run Step

Step 10 - Start the portal server
Run Step
```

## Backup do profile
```bash
kubectl exec -it dx-core-0 -c core -- bash -c '/opt/HCL/wp_profile/bin/backupConfig.sh /opt/HCL/profiles/BACKUP/backup-wp-profile-cf214-db2-$(date +%Y%m%d).zip \
  -profileName wp_profile \
  -username $USER_WPS \
  -password $PASSWORD_DB \
  -nostop'
```

## Próxima etapa: Configuração com LDAP
Ir para [Próxima etapa: Configuração com LDAP](setup-ldap.md).
