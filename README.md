# Projeto Infra

# Introdução

Este projeto tem como objetivo simular uma infraestutura para consolidar os conhecimentos
adquiridos durante o estágio. Os pontos principais do prejeto são no sistema operativo Linux,
utilização de Docker e configuração de aplicações.

A infraestutura vai ser montada numa máquina virtual Ubunto server 24.04.2
A restante infraestutura vai ser montada dentro do Docker através de containers.

### Arquitetura

VM com Ubuntu Server
  Docker
    Traefik como reverse proxy
    Web server com domínio(Nginx)
    Prometheus + Grafana

O Ubunto Server foi escolhido devido a ser um sistema operativo fiável e com um baixo 
consumo de recursos, considerado uma ótima distribuição para alojar servidores.
A infraestrutura vai ser criada em Docker para garantir a portabilidade , isolamente e 
facilidade na gestão. Permite também facilmente replicar a infraestrutura noutros ambientes.
A implementação de um Web server com um domínio local visa à consolidação do conhecimento
sobre a configuração da aplicação escolhida.
O Prometheus e o Grafana são dois softwares utizados para monitorização e representação
de dados. O Prometheus tem a função de recolher métricas sobre o sistema e o 
Grafana vai representar os dados através de um painel com gráficos e quadros.

# Processo

## 1 - Criação do ambiente

### Criar a VM

O primeiro passo do projeto é ir ao site oficial do Ubunto e fazer download do ISO Ubunto server 24.04.2.
Após ter a imagem do sistema operativo procedemos à criação da máquina virtual, o virtualizador
utilizado foi o VmWare workstation.

Com o Ubunto instalado passei à configuração da rede , em que vou atribuir um IP fixo á VM.

A rede utilizada neste laboratório é uma rede virtual do VmWare que funciona com NAT. O IP da rede é 192.168.170.1/24 e o IP que vai ser atribuido ao servidor  é o 192.168.170.150/24.

Para configurar as definições de rede do Ubunto é utilizado o netplan que funcina com ficheiros de configuração yaml.  Para atribuir IP fixo ao servidor fazemos um backup do ficheiro original.

```bash
sudo cp /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.backup
```

E depois editamos o ficheiro  /etc/netplan/50-cloud-init.yaml com as configurações:

```bash
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses: [192.168.170.150/24]
      routes:
        - to : default
          via: 192.168.170.2
      nameservers:
          addresses: [8.8.8.8, 8.8.4.4]

```

