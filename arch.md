# 1. Arquitetura

O objetivo deste documento é mapear o back-end atual para permitir sua análise e futura
reestruturação rumo a uma arquitetura com processamento em borda. Nesse novo modelo, boa parte
do back-end será migrada para uma aplicação Desktop que rodará dentro da rede local do
estabelecimento. Para mais detalhes desta proposta de arquitetura, [acesse este link]().

## 1.1. Modelo em Camadas

A arquitetura atual do back-end, hospedado na Azure, é um monolito desenvolvido em Kotlin com
Quarkus, compilado com GraalVM e implantado em contêineres Docker. Ela segue o modelo em camadas,
no qual temos:

- **Camada de apresentação REST**: Responsável por fornecer todos os endpoints necessários para
  o **módulo de cardápio** (acessado pelo cliente do estabelecimento), para o **módulo do app
  mobile** (usado por garçons, caixa, cozinha e gestores) e para o **módulo de integração**
  (utilizado por sistemas parceiros para receber pedidos, atualmente integrado apenas com o
  Real Software). Hoje, esta camada também implementa regras de negócio que, idealmente,
  pertenceriam ao Caso de Uso.

- **Camada de negócio**: Possui regras de negócio bem definidas e reutilizáveis por mais de um
  componente da camada de apresentação. Um componente dessa camada pode acessar outro da mesma
  camada ou de camadas inferiores, mas nunca a camada de apresentação.

- **Camada de Integração**: Responsável por persistir ou obter dados em bases de dados ou em
  sistemas externos.

## 1.2. Detalhando a Camada de Integração

Podemos subdividir a camada de integração em duas grandes responsabilidades:

- **Persistência em Banco de Dados**: Todos os dados gerados pelo MENUR são armazenados em um
  banco de dados PostgreSQL 16. As tabelas estão mapeadas em entidades (classes) com JPA, e o
  acesso ocorre por meio de classes DAO (Data Access Object), usando JPQL ou SQL nativo em
  alguns casos. Os dados carregados do banco transitam por todas as camadas como objetos
  gerenciados pelo Hibernate JPA. Isso exige que as camadas entendam que qualquer manipulação
  na entidade gerará um `UPDATE` no banco. Da mesma forma, se um atributo mapeado como `LAZY`
  for acessado, um novo `SELECT` será executado. Arquiteturalmente, esse comportamento provoca
  muito acoplamento e expõe o Hibernate a todas as camadas, sendo um ponto de melhoria para a
  próxima [arquitetura proposta neste link]().

- **Cliente de outros sistemas**: O MENUR integra com diversos sistemas e plataformas para, por
  exemplo, emitir e processar pagamentos (Pix e cartão), permitir login com Apple ID e Google ID,
  enviar e-mails, SMS e notificações push, entre outros. Todos os detalhes de acesso a sistemas
  externos estão centralizados nesta camada, usando o Resteasy Client. Os parâmetros de
  integração podem ser configurados via variáveis de ambiente, o que facilita a configuração e
  manutenção da aplicação na Azure.

## 1.3. Organização das Pastas e Packages

Talvez não seja óbvia a relação entre as camadas e a organização das pastas e packages do
back-end. Por isso, segue um detalhamento para melhor entendimento.

Em um **Primeiro Nível**, na raiz do projeto, existem as pastas `src`, `doc` e `target`:

- `src`: Contém todo o código-fonte da aplicação, além de scripts e artefatos relacionados
  diretamente ao funcionamento e build do projeto. É a pasta principal.
- `doc`: Contém toda a documentação do projeto, incluindo este documento.
- `target`: Contém os artefatos gerados pelo build do projeto, como arquivos `.class`, `.jar` e
  demais derivados do build com Maven.

Em um **Segundo Nível**, dentro de `src`, que é a pasta mais importante, encontramos:

- `src/main`: Contém o código-fonte da aplicação, scripts e artefatos relacionados ao seu
  funcionamento e build.
- `src/test`: Deveria conter testes unitários e de integração, mas no MENUR não há testes
  implementados.
- `src/etc`: Reúne artefatos e scripts importantes para o desenvolvimento do MENUR, como a
  modelagem do banco de dados com pgModeler e scripts de migração. Não é muito relevante no
  dia a dia.

Em um **Terceiro Nível**, dentro de `src/main/kotlin`, podemos identificar as camadas da
arquitetura:

- `src/main/kotlin/core`: Abriga a camada de negócios (`business`), a camada de persistência
  (`persistence`), as entidades JPA (`entity`), que permeiam todas as camadas de forma
  transversal, e as exceções (`exception`) específicas da regra de negócio.
- `src/main/kotlin/client`: Toda a lógica de integração com sistemas externos fica aqui. Cada
  sistema possui sua própria pasta.
- `src/main/kotlin/rest`: Camada de apresentação REST. Contém todos os endpoints da aplicação,
  versionados. Os dados retornados estão em `data`, enquanto os endpoints em si estão em
  `service`.
- `src/main/kotlin/util`: Reúne utilitários usados por mais de uma camada, como a classe
  `Logger`, empregada em todas as camadas para registro de logs.
- `src/main/kotlin/sdk`: Contém a configuração ou implementação de funcionalidades transversais,
  como monitoramento do back-end, cache, controle de acesso e transações, entre outros.
- Demais pastas: Não serão detalhadas neste documento, pois são muito específicas e não
  contribuem significativamente para o entendimento geral da arquitetura do MENUR.

# Domínios e Sub-domínios

A plataforma MENUR possui complexidade relativamente alta, não por grandes desafios tecnológicos,
mas pela quantidade de funcionalidades e detalhes de implementação que visam oferecer a melhor
usabilidade possível — tanto para clientes quanto para funcionários — sem abrir mão da segurança
dos processos e dados.

Para ter uma visão geral das funcionalidades do MENUR, listamos os domínios e subdomínios que
fazem parte do back-end atual hospedado na Azure:

- **Cardápio**: Sob a ótica do cliente, é por onde tudo começa.

  - **Exibição do Cardápio**: Responsável por buscar dados que compõem o cardápio, como categorias,
    produtos, fotos, opções, observações, personalizações de layout (por exemplo, logotipo e cor),
    ocultamento de itens em falta e escolha do idioma correto. Esses endpoints são acessados pelos
    clientes no próprio celular.
  - **Cadastro dos Produtos**: Permite criar novos produtos ou alterar os existentes, incluindo
    categorias, fotos, preços e códigos de integração com sistemas de gestão (atualmente, apenas
    o Real Software). Esses endpoints são acessados pelo app mobile MENUR.
  - **Ocultamento de Produtos**: Possibilita ao estabelecimento ocultar produtos em falta ou que
    não devam mais ser exibidos no cardápio para os clientes.

- **Pedidos**: Sob a ótica dos funcionários do estabelecimento, é por onde tudo começa.

  - **Lançamento de Pedidos**: Tanto o cliente quanto a equipe podem lançar pedidos. No
    autoatendimento, o cliente faz isso diretamente no cardápio via celular. No caso do atendente,
    é usado o app mobile MENUR, exclusivo para a equipe, pois nem sempre o cliente quer ou pode
    fazer o pedido sozinho.
  - **Recebimento de Pedidos**: Recebe pedidos enviados pelo cardápio ou pelo próprio app MENUR
    usado pelos funcionários. Esses endpoints são acessados pelo app mobile MENUR.
  - **Status dos Pedidos**: Permite visualizar e alterar o status, do recebimento até a entrega.
    Esses endpoints são acessados pelo app mobile MENUR, mas o status também é visível para os
    clientes no cardápio.
  - **Cancelamento de Pedidos**: O cancelamento é um processo crítico que afeta produção e
    controle de caixa. Pedidos recebidos não podem mais ser cancelados pelo cliente e, se já
    estiverem com o estabelecimento, só podem ser cancelados pelo app mobile MENUR. Pedidos
    pagos geram um cancelamento do pagamento. Para mais detalhes, veja [Pagamentos]().

- **Identidade e Acesso**: Garante a segurança da plataforma, gerenciando acesso e confidencialidade.

  - **Convites para novos funcionários**: Para usar o app mobile MENUR, funcionários precisam de
    um convite de alguém com nível de acesso superior ou igual. Ao ser convidado, o novo
    funcionário recebe um e-mail com link para baixar o app e criar senha.
  - **Nível de acesso dos funcionários**: Nem todos podem ver todas as informações ou acessar
    todas as funcionalidades. O nível de acesso é controlado por um administrador (funcionário
    ou proprietário). Um funcionário não pode conceder a si mesmo ou a outros um nível acima
    do seu. Alguns níveis têm acesso a dados financeiros, cancelamento de pedidos e configurações
    de pagamento, enquanto outros não. Esses endpoints são acessados pelo app mobile MENUR.
  - **Login dos funcionários**: Para acessar o app, é preciso login com e-mail e senha. Em caso
    de esquecimento, é possível resetar a senha, e um e-mail com instruções é enviado. Esses
    endpoints são acessados pelo app mobile MENUR.
  - **Login dos clientes**: Antes de fazer o primeiro pedido via autoatendimento, o cliente
    precisa se identificar. Para login via cardápio, o MENUR integra com Google ID e Apple ID.
    Esses endpoints são acessados diretamente pelo cardápio.
  - **Acesso dos integradores**: O MENUR não faz controle de estoque, ficha técnica, controle
    fiscal, compras de insumos nem relatórios gerenciais. Em vez disso, oferece uma API de
    integração nos moldes do iFood e do Open Delivery da Abrasel, para que sistemas de gestão
    obtenham dados de pedidos e pagamentos. Para acessar a API de um estabelecimento, é preciso
    gerar uma chave de acesso para o integrador.

