# TUNNING no Portal

O tunning neste documento é específico para DX Portal com função Autoria, refere-se ao processo de criação de conteúdo em um ambiente de portal digital.

Tabela de conteúdo
==================

- [Conectar no cluster](#conectar-no-cluster)
- [Configurar o Semaphore](#configurar-o-semaphore)
- [Configurar o WebSphere](#configurar-o-websphere)
- [Reiniciar o POD](#reiniciar-o-pod)

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

user.cache.enable = true
resourceserver.browserCacheMaxAge = 600

# VMM Context Pool
WebSphere Integrated Solutions Console > Security > Global security > Federated repositories > prd > Performance

Initial size = 10
Preferred size = 40
Maximum size = 40
Attributes Cache size = 8000
Search results Cache size = 8000

# JVM Heap Sizes
WebSphere Integrated Solutions Console > Servers > Server Types > WebSphere applications servers > WebSphere_Portal > Java and Process Management > Process definition > Java Virtual Machine 

Initial heap size = 3584
Maximum heap size = 3584
Generic JVM arguments = Add -Xmn1024M

# Data Source
WebSphere Integrated Solutions Console > Resources > JDBC > Data sources > wpdbDS > Connection pools

Maximum connections = 100
Minimum connections = 0
Unused timeout = 600

# Prepared Statement Cache Size
WebSphere Integrated Solutions Console > Resources > JDBC > Data sources > wpdbDS > WebSphere Application Server data source properties 

Statement cache size = 15

# Web Container Thread Pool Size
WebSphere Integrated Solutions Console > Servers > Server Types > WebSphere applications servers > WebSphere_Portal > Thread pools

WebContainer Minimum Size = 60
WebContainer Maximum Size = 60

# WCM Object Cache
WebSphere Integrated Solutions Console > Resources > Cache instances > Object cache instances 

abspath cache size = 8000
abspathreverse cache size = 8000
processing cache size = 10000
session cache size = 6000
strategy cache size = 8000
summary cache size = 2000

# Cache Manager Service
WebSphere Integrated Solutions Console > Resources > Resource Environment > Resource Environment Providers > WP CacheManagerService > Custom properties Name

cacheinstance.com.ibm.wps.ac.CommonRolesCache.size = 50000
cacheinstance.com.ibm.wps.ac.ProtectedResourceCache.size = 20000
cacheinstance.com.ibm.wps.cp.models.ModelCache.CategoryModel.lifetime = 28800
cacheinstance.com.ibm.wps.cp.models.ModelCache.ResourceModel.lifetime = 28800
cacheinstance.com.ibm.wps.cp.models.ModelCache.ResourceModel.size = 2000
cacheinstance.com.ibm.wps.cp.models.ModelCache.TagModel.lifetime = 28800
cacheinstance.com.ibm.wps.cp.models.ModelCache.TagModel.size = 2000
cacheinstance.com.ibm.wps.pe.portletentitycounter.size = 5000
cacheinstance.com.ibm.wps.resolver.resource.AbstractRequestDispatcherFactory.size = 200

# SESSIONID
Servers > Server Types > WebSphere applications servers > WebSphere_Portal > Session management > custom properties

name: HttpSessionIdLength
value: 128

OK
SAVE
```

## Reiniciar o POD
```bash
kubectl -ndxdev rollout restart sts dx-core
```