# Implementação em containers(gNB/UEs)

Este tutorial é uma adaptação da [documentação](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/ci-scripts/yaml_files/5g_rfsimulator/README.md) compartilhada pela OAI, foram adaptados alguns pontos cruciais para a integração separada entre o compornentes da arquitetura O-RAN, tais alterações estão listadas a baixo:
* Imagem Docker da gNB;
* Documento "docker-compose.yaml" para CORE e RAN;

> [!NOTE]
> Foi necessária a compilação de uma nova imagem da gNB devido a ausência dos modelos de serviço que são necessários para a conexão com o Near-RT RIC utilizado na nosso arquitetura (Flexric).

> [!IMPORTANT]    
> Esta versão da RAN funciona apenas com a versão v2.0.0 da CN5G.

## Pré-Requisitos
* **Geral**
    * Docker;
* **gNodeb**
    * Sistema Operacional: Ubuntu 22.04 LTS;
    * Outros requisitos não foram listados;
* **UE (User Equipament)**
    * Sistema Operacional: Ubuntu 22.04 LTS;
    * CPU: 8 cores x86_64 @ 3.5 GHz;
    * RAM: 8GB.

## Implementação dos containeres
Vamos tratar a implementação desagregada entre RAN, CORE e Near-RT RIC, caso queira implementar em uma mesma máquina siga o [tutorial oficial da OAI](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/ci-scripts/yaml_files/5g_rfsimulator/README.md).
> [!IMPORTANT] 
> Nesse ponto contamos que esteja com o Near-RT RIC implementado em uma máquina diferente.

```sh
# Comando deve ser executado nas máquina da gNB e Core (vamos utilizar no tutorial os nomes gNB-host e Core-host) 
git clone https://gitlab.eurecom.fr/oai/openairinterface5g.git
git clone git@github.com:gercom-ufpa/oai-5g-ran.git
# Habilitar o encaminhamento de ip
sudo sysctl net.ipv4.conf.all.forwarding=1
sudo iptables -P FORWARD ACCEPT

```
## OAI 5G Core
Agora devemos iniciar o OAI 5G Core (minimal) na Core-host:

```sh
# Comando deve ser executado em Core-host após verificar que estão presentes os 2 diretórios "openairinterface5g" e "OAI-5G-RAN"
# Essa comando irá substituir o script do docker compose do core onde temos algumas altereções como a imagem da gnb com os modelos de serviço
cp oai-5g-ran/Custom/5GC-OAI/docker-compose.yaml openairinterface5g/ci-scripts/yaml_files/5g_rfsimulator
cd openairinterface5g/
cd ci-scripts/yaml_files/5g_rfsimulator
docker compose up -d mysql oai-amf oai-smf oai-upf oai-ext-dn
# Verificar o estado dos containers, todos devem ficar em Healthy
watch docker compose ps -a
# Verificar se as interfaces "oai-cn5g" e "rfsim5g-traffic" foram criadas 
ifconfig
```
Após isso deixe separado o terminal da Core-host para o passo ... .


## RAN: gNB Monolítica
No terminal da gNB-Host antes de realizar o deploy é necessário verificar e alterar algumas informações:

```sh
# Comando deve ser executado em gNB-host após verificar que estão presentes os 2 diretórios "openairinterface5g" e "OAI-5G-RAN"
cp oai-5g-ran/Custom/RAN/gnb.sa.band78.106prb.rfsim.conf openairinterface5g/ci-scripts/conf_files
```
Esse arquivo é utilizado na implementação da gNB para carregar algumas informações de conexão, desse modo, dependendo do caso devemos mudar alguns campos para realizar a coneão da gNB com o Near-RT RIC e Core.

* Para conexão com o amf (Core):
```sh
////////// AMF parameters:
        amf_ip_address = ({ ipv4 = "IP-Core-Host"; });

        NETWORK_INTERFACES :
        {
           GNB_INTERFACE_NAME_FOR_NG_AMF            = "gnb-public"; # Interface de rede que é criada durante o deploy da gNB 
           GNB_IPV4_ADDRESS_FOR_NG_AMF              = "192.168.27.150/24";
           GNB_INTERFACE_NAME_FOR_NGU               = "gnb-public";
           GNB_IPV4_ADDRESS_FOR_NGU                 = "192.168.27.150/24";
           GNB_PORT_FOR_S1U                         = 2152; # Spec 2152
        };

```
* Para conexão com o Near-RT RIC (Flexric):
```sh
e2_agent = {
  near_ric_ip_addr = "IP-NearRT_RIC-Host";
  #sm_dir = "/path/where/the/SMs/are/located/"
  sm_dir = "/usr/local/lib/flexric/";
};
```
O próximo passo é copiar o arquivo docker-compose.yaml modificado para o diretório do OAI;

```sh
# Comando deve ser executado em gNB-host após verificar que estão presentes os 2 diretórios "openairinterface5g" e "OAI-5G-RAN"
cp oai-5g-ran/Custom/RAN/docker-compose.yaml openairinterface5g/ci-scripts/yaml_files/5g_rfsimulator
```

Nesse docker compose temos definidas as configurações da GNB e UEs, bem como a rede gnb-public e o endereço a gNB monolítica:

```sh
        networks:
            public_net_gnb:
                ipv4_address: 192.168.27.150

```

> [!IMPORTANT]
> Antes de iniciar a gNB devemos criar a rota de comunicação entre gNB-Host e Core-Host

