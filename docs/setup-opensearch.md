# Configuração com OpenSearch
Antes de iniciar a configuração, tem que ter concluído à [Instalação do OpenSearch](docs/install-opensearch.md).

Tabela de conteúdo
==================

- [Downloads](#downloads)
  - [Baixar o arquivo](#baixar-o-arquivo)
  - [Extrair o binário](#extrair-o-binário)
- [Custom-values-cf221.yaml](#custom-values-cf221yaml)
  - [Editar custom-values-cf221.yaml](#editar-custom-values-cf221yaml)
- [Helm upgrade](#helm-upgrade)
- [Validação](#validação)
  - [POD](#pod)
  - [API](#api)

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

## Custom-values-cf221.yaml
  
### Editar custom-values-cf221.yaml
```bash
applications:
  remoteSearch: false

haproxy:
    ssl: false
    serviceType: "ClusterIP"
    searchMiddlewareService: "dx-search-search-middleware-query"
```

## Helm upgrade
```bash
helm upgrade dx -n dxdev -f custom-values-cf221.yaml /tmp/install/images/hcl-dx-deployment-v2.30.0_20240709-2027.tgz
```

## Validação

### POD
```bash
kubectl get pods -n dxdev |grep 'search'
NAME                                                 READY   STATUS    RESTARTS        AGE
dx-search-open-search-manager-0                      1/1     Running   1 (40h ago)     2d16h
dx-search-search-middleware-query-6b85cc8684-r5d22   1/1     Running   0               2d16h
```

### API
```bash
http://www-dev.bradseg.com.br/dx/api/search/v2/explorer
```