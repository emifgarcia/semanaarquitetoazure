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

### Fase 2
Nesta fase, estamos planejando migrar o banco de dados SQL Server local, atualmente instalado na VM Windows Server, para o Azure como um serviço PaaS, utilizando o SQL Server e o SQL Database. É importante destacar que durante essa migração, não será possível manter um banco de dados no Azure simultaneamente ao banco local. Isso ocorre porque escrever dados em ambos os bancos ao mesmo tempo pode causar problemas durante a fusão, potencialmente resultando na perda de dados importantes.

Para minimizar o tempo de inatividade, nossa estratégia será criar um novo banco de dados no Azure e, em seguida, migrar os dados do banco local para ele. No entanto, é importante ressaltar que mesmo com essa abordagem, haverá um período de inatividade, embora seja mantido o mais curto possível.

Vamos iniciar criando os recursos SQL Database e SQL Server no Azure. Neste estágio inicial, o banco ainda não está acessível publicamente, pois, por padrão, o Azure bloqueia o acesso público aos bancos e não há uma interface de rede (NIC) anexada a ele. Isso é normal, pois os recursos PaaS no Azure não possuem NICs criadas automaticamente.

Para tornar o banco acessível, iremos criar um endpoint privado dentro da subnet da VM temporária para permitir a comunicação com o SQL Server. Ao criarmos um endpoint privado, uma zona DNS privada é gerada, já que o endpoint precisa ser chamado pelo nome do recurso, não pelo IP. Isso garante que o acesso ao nosso banco de dados seja resolvido para um endereço IP privado, não público.

Posteriormente, iremos adicionar dois "virtual links" para as VNets na zona DNS privada apontando para as Vnets locais: um para a VNet da aplicação e outro para a VNet do firewall. Isso permitirá que essas redes consultem o serviço de DNS privado antes de enviar tráfego para o Azure SQL Server.

Em seguida, realizaremos a instalação e configuração do Azure Data Migration Assistant na máquina local (SQL) para realizar uma avaliação e migração do banco para o Azure. Finalmente, atualizaremos a connection string para que as aplicações "Sistema” e "Dashboard" se conectem ao banco no Azure, em vez do banco local. Após essas etapas, poderemos desativar o banco local, pois o SQL Database estará ativo e operacional. As etapas descritas acima estão representadas na arquitetura ilustrada na imagem abaixo:

