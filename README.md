# Deploy OAI-RAN

Nesse documento são tratados os diferentes tipos de implementação da RAN provida pela OAI (Open Air Interface) que é composta pela gNodeB e UE.

## Pré-requisitos Gerais
* **gNodeb**
    * Sistema Operacional: Ubuntu 22.04 LTS;
    * Outros requisitos não foram listados;
* **UE (User Equipament)**
    * Sistema Operacional: Ubuntu 22.04 LTS;
    * CPU: 8 cores x86_64 @ 3.5 GHz;
    * RAM: 8GB.
* **Geral**
    * Docker;
    * CMake &ge;v3.15;
> [!TIP]
> Cmake pode ser instalado usando o [tutorial](https://apt.kitware.com/).

> [!NOTE]
> Referências tiradas do [GIT do projeto](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/NR_SA_Tutorial_OAI_nrUE.md).

## Pré-requisitos de deploy
* **5GCN** 
    * Primeiros instalar e configurar a 5G Core Network da OAI.
> [!NOTE]
> Ler da documentação no [GIT]().

* **Build do UHD**    
    ```sh
    sudo apt -y install libsctp-dev python3.8 cmake-curses-gui libpcre2-dev autoconf automake build-essential ccache cmake cpufrequtils doxygen ethtool g++ git inetutils-tools libboost-all-dev libncurses5 libncurses5-dev libusb-1.0-0 libusb-1.0-0-dev libusb-dev python3-dev python3-mako python3-numpy python3-requests python3-scipy python3-setuptools python3-ruamel.yaml

    git clone https://github.com/EttusResearch/uhd.git ~/uhd
    cd ~/uhd
    git checkout v4.6.0.0
    cd host
    mkdir build
    cd build
    cmake ../
    make -j $(nproc)
    make test # Opcional
    sudo make install
    sudo ldconfig
    sudo uhd_images_downloader

* **Build gNB-OAI e nrUE-OAI** 
    
    ```sh
    # Clonar o repositório fonte do projeto OAI
    git clone https://gitlab.eurecom.fr/oai/openairinterface5g.git ~/openairinterface5g
    git checkout develop
    ```

> [!IMPORTANT]    
> Por padrão o OAI utiliza o E2AP V2 e KPM V2, o nosso Near-RT RIC (Flexric) foi gerado utilizando a versão 3.

    ```sh
    # Modificar os parâmetros da E2AP E KPM
    nano CMakeLists.txt

    #Substituir para
    E2AP_VERSION "E2AP_V3"
    KPM_VERSION "KPM_V3_00"

    # Installar as dependencias do OAI
    cd ~/openairinterface5g/cmake_targets
    ./build_oai -I

    # Dependencias nrscope
    sudo apt install -y libforms-dev libforms-bin

    # Build OAI gNB e nrUE
    ./build_oai -w USRP --ninja --nrUE --gNB --build-e2 --build-lib "nrscope" -C
    ```

* **Modelos de Serviço**

    Outro componente fundamental para a conexão entre gNodeB e Near RT-RIC é a presença do modelo de serviço na máquina.

    ```sh
    # Adicionar o repositório do projeto
    git clone ~adicionar nosso repositório~
    cd ~adicionar nosso repositório/custom~
    sudo cp flexric/ /usr/local/lib/flexric 
    ``` 
## Execução OAI gNB e nrUE

* Ao chegar nesse ponto esperamos que o 5GCN OAI já esteja de pé. Após isso, é necessário verificar o IP do Near-RT RIC e retornar ao diretório do OAI.
    
    ```sh
    cd ~/openairinterface5g/
    nano nano targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf

    # Mudar o campo do e2_agent

    e2_agent = {
        near_ric_ip_addr = "IP-NearRT-RIC";
        #sm_dir = "/path/where/the/SMs/are/located/"
        sm_dir = "/usr/local/lib/flexric/";
    };
    ```


    ```sh
    #Retornar ao diretório do projeto oai
    cd ~/openairinterface5g/cmake_targets/ran_build/build

    # Executar a gNB
    sudo ./nr-softmodem -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf --gNBs.[0].min_rxtxtime 6 --rfsim --sa

    # Executar a nrUE com o RFsimulator
    sudo ./nr-uesoftmodem -r 106 --numerology 1 --band 78 -C 3619200000 --sa --uicc0.imsi 001010000000001 --rfsim

    ```
