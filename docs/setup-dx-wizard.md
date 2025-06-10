# Configuração DX WIZARD com LDAP

O HCL DX Wizard é uma ferramenta projetada para facilitar a configuração e o gerenciamento de servidores e implantações. 

Tabela de conteúdo
==================

- [Conectar no cluster](#conectar-no-cluster)
- [Configurar o Semaphore](#configurar-o-semaphore)
- [Forward](#forward)
- [Wizard](#wizard)
  - [Acessar Console WebSphere Wizard](#acessar-console-websphere-wizard)
  - [Configuração Wizard com LDAP](#configuração-wizard-com-ldap)
  - [Reinicialização Wizard](#reinicialização-wizard)
    - [Parar](#parar)
    - [Iniciar](#iniciar)
  - [Acessar Console WebSphere Wizard](#acessar-console-websphere-wizard)
  - [Configurar Aplicações Wizard](#configurar-aplicações-wizard)

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

## Forward 
```bash
kubectl port-forward svc/dx-core 10203:10203 
```

## Wizard

### Acessar Console WebSphere Wizard
```url
https://localhost:10203/ibm/console
```

### Configuração Wizard com LDAP
```text
SECURITY > GLOBAL SECURITY > CONFIGURE > ADD REPOSITORY (LDAP) > NEW REPOSITORY > LDAP REPOSITORY 

Primary host name: 192.168.160.11
Port: 389
Bind distinguished name (DN): uid=wpadmin,ou=users,DC=BRADSEG,DC=COM,DC=BR
Bind password: <senha>
Base Entry: DC=BRADSEG,DC=COM,DC=BR

OK

PERFORMANCE

Limit search time: 120000
Limit search returns: 500

Enable context pool

Initial size: 10
Preferred size: 40
Maximum size: 40

Cache the attributes
Cache size: 8000

Cache the search results
Cache size: 8000

OK

SAVE

Global security > Federated repositories > prd > Federated repositories entity types to LDAP object classes mapping

Group = groupOfUniqueNames

OK

SAVE

SECURITY > GLOBAL SECURITY > Set as current

SAVE
```
### Reinicialização Wizard

#### Parar
```bash
kubectl exec -it dx-core-0 -- bash -c '/opt/HCL/AppServer/profiles/cw_profile/bin/stopServer.sh server1 -username wpsadmin -password <senha>'
```

#### Iniciar
```bash
kubectl exec -it dx-core-0 -- bash -c '/opt/HCL/AppServer/profiles/cw_profile/bin/startServer.sh server1'
```

### Acessar Console WebSphere Wizard
```url
https://localhost:10203/ibm/console
```

### Configurar Aplicações Wizard

APPLICATIONS > APPLICATIONS TYPE > WEBSPHERE ENTERPRISE APPLICATIONS > WIZARD > SECURITY ROLE TO USER/GROUP MAPPPING 

SELECT WASAdmin

CLICK MAP GROUPS

SEARCH STRING = wpsadmins

OK

SELECT WASAdmin

CLICK MAP SPECIAL SUBJECTS 

SEARCH STRING = All Authenticated in Application's Realm

OK


APPLICATIONS > APPLICATIONS TYPE > WEBSPHERE ENTERPRISE APPLICATIONS > dxconnect > SECURITY ROLE TO USER/GROUP MAPPPING 

SELECT WASAdmin

CLICK MAP GROUPS

SEARCH STRING = wpsadmins

OK

SELECT WASAdmin

CLICK MAP SPECIAL SUBJECTS 

SEARCH STRING = All Authenticated in Application's Realm

OK
```