![image](https://github.com/emifgarcia/semanaarquitetoazure/assets/80786486/062bf0ab-c0c4-4867-8531-0d059fab32f5)

### Fase 3
Na fase anterior, acessávamos nossas aplicações tanto através do App Service (Sistema) quanto da VM App localizada no ambiente on-premises (Dashboard). Ambas as aplicações estavam expostas à internet sem nenhum mecanismo de proteção, deixando-as vulneráveis a ataques e vazamento de dados. Para solucionar esse problema, na fase 3, implementaremos uma camada de segurança para nossas aplicações utilizando o Application Gateway, que oferece suporte a um Web Application Firewall (WAF), protegendo-as contra uma variedade de ameaças comuns à segurança na web, como ataques de injeção de SQL, cross-site scripting (XSS) e outros ataques de aplicativos da web.

Assim, as requisições serão encaminhadas para o Application Gateway, que direcionará o tráfego para o backend correspondente. Configuraremos regras para redirecionar requisições na porta 80 para a porta 443 HTTPS, utilizando a URL do App Service para acessar a aplicação do Sistema, e na porta 8080 para a porta 80 HTTP, para acessar a aplicação do Dashboard na VM local.

Após a implementação da fase 3, as aplicações ainda estarão acessíveis diretamente pelo App Service e pelo endereço público da VM App, conforme ilustrado nas linhas em vermelho na imagem abaixo. Entretanto, buscamos evitar essa exposição direta, e na fase 4 realizaremos ajustes de segurança para garantir que as aplicações só estejam acessíveis através do Application Gateway.

![image](https://github.com/emifgarcia/semanaarquitetoazure/assets/80786486/90ff4102-4a30-42f2-8d26-63a26d6fbf3c)

### Fase 4
Nesta etapa, nosso objetivo é permitir que os usuários acessem exclusivamente as aplicações por meio do Application Gateway, em vez de acessá-las diretamente pelo App Service ou pelo endereço IP público da VM App. Isso é necessário porque, como discutido anteriormente, as aplicações não devem ser expostas publicamente sem uma camada de segurança intermediária, como o Application Gateway.

Para alcançar esse objetivo, iniciaremos desabilitando o acesso público ao App Service e criando um private endpoint para ele na subnet cltec-ti-snet-saa-prod-uksouth-001. Dessa forma, todo o tráfego destinado ao App Service será redirecionado para seu endereço privado. Em seguida, desassociaremos o endereço IP público da NIC da VM App, restringindo o acesso à aplicação "Dashboard" exclusivamente via Application Gateway.

Como último passo, configuraremos o Application Gateway para ouvir nas portas 80 e 8080 e redirecionar o tráfego recebido na porta 80 para o App Service, enquanto o tráfego recebido na porta 8080 será direcionado para a VM interna de App com o endereço IP 10.0.1.4.

![image](https://github.com/emifgarcia/semanaarquitetoazure/assets/80786486/ec3e8185-34dd-4d6b-9261-43c52d502abc)

## Componentes da Arquitetura

Nesta seção, iremos destacar os principais componentes da nova arquitetura e os motivos que nos levaram a escolher cada um deles em relação aos seus concorrentes no Azure.

| RECURSO | FUNÇÃO | VANTAGENS |
|----------|----------|----------|
| App Service  | Hospedar e executar a aplicação "Sistema" na nuvem Azure.  | **Gerenciamento simplificado:** gerencia automaticamente muitos aspectos da infraestrutura. Isso reduz significativamente a carga de trabalho administrativa em comparação com a gestão manual de uma máquina virtual. **Escalabilidade automática:** facilita a escalabilidade automática da aplicação com base na demanda. Isso significa que a aplicação pode lidar com picos de tráfego sem intervenção manual, garantindo alta disponibilidade e desempenho consistente. **Foco no desenvolvimento:** os desenvolvedores podem se concentrar mais no desenvolvimento de aplicativos e menos na configuração e gerenciamento da infraestrutura. Isso acelera o ciclo de desenvolvimento e permite lançar novas funcionalidades mais rapidamente. **Integração com outros serviços do Azure:** integra perfeitamente com outros serviços do Azure, como bancos de dados, armazenamento, autenticação e monitoramento. **Fácil implantação e atualização:** oferece várias opções para implantar e atualizar aplicativos, incluindo integração contínua/desenvolvimento contínuo (CI/CD) e implantação com um clique.  |
| Virtual Network Gateway  | Estabelecer uma conexão segura e privada entre o ambiente de nuvem e o ambiente local.  | **Conectividade de baixo custo:** em comparação com o Express Route, que requer um circuito de rede dedicado e pago, o VPN S2S é uma opção mais econômica para o projeto, visto que não requer largura de banda extremamente alta ou requisitos específicos de latência. **Flexibilidade geográfica:** pode ser implementado em qualquer região do Azure. **Implementação mais rápida:** a configuração e implementação de uma conexão VPN S2S geralmente são mais rápidas e simples em comparação com a configuração de um circuito Express Route, que pode exigir coordenação com provedores de telecomunicações e implantação física de hardware adicional. **Conectividade baseada na Internet:** utiliza a infraestrutura de Internet pública para estabelecer a conexão entre a rede local e o Azure. Para o projeto, a conectividade baseada na Internet é suficiente e mais conveniente, visto que o cliente já usa a Internet para outras comunicações.  |
| Network Virtual Appliance | Estabelecer uma conexão VPN com o Virtual Network Gateway do Azure para fornecer uma conexão segura entre a rede Azure e a rede local. | **Integração com a infraestrutura existente:** será facilmente integrada à infraestrutura existente da nuvem. **Customização e controle total:** a VM oferece controle total sobre a configuração e personalização do firewall de acordo com as necessidades específicas de segurança do cliente. **Economia de custos:** para nosso caso de uso, a VM será mais econômico do que adquirir um appliance de firewall dedicado.
| Vnet Peering | Fornecer conectividade direta, baixa latência, segurança e simplicidade para  a comunicação entre os recursos hospedados nas VNets de App e Firewall. | **Facilidade de configuração:** é mais fácil de configurar do que uma VPN Vnet-Vnet, pois envolve simplesmente estabelecer uma conexão direta entre duas VNets dentro da mesma região do Azure. **Latência mais baixa:** como VNet Peering estabelece uma conexão direta entre VNets dentro da mesma região do Azure, geralmente oferece menor latência em comparação com a VPN entre VNets. **Escalabilidade:** é altamente escalável e pode ser facilmente configurado para conectar múltiplos VNets dentro da mesma região do Azure. **Maior largura de banda:** A largura de banda de uma conexão VPN entre VNets pode ser limitada pela capacidade do Gateway VPN e pela largura de banda da conexão de Internet subjacente.
| Application Gateway | Fornecer uma camada de proteção para as aplicações "Sistema" e "Dashboard". | **Proteção contra ameaças na camada de Aplicação:** O Application Gateway WAF, combinado com as regras de segurança da Rule-WAF, oferece uma defesa robusta contra uma ampla variedade de ataques na camada de aplicação, como injeção de SQL, cross-site scripting (XSS), falsificação de solicitação entre sites (CSRF) e outros ataques comuns.

## Padrão de Taxonomia

Esta seção apresenta o padrão de nomenclatura proposto para os recursos e componentes relacionados à migração da aplicação "Sistema” e do banco de dados do ambiente on-premise para a plataforma Azure. O objetivo deste padrão é estabelecer uma estrutura consistente e compreensível para identificar e categorizar os diversos elementos da arquitetura na nuvem. É essencial que todos os envolvidos na migração da aplicação compreendam e adotem este padrão de nomenclatura, a fim de facilitar a comunicação, o gerenciamento e a manutenção dos recursos na Azure. O padrão proposto adota as melhores práticas de nomenclatura, seguindo convenções amplamente aceitas, além de atender às necessidades específicas e exigências de conformidade de  do cliente. É importante ressaltar que este padrão de nomenclatura é flexível e pode ser ajustado para atender às necessidades específicas da aplicação, das equipes e das políticas organizacionais. No entanto, recomenda-se que qualquer modificação seja documentada e comunicada adequadamente a todas as partes interessadas. A seguir, apresentamos os principais elementos do padrão de nomenclatura, juntamente com exemplos ilustrativos para facilitar a compreensão e a aplicação prática deste padrão para o projeto.

![image](https://github.com/emifgarcia/semanaarquitetoazure/assets/80786486/335146ec-0155-4def-8a8b-5b23bac8bc84)

| Componente de nomenclatura | Descrição |
|----------------------------|------------|
| Organização* | Nome de nível superior da organização, normalmente utilizado como o grupo de gerenciamento superior ou, em organizações menores, parte da convenção de nomenclatura. Exemplo: `cltec` |
| Unidade de negócio ou departamento* | Divisão de nível superior da sua empresa proprietária da assinatura ou da carga de trabalho à qual o recurso pertence. Em organizações menores, esse componente pode representar um único elemento organizacional corporativo de nível superior. Exemplos: `fin`, `mktg`, `product`, `it`, `corp`|
| Tipo de recurso* | Uma abreviação que representa o tipo de recurso ou ativo do Azure. Esse componente geralmente é um prefixo ou sufixo no nome. Exemplos: `rg`, `vm`|
| Nome do projeto, aplicativo ou serviço* | Nome de um projeto, aplicativo ou serviço do qual o recurso faz parte. Exemplos: `navigator`, `emissions`, `sharepoint`, `hadoop` |
| Ambiente* | A fase do ciclo de vida de desenvolvimento da carga de trabalho compatível com o recurso. Exemplos: `prod`, `dev`, `qa`, `stage`, `test` |
| Localidade* | A região ou o provedor de nuvem onde o recurso está implantado. Exemplos: `westus`, `eastus2`, `westeu`, `usva`, `ustx` |
| Função | Identificador da finalidade do recurso ou alguma indicação de onde o recurso está anexado. Exemplos: `db (banco de dados)`, `ws (servidor web)`, `ps (servidor de impressão)` |
| Instância* | A contagem de instâncias para um recurso específico, para diferenciá-lo de outros recursos que têm a mesma convenção de nomenclatura e componentes de nomenclatura. Exemplos, `01`, `001` |

## Estimativa de precificação

A seguir, apresentamos uma estimativa de precificação para a migração da aplicação "Sistema" do ambiente on-premise para a plataforma Azure. Essa estimativa foi desenvolvida com base em uma análise detalhada dos requisitos da aplicação, da arquitetura proposta e dos preços dos serviços da Azure. É importante observar que a estimativa de precificação apresentada abaixo é uma projeção aproximada e pode variar com base em diversos fatores, como o consumo real de recursos, configurações específicas, descontos aplicáveis, flutuações nos preços do mercado e mudanças nas estratégias de implementação ao longo do tempo. Além disso, esta estimativa foi elaborada com base em dados disponíveis no momento da sua criação e está sujeita a alterações de acordo com as políticas de preços e ofertas futuras da Azure.


<img width="1264" alt="image" src="https://github.com/emifgarcia/semanaarquitetoazure/assets/80786486/2423bac3-761f-4370-ad76-723363b01af9">

## Validação CPSDM

Fatores:
`Custo` peso=2
`Performance` peso=1.5
`Seguro` peso=1.5
`Disponível` peso=1
`Moderno` peso=1

Pontuações:

1 = Não aderente

2 = Parcialmente aderente

3 = Totalmente aderente

Resultados:

21 - 15 recurso altamente aderente às necessidades do cliente

14 - 12 recurso moderadamente aderente às necessidades do cliente

07 - 12 recurso não aderente às necessidades do cliente

Avaliação do CPSDM para o Projeto:


