# Configuração com LDAP

Antes de iniciar a configuração, tem que ter concluído à [Instalação do LDAP](install-ldap.md).

Tabela de conteúdo
==================

- [Conectar no cluster](#conectar-no-cluster)
- [Configurar o Semaphore](#configurar-o-semaphore)
- [Wizard](#wizard)
  - [Acessar o Wizard](#acessar-o-wizard)
  - [Configuração do Wizard](#configuração-do-wizard)
- [Backup do profile](#backup-do-profile)
- [Próxima etapa: Upgrade - CF221](#próxima-etapa-upgrade---cf221)

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
## Wizard

### Acessar o Wizard
```text
https://www-dev.bradseg.com.br/hcl/wizard
```

### Configuração do Wizard
```text
export DC_LDAP="DC=bradseg,DC=COM,DC=BR"
export USER_LDAP="uid=wpadmin,ou=users,DC=bradseg,DC=COM,DC=BR"
export PASSWORD_LDAP="senha_ldap"
export GROUP_LDAP="cn=wpadmins,ou=groups,DC=bradseg,DC=COM,DC=BR"
export NAME_LDAP="prd"
export HOST_LDAP="192.168.160.11"
export PORT_LDAP="389"
export USER_WPS="wpsadmin"
export PASSWORD_WPS="senha_wps"


SET UP A STAND-ALONE SERVER > ENABLE FEDERATED SECURITY

System Information

Target operating system: Linux
Target portal profile name: wp_profile
Target portal profile home directory: /opt/HCL/wp_profile

NEXT

Security Settings

User registry sofware: IBM Directory Server
Do you need SSL between the portal server and the user registry: No, do not enable SSL
Can portal update entries in your LDAP user registry: Yes, portal can create, update, and delete entries
Use Administrator IDs stored in your LDAP user registry: Yes, update the IDs using the new LDAP user registry
Validate LDAP user registry entries: Yes

NEXT

Existing Administrator Information

WebSphere Application Server administrator ID: $USER_WPS
WebSphere Application Server administrator password: $PASSWORD_WPS
Re-enter the password: $PASSWORD_WPS
Digital Experience Portal administrator ID: $USER_WPS
Digital Experience Portal administrator password: $USER_WPS
Re-enter the password: $PASSWORD_WPS

NEXT

User Registry Information

LDAP Repository ID: $NAME_LDAP
LDAP host name: $HOST_LDAP
LDAP port: $PORT_LDAP

NEXT

User Registry Credentials

Bind DN: $USER_LDAP
Bind password: $PASSWORD_LDAP

NEXT

Detailed User Registry Information

Base DN: $DC_LDAP
Administrator group DN from LDAP: $GROUP_LDAP
Administrator DN from LDAP: $USER_LDAP
Administrator password from LDAP: $PASSWORD_LDAP
Default parent for group: 
Default parent for PersonAccount: 

NEXT

Download Configuration Scripts

kubectl cp WorkflowInstanceScriptsAll.zip dx-core-0:/tmp -c core
kubectl exec -it dx-core-0 -c core -- bash
unzip -q /tmp/WorkflowInstanceScriptsAll.zip -d /tmp/scripts
cd /tmp/scripts/scripts

1 Step - Create a backup of the Digital Experience Portal profile before modifying the cell security.
./BackupWebSpherePortalProfile.sh

2 Step - Validate your LDAP server settings.
./ValidateFederatedLDAP.sh

3 Step - Add an LDAP user registry to the existing federated repository.
./EnableFederatedLDAPSecurity.sh

4 Step - Register the WebSphere Application Server scheduler tasks.
./ReregisterSchedulerTasks.sh

5 Step - Replace the existing Digital Experience Portal and WebSphere Application Server users and groups with new users and groups from your LDAP server.
./ChangeWASAdminUser.sh

6 Step - Update the user registry where new users and groups are stored.
./SetEntityTypes.sh

7 Step - Recycle the servers after a security change.
./RecycleAfterSecurityChangeFirst.sh

8 Step - Update the search administration user.
./UpdateSearchAdminUser.sh

9 Step - After you change the security model, the servers need to be restarted. Restart the portal server.
./RecycleAfterSecurityChange.sh

10 Step -  	
Verify that all defined attributes are available in the configured LDAP user registry.
./ValidateFederatedLDAPAttributes.sh

11 Step - Manual Step: Update the MemberFixerModule.properties file with the values for your LDAP users.

 # Add linha no arquivo
 echo "uid=wpsadmin,o=defaultWIMFileBasedRealm -> uid=wpadmin,ou=users,DC=bradseg,DC=COM,DC=BR" | tee -a /opt/HCL/wp_profile/PortalServer/wcm/shared/app/config/wcmservices/MemberFixerModule.properties

12 Step - Run the member fixer tool.
./RunWcmAdminTaskMemberFixer.sh

13 Step - Restart the Digital Experience Portal server.
./RestartPortalServer.sh

14 Step - Manual Step: Map attributes to ensure proper communication between Digital Experience Portal and the LDAP server.
Skip Step

FINISH

```
## Backup do profile
```bash
kubectl exec -it dx-core-0 -c core -- bash -c '/opt/HCL/wp_profile/bin/backupConfig.sh /opt/HCL/profiles/BACKUP/backup-wp-profile-cf214-ldap-$(date +%Y%m%d).zip \
  -profileName wp_profile \
  -username $USER_WPS \
  -password $PASSWORD_WPS \
  -nostop'
```

## Próxima etapa: Upgrade - CF221
Ir para [Upgrade - CF221](upgrade-cf221.md).
