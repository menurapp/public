# 1. Arquitetura

O objetivo deste documento é mapear o back-end atual para que seja possível fazer uma análise e restruturação para uma
arquitetura com processamento em borda, onde grande parte do back-end será migrado para uma aplicação Desktop que rodará
dentro da rede local do estabelecimento. Para mais detalhes desta proposta de arquitetura,
[acesse este link]().

## 1.1. Modelo em Camadas

A arquitetura do atual back-end hospedado na Azure, que é um monolito desenvolvido em Kotlin com Quarkus, compilado com
GraalVM, e implantado
em containers Docker, segue o modelo em camadas, onde há:

- **Camada de apresentação REST**, que fornece todos os endpoints necessários para: o **módulo do cardápio**, que é
  acessado
  pelo cliente do estabelecimento; o módulo **app mobile**, que é utilizado pelos garçons, caixa, cozinha e gestores do
  estabelecimento e;
  o **módulo de integração**, que é acessado pelos sistemas parceiros para receber os pedidos (atualmente integrado
  apenas com
  o Real Software. Atualmente esta cama contém também a implementação de regras de funcionamento do Caso de Uso.
- **Camada de negócio**, que possui regras de negócio bem definidas e que podem ser reutilizadas por mais de um
  componente da
  Camada de Apresentação. Um componente da camada de negócio pode acessar outro componente da mesma camada, ou de
  camadas inferiores,
  mas nunca da Camada de Apresentação.
- **Camada de Integração**, que é responsável por persistir, ou obter, os dados em bases de dados ou em sistemas
  externos.

## 1.2. Detalhando a Camada de Integração

Podemos dividir da Camada de Integração em duas grandes responsabilidades:

- **Persistência em Banco de Dados**: Todos os dados que sao gerados pelo MENUR são armazenados em um banco de dados
  PostgreSQL 16.
  Todas as tabelas estão mapeadas em Entidades (classes) com JPA. O acesso a estes dados são feitos por meio de Classes
  DAO
  (Data Access Object) utilizando JPQL (Java Persistence Query Language) ou SQL nativo em alguns casos. Os dados
  carregados do
  banco de dados transitam transversalmente por todas as camadas da aplicação como objetos gerenciados pelo Hibernate
  JPA, o que
  exige que as camadas entendam que qualquer manipulação na entidade gerará um UPDATE no banco de dados. Da mesma forma,
  caso
  algum atributo mapeado como LAZY seja acessado, será gerada um novo SELECT no banco de dados. Arquiteturalmente este
  comportamento
  traz muito acoplamento e falta de abstração do comportamento do Hibernate em todas as camadas. Este é um ponto de
  melhoria para
  a próxima [arquitetura proposta neste link]().
- **Client de outros sistemas**: O MENUR integra com diversos outros sistemas e plataformas para, por exemplo: emitir e
  processar
  pagamentos Pix e Cartão, permitir login com Apple ID e Google ID, enviar e-mails, SMS, notificações push, etc. Todos
  os detalhes
  de acesso aos sistemas externos estão abstraídos nesta camada utilizando o Resteasy Client. Todos os parâmetros para
  integração
  podem ser configurados via variável de ambiente, o que facilita a configuração e manutenção da aplicação na Azure.

## 1.3 Organização das pastas e packages

Talvez não seja tão óbvio fazer a relação entre as camadas e a organização das pastas e packages do back-end. Por isso,
segue
um detalhamento para melhor entendimento.

Em um **Primeiro Nível**, na raiz do projeto existe a pasta `src`, `doc` e `target`:

- `src`: Contém todo o código-fonte da aplicação, scripts e artefatos relacionados diretamente com o funcionamento e
  build
  do projeto. É a principal pasta.
- `doc`: Contém toda a documentação do projeto, como este documento.
- `target`: Contém os artefatos gerados pelo build do projeto, como os arquivos `.class`, `.jar` e demais derivados do
  build
  com Maven.

Em um **Segundo Nível**, dentro de `src`, a pasta mais importante, pode-se encontrar:

- `src/main`: Contém o código-fonte da aplicação, scripts e artefatos relacionados diretamente com o funcionamento e
  build.
- `src/test`: Deveria conter os testes unitários e de integração da aplicação, mas não é o caso do MENUR. Infelizmente
  não há testes.
- `src/etc`: Contém artefatos e scripts importantes para o desenvolvimento do MENUR, como o caso da modelagem do bando
  de dados utilizando
  o pgModeler, scripts de migração do bando de dados, etc. Não é uma pasta muito importante no dia-a-dia, na verdade.

Em um **Terceiro Nível**, dentro de `src/main/kotlin` podemos identificar as Camadas da Arquitetura:

- `src/main/kotlin/core`: Contém a Camada de Negócios (`business`), a Camada de Persistência no Banco de
  Dados (`persistence`),
  as Entidades mapeadas com JPA (`entity`) que permeiam todas as camadas de forma transversal, e exceções (`exception`)
  que são específicas da regra de negócio.
- `src/main/kotlin/client`: Toda a lógica de integração com sistemas externos está nesta pasta. Cada sistema externo
  possui uma pasta própria.
- `src/main/kotlin/rest`: É a camada de apresentação REST. Contém todos os endpoints da aplicação, versionados. Os dados
  retornados pelos endpoints estão na pasta `data`, enquanto os endpoints em si estão na pasta `service`.
- `src/main/kotlin/util`: Possui utilitários que são utilizados por mais de uma camada, como por exemplo, a classe
  `Logger` que é utilizada em todas as camadas para logar mensagens.
- `src/main/kotlin/sdk`: Contém a configuração ou a implementação de funcionalidades transversais como a monitoração do
  back-end, mecanismos de cache, controle de acesso, controle transacional, etc.
- Demais pastas: Não serão detalhas neste documento por serem muito específicas e não contribuírem significativamente
  com
  para explicação e entendimento geral da arquitetura do MENUR.

# Domínios e Sub-domínios

A plataforma MENUR possui uma complexidade relativamente alta, mas não por conta de grandes desafios tecnológicos, e sim
por conta da quantidade de funcionalidades e detalhes de implementação para garantir a melhor usabilidade possível para
os usuários (tanto os clientes quando os funcionários dos estabelecimentos), sem abrir mão da segurança dos processos e
dos dados.

Para ter uma visão geral das Funcionalidades do MENUR, os domínios e sub-domínios que fazem parte do atual back-end
hospedado na Azure:

- **Cardápio**: Sob a ótica do cliente dos estabelecimentos, é por onde tudo começa.
    - **Exibição do Cardápio**: Responsável por resgatar os dados que compõem o cardápio, como categorias
      produtos, fotos, opções, observações, personalizações de layout estabelecimento (ex: logotipo e cor), ocultamento
      de itens em
      falta, e escolha do idioma correto. Estes endpoints são acessados pelo cardápio que o cliente dos estabelecimentos
      acessam usando seus próprios celulares.
    - **Cadastro dos Produtos**: Possibilita o cadastro de novos produtos ou mudanças nos existentes. Engloba desde o
      cadastro e ordem de exibição das categorias, produtos, fotos, opções, preços e código de integração com sistemas
      de gestão
      integrados (atualmente apenas o Real Software). Estes endpoints são acessados pelo app mobile MENUR.
    - **Ocultamento de Produtos**: Permite que o estabelecimento oculte produtos que estão em falta ou que não deseja
      mais
      exibir no cardápio para os clientes.


- **Gestão de Pedidos**: Sob a ótica dos funcionários dos estabelecimentos, é por onde tudo começa.
    - **Lançamento de Pedidos**: É possível que tanto o cliente quando a equipe do estabelecimento lancem pedidos na
      plataforma.
      Para o caso do auto-pedido, o próprio cliente lança usando seu próprio celular. Para ao caso do lançamento pelo
      atendente,
      é utilizado o app mobile MENUR, que é de uso exclusivo pela equipe do estabelecimento.
    - **Recebimento de Pedidos**: Responsável por receber os pedidos dos clientes, seja por meio do cardápio ou por meio
      do lançamento pelo próprio app MENUR utilizado pelos funcionários do estabelecimento. Estes endpoints são
      acessados pelo app mobile MENUR.
    - **Status dos Pedidos**: Permite que os funcionários dos estabelecimentos visualizem e modifiquem o status dos
      pedidos, desde o recebimento até a entrega. Estes endpoints são acessados pelo app mobile MENUR, porém os status
      dos pedidos são vistos também pelos clientes no cardápio.
    - **Cancelamento de Pedidos**: O cancelamento de pedidos é um processo crítico e que precisa de uma atenção
      especial,
      pois interfere na dinâmica da produção e no controle de caixa. Pedidos já recebidos pelo estabeleicmento não podem
      ser
      mais cancelados pelo próprio cliente. Pedidos que já estão no estabelecimento só podem ser cancelados utilizando o
      app
      mobile MENUR pela própria equipe do estabelecimento. Pedidos já pagos, ao serem cancelados geram um cancelamento
      do pagamento.
      Para mais detalhes sobre pagamentos e estornos, veja o domínio [Pagamentos]().


- **Identidade e Acesso**: Tem a grande responsabilidade de manter a plataforma em segurança, gerenciar o acesso e
  garantir
  a confidencialidade dos dados.
    - **Convites para novos funcionários**: Para acessar todas as funcionalidades do app mobile MENUR, os funcionários
      precisam
      ter acesso ao app. Os convites podem ser feitos por outros funcionários que já tem acesso ao app, respeitando o
      nível de acesso. Quando o convite é feito, o novo funcionário recebe um e-mail com um link para baixar o app e são
      instruidos a criar sua própria senha de acesso.
    - **Nível de acesso dos funcionários**: Nem todos os funcionários podem acessar todas as informações ou ter acesso a
      todas
      as funcionalidades. O nível de acesso é controlado por um administrador do estabelecimento, que pode ser um
      funcionário ou
      o próprio proprietário. Um funcionário não pode modificar e nem conceder um nível de acesso mais alto
      ao seu próprio nível. Alguns níveis podem ter acesso a informações financeiras, cancelamento de pedidos e
      configuração de
      contas e meios de pagamento, por exemplo, enquanto outros não. Estes endpoints são acessados pelo app mobile
      MENUR.
    - **Login dos funcionários**: Para acessar o app mobile MENUR, os funcionários precisam fazer login com seu e-mail e
      senha
      diretamente no app mobile MENUR. Em caso de esquecimento de senha, o usuário pode solicitar o reset e um e-mail
      com as
      instruções para criar uma nova senha é enviado para o e-mail cadastrado. Estes endpoints são acessados pelo app
      mobile MENUR.
    - **Login dos clientes**: Quando um cliente vai fazer o seu primeiro pedido via auto-atendimento no cardápio
      utilizando
      seu próprio celular, ele precisa fazer uma identificação básica. Para o login via cardápio, o MENUR integra com o
      Google ID
      e com o Apple ID. Estes endpoints são acessados pelo cardápio.
    - **Acesso dos integradores**: O MENUR não faz controle de estoque, nem composição de produtos para ficha técnica,
      nem possui funcionalidades de controle fiscal, nem módulo de compras de insumos, nem relatórios gerenciais.
      Para isso, o MENUR disponibiliza
      uma API de integração aos moldes da API do iFood e Open Delivery da Abrasel, possibilitando que sistemas de gestão
      puxem os dados de pedidos e pagamento do MENUR. Para ter acesso aos dados da API do estabelecimento, é preciso
      gerar uma chave de acesso para o integrador.


- **Pagamento**: o MENUR disponibiliza como um grande diferencial a possibilidade de auto-pagamento utilizando Pix ou
  até
  mesmo Google Pay e Apple Pay, com baixa e conciliação automática. Além disso, é possível utilizar os meios de
  pagamentos
  tradicionais não integrados e, em seguida, lançar registrar manualmente o pagamento no app mobile MENUR par controle.
    - **Pagamento com Pix**: Como uma das alternativas de auto-pagamento, o MENUR disponibiliza a opção de pagamento com
      Pix. O cliente inicializa o pagamento no cardápio, copia o código Copia&Cola e faz o pagamento no seu próprio
      aplicativo
      de banco. O pagamento é confirmado automaticamente no cardápio, e sinalizado para a equipe do estabelecimento no
      app
      mobile MENUR. Todo o processo é intermediado pelo Banco Central e quando o pagamento é confirmado, o banco parceiro,
      a Efí Bank, que processa os pagamentos Pix envia uma notificação para o MENUR.
    - **Pagamento com Carteira Digital**: O MENUR integra com o Google Pay e Apple Pay para permitir que o cliente faça
      o pagamento utilizando a carteira digital do seu próprio celular. O cliente inicia o auto-pagamento no cardápio,
      e escolhe o cartão de crédito cadastrado no Google Pay ou Apple Pay. O pagamento é confirmado automaticamente no
        cardápio, e sinalizado para a equipe do estabelecimento no app mobile MENUR. Todo o processo é intermediado pela
        plataforma de pagamentos parceira, a Stripe Payments.
    - **Registro Manual de Pagamento**: Nem sempre o cliente do estabelecimento vai querer fazer o auto-pagamento. Para
      isso, o
      estabelecimento pode receber o pagamento da forma tradicional (utilizando a máquina de cartão de crédito, ou em dinheiro, por ex)
      e, em seguida, registrar manualmente no app mobile MENUR. O pagamento é confirmado automaticamente
      no cardápio, e sinalizado para toda equipe do estabelecimento no app mobile MENUR.

- **Sincronização**: Escrever...

- **Sincronização**: Escrever...

- **Mesas**: Escrever...

- **QR Codes**: Escrever...      

- **Integração**: Escrever...

- **Gestão**: Escrever...