- **Pagamento**: O MENUR oferece a possibilidade de auto-pagamento por Pix, Google Pay e Apple Pay,
  com baixa e conciliação automática. Também é possível usar métodos tradicionais (não integrados)
  e registrar manualmente o pagamento depois.

  - **Pagamento com Pix**: Opção de auto-pagamento em que o cliente copia o código Copia&Cola e
    faz o pagamento no próprio app bancário. A confirmação ocorre automaticamente no cardápio e no
    app mobile MENUR. O processo é intermediado pelo Banco Central e, ao ser confirmado, o banco
    parceiro (Efí Bank) notifica o MENUR.
  - **Pagamento com Carteira Digital**: Integração com Google Pay e Apple Pay. O cliente inicia
    o pagamento no cardápio e escolhe o cartão cadastrado na carteira digital. A confirmação
    ocorre no cardápio e no app mobile MENUR. A Stripe Payments intermedia esse processo.
  - **Registro Manual de Pagamento**: Caso o cliente não use o auto-pagamento, o estabelecimento
    pode receber por meios tradicionais (máquina de cartão ou dinheiro, por exemplo) e registrar
    manualmente no app mobile MENUR. A confirmação é sinalizada para o cardápio e para toda a
    equipe.

- **Sincronização**: O MENUR mobile segue o conceito de app local-first, com UI de
  responsividade instantânea. Para isso, todos os dados necessários precisam estar disponíveis
  localmente no app. O back-end oferece uma API REST para que os apps busquem dados atualizados
  a partir de um determinado timestamp. Para que esse serviço tenha um tempo de resposta viável,
  cada uma das mais de 30 tabelas é acessada em paralelo, e o resultado é unificado em uma única
  resposta. Por questão de segurança, nem todos os dados são disponibilizados, nem permanecem
  com a mesma estrutura. Cabe ao app mobile MENUR fazer as requisições e interpretar os dados
  para carregá-los na base local do dispositivo.

- **Mesas**: _Escrever..._

  - **Cadastro de Mesas**: Escrever..
  - **Junção de Mesas**: Escrever..
  - **Pagamento antecipado**: Escrever..

- **QR Codes**: _Escrever..._

- **Integração**: Como o MENUR não oferece funcionalidades de controle de estoque, ficha técnica,
  controle fiscal, compras de insumos nem relatórios gerenciais, estabelecimentos que não são
  MEI normalmente precisam contratar um sistema de gestão. Para que esses sistemas possam
  acessar os dados gerados no MENUR — como pedidos e pagamentos — é disponibilizada uma API de
  integração de acesso restrito aos integradores parceiros. Atualmente, o único sistema de gestão
  integrado é o Real Software. A estrutura de dados para integração segue padrões de mercado,
  mas com detalhamento avançado do status de cada item do pedido.

  - **Padrão iFood**: O iFood disponibiliza uma API muito bem documentada e amplamente usada no
    mercado por integradores parceiros. O MENUR segue o mesmo padrão para reduzir a curva de
    aprendizado dos parceiros.
  - **Padrão Open Delivery Abrasel**: A Abrasel (Associação Brasileira de Bares e Restaurantes)
    lançou uma especificação para padronizar as integrações entre sistemas de Delivery e de
    Gestão. Por enquanto, não é muito aceita pelo mercado, mas a Abrasel continua investindo para
    que o padrão seja adotado. Atualmente, não há implementação específica do Open Delivery no
    MENUR, mas vale ficar atento a esse movimento.

  > **_OBS:_** É importante destacar que o módulo de integração não contempla as integrações com
  > plataformas de pagamento, nem com Google ou Apple para autenticação, nem outras integrações
  > que o MENUR estabelece com terceiros.

- **Endereço**: Para cadastro e busca por endereços, o MENUR usa a API do Google Maps em várias
  partes da plataforma, melhorando a usabilidade ao cadastrar endereços, definir locais de
  entrega e áreas de restrição.

  - **Endereço do estabelecimento**: Para calcular a taxa de entrega, é essencial saber a
    localização do estabelecimento. Ao cadastrar esse endereço, ocorre uma pré-carga de dados
    como Estado, Cidade e Bairro na base de dados do MENUR. Esses endpoints são acessados pelo
    app mobile MENUR.
  - **Endereço do usuário**: Para calcular a taxa de entrega, também é importante conhecer o
    endereço do cliente. O cadastro dispara uma pré-carga de dados como Estado, Cidade e Bairro
    na base do MENUR. Esses endpoints são acessados pelos clientes no próprio celular.
  - **Taxas e locais de entrega**: É possível definir a área de entrega e suas respectivas taxas.
    O cadastro da área de entrega usa o Google Maps para buscar por Estado, Cidade, Bairro, Rua
    ou até mesmo um endereço específico. Da mesma forma, podem ser definidos locais restritos
    onde não há cobertura de entrega.

- **Gestão**: _Escrever..._

> **OBS**: Escrever sobre i18n
