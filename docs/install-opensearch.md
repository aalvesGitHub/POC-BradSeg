# Instalação do OpenSearch
Antes de iniciar a instalação, tem que ter concluído o [Upgrade com CF221](docs/upgrade-cf221.md).

Tabela de conteúdo
==================

- [Downloads](#downloads)
  - [Baixar o arquivo](#baixar-o-arquivo)
  - [Extrair o binário](#extrair-o-binário)
- [Images](#images)
  - [Autenticação no docker.io](#autenticação-no-dockerio)
  - [Carrega as imagens](#carrega-as-imagens)
  - [Cria tags](#cria-tags)
  - [Publica imagens](#publica-imagens)
- [Certificados](#certificados)
- [Custom-values-opensearch.yaml](#custom-values-opensearchyaml)
  - [Gerar custom-values-opensearch.yaml](#gerar-custom-values-opensearchyaml)
  - [Editar custom-values-opensearch.yaml](#editar-custom-values-opensearchyaml)
- [Helm install](#helm-install)
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

## Images

### Autenticação no docker.io

```bash
export USER_DOCKER="usuario_docker"
docker login docker.io -u $USER_DOCKER
```

### Carrega as imagens
```bash
cd /tmp/install/images
ls -f | grep -E 'opensearch|middleware'| xargs -L 1 docker load -i
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

## certificados
```bash
# Root CA for certificates
openssl genrsa -out root-ca-key.pem 2048
openssl req -new -x509 -sha256 -key root-ca-key.pem -subj "/C=US/O=ORG/OU=UNIT/CN=opensearch" -out root-ca.pem -days 3650

# Admin cert for OpenSearch configuration
openssl genrsa -out admin-key-temp.pem 2048
openssl pkcs8 -inform PEM -outform PEM -in admin-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out admin-key.pem
openssl req -new -key admin-key.pem -subj "/C=US/O=ORG/OU=UNIT/CN=A" -out admin.csr
openssl x509 -req -in admin.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -sha256 -out admin.pem -days 3650

# Node cert for inter node communication
openssl genrsa -out node-key-temp.pem 2048
openssl pkcs8 -inform PEM -outform PEM -in node-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out node-key.pem
openssl req -new -key node-key.pem -subj "/C=US/O=ORG/OU=UNIT/CN=opensearch-node" -out node.csr
openssl x509 -req -in node.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -sha256 -out node.pem -days 3650

# Client cert for application authentication
openssl genrsa -out client-key-temp.pem 2048
openssl pkcs8 -inform PEM -outform PEM -in client-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out client-key.pem
openssl req -new -key client-key.pem -subj "/C=US/O=ORG/OU=UNIT/CN=opensearch-client" -out client.csr
openssl x509 -req -in client.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -sha256 -out client.pem -days 3650

# Create kubernetes secrets
kubectl create secret generic search-admin-cert --from-file=admin.pem --from-file=admin-key.pem --from-file=root-ca.pem -n dxdev
kubectl create secret generic search-node-cert --from-file=node.pem --from-file=node-key.pem --from-file=root-ca.pem -n dxdev
kubectl create secret generic search-client-cert --from-file=client.pem --from-file=client-key.pem --from-file=root-ca.pem -n dxdev
```

## Custom-values-opensearch.yaml

### Gerar custom-values-opensearch.yaml
```bash
helm show values hcl-dx-search-v2.19.0_20240708-2015.tgz | grep -v \# > custom-values-opensearch.yaml
```

### Editar custom-values-opensearch.yaml
```bash

images:
  repository: "dockerbradseg"
  tags:
    openSearch: "v95_CF221_20240708-1958"
    searchMiddleware: "v2.0.0_20240708-0052"
  names:
    openSearch: "opensearch"
    searchMiddleware: "search-middleware"

volumes:
  openSearchManager:
    data:
      storageClassName: "infra-sc"
      requests:
        storage: "10Gi"

  openSearchData:
    data:
      storageClassName: "infra-sc"
      requests:
        storage: "10Gi"
```

## Helm install
```bash
helm install -n dxdev -f custom-values-opensearch.yaml dx-search hcl-dx-search-v2.19.0_20240708-2015.tgz
```

## Próxima etapa: Configuração com OpenSearch
Ir para [Próxima etapa: Configuração com OpenSearch](docs/setup-opensearch.md)
