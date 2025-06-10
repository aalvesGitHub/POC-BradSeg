# Instalação LDAP

Antes de iniciar a instalação do LDAP, deve ter acesso ao **usuário support** e o **sudoers com permissão total** no servidor.

Tabela de conteúdo
==================

- [Conecta no servidor](#conecta-no-servidor)
- [Pré-requisitos](#pré-requisitos)
  - [Sistema operacional](#sistema-operacional)
- [Downloads](#downloads)
  - [Baixar os arquivos](#baixar-os-arquivos)
  - [Extrair os binários](#extrair-os-binários)
- [Instalação](#instalação)
  - [DB2](#db2)
  - [ISDS](#isds)
  - [Web Administration Tools](#web-administration-tools)
    - [WebSphere Application Server](#websphere-application-server)
      - [Arquivo ports.props](#arquivo-portsprops)
      - [Profile TDSWebAdminProfile](#profile-tdswebadminprofile)
      - [Iniciar WebSphere](#iniciar-websphere)
      - [Deploy IDSWebApp](#deploy-idswebapp)
- [Base LDAP](#base-ldap)
  - [DC](#dc)
  - [Users](#users)
  - [Groups](#groups)
  - [Template](#template)
- [Service](#service)
  - [Arquivo db2.init](#arquivo-db2init)
  - [Arquivo ldap.service](#arquivo-ldapservice)
  - [Habilitar ldap.service](#habilitar-ldapservice)
- [Reinicialização](#reinicialização)
  - [Parar LDAP](#parar-ldap)
  - [Iniciar LDAP](#iniciar-ldap)
- [Próxima etapa: Configuração com LDAP](#próxima-etapa-configuração-com-ldap)

## Conecta no Servidor
```bash
ssh -l support 192.168.160.11

# Tornar usuário root
sudo su - 
```

## Pré-requisitos

### Sistema operacional
```bash
# Habilita repositório EPEL
dnf config-manager --set-enabled crb
dnf install epel-release epel-next-release
dnf install inxi

# Instalação de pacotes
dnf makecache --refresh
yum whatprovides /*/libpam.so
yum install lvm2 \
  net-tools \
  ksh libstdc++.i686 \
  libstdc++-devel.i686 \
  pam-devel-1.5.1-20.el9.i686 \
  binutils \
  mksh \
  pip \
  unzip \
  chrony \
  ksh \
  psmisc \
  rsync \
  libnsl \
  xorg-x11-utils \
  xorg-x11-xauth \
  libXtst.i686 \
  libXtst.x86_64 \
  gtk2.i686 \
  gtk2.x86_64 \
  gtk2-engines.x86_64 -y

# Instalação do gdown
pip install gdown

# Desativar o selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

# Cria o diretório
mkdir -p /db2/idsldap

# Criação do grupo
groupadd -g 1014 idsldap

# Criação do usuário
useradd -g idsldap -d /db2/idsldap -M idsldap
chown -R idsldap:idsldap /db2/idsldap
chmod -R 775 /db2/idsldap

# Tunning de limits
echo root hard nofile 65536 |tee -a /etc/security/limits.conf
echo root soft nofile 65536 |tee -a /etc/security/limits.conf

# Atualizar a senha do usuário
passwd idsldap
senha: $PASSWORD_IDSLDAP

# Tornar o usuário idsldap
su - idsldap

# Configuração do .bash_profile
cat > .bash_profile<<EOF
# .bash_profile

# Get the aliases and functions
if [ -f ~/.bashrc ]; then
        . ~/.bashrc
fi

# User specific environment and startup programs
EOF

# Configuração do .bashrc
cat > .bashrc <<EOF
# .bashrc

# Source global definitions
if [ -f /etc/bashrc ]; then
        . /etc/bashrc
fi

# User specific environment
if ! [[ "$PATH" =~ "$HOME/.local/bin:$HOME/bin:" ]]
then
    PATH="$HOME/.local/bin:$HOME/bin:$PATH"
fi
export PATH

# Uncomment the following line if you don't like systemctl's auto-paging feature:
# export SYSTEMD_PAGER=

# User specific aliases and functions
### CARREGA PROFILE DB2
[[ -f $HOME/sqllib/db2profile ]] && source $HOME/sqllib/db2profile

EOF

# Serviço chronyd
systemctl enable --now chronyd
systemctl restart chronyd

# Reiniciar o servidor
shutdown -r now
```

## DOWNLOADS

### Baixar os arquivos
```bash
# IBM DB2 11.5.4
gdown --id 1R-QR8CnjQup-7BSBibEet7NCM0NyOK_3 -O ~/DB2S_11.5.4_MPML.tar.gz

# IBM DB2 License
gdown --id 1TGXibJCgrRZULpu_nJGVStkhA2saHPuO -O ~/DB2_ESEPVU_OA_11.5.4_MPML.zip

# IBM DB2 11.5 FP9
gdown --id 1ceuvgZMKVk9_GsYdmlgsgPmordAvGTzt -O ~/v11.5.9_linuxx64_server_dec.tar.gz

# ISDS 6.5.0.28
gdown --id 1smxLIvCHFaEKJK0ghXZKe1BL-JvcGVw_ -O ~/6.4.0.28-ISS-ISDS-LinuxX64-IF0028.tar.gz

# IBM GSKIT
gdown --id 1i4VIV0hE4wlBwthjp9HvfG3nlOm-VFES -O ~/8.0.55.31-ISS-GSKIT-LinuxX64-FP0031.tar.gz

# ISDS Premium Feature Activation
gdown --id 1X9Xly2RfbGAx1tt2hL86akfRlgTc-2sw -O ~/CN47LML.tar

# IBM JAVA SDK
gdown --id 1FLKUScY6w1oF1kABguxgcvhFZxWaIjXb -O ~/8.0.8.5-ISS-JAVA-LinuxX64-FP005.tar

# HCL-Portal-95_WCM_SETUP PARTE 1
gdown --id 16MmL-h0yI8EArX3l0gBT8Mzi7ZrkrcSK -O ~/HCL-Portal-95_WCM_SETUP-01-SL.zip

# HCL-Portal-95_WCM_SETUP PARTE 2
gdown --id 158pMyKIPA6Qsz5vfjU62-glj0qCBHvFF -O ~/HCL-Portal-95_WCM_SETUP-02-SL.zip

# HCL-Portal-95_WCM_SETUP PARTE 3
gdown --id 1ShkJl3PdupQ1y-ySI1tjWn3AtegGAsT1 -O ~/HCL-Portal-95_WCM_SETUP-03.zip

# HCL-Portal-95_WCM_SETUP PARTE 4
gdown --id 1dGt1A7n4-QgyWfYt5Y7L66Rz-T1qmsKD -O ~/HCL-Portal-95_WCM_SETUP-04.zip

# JDK 8
gdown --id 1roUp1mYpFPeoj9EpV6JuMSBKC7K-oC3i -O ~/ibm-java-sdk-8.0-5.30-linux-x64-installmgr.zip
```

>**NOTA:**
>
> Caso não possua os binários, deve realizar os downloads pelo site do fabricante [Passport Advantage Online for Customers](https://www.ibm.com/software/passportadvantage/pao_customer.html)

### Extrair os binários
```bash
# Criar o diretório
mkdir -p /tmp/install/fp1159

# IBM DB2 11.5.4
tar xvf ~/DB2S_11.5.4_MPML.tar.gz -C /tmp/install

# IBM DB2 11.5 FP9
tar xvf ~/v11.5.9_linuxx64_server_dec.tar.gz -C /tmp/install/fp1159

# IBM DB2 License
unzip -q ~/DB2_ESEPVU_OA_11.5.4_MPML.zip -d /tmp/install

# ISDS 6.5.0.28
tar xvf ~/6.4.0.28-ISS-ISDS-LinuxX64-IF0028.tar.gz -C /tmp/install

# IBM GSKIT
tar xvf ~/8.0.55.31-ISS-GSKIT-LinuxX64-FP0031.tar.gz -C /tmp/install

# ISDS Premium Feature Activation
tar xvf ~/CN47LML.tar -C /tmp/install

# IBM JAVA SDK
tar xvf ~/8.0.8.5-ISS-JAVA-LinuxX64-FP005.tar -C /tmp/install

# HCL-Portal-95_WCM_SETUP PARTE 1
unzip -q ~/HCL-Portal-95_WCM_SETUP-01-SL.zip -d /tmp/install

# HCL-Portal-95_WCM_SETUP PARTE 2
unzip -q ~/HCL-Portal-95_WCM_SETUP-02-SL.zip -d /tmp/install

# HCL-Portal-95_WCM_SETUP PARTE 3
unzip -q ~/HCL-Portal-95_WCM_SETUP-03.zip -d /tmp/install

# HCL-Portal-95_WCM_SETUP PARTE 4
unzip -q ~/HCL-Portal-95_WCM_SETUP-04.zip -d /tmp/install

# JDK8
unzip -q ~/ibm-java-sdk-8.0-5.30-linux-x64-installmgr.zip -d /tmp/install/JDK805
```
>**NOTA:**
>
> O diretório /tmp precisa ter no mínimo 12GB disponível.

## Instalação

### DB2
```bash
# Verifica dependência da instalação
cp /etc/redhat-release /etc/redhat-release.orig
echo 'Red Hat Enterprise Linux Server release 7.9 (Maipo)' > /etc/redhat-release
/tmp/install/server_dec/db2prereqcheck -l -i

# Instalação dos binários db2
printf "o\nyes\nno\n" | /tmp/install/server_dec/db2_install -p SERVER -b /db2/V115

# Lista a versão do db2
db2ls

# Instala a licença
/db2/V115/adm/db2licm -a /tmp/install/ese_c/db2/license/db2ese_c.lic

# Lista a licença
/db2/V115/adm/db2licm -l

# Executa o installFixPack
printf "o\nyes\n" | /tmp/install/fp1159/server_dec/installFixPack -b /db2/V115
```

## ISDS

```bash
# Instalação da licença ISDS
printf "o\n1\n" | /tmp/install/6.4.0.28-ISS-ISDS-LinuxX64-IF0028/license/idsLicense

# Instalação do GSKIT
cd /tmp/install/8.0.55.31-ISS-GSKIT-LinuxX64-FP0031/64
rpm -ihv *

# Instalação do ISDS
```bash
cd /tmp/install/6.4.0.28-ISS-ISDS-LinuxX64-IF0028/images
rpm -ihv idsldap-*

# ISDS Premium Feature Activation
cd /tmp/install/x64_linux/entitlement
rpm -ihv --nodeps idsldap-ent64-6.4.0-0.x86_64.rpm

# IBM SDK JAVA
rsync -avl --progress /tmp/install/java /opt/IBM/ldap/V6.4/

# Cria links do ISDS
/opt/ibm/ldap/V6.4/bin/idslink -i -s fullsrv

# Criar usuário idsldap
idsadduser -u idsldap -w P@ssw0rd -l /db2/idsldap -g idsldap

# Criar usuário ldap01
idsadduser -u ldap01 -w P@ssw0rd -l /db2/ldap01 -g idsldap

# Criar instância idsldap
idsicrt -I idsldap -p 389 -s 636 -t idsldap -e Myseed@12345 -g Mysalt@12345

# Criar a base db2 ldap01
idscfgdb -n -I idsldap -a ldap01 -t ldap01 -w P@ssw0rd -l /db2/ldap01

# Configura senha do cn=root
idsdnpw idsldap -u cn=root -p <senha>

# Criar suffix
idscfgsuf -I idsldap -s "dc=bradseg,dc=com,dc=br" -n

# Iniciar instância
ibmslapd -I idsldap
```

## Web Administration Tools

### WebSphere Application Server
```bash
# Configurar permissão
chmod 755 -R /tmp/install/SETUP/IIM/linux_x86_64

# Instalar o IBM Installation Manager
/tmp/install/SETUP/IIM/linux_x86_64/install

# Instalar o WebSphere Application Server
/opt/IBM/InstallationManager/eclipse/IBMIM

File > Preferences > Add Repository
/tmp/install/SETUP/products/WASND905/repository.config
/tmp/install/JDK805/repository.config

Install

(instalação padrão)
```

####  Arquivo ports.props
```bash
cat > /tmp/ports.props <<EOF
BOOTSTRAP_ADDRESS=12102
SOAP_CONNECTOR_ADDRESS=12103
ORB_LISTENER_ADDRESS=10034 
SAS_SSL_SERVERAUTH_LISTENER_ADDRESS=10041
CSIV2_SSL_SERVERAUTH_LISTENER_ADDRESS=10036
CSIV2_SSL_MUTUALAUTH_LISTENER_ADDRESS=10033
WC_adminhost=12100
WC_defaulthost=12101
DCS_UNICAST_ADDRESS=10030
WC_adminhost_secure=12104
WC_defaulthost_secure=12105
SIP_DEFAULTHOST=10027
SIP_DEFAULTHOST_SECURE=10026
IPC_CONNECTOR_ADDRESS=10037
SIB_ENDPOINT_ADDRESS=10028
SIB_ENDPOINT_SECURE_ADDRESS=10038
SIB_MQ_ENDPOINT_ADDRESS=10040
SIB_MQ_ENDPOINT_SECURE_ADDRESS=10031
EOF
```

#### Profile TDSWebAdminProfile
```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh -create \
  -templatePath /opt/IBM/WebSphere/AppServer/profileTemplates/default \
  -profileName TDSWebAdminProfile \
  -profilePath /opt/IBM/WebSphere/AppServer/profiles/TDSWebAdminProfile \
  -cellName TDScell \
  -nodeName TDSnode \
  -enableAdminSecurity true \
  -adminUserName wasadmin \
  -adminPassword P@ssw0rd \
  -portsFile /tmp/ports.props
```

#### Iniciar WebSphere
```bash
/opt/IBM/WebSphere/AppServer/profiles/TDSWebAdminProfile/bin/startServer.sh server1
```

#### Deploy IDSWebApp
```bash
# Acessar a console WebSphere
https://192.168.160.11:12104/ibm/console
Username: wasadmin
Password: P@ssw0rd

# Deploy
cp /opt/ibm/ldap/V6.4/idstools/IDSWebApp.war /opt/IBM/WebSphere/AppServer/temp/
Applications > New Application 
Context Root: /IDSWebApp

# Acessar a console Web Administration Tools
http://192.168.160.11:12101/IDSWebApp
User ID: superadmin
Password: P@ssw0rd
```

### Base LDAP

#### DC
```bash
cat > /tmp/createDomain.ldif<<EOF
dn: dc=bradseg,dc=com,dc=br
objectClass: top
objectClass: domain
EOF
idsldapadd -D cn=root -w <senha> -f /tmp/createDomain.ldif
```

#### Users
```bash
idsldapadd -D cn=root -w <senha> -f /tmp/users_bradseg-dev.ldif
```

#### Groups
```bash
idsldapadd -D cn=root -w <senha> -f /tmp/groups_bradseg-dev.ldif
```

#### Template
```bash
cat > /tmp/template-user.ldif<<EOF
# Entry 5: uid=$userid,ou=users,dc=bradseg,dc=com,dc=br
dn: uid=$userid,ou=users,dc=bradseg,dc=com,dc=br
cn: $userfull
displayname: $userid
objectclass: inetOrgPerson
objectclass: organizationalPerson
objectclass: person
objectclass: top
businessCategory: $group
city: Sao Paulo
mail: $userid@bradseg.com.br
sn: $userid
uid: $userid
userpassword: ALM@V1V@
EOF
idsldapadd -D cn=root -w <senha> -f /tmp/template-user.ldif
```
## Service

### Arquivo db2.init
```bash
cat > ldap.init
#!/bin/bash
#
# Service: ldap.init
# Descrição: Este serviço de inicialização do Ldap.
# description: Start up the Tivoli Directory Server V6.4.
RETVAL=0

case "$1" in
  start)
    echo "-----------------------------------------------------------------------"
    echo "Iniciando Ldap"
    echo "-----------------------------------------------------------------------"
    ibmslapd -I idsldap ;;
stop)
   echo "-----------------------------------------------------------------------"
   echo "Parando Ldap"
   echo "-----------------------------------------------------------------------"
   ibmslapd -I idsldap -k ;;
*) echo $"Usage: $0 {start|stop}"
  exit 1 ;;
esac
exit 
```

### Arquivo ldap.service
```bash
cat > /etc/systemd/system/ldap.service<<EOF
[Unit]
Description=Startup for ISDS 6.4

[Service]
Type=oneshot
ExecStart=/etc/init.d/ldap.init start
ExecStop=/etc/init.d/ldap.init stop
User=root
RemainAfterExit=yes

[Install]
WantedBy=default.target
EOF
```

### Habilitar ldap.service
```bash
systemctl enable ldap.service
systemctl daemon-reload
```

## Reinicialização

### Parar LDAP
```bash
systemctl stop ldap.service
```

### Iniciar LDAP
```bash
systemctl start ldap.service
```

## Próxima etapa: Configuração com LDAP
Ir para [Configuração com LDAP](setup-ldap.md).
