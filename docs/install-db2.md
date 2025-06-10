# Instalação DB2

Antes de iniciar a instalação do db2, deve ter acesso ao **usuário support** e o **sudoers com permissão total** no servidor.

Tabela de conteúdo
==================

- [Conecta no servidor](#conecta-no-servidor)
- [Pré-requisitos](#pré-requisitos)
  - [Sistema operacional](#sistema-operacional)
  - [LVM](#lvm)
- [Downloads](#downloads)
  - [Baixar o arquivo](#baixar-o-arquivo)
  - [Extrair o binário](#extrair-o-binário)
- [Instalação](#instalação)
  - [Arquivo db2.init](#arquivo-db2init)
  - [Arquivo db2.service](#arquivo-db2service)
  - [Parar DB2](#parar-db2)
  - [Iniciar DB2](#iniciar-db2)
- [Próxima etapa: Configuração com DB2](#próxima-etapa-configuração-com-db2)

## Conecta no Servidor
```bash
ssh -l support 192.168.160.9

# Tornar usuário root
sudo su - 
```

## Pré-requisitos

### Sistema operacional
```bash
# Atualizar repositório
dnf makecache --refresh

# Procura pela biblioteca para mostrar a release do pacote
yum whatprovides /*/libpam.so

# Instalação de pacotes
yum install lvm2 net-tools ksh libstdc++.i686 libstdc++-devel.i686 pam-devel-1.3.1-22.el8.i686 binutils mksh unzip -y

# Desativa o selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

# Cria o grupo dba
groupadd -g 1007 dba

# Cria o usuário dbdev
useradd -g dba -d /db2/dbdev -M dbdev

# Configura permissão no diretório
chown -R dbdev:dba /db2/dbdev
chown -R dbprd:dba /db2/dbprd

# Torna usuário dbdev
su - dbdev

# Configura o bash
cat > .bash_profile<<EOF
# .bash_profile

# Get the aliases and functions
if [ -f ~/.bashrc ]; then
        . ~/.bashrc
fi

# User specific environment and startup programs
EOF
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
```

### LVM
```bash
# Procurar o disco 150 GB
fdisk -l |grep 150

# Criar o PV
pvcreate /dev/sdb

# Criar o VG
vgcreate dbvol /dev/sdb

# Lista o VG
vgs

# Criar a LV
lvcreate -L 10G -n dbbin dbvol
lvcreate -l100%FREE -n dbdev dbvol

# lista a LV
lvs

# Formata a partição LVM
mkfs.ext4 /dev/dbvol/dbbin
mkfs.ext4 /dev/dbvol/dbdev

# Cria o diretório
mkdir /db2

# Cadastra entrada no fstab
echo "/dev/mapper/dbvol-dbbin /db2    ext4    defaults        0       1" |tee -a /etc/fstab

# Atualiza o systemctl
systemctl daemon-reload

# Mounta o filesystem
mount -a

# Cria o diretório
mkdir /db2/dbdev

echo "/dev/mapper/dbvol-dbdev /db2/dbdev    ext4    defaults        0       2" |tee -a /etc/fstab

# Atualiza o systemctl
systemctl daemon-reload

# Mounta o filesystem
mount -a

# Lista o filsystem
df -h
```

## Downloads

### Baixar o arquivo
```bash
gdown --id "xxxxxxxxxx" -O /tmp/install/DB2S_11.5.4_MPML.tar.gz
gdown --id "xxxxxxxxxx" -O /tmp/install/v11.5.9_linuxx64_universal_fixpack.tar.gz
```

### Extrair o binário
```bash
tar xvf /tmp/install/DB2S_11.5.4_MPML.tar.gz -C /tmp/install
tar xvf /tmp/install/v11.5.9_linuxx64_universal_fixpack.tar.gz -C /tmp/install/fixpack
```

### Instalação
```bash
# Verifica dependências da instalação
cp /etc/redhat-release /etc/redhat-release.orig
echo 'Red Hat Enterprise Linux Server release 7.9 (Maipo)' > /etc/redhat-release

/tmp/server_dec/db2prereqcheck -l -i

# Executa o db2_install
printf "o\nyes\nno\n" | /tmp/server_dec/db2_install -p SERVER -b /db2/V115

# Lista a versão do db2
db2ls

# Lista a licença db2
/db2/V115/adm/db2licm -l

# Executa o installFixPack
printf "o\nyes\n" | ./installFixPack -b /db2/V115

# Criar instancia dbdev (Port: 25010)
/db2/V115/instance/db2icrt -u dbdev dbdev

# Tornar usuário dbdev
su - dbdev

# Lista instancia
db2 get instance

# Lista a versão 
db2level

# Lista as configurações da instância
db2 get dbm cfg

# Cria database sample
db2sampl

# Inicia o db2
db2start

# Conecta no database sample
db2 connect to sample

# Query na tabela staff
db2 "select * from staff"

# Lista processos no database
db2 list applicationscp /etc/redhat-release /etc/redhat-release.orig
echo 'Red Hat Enterprise Linux Server release 7.9 (Maipo)' > /etc/redhat-release

# Lista tabespaces do database
db2 list tablespaces show detail

# Detalha todos os processos no database
db2 list applications for database sample show detail

# Info do SO
db2pd -osinfo

# Lista as conexões ativas no database
db2 list active databases

# Lista os diretórios db2
db2 list db directory

# lista o diretory pela path
db2 list database directory on /db2/dbdev

# Lista o tamanho do database
db2 'call get_dbsize_info(?,?,?,0)'

# Configurações do database
db2 get db cfg

# Configurações da instancia
db2 get dbm cfg 

# Extrair o pacote
unzip -q /install/DB2_ESEPVU_OA_11.5.4_MPML.zip -d /tmp

# Instala a licença
/db2/V115/adm/db2licm -a /tmp/ese_c/db2/license/db2ese_c.lic

# Lista a licença
/db2/V115/adm/db2licm -l
```

#### Arquivo db2.init
```bash
cat > /etc/init.d/db2.init
#!/bin/bash
#
# Service: db2.init
# Descrição: Este serviço de inicialização do DB2.
# description: Start up the DB2 11.5.9.0.
RETVAL=0

case "$1" in
  start)
    echo "-----------------------------------------------------------------------"
    echo "Iniciando DB2"
    echo "-----------------------------------------------------------------------"
    su - dbdev -c /db2/dbdev/sqllib/adm/db2start ;;
stop)
   echo "-----------------------------------------------------------------------"
   echo "Parando DB2"
   echo "-----------------------------------------------------------------------"
   su - dbdev -c "db2 force applications all"
   su - dbdev -c "db2 terminate"
   su - dbdev -c /db2/dbdev/sqllib/adm/db2stop
   su - dbdev -c "db2licd -end"
   su - dbdev -c /db2/dbdev/sqllib/bin/ipclean ;;
*) echo $"Usage: $0 {start|stop}"
  exit 1 ;;
esac
exit 
```

#### Arquivo db2.service
```bash
cat > /etc/systemd/system/db2.service<<EOF
[Unit]
Description=Startup for DB2 11.5.9.0

[Service]
Type=oneshot
ExecStart=/etc/init.d/db2.init start
ExecStop=/etc/init.d/db2.init stop
User=root
RemainAfterExit=yes

[Install]
WantedBy=default.target
EOF
```

#### Habilitar db2.service
```bash
systemctl enable db2.service
systemctl daemon-reload
```

#### Parar DB2
```bash
systemctl stop db2.service
```

#### Iniciar DB2
```bash
systemctl start db2.service
```

## Próxima etapa: Configuração com DB2
Ir para [Próxima etapa: Configuração com DB2](setup-db2.md).
