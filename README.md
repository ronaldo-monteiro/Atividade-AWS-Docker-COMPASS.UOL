<h1 align="center">Projeto AWS - Docker </h1>

<h2 align="center">Passo a passo do projeto </h2>
<ol>

Desafio do do projeto:

1 - Instalar e configurar o DOCKER ou CONTAINERD no host EC2; conforme
projeto Ponto adicional para o trabalho utilizar a instalação via script de
Start Instance (user_data.sh)

2 - Efetuar Deploy de uma aplicação Wordpress com:
container de aplicação RDS database Mysql

3 - Configuração da utilização do serviço EFS AWS para estáticos
do container de aplicação Wordpress

4 - configuração do serviço de Load Balancer AWS para a aplicação
Wordpress

# <h2>Desenvolvimento do projeto</h2>

 - Criar uma VPC com duas sub-redes, cada uma em uma zona de disponibilidad distintas.
 - Criar uma Tabela de Rotas epermitir o tráfego de internet e associar as duas sub-redes criadas a ela.
 - Criar duas Instâncias EC2, cada uma para uma zona de disponibilidade, utilizar os seguintes REGRAS:

    - <h4>SECURITY GROUP PUBLICO</h4>
    
    | TIPO | PROTOCOLO | PORTA | ORIGEM |
    | --- | --- | --- | --- |
    | HTTPS | TPC | 443 | /0|
    | HTTP | TCP | 80 | 0.0.0.0/0 |
    
    - <h4>SECURITY GROUP PRIVADO</h4>
    
    | TIPO | PROTOCOLO | PORTA | ORIGEM |
    | --- | --- | --- | --- |
    | HTTP | TPC | 80 | SG PUBLICO |
    | SSH | TCP | 22 | SG PUBLICO |
    

    - <h4>SECURITY GROUP RDS</h4>
    
    | TIPO | PROTOCOLO | PORTA | ORIGEM |
    | --- | --- | --- | --- |
    | MYSQL/AURORA | TCP | 3306 | SG PRIVADO|

    - <h4>SECURITY GROUP EFS</h4>
    
    | TIPO | PROTOCOLO | PORTA | ORIGEM |
    | --- | --- | --- | --- |
    | NFS | TCP | 2049 | SG PRIVADO |


 <h2> Criar o EFS Elastic File System</h2>
 
Devemos criar um Elastic File sistem  para uso  dos diretórios públicos e estáticos do wordpress, do container de aplicação Wordpress.


<h2>Criar o Relational Database Service</h2>

- devemos criar um banco de dados RDS - Mysql para hospedar a aplicaço do container

- IMPORTANTE utilizar o  Template MySQL que é Free Tier.

- Escolher a rede VPC e o Security Group criados.
Criar um nome inicial do Database.

<h2>Criar o Template de Instâncias</h2>


- Utilizaremos uma EC2 

- Utilzaremos uma Amazon Linux t3.small

- Devemos criar uma chave SSH


Criar o script user_data.sh para instalar o docker, wordpress e nfs-utils;\
montar o efs e criar o yaml para conectar ao rds.

#!/bin/bash
sudo su\
yum update -y\
yum install docker -y\
systemctl start docker\
systemctl enable docker\
usermod -aG docker ec2-user\
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose\
chmod +x /usr/local/bin/docker-compose\
mv /usr/local/bin/docker-compose /bin/docker-compose\
yum install nfs-utils -y\
mkdir /mnt/efs/\
chmod +rwx /mnt/efs/\
mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport ID_DO_SEU_EFS.efs.us-east-1.amazonaws.com:/ /mnt/efs\
echo "ID_DO_SEU_EFS.efs.us-east-1.amazonaws.com:/ /mnt/efs nfs defaults 0 0" >> /etc/fstab
echo "version: '3.8'\
services:\
  wordpress:\
    image: wordpress:latest\
    volumes:\
      - /mnt/efs/wordpress:/var/www/html\
    ports:
      - 80:80\
    restart: always\
    environment:\
      WORDPRESS_DB_HOST: ENDPOINT XXXXX\
      WORDPRESS_DB_USER: MASTER USERNAME XXXXX\
      WORDPRESS_DB_PASSWORD: MASTER PASSWORD XXXXX\
      WORDPRESS_DB_NAME: INITIAL NAME XXXXX\
      WORDPRESS_TABLE_CONFIG: wp_" | sudo tee /mnt/efs/docker-compose.yml\
cd /mnt/efs && sudo docker-compose up -d\

<h2>Criar o Load Balancer</h2>

- Precisamos criar um Load Balancer.
- Para isso vamos em EC2 -> Load Balancing -> Load Balancers -> Create load balancer.
- Foi sugerido usar o Classic Load Balancer.
- Em basic information seleciona o scheme Internet-facing
- Selecione A VPC criada anteriormente
- Em Mappings marque as duas Az's e selecione as duas Subnets Públicas
- Selecione o SG-PUBLIC para acesso à internet
- Em Listeners e Routing, vou colocar as portas http:80 e tcp:22
- Em Health checks, vou selecionar tcp:80

<h2>Criar o EndPoint</h2>

- Este Endpoint serve de túnel da internet para às instâncias via SSH que não têm IP público.
- Neste passo, para criá-lo vamos em VPC -> Endpoints -> Create endpoint
- Em Service Category selecionar EC2 Instance Connect Endpoint
- Em VPC, selecionar a anterior criada
- Em Security Group selecionar a SG-PUBLIC
- Em subnet, selecionar a private-1a

 <h2>Criar o Auto Scaling</h2>
 
- Para criar o ASG iremos em EC2 -> Auto Scaling -> Auto Scaling Groups e Create Auto Scaling Groups.
- Selecione o Template criado anteriormente, a VPC e as 2 Subnets Privadas.
- Attach to an existing load balancer - Choose from Classic Load Balancers, escolher o load Balancer criado
- Health Check é recomendado ligar o ELB health checks
- Para o group Size vou colocar como Desire capacity = 2, Min = 2 e Max = 4.
- Em target tracking, vou ativar o automatic scaling para 80% de utilização da CPU.

Tudo estiver configurado corretamente, podemos acessar a aplicação pelo DNS do Load Balancer (e não pelo IP das instâncias).

<h2>Fim do projeto!</h2>