```sh
# Na máquina gNB-Host
sudo ip route add 192.168.70.128/26 via IP_ADD_OF_CORE_HOST dev NETWORK_INTERFACE_NAME_OF_gNB_HOST
# IP_ADD_OF_CORE_HOST: É o IP-Core-Host, ou seja o ip da máquina que está implementado o Core da OAI
# NETWORK_INTERFACE_NAME_OF_gNB_HOST: interface de rede padrão da máquina que está sendo implementado a gNB ou seja, o gNB-Host
```

De forma semelhante o mesmo processo deve ser realizado na máquina do Core-Host:
```sh
sudo ip route add 192.168.27.0/24 via IP_ADD_OF_gNB_HOST dev NETWORK_INTERFACE_NAME_OF_CORE_HOST
# IP_ADD_OF_gNB_HOST: É o IP-gNB-Host, ou seja o ip da máquina que está implementado a gNB da OAI
# NETWORK_INTERFACE_NAME_OF_Core_HOST: interface de rede padrão da máquina que está sendo implementado o Core 5G ou seja, o Core-Host
```

> [!TIP]
> Para verificar o sucesso da conexão podemos utilizar o "ping 192.168.70.129" na gNB-Host

Após isso, podemos continuar a implementação da gNB monolítica:

```sh
# Na máquina gNB-Host
cd openairinterface5g/ci-scripts/yaml_files/5g_rfsimulator
docker compose up -d oai gnb
# verificar o estatdo do container
docker compose ps -a
# Agora na máquina Core-Host é possível verificar a conexão da gnb com o amf
docker logs rfsim5g-oai-amf
# Devem aparecer as informações de Index, Status (Connected), Global ID gNB Name e PLMN
```
> [!TIP]
> É possível realizar um ping de dentro do container da gNB para verificar a conexão:

```sh
# Na máquina gNB-Host
docker exec -it rfsim5g-oai-gnb bash
ping 192.168.70.129
```

## RAN: UEs

Após o sucesso dos passos anteriores é possível realizar a implementação das UEs, no [arquivo](Custom/RAN/docker-compose.yaml) são definidas até 10 UEs, porém mais podem ser configuradas mudando os seguintes parâmetros:

```sh
# Caso queiram adicionar mais UEs, colocar após a UE10 no arquivo
oai-nr-ue"Número-da-UE":
        image: oaisoftwarealliance/oai-nr-ue:develop
        privileged: true
        container_name: rfsim5g-oai-nr-ue
        # mudar o imsi
        environment:
            USE_ADDITIONAL_OPTIONS: -E --sa --rfsim -r 106 --numerology 1 --uicc0.imsi 208990100001100 -C 3619200000 --rfsimulator.serveraddr 192.168.27.150 --log_config.global_log_options level,nocolor,time
        depends_on:
            - oai-gnb
            #mudar o ip
        networks:
            public_net_gnb:
                ipv4_address: 192.168.27.160
        devices:
             - /dev/net/tun:/dev/net/tun
        volumes:
            - ../../conf_files/nrue.uicc.conf:/opt/oai-nr-ue/etc/nr-ue.conf
        healthcheck:
            test: /bin/bash -c "pgrep nr-uesoftmodem"
            interval: 10s
            timeout: 5s
            retries: 5

```
> [!IMPORTANT]
> Verificar se o IP do campo "--rfsimulator.serveraddr" condiz com o endereço da gNB

Depois de verificar o endereço da gNB podemos seguir com o deploy da UE:

```sh
cd openairinterface5g/ci-scripts/yaml_files/5g_rfsimulator
# Para mais de uma UE vá adicionado o nome de cada uma respectivamente no comando
docker compose up -d oai-nr-ue
# Verificar o status da UE
docker compose ps -a
```

Com o término do deploy da UE devemos garantir que ela está conectada:

```sh
# Na máquina gNB-Host
# Devemos verificar se a interface de rede "oaitun_ue1" foi criada no container, pois ela é configurada pelo container "ext-dn" presente no core 
docker exec -it rfsim5g-oai-nr-ue /bin/bash
ifconfig
# Na máquina Core-Host
# Novamente verifica no amf se a UE foi registrada
docker logs rfsim5g-oai-amf
```
Com o término da verificação e caso tudo esteja de acordo podemos prosseguir para o último passo.

## Teste de conectividade 

Primeiro podemos utilizar o ping para verificar a conectividade entre UE e Core:

```sh
# Na gNB-Host
docker exec -it rfsim5g-oai-nr-ue /bin/bash
# Fazer um ping utilizando a interface "oaitun_ue1" para o container "ext-dn" que é utilizado para os próximos testes
ping -I oaitun_ue1 -c 2 192.168.72.135

```

Outro teste que pode ser realizado é de Donwlink (ext-dn->ue) e Uplink (Ue->ext-dn) utilizando iperf:

* Downlink
```sh
# Na máquina gNB-Host
# Iniciar um servidor iperf dentro do container NR-UE
docker exec -it rfsim5g-oai-nr-ue /bin/bash
iperf -B 12.1.1.2 -u -i 1 -s
# Na máquina Core-Host
# Iniciar um cliente iperf dentro do container ext-dn
docker exec -it rfsim5g-oai-ext-dn /bin/bash
iperf -c 12.1.1.2 -u -i 1 -t 20 -b 500K
```

* Uplink
```sh
# Na máquina Core-Host
# Iniciar um cliente iperf dentro do container ext-dn
docker exec -it rfsim5g-oai-ext-dn /bin/bash
# Verificar
iperf -B 192.168.72.135 -u -i 1 -s
# Na máquina gNB-Host
# Iniciar um servidor iperf dentro do container NR-UE
docker exec -it rfsim5g-oai-nr-ue /bin/bash
iperf -c 192.168.72.135 -u -i 1 -t 200 -b 10G
```