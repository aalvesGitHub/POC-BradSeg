# Configuração do Leap

Antes de iniciar a configuração, tem que ter concluído à [Instalação do Leap](docs/install-leap.md).

Tabela de conteúdo
==================

- [Popular banco](#popular-banco)
  - [Forward](#forward)
  - [Acesso da console](#acesso-da-console)
  - [Tunning no banco](#tunning-no-banco)
      - [Acessar o servidor](#acessar-o-servidor)
      - [Execute os comandos](#execute-os-comandos)
- [Ingress](#ingress)
- [URLs](#urls)
- [Configuração com LDAP](#configuração-com-ldap)
  - [Editar custom-values-leap.yaml](#editar-custom-values-leapyaml)
- [Helm upgrade](#helm-upgrade)

## Popular banco

### Forward
```bash
kubectl port-forward svc/hcl-leap 9443:9443 -n leap
```

### Acesso da console
```text
https://localhosts:9443/apps

 - Ao abrir a pagina
 - Clique no botão CORRIGIR
 - Aguarde a finalização da população do banco
 - Clique no botão Continuar
 - Depois feche a pagina
```

###  Tunning no banco

#### Acessar o servidor
```bash
ssh -l support 192.168.160.9

# tornar usuário dbdev
sudo su - 
su - dbdev
```

#### Execute os comandos
```bash
export PASSWORD_DB="senha do banco"
db2 connect to LEAPDB user dbdev using $PASSWORD_DB
db2 "CREATE BUFFERPOOL FEB4KBP IMMEDIATE SIZE 250 AUTOMATIC PAGESIZE 4 K"
db2 "CREATE LARGE TABLESPACE USERSPACE4K PAGESIZE 4 K MANAGED BY AUTOMATIC STORAGE BUFFERPOOL FEB4KBP"
db2 "CREATE BUFFERPOOL FEB8KBP IMMEDIATE SIZE 250 AUTOMATIC PAGESIZE 8 K"
db2 "CREATE LARGE TABLESPACE USERSPACE8K PAGESIZE 8 K MANAGED BY AUTOMATIC STORAGE BUFFERPOOL FEB8KBP"
db2 "CREATE BUFFERPOOL FEB16KBP IMMEDIATE SIZE 250 AUTOMATIC PAGESIZE 16 K"
db2 "CREATE LARGE TABLESPACE USERSPACE16K PAGESIZE 16 K MANAGED BY AUTOMATIC STORAGE BUFFERPOOL FEB16KBP"
```

## Ingress
```bash
kubectl apply -f -<<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hcl-leap
  namespace: "leap"
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: 100m
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "120"
    nginx.ingress.kubernetes.io/app-root: /apps
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - form-dev.bradseg.com.br
  rules:
  - host: form-dev.bradseg.com.br
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hcl-leap
            port:
              number: 9080
EOF
```

## URLs
| Nome            | Descrição                                                |
|-----------------|----------------------------------------------------------|
| URL Leap        | http://form-dev.bradseg.com.br/apps            |
| URL Rest API    | http://form-dev.bradseg.com.br/apps-basic      |
| URL OpenLibertyt| http://form-dev.bradseg.com.br/adminCenter     |

## Configuração com LDAP

### Editar custom-values-leap.yaml
```bash
vi custom-values-leap.yaml

#### Configuração LDAP
    configOverrideFiles:
      ldapOverride: | 
        <server description="leapServer"> 
          <federatedRepository id="leapRepo"> 
            <primaryRealm name="FEDREALM"> 
              <participatingBaseEntry name="DC=BRADSEG,DC=COM,DC=BR"/> 
              <userSecurityNameMapping outputProperty="uid" /> 
              <groupSecurityNameMapping outputProperty="cn" /> 
            </primaryRealm> 
          </federatedRepository>
            <ldapRegistry id="OpenLdap" 
              name="DC=BRADSEG,DC=COM,DC=BR" 
              ldapType="Custom" 
              host="192.168.160.11" port="389" 
              baseDN="DC=BRADSEG,DC=COM,DC=BR" 
              searchTimeout="8m" 
              ignoreCase="true" 
              bindDN="uid=wpadmin,ou=users,DC=BRADSEG,DC=COM,DC=BR" 
              bindPassword="<senha>" 
              sslEnabled="false" 
              derefAliases="never"> 
                <loginProperty name="uid"></loginProperty> 
                <ldapEntityType name="PersonAccount"> 
                  <objectClass>inetOrgPerson</objectClass> 
                  <searchBase>ou=users,DC=BRADSEG,DC=COM,DC=BR</searchBase> 
                </ldapEntityType> 
                <ldapEntityType name="Group"> 
                  <objectClass>groupOfUniqueNames</objectClass> 
                  <searchBase>ou=groups,DC=BRADSEG,DC=COM,DC=BR</searchBase> 
                </ldapEntityType> 
                <customFilters userIdMap="*:uid" groupIdMap="*:cn" groupMemberIdMap="*:uniqueMember" userFilter="(&amp;(uid=%v)(objectclass=inetOrgPerson))" groupFilter="(&amp;(cn=%v)(objectclass=groupOfUniqueNames))"/> 
            </ldapRegistry> 
        </server>
```

## Helm upgrade
```bash
helm upgrade -n leap -f custom-values-leap.yaml hcl images/hcl-leap-deployment-v1.1.0_20240717-1648-9.3.7.18.tgz
```