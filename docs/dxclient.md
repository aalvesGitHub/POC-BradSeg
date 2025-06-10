# DXCLIENT

DXClient é uma ferramenta de linha de comando que oferece uma interface unificada para todas as tarefas de automação e CI/CD relacionadas ao HCL DX. 

Tabela de conteúdo
==================

- [Docker](#docker)
- [CLI](#cli)

## DOCKER
```bash
# Donwload da imagem dxclient
gdown --id 1aX-ZRaupEpfdraVyRVS5mT2rLvKQdDCz -O /tmp/hcl-dxclient-image-v95_CF221_20240708-2022.zip

# Extrai o arquivo
unzip -q /tmp/hcl-dxclient-image-v95_CF221_20240708-2022.zip -d /tmp

# Carrega a imagem docker
docker load -i /tmp/dxclient/dxclient.tar.gz

# Carrega variáveis
export USER_WPS="usuario_wps"
export PASSWORD_WPS="senha_wps"

# Cria o container dxclient
docker run --name dxclient -it hcl/dx/client:v95_CF221_20240708-2007 bin/dxclient restart-dx-core --hostname www-dev.bradseg.com.br \
--dxUsername $USER_WPS \
--dxProfileName wp_profile \
--dxProfilePath /opt/HCL/wp_profile \
--dxPassword $PASSWORD_WPS \
--dxConnectUsername $USER_WPS \
--dxConnectPassword $PASSWORD_WPS \
--dxConnectPort 443
```
## CLI
```bash
# Donwload da imagem dxclient
gdown --id 1I8SdcD3jImvaRXDgbrB4XbCwrwqjkbpb -O /tmp/hcl-dxclient-v95_CF221_20240708-2022.zip

# Extrai o arquivo
unzip -q /tmp/hcl-dxclient-v95_CF221_20240708-2022.zip -d /opt

# Ir para o diretório
cd /opt/dxclient

# Compila o pacote
make install

# Carrega variáveis
export PATH:$PATH:/opt/dxclient/bin
export USER_WPS="usuario_wps"
export PASSWORD_WPS="senha_wps"

# Exemplos: dxclient

# Reinicia o POD dx-core
dxclient restart-dx-core \
--hostname dxclient.bradseg.com.br \
--dxUsername $USER_WPS \
--dxProfileName wp_profile \
--dxProfilePath /opt/HCL/wp_profile \
--dxPassword $PASSWORD_WPS \
--dxConnectUsername $USER_WPS \
--dxConnectPassword $PASSWORD_WPS \
--dxConnectPort 443

# Deploy do Tema bradseg
dxclient deploy-theme \
--dxPort 443 \
--dxProtocol https \
--dxUsername $USER_WPS \
--dxPassword $PASSWORD_WPS \
--dxSoapPort 10033 \
--hostname www-dev.bradseg.com.br \
--xmlConfigPath /config \
--dxConnectPort 443 \
--dxConnectUsername $USER_WPS \
--dxConnectPassword $PASSWORD_WPS \
--xmlFile /home/thiago/Gitlab/Bradseg/tema/bradsegThemeDynamic/WebContent/WEB-INF/registerTheme.xml \
--applicationFile /home/thiago/Gitlab/Bradseg/tema/bradsegThemeEAR/target/bradsegThemeEAR.ear \
--applicationName "bradsegThemeEAR" \
--themeName "Alma Viva Solutions - Theme" \
--themePath /home/thiago/Gitlab/Bradseg/tema/bradsegThemeDynamic/target \
--dxProfileName wp_profile

# Importa Biblioteca WCM
bin/dxclient wcm-library-import \                                          
-hostname dxclient.bradseg.com.br \
-dxUsername $USER_WPS \
-dxPassword $PASSWORD_WPS  \
-dxConnectUsername $USER_WPS \
-dxConnectPassword $PASSWORD_WPS \
-dxConnectPort 443 \
-dxWASUsername $USER_WPS \
-dxWASPassword $PASSWORD_WPS \
-dxProfileName wp_profile \
-libFilePath /home/thiago/Downloads/wcm-library-export-20240920143809.tar.gz
```