Fonte: [https://documentation.ubuntu.com/server/](https://documentation.ubuntu.com/server/)

### Firewall

Após atribuido o IP vamos configurar a firewall de forma a permitir que os portos utilizados pela infraestrutura recebam ligações. O Ubuntu usa o ufw. É necessário correr os seguintes comandos :

```bash
sudo ufw allow 22
sudo ufw allow 80
sudo ufw allow 443
sudo ufw enable
sudo ufw restart
```

Através desta configuração vamos então abrir os portos 22  para ligações remotas ssh , 80 para ligações HTTP e o 443 para ligações HTTPS. Depois ativamos e reiniciamos a firewall para aplicar as configurações.

Após concluir estas configurações comecei a pesquisar sobre a instalção do docker no ubunto e deparei-me com a informação que devido ao modo de operação do docker. O tráfego é direcionado de e para os containers antes de passar pelas regras do ufw. O que leva a que as regras de firewall sejam ignoradas nas ligações aos containers.

Para resolver esta questão  foi utilizado configurando o iptables.

Para configurar o iptables é necessário instalar um pacote para as regras criadas ficarem guardadas persistêntemente:

```bash
sudo apt install iptables-persistent
```

Depois configurar a iptables com os seguintes comandos:

```bash

# Limpar regras existentes (se necessário)
sudo iptables -F
sudo iptables -X
sudo iptables -Z

# Permitir tráfego de loopback
sudo iptables -A INPUT -i lo -j ACCEPT

# Permitir respostas a conexões já estabelecidas
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Abrir portas específicas
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Definir políticas padrão
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# Salvar as regras
sudo netfilter-persistent save
```

NOTA: Ao configurar o iptables é necessário em primeiro lugar configurar as regras pretendidas e só depois definir as políticas padrão das chains. Sob pena de por exemplo se fechar o acesso à porta 22 e perder o acesso remoto à máquina.

fontes:

[https://documentation.ubuntu.com/server/](https://documentation.ubuntu.com/server/)

[https://docs.docker.com/engine/network/packet-filtering-firewalls/#docker-and-ufw](https://docs.docker.com/engine/network/packet-filtering-firewalls/#docker-and-ufw)

### Docker

Antes de começar a instalar o Docker Engine é preciso confirmar que não existe nenhum pacote que entre em conflito . Esta verificação é feita através do comando:

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

O que vai ser executado neste comando é loop onde em cada ciclo é removido o pacote em questão se existir.

A instalação do Engine vai ser feita através do repositório apt, o que significa que é necessário aceder ao mesmo:

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

Depois do repositório do Docker ser adicionado passar à instalação dos pacotes necessários na máquina:

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Para confirmar o sucesso da instalação do docker corremos o container de teste da docker.

```bash
sudo docker run hello-world
```

Este container vai fazer o pull da imagem e correr o container , que nos vai mostrar uma mensagem a confirmar que a execução deste container significa que o docker foi instalado com sucesso.

Apartir deste momento o Docker já está instalado na máquina mas devido à utilização dos sockets de comunicação pelo daemon do docker só o root é que tem permissões. De forma a evitar estar a correr todos o comando docker com sudo, ao instalar o docker é criado um grupo “docker” que vai ter as permissões necessárias para executar os comandos docker sem sudo.

Para confirmar o grupo é utlizados o comando:

```bash
sudo getent group docker
```

Que nos vai confirmar a existencia do grupo e seu gid.

O próximo passo é então adicionar o user pretendido ao grupo docker

```bash
sudo usermod -aG docker <user>
```

Se o sistema operativo não assumir a alteração executar um reboot à máquina.

Após o reboot já é expectável que seja possível executar comandos docker com o user adicionado ao grupo sem utilizar o sudo.

fontes:

 [https://docs.docker.com/engine/install/ubuntu/](https://docs.docker.com/engine/install/ubuntu/)

[https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)

[https://docs.docker.com/engine/install/linux-postinstall/](https://docs.docker.com/engine/install/linux-postinstall/)

### **Organizar Diretorias e Estrutura do Projeto**

Para criar um projeto organizado e de fácil compreensão vai ser criada uma estrutura de diretorias.

Sendo o principal caminho:

```bash
mkdir projetoinfra
```

Dentro da diretorias projetoinfra vao ser criados os seguintes diretorias:

```bash
mkdir traefik
mkdir webserver
mkdir prometheus

```

Dentro de cada um destas diretorias seram guardados os ficheiros de configuração da respetiva aplicação.

## 2-Serviços

Nesta fase vamos entrar no processo de começar a criar as aplicações da infraestrutura através de um ficheiro de docker compose. Vamos começar pela implementação do traefik como reverse proxy , passando para um webserver nginx e terminar com as ferramentas de monitorização prometheus e grafana.

Na descrição de cada serviço vai ser mostrado à parte do docker-compose.yml correspondente a esse serviço.

Para dar inicio ao projeto e garantir que todos os containers ficam na mesma rede docker vai ser criada a rede infranetwork:

 

```bash
docker network create --attachable -d bridge infranetwork
```

## Traefik

Este software foi escolhido para funcionar como reverse proxy e a sua função na infraestrutura é receber as ligações na porta 80 e 433 e encaminhar a comunicação para o respetivo container. Deste modo aumentamos a segurança porque os containers que necessitam de comunicação exteiror não precisam de portos publicados. 

![image.png](image.png)

Serviço do traefik do docker compose:

```yaml
traefik:
    image: traefik:v3.4
    container_name: traefik
    ports:
      - "80:80" 
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock # permite o acesso do traefik ao sockets do docker
      - ./traefik/traefik.yml:/etc/traefik/traefik.yml:ro # Passagem do ficheiro de configuração
    
    restart: unless-stopped # Iniciar/reiniciar o serviço a não ser que seja parado
    labels:
      - "traefik.enable=true" # ativa o serviço do traefik para encaminhar requisições para os containers
      - "traefik.http.routers.dashboard.entrypoints=web" # Indica que o tráfego vem da interface "web" ( porto 80 )
      - "traefik.http.routers.dashboard.rule=Host(`dashboard.projetoinfra.com`)" # label que vai resolver o nome
      - "traefik.http.routers.dashboard.middlewares=dashboard-auth" # Indica o middleware que vai tratar da autenticação
      - "traefik.http.middlewares.dashboard-auth.basicauth.users=admin:$$apr1$$2McayjLp$$oU3szFX6y0LaK8AExdB8b0" # User para autenticação
      - "traefik.http.services.traefik-traefik.loadbalancer.server.port=8080" # Configuração do serviço para enviar o tráfego da API
      - "traefik.http.routers.dashboard.service=api@internal" # Configuração para ativar o dashboard
    networks: 
      - infranetwork # configura o container na rede

```

Dentro da diretoria projetoinfra/traefik vai ser criado o ficheiro de configuração traefik.yml , ficheiro cujo vai passar as configurações para dentro do container:

```yaml
global:
  checkNewVersion: false # não permite atualizações
  sendAnonymousUsage: false # não permite enviar dados anonimamente
log:
  level: DEBUG # aumenta a verbose dos logs
api:
  dashboard: true # ativa o frontend
entryPoints: # os entryPoints vão definir os portos e os protocolos no qual o traefik vai receber pacotes
  web:
    address: :80
  websecure:
    address: :443
providers:
  docker: # vai permitir examinar os containers do docker
    endpoint: "unix:///var/run/docker.sock"

```

Visto que API do traefik vai estar exposta , vamos aumentar a segurança através da autenticação no acesso ao dashboard. Para o efeito é necessário criar um user e uma password para fazer a autenticação por http através do comando htpasswd.

A instalação do pacote com essa ferramenta é feita através do comando:

```bash
sudo apt install apache2-utils
```

Com o pacote instalado vamos então gerar um par user/password e o respetivo hash da password

```bash
htpasswd -nb admin "admin12345" | sed -e 's/\$/\$\$/g'
```

O output do comando vai ser o user e o respetivo hash da password

```bash
admin:$$apr1$$2McayjLp$$oU3szFX6y0LaK8AExdB8b0
```

Este hash vai ser copiado para a label do docker compose que declara os dados do user para autenticação

O neste caso o dashboard do traefik é acedido através do : [dashboard.projetoinfra.com] , ao aceder-mos a este link vai ser pedido um user e uma password para aceder ao dashboard.

Para o traefik encaminhar os pedidos para o respetivo container , é preciso adicionar labels na criação dos contaiers.


fontes:

[https://doc.traefik.io/traefik/v3.5](https://doc.traefik.io/traefik/v3.5)

[https://www.youtube.com/watch?v=-hfejNXqOzA](https://www.youtube.com/watch?v=-hfejNXqOzA)

[https://doc.traefik.io/traefik/setup/docker/](https://doc.traefik.io/traefik/setup/docker/)

## Web server

Após o reverse proxy estar configurado vamos então criar o servidor Web. O servidor web vai hospedar uma página com a minha apresentação e vai ter um link para fazer download do meu currículo.  A aplicação que vai ser utilizada é o Nginx. Vamos começar por criar um diretório webserver/html. 

Dentro desse diretório vamos criar um ficheiro index.html:

```html
<!DOCTYPE html>
<html lang="pt">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Currículo de Álvaro</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f4f4f4;
            text-align: center;
            padding: 40px;
        }

        .container {
            background: white;
            padding: 30px;
            border-radius: 10px;
            display: inline-block;
            box-shadow: 0 0 10px rgba(0,0,0,0.1);
        }

        h1 {
            color: #333;
        }

        a.button {
            display: inline-block;
            margin-top: 20px;
            padding: 12px 20px;
            font-size: 16px;
            background-color: #007bff;
            color: white;
            text-decoration: none;
            border-radius: 5px;
        }

        a.button:hover {
            background-color: #0056b3;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Olá, eu sou o Álvaro</h1>
        <p>Estagiário em Administração de sistemas Linux. Abaixo podes descarregar o meu currículo.</p>
        <a href="curriculo_alvarocastanheira.pdf" download class="button">Download do Currículo</a>
    </div>
</body>
</html>

```

Foi adicionado um ficheiro curriculo_teste.txt dentro da diretoria do html para quando aceder ao servidor web e fazer click no botão o é feito o download do ficheiro.

Em simultaneo com a criação deste web server é também criado um segundo web server com a finalidade de testar se o traefik está realmente a encaminhar o tráfego para os respetivos containers através do nome.

Configuração dos serviços no docker compose:

```yaml
nginx:
    container_name: nginx-server
    image: nginx:1.29.0-alpine
    networks:
      - infranetwork # configura o container na rede
    volumes:
      - ./webserver/html:/usr/share/nginx/html # montar o diretório com a página e currículo dentro do container
    labels: # labels para permitir o traefik encaminhar as requisições
      - traefik.enable=true # ativa o serviço do traefik para encaminhar requisições para os containers
      - traefik.http.routers.nginx-http.entrypoints=web
      - traefik.http.routers.nginx-http.rule=Host("alvaro.pt") # label que vai resolver o nome
    restart: unless-stopped # Iniciar/reiniciar o serviço a não ser que seja parado

  nginx-teste: # serviço nginx default criado para testar se o traefik está a resolver as requisições pelo domínio
    container_name: nginx-teste2 
    image: nginx:latest
    networks:
      - infranetwork # configura o container na rede
    labels: # labels para permitir o traefik encaminhar as requisições
      - traefik.enable=true # ativa o serviço do traefik para encaminhar requisições para os containers
      - traefik.http.routers.nginx-http2.rule=Host("projetoinfra.com") # label que vai resolver o nome
      - traefik.http.routers.nginx-http2.entrypoints=web 
    restart: unless-stopped # Iniciar/reiniciar o serviço a não ser que seja parado
```


## Prometheus

O Prometeus foi escolhido para fazer a monitorização tantos dos serviços como da máquina host. É uma aplicação que faz a recolha de dados regular em intrevalos de tempo , o que nos vai permitir analisar a performace dos serviços e do próprio sistema. O Prometheus tem vários componentes , sendo o principal o “Prometheus server” que vai recolher os dados.

Serviço do Prometheus no docker compose :

```yaml
 # Monitorização
  prometheus:
    container_name: prometheus
    image: prom/prometheus:v3.5.0
    volumes:
      - "./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml" # montar o ficheiro de configuração 
      - prometheus_data:/prometheus # montar um volume persistente para guardar os dados declarado no final do ficheiro
    labels: # labels para permitir o traefik encaminhar as requisições
     - traefik.enable=true # ativa o serviço do traefik para encaminhar requisições para os containers
     - traefik.http.routers.prometheus.entrypoints=web
     - traefik.http.routers.prometheus.rule=Host("prometheus.projetoinfra.com") # label que vai resolver o nome
    networks:
      - infranetwork
    restart: unless-stopped # Iniciar/reiniciar o serviço a não ser que seja parado

```

Exporters

O Prometheus faz a recolha das métricas através de exporters , devido à amplitude de sistemas que pode monitorizar. Os por norma os dados recolhidos não foram projetados para expor métricas, os exporters vão recolher essas métricas nos alvos e converter esses dados no formato padrão do Prometheus. 

Neste ambiente vamos utilizar dois exporters. O primeiro é o “node exporter” que vai recolher as métricas da própria máquina. E o segundo é o cAdivsor que vai recolher métricas sobre os recursos utilizados pelos containers.

Apesar dos exporters pertencerem à estrutura do Prometheus são criados em containers separados:

```yaml
   # Exporters do prometheus 
  node_exporter:
    container_name: node_exporter
    image: quay.io/prometheus/node-exporter:v1.9.1
    command: "--path.rootfs=/host"
    pid: host
    volumes:
      - /:/host:ro,rslave # montado o volume do "/" dentro do container em modo read-only para que o container possa recolher as métricas da máquina
    networks:
      - infranetwork
    restart: unless-stopped # Iniciar/reiniciar o serviço a não ser que seja parado

  cadvisor:
    container_name: cadvisor
    image: gcr.io/cadvisor/cadvisor:v0.52.1
    volumes:
      - /:/rootfs:ro
      - /run:/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    devices:
      - /dev/kmsg
    privileged: true
    networks:
      - infranetwork
    restart: unless-stopped # Iniciar/reiniciar o serviço a não ser que seja parado
```

Após configurado o docker compose crimamos então o prometheus.yml na diretoria do prometheus dentro do projetoinfra:

```yaml
global:
  scrape_interval: 15s  # By default, scrape targets every 15 seconds.

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  # external_labels:
  #  monitor: 'codelab-monitor'

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'
    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']

# Example job for node_exporter
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['node_exporter:9100']  
      # Example job for cadvisor
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']  
```


Fontes:

[https://prometheus.io/docs/introduction/overview/](https://prometheus.io/docs/introduction/overview/)

[https://mxulises.medium.com/simple-prometheus-setup-on-docker-compose-f702d5f98579](https://mxulises.medium.com/simple-prometheus-setup-on-docker-compose-f702d5f98579)

[https://www.youtube.com/watch?v=9TJx7QTrTyo&t=199s](https://www.youtube.com/watch?v=9TJx7QTrTyo&t=199s)

[https://github.com/google/cadvisor](https://github.com/google/cadvisor)

[https://prometheus.io/docs/guides/node-exporter/](https://prometheus.io/docs/guides/node-exporter/)

[https://github.com/ChristianLempa/boilerplates](https://github.com/ChristianLempa/boilerplates)

## Grafana

Após configurar o Prometheus para recolher as métricas a ferramenta utilizada para apresentar os dados é o Grafana devido a ter uma integração simples com o Prometheus.

O serviço do Grafana não requer ficheiros de configuração , apenas o serviço no ficheiro do docker compose:

```yaml
# Visualização das metricas
  grafana:
    image: grafana/grafana:12.0.2-security-01-ubuntu
    container_name: grafana
    labels: # labels para permitir o traefik encaminhar as requisições
     - traefik.enable=true # ativa o serviço do traefik para encaminhar requisições para os containers
     - traefik.http.routers.grafana.entrypoints=web
     - traefik.http.routers.grafana.rule=Host("grafana.projetoinfra.com") # label que vai resolver o nome
    volumes:
      - grafana_data:/var/lib/grafana # montar um volume persistente para guardar os dados declarado no final do ficheiro
    networks:
      - infranetwork
    restart: unless-stopped # Iniciar/reiniciar o serviço a não ser que seja parado
```


Para aceder ao dashboard do grafana é através do link:

```yaml
grafana.projetoinfra.com
```

Ao aceder ao link é apresentada a página de Log in do grafana. A autenticação configurada por defeito é o username e a password :“admin” , após o login in é pedido para mudar a password.

No caso deste laboratório atribui a password “admin12345” .

Ao fazer login é necessário adicionar o Prometheus como fonte de dados. Na coluna do lado esquerdo na aba “Connectios” depois clickar na opção “data sources” e procurar por “Prometheus”.

Ao entrar nesta página configurar  “Prometheus server URL”

 

```yaml
[http://prometheus:9090](http://prometheus:9090/)
```

Devido aos serviços estarem na mesma stack do docker compose os containers conseguem resolver os URLs pelo nome.

O próximo passo é criar um dashboard para as métricas serem visualizadas. Devido à flexibilidade e complexidade do Grafana existem templates de dashboards disponibilizados pela comunidade. Estes templates servem como base e depois podem ser adptados às necessidades de cada infraestrutura. 

O dashboard que vamos importar é o “Node Exporter Full” atarvés do ID fornencido no link:

```yaml
[https://grafana.com/grafana/dashboards/1860-node-exporter-full/](https://grafana.com/grafana/dashboards/1860-node-exporter-full/)
```

Após copiar o link na aba “Dashboards” selecionar a opção “Import a dashboard” e inserir o link copiado e clickar “Load”.

Após importar e carregar o tamplate é aceder à aba “Dashboards” para visualizar as métricas.

Fontes:

[https://grafana.com/docs/grafana/latest/setup-grafana/installation/docker/](https://grafana.com/docs/grafana/latest/setup-grafana/installation/docker/)

[https://www.youtube.com/watch?v=9TJx7QTrTyo&t=199s](https://www.youtube.com/watch?v=9TJx7QTrTyo&t=199s)

[https://grafana.com/grafana/dashboards/1860-node-exporter-full/](https://grafana.com/grafana/dashboards/1860-node-exporter-full/)


NOTA: Para efetuar os testar o acesso aos serviços é necessários resolver os nomes a nível local na máquina , adicionadas as entradas indicadas no ficheiro "hosts_adicionar.txt".