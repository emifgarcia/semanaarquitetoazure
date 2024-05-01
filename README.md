# Projeto Descritivo Semana Arquiteto Azure
## Objetivo

O objetivo desta documentação é fornecer uma visão detalhada da arquitetura proposta para a migração da aplicação "Sistema" e do banco de dados "SQL Server” do ambiente on-premise para a plataforma Azure. Esta documentação destina-se a fornecer orientações claras e detalhadas para os arquitetos, engenheiros e partes interessadas envolvidos no processo de migração, permitindo uma compreensão completa da arquitetura proposta, suas vantagens, considerações-chave e o plano estratégico para a implementação e migração bem-sucedida da aplicação para a nuvem Azure. Além disso, visa garantir que todas as partes interessadas tenham uma visão unificada e compartilhada dos objetivos, requisitos e abordagens para a migração, facilitando uma colaboração eficaz e a tomada de decisões informadas ao longo do processo de implementação. Vale ressaltar que a empresa fictícia que está conduzindo essa migração é denominada "Café com Leite Tech”, e este projeto é meramente uma abordagem para fins de estudo.

## Arquitetura Existente

A arquitetura atual reside em um ambiente on-premise, onde as duas aplicações do cliente estão hospedadas em um servidor local com Windows Server 2022, sendo executadas através do serviço IIS (Internet Information Services). A aplicação "Sistema” opera na porta 80, enquanto a aplicação "Dashboard” está configurada para a porta 8080. Ambas as aplicações se comunicam com um banco de dados SQL Server 2022 instalado em outro servidor também executando o Windows Server 2022, localizado em uma sub-rede virtual separada do servidor das aplicações. As aplicações podem ser acessadas externamente através de um endereço de IP público vinculado à máquina virtual de aplicação. Por outro lado, embora a máquina de banco de dados também possua um endereço IP público associado à sua placa de rede (NIC), não está acessível externamente por razões de segurança. Este endereço destina-se exclusivamente a possibilitar o acesso via RDP à VM para a configuração do SQL Server. Um Grupo de Segurança de Rede (NSG) foi configurado e associado às duas sub-redes para permitir exclusivamente acessos via RDP na porta 3389 às VMs e acesso HTTP nas portas 80 e 8080 à VM de aplicação. Este ambiente local é representado por uma rede virtual e duas sub-redes implementadas no Azure, localizada na região East US, conforme mostrado na imagem abaixo. Decidimos representar esse ambiente na Azure em vez de utilizar uma ferramenta de virtualização, pois a nuvem oferece uma série de vantagens em comparação com soluções locais de virtualização. Optar pelo ambiente de nuvem nos permite maior facilidade na simulação e teste de ambientes, agilizando o processo de migração e garantindo uma transição suave para a infraestrutura em nuvem.

![image](https://github.com/emifgarcia/semanaarquitetoazure/assets/80786486/0b344e22-4b8c-40a7-849a-12448240f26c)

## Motivações e Benefícios

A necessidade de migrar este ambiente "local" para a nuvem é multilateral. Primeiramente, a escalabilidade é uma demanda crucial, uma vez que o cliente deseja a flexibilidade de aumentar e diminuir os recursos computacionais conforme a demanda, sem lidar com despesas de capital (CAPEX). Além disso, a flexibilidade é fundamental, pois o cliente busca adaptar sua aplicação às oscilações e requisitos em constante mudança do seu negócio, aproveitando a ampla gama de serviços e ferramentas oferecidos pela nuvem.

Outro ponto crucial é a disponibilidade e confiabilidade. O cliente requer uma infraestrutura altamente disponível e confiável, garantindo tempos de atividade e redundância integrados, benefícios que a Azure oferece com suas garantias de serviço. Em termos de segurança, a proteção dos dados é uma prioridade, e a Azure proporciona controles avançados de segurança, criptografia de dados e conformidade com os mais rigorosos padrões de segurança internacional.

A economia de custos também é um fator determinante para a migração. Evitar despesas de capital associadas à compra e manutenção de hardware local é uma vantagem significativa, assim como pagar apenas pelos recursos efetivamente utilizados na nuvem. Com uma gestão simplificada, o cliente busca facilitar a administração do seu ambiente. Neste sentido, a Azure oferece ferramentas avançadas de gerenciamento e monitoramento que simplificam essa administração, reduzindo a sobrecarga operacional.

Por fim, a inovação contínua é um aspecto fundamental. O cliente deseja beneficiar-se dos avanços constantes nos serviços de nuvem proporcionados pela Azure, sem a preocupação com a manutenção de uma infraestrutura legada. Essa capacidade de evoluir com as mais recentes tecnologias e práticas de forma contínua e sem interrupções é um dos principais impulsionadores da migração para a nuvem.

## Visão Geral da Arquitetura Proposta

O projeto prevê a implementação e migração dos serviços e recursos de forma faseada, onde cada fase já terá uma arquitetura funcional e a fase subsequente irá aprimorar a anterior. Essa abordagem foi adotada considerando o requisito do cliente de minimizar o tempo de inatividade (downtime) ao máximo.

### Fase 1
Nesta fase, a aplicação "Sistema" será migrada do servidor local para o App Service no Azure. Os acessos a essa aplicação serão feitos de forma pública, através da internet, utilizando a URL do App Service. O App Service por sua vez, se comunicará com o banco SQL, que ainda estará no ambiente local nesta etapa, por meio da VPN Site-to-Site (S2S) e utilizando os serviços de integração de rede virtual e do VNet peering entre as redes locais, conforme ilustrado na figura 2 abaixo. Enquanto isso, a aplicação "Dashboard" permanecerá no ambiente local, mas seu acesso agora será realizado pela porta 80, uma vez que a aplicação "Sistema" terá sido migrada para a nuvem. O acesso da aplicação "Dashboard" ao banco de dados ocorrerá pela rede local (por enquanto sem mudanças), através da comunicação interna entre as sub-redes.

![image](https://github.com/emifgarcia/semanaarquitetoazure/assets/80786486/dcd6bdd1-e03a-4c70-be78-0a156ccb7a7e)

