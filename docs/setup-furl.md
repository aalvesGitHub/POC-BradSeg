# URLs amigáveis

Antes de iniciar a configuração, tem que ter concluído o [Upgrade - CF221](upgrade-cf221.md).

Tabela de conteúdo
==================

- [Downloads](#downloads)
  - [Baixar o arquivo](#baixar-o-arquivo)
  - [Extrair o binário](#extrair-o-binário)
- [Conectar no cluster](#conectar-no-cluster)
- [Configurar o Semaphore](#configurar-o-semaphore)
- [Wizard](#wizard)
  - [Acessar o Wizard](#acessar-o-wizard)
  - [Configuração do Wizard](#configuração-do-wizard)
- [Custom-values.yaml](#custom-valuesyaml)
  - [Editar custom-values-cf221.yaml](#editar-custom-values-cf221yaml)
- [Próxima etapa: Configuração com OpenSearch](#próxima-etapa-configuração-com-opensearch)

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
https://www-dev.almavivasolutions.com.br/hcl/wizard
```

### Configuração do Wizard
```text
export USER_WPS="wpadmin"
export PASSWORD_WPS="senha_wps"

SET UP A STAND-ALONE SERVER > MODIFY SITE URLS FOR SEARCH ENGINE OPTIMEZATION

System Information

Target operating system: Linux
Target portal profile name: wp_profile
Target portal profile home directory: /opt/HCL/wp_profile

NEXT

Environment Settings

Do you use an external web server: No
Do you use HCL Web Content Manager: yes

NEXT

URL Settings

Do you want your portal URL to contain navigational state information: No
Do you want to modify or remove your context root: Yes
Connect to the portal server to validate settings: Yes

NEXT

Site Information

WebSphere Application Server administrator ID: $USER_WPS
WebSphere Application Server administrator password: $PASSWORD_WPS
Re-enter the password: $PASSWORD_WPS
Digital Experience Portal administrator ID: $USER_WPS
Digital Experience Portal administrator password: $PASSWORD_WPS
Re-enter the password: $PASSWORD_WPS
Context root: 
Default home: 
Personalized home: myweb
WSRP context home: websrp

NEXT

Download Configuration Scripts

kubectl cp ~/Downloads/WorkflowInstanceScriptsAll.zip dx-core-0:/tmp -c core
kubectl exec -it dx-core-0 -c core -- bash
unzip -q /tmp/WorkflowInstanceScriptsAll.zip -d /tmp/scripts
cd /tmp/scripts/scripts

1 Step - Validate the context root values selected.
./ValidateContextRoot.sh

2 Step - Stop the portal server.
./StopPortalServer.sh

3 Step - Modify the servlet paths for all applications and portlets.
./ModifyServletPath.sh

4 Step - Remove navigational state information from your friendly URL.
./RemoveStatelessURI.sh

5 Step - Manual step: Change the JSP components in the Web Resources v70 Library.
Skip Step

6 Step - Optional manual step: If your custom themes use Dojo, update your themes to reference the correct Dojo context root.
Skip Step
Obs: The default Dojo context root in Digital Experience Portal is /wps/portal_dojo. 
WpsContextRoot=null deve atualizar a path para /portal_dojo.

7 Step - Manual step: Refresh your search collection and regather documents.
Skip Step

8 Step - Optional manual step: Update syndicator and subscriber servers that reference your modified site URL. If you do not use syndication, skip this step.
Skip Step

9 Step - Optional manual step: Disable friendly URL redirects.
# Executar manual
Log in to the WebSphere Integrated Solutions Console.
Click Resources > Resource Environment > Resource Environment Providers.
Find the WP ConfigService Resource Environment Provider.
Select WP ConfigService > Custom properties.
Find the property friendly.redirect.enabled and set the value to false If this property does not exist, add it as a new custom property and set the value to false.
Apply the changes and save them to the Master Configuration.

10 Step - Optional manual step: If you are using a custom theme, then run manual steps for enabling hasBaseURL.
# Executar manual no tema customizado
[Enable hasBaseURL in your custom theme](6.1-custom-theme.md)

11 Step - Optional manual step: Update the personalization publishing server with the new site URL.
Skip Step

12 Step - Optional manual step: Redeploy the Web Application Bridge to a virtual host.
Skip Step

13 Step - Restart the Digital Experience Portal server.
./RestartPortalServer.sh

FINISH
```
## Custom-values.yaml
  
### Gerar custom-values.yaml

### Editar custom-values-cf221.yaml
```text
networking:
  core:
    contextRoot: ""
    personalizedHome: "myweb"
    home: ""
```

# Executar o upgrade
```bash
helm upgrade -n dxdev -f custom-values-cf221.yaml dx /tmp/install/images/hcl-dx-deployment-v2.30.0_20240709-2027.tgz
```

## Próxima etapa: Configuração com OpenSearch
Ir para [Próxima etapa: Configuração com OpenSearch](setup-opensearch.md).
