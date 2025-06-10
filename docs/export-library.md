# Exportar biblioteca

Antes de começar, certifique-se que tem instalado o [dxclient cli](docs/dxclient.md).

Tabela de conteúdo
==================

- [Conectar no cluster](#conectar-no-cluster)
- [Configurar o Semaphore](#configurar-o-semaphore)
- [Configurar o WebSphere](#configurar-o-websphere)
- [Reiniciar o POD](#reiniciar-o-pod)
- [Enviar diretório](#enviar-diretório)
- [Exportar](#exportar)

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

## Configurar o WebSphere

https://www-dev.bradseg.com.br/ibm/console

```text
# WCM Authoring
WebSphere Integrated Solutions Console > Resources > Resource Environment > Resource Environment Providers > WCM WCMConfigService > Custom properties Name

export.directory = "diretorio"
```
>**NOTA:**
>
> "diretorio": Diretório com conteúdo da biblioteca para exportação.
>
> Exemplo: /opt/HCL/profiles/prof_95_CF221/PortalServer/wcm/ilwwcm/system/export/avs_internet_content

## Reiniciar o POD
```bash

export USER_WPS="usuario_wps"
export PASSWORD_WPS="senha_wps"

cd /opt/dxclient
bin/dxclient restart-dx-core -hostname dxclient.bradseg.com.br \
  -dxUsername $USER_WPS \
  -dxPassword $PASSWORD_WPS \
  -dxProfileName wp_profile \
  -dxProfilePath /opt/HCL/wp_profile \
  -dxConnectUsername $USER_WPS \
  -dxConnectPassword $PASSWORD_WPS \
  -dxConnectPort 443
```

## Enviar diretório
```bash
kubectl cp "diretorio" dx-core-0:/opt/HCL/wp_profile/PortalServer/wcm/ilwwcm/system/export/"diretorio"
```

>**NOTA:**
>
> "diretorio": Diretório com nome da biblioteca a ser exportada.
>
> Exemplo: /opt/HCL/profiles/prof_95_CF221/PortalServer/wcm/ilwwcm/system/export/avs_internet_content

## Exportar
```bash

export USER_WPS="usuario_wps"
export PASSWORD_WPS="senha_wps"

cd /opt/dxclient
bin/dxclient wcm-library-export \
-hostname dxclient.bradseg.com.br \
-dxUsername $USER_WPS \
-dxPassword $PASSWORD_WPS  \
-dxConnectUsername $USER_WPS \
-dxConnectPassword $PASSWORD_WPS \
-dxConnectPort 443 \
-dxWASUsername $USER_WPS \
-dxWASPassword $PASSWORD_WPS \
-dxProfileName wp_profile \
-librariesName "nome_biblioteca" 
```
>**NOTA:**
>
> "nome_biblioteca": Nome da biblioteca.
>
> Exemplo: /opt/HCL/profiles/prof_95_CF221/PortalServer/wcm/ilwwcm/system/export/avs_internet_content
