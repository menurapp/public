# 1. Arquitetura

O objetivo deste documento é mapear o back-end atual para permitir a análise e a reestruturação
para uma arquitetura com processamento em borda, na qual grande parte do back-end será migrada
para uma aplicação Desktop que rodará dentro da rede local do estabelecimento. Para mais detalhes
desta proposta de arquitetura, [acesse este link]().

## 1.1. Modelo em Camadas

A arquitetura do back-end atual, hospedado na Azure, é um monolito desenvolvido em Kotlin com
Quarkus, compilado com GraalVM e implantado em contêineres Docker. Ela segue o modelo em camadas,
no qual temos:

- **Camada de apresentação REST**: Responsável por fornecer todos os endpoints necessários para o
  **módulo de cardápio**, acessado pelo cliente do estabelecimento; para o **módulo do app mobile**,
  usado por garçons, caixa, cozinha e gestores; e para o **módulo de integração**, utilizado por
  sistemas parceiros para receber pedidos (atualmente, integrado apenas com o Real Software).
  Atualmente, esta camada também implementa regras de funcionamento do Caso de Uso.

- **Camada de negócio**: Possui regras de negócio bem definidas e reutilizáveis por mais de um
  componente da camada de apresentação. Um componente dessa camada pode acessar outro da mesma
  camada ou de camadas inferiores, mas nunca a camada de apresentação.

- **Camada de Integração**: Responsável por persistir ou obter dados em bases de dados ou em
  sistemas externos.

## 1.2. Detalhando a Camada de Integração

Podemos subdividir a camada de integração em duas grandes responsabilidades:

- **Persistência em Banco de Dados**: Todos os dados gerados pelo MENUR são armazenados em um
  banco de dados PostgreSQL 16. Todas as tabelas estão mapeadas em entidades (classes) com JPA.
  O acesso a esses dados é feito por meio de classes DAO (Data Access Object), utilizando JPQL
  (Java Persistence Query Language) ou SQL nativo em alguns casos. Os dados carregados do banco
  transitam transversalmente por todas as camadas da aplicação como objetos gerenciados pelo
  Hibernate JPA. Isso exige que as camadas entendam que qualquer manipulação na entidade gerará
  um UPDATE no banco. Da mesma forma, se um atributo mapeado como LAZY for acessado, um novo
  SELECT será executado. Arquiteturalmente, esse comportamento traz muito acoplamento e falta de
  abstração do Hibernate em todas as camadas, sendo um ponto de melhoria para a próxima
  [arquitetura proposta neste link]().

- **Client de outros sistemas**: O MENUR integra com diversos sistemas e plataformas para, por
  exemplo, emitir e processar pagamentos Pix e cartão, permitir login com Apple ID e Google ID,
  enviar e-mails, SMS e notificações push, entre outros. Todos os detalhes de acesso a sistemas
  externos estão abstraídos nesta camada, usando o Resteasy Client. Os parâmetros de integração
  podem ser configurados via variáveis de ambiente, o que facilita a configuração e manutenção
  da aplicação na Azure.

## 1.3 Organização das pastas e packages

Talvez não seja óbvia a relação entre as camadas e a organização das pastas e packages do back-end.
Por isso, segue um detalhamento para melhor entendimento.

Em um **Primeiro Nível**, na raiz do projeto, existem as pastas `src`, `doc` e `target`:

- `src`: Contém todo o código-fonte da aplicação, além de scripts e artefatos relacionados
  diretamente ao funcionamento e build do projeto. É a pasta principal.
- `doc`: Contém toda a documentação do projeto, incluindo este documento.
- `target`: Contém os artefatos gerados pelo build do projeto, como arquivos `.class`, `.jar`
  e demais derivados do build com Maven.

Em um **Segundo Nível**, dentro de `src`, que é a pasta mais importante, encontramos:

- `src/main`: Contém o código-fonte da aplicação, scripts e artefatos relacionados ao seu
  funcionamento e build.
- `src/test`: Deveria conter testes unitários e de integração, mas no MENUR não há testes
  implementados.
- `src/etc`: Reúne artefatos e scripts importantes para o desenvolvimento do MENUR, como a
  modelagem do banco de dados com pgModeler e scripts de migração. Não é muito relevante no
  dia a dia.

Em um **Terceiro Nível**, dentro de `src/main/kotlin`, podemos identificar as camadas da arquitetura:

- `src/main/kotlin/core`: Abriga a camada de negócios (`business`), a camada de persistência
  (`persistence`), as entidades JPA (`entity`), que permeiam todas as camadas de forma transversal,
  e as exceções (`exception`) específicas da regra de negócio.
- `src/main/kotlin/client`: Toda a lógica de integração com sistemas externos fica aqui. Cada
  sistema possui sua própria pasta.
- `src/main/kotlin/rest`: Camada de apresentação REST. Contém todos os endpoints da aplicação,
  versionados. Os dados retornados estão em `data`, enquanto os endpoints em si estão em `service`.
- `src/main/kotlin/util`: Reúne utilitários usados por mais de uma camada, como a classe `Logger`,
  empregada em todas as camadas para registro de logs.
- `src/main/kotlin/sdk`: Contém a configuração ou implementação de funcionalidades transversais,
  como monitoramento do back-end, cache, controle de acesso e transações, entre outros.
- Demais pastas: Não serão detalhadas neste documento, pois são muito específicas e não contribuem
  significativamente para o entendimento geral da arquitetura do MENUR.

# Domínios e Sub-domínios

A plataforma MENUR possui complexidade relativamente alta, não por grandes desafios tecnológicos,
mas pela quantidade de funcionalidades e detalhes de implementação para garantir a melhor
usabilidade possível — tanto para clientes quanto para funcionários — sem abrir mão da segurança
dos processos e dados.

Para se ter uma visão geral das funcionalidades do MENUR, listamos os domínios e subdomínios que
fazem parte do back-end atual hospedado na Azure:

- **Cardápio**: Sob a ótica do cliente, é por onde tudo começa.
    - **Exibição do Cardápio**: Responsável por buscar os dados que compõem o cardápio, como categorias,
      produtos, fotos, opções, observações, personalizações de layout (por exemplo, logotipo e cor),
      ocultamento de itens em falta e escolha do idioma correto. Esses endpoints são acessados pelos
      clientes no próprio celular.
    - **Cadastro dos Produtos**: Permite criar novos produtos ou alterar os existentes, incluindo
      categorias, fotos, preços e códigos de integração com sistemas de gestão (atualmente, apenas o
      Real Software). Esses endpoints são acessados pelo app mobile MENUR.
    - **Ocultamento de Produtos**: Possibilita ao estabelecimento ocultar produtos em falta ou que
      não devam mais ser exibidos no cardápio para os clientes.

- **Gestão de Pedidos**: Sob a ótica dos funcionários do estabelecimento, é por onde tudo começa.
    - **Lançamento de Pedidos**: Tanto o cliente quanto a equipe podem lançar pedidos. No
      autoatendimento, o cliente faz isso diretamente no cardápio via celular. No caso do atendente,
      é usado o app mobile MENUR, exclusivo para a equipe do estabelecimento.
    - **Recebimento de Pedidos**: Recebe os pedidos enviados pelo cardápio ou pelo próprio app MENUR,
      utilizados pelos funcionários. Esses endpoints são acessados pelo app mobile MENUR.
    - **Status dos Pedidos**: Permite visualizar e alterar o status, do recebimento até a entrega.
      Esses endpoints são acessados pelo app mobile MENUR, mas o status também é visível para os
      clientes no cardápio.
    - **Cancelamento de Pedidos**: O cancelamento é um processo crítico que afeta produção e
      controle de caixa. Pedidos recebidos não podem mais ser cancelados pelo cliente e, se já
      estiverem em posse do estabelecimento, só podem ser cancelados pelo app mobile MENUR. Pedidos
      pagos geram um cancelamento do pagamento. Para mais detalhes, veja [Pagamentos]().

- **Identidade e Acesso**: Garante a segurança da plataforma, gerenciando acesso e confidencialidade.
    - **Convites para novos funcionários**: Para usar o app mobile MENUR, funcionários precisam de
      convite de alguém com nível de acesso superior ou igual. Ao ser convidado, o novo funcionário
      recebe um e-mail com link para baixar o app e criar senha.
    - **Nível de acesso dos funcionários**: Nem todos podem ver todas as informações ou ter acesso
      a todas as funcionalidades. O nível de acesso é controlado por um administrador (funcionário
      ou proprietário). Um funcionário não pode conceder a si mesmo ou a outros um nível acima do
      seu. Alguns níveis têm acesso a dados financeiros, cancelamento de pedidos e configurações
      de pagamento, enquanto outros não. Esses endpoints são acessados pelo app mobile MENUR.
    - **Login dos funcionários**: Para acessar o app, é preciso login com e-mail e senha. Em caso de
      esquecimento, é possível resetar a senha, e um e-mail com instruções é enviado. Esses endpoints
      são acessados pelo app mobile MENUR.
    - **Login dos clientes**: Antes de fazer o primeiro pedido via autoatendimento, o cliente precisa
      se identificar. Para login via cardápio, o MENUR integra com Google ID e Apple ID. Esses
      endpoints são acessados diretamente pelo cardápio.
    - **Acesso dos integradores**: O MENUR não faz controle de estoque, ficha técnica, controle fiscal,
      compras de insumos ou relatórios gerenciais. Em vez disso, oferece uma API de integração nos
      moldes do iFood e Open Delivery da Abrasel, permitindo que sistemas de gestão puxem dados de
      pedidos e pagamentos. Para acessar a API de um estabelecimento, é preciso gerar uma chave
      de acesso para o integrador.

- **Pagamento**: O MENUR oferece a possibilidade de auto-pagamento por Pix, Google Pay e Apple Pay,
  com baixa e conciliação automática. Também é possível usar métodos tradicionais (não integrados)
  e depois registrar manualmente o pagamento.
    - **Pagamento com Pix**: Opção de auto-pagamento em que o cliente copia o código Copia&Cola e faz
      o pagamento no próprio app bancário. A confirmação ocorre automaticamente no cardápio e no app
      mobile MENUR. O processo é intermediado pelo Banco Central e, ao ser confirmado, o banco
      parceiro (Efí Bank) notifica o MENUR.
    - **Pagamento com Carteira Digital**: Integração com Google Pay e Apple Pay. O cliente inicia o
      pagamento no cardápio e escolhe o cartão cadastrado na carteira digital. A confirmação ocorre
      no cardápio e no app mobile MENUR. A Stripe Payments intermedia esse processo.
    - **Registro Manual de Pagamento**: Caso o cliente não use o auto-pagamento, o estabelecimento
      pode receber por meios tradicionais (máquina de cartão ou dinheiro, por exemplo) e registrar
      manualmente no app mobile MENUR. A confirmação é sinalizada para o cardápio e para toda a
      equipe.

- **Sincronização**: Escrever...

- **Sincronização**: Escrever...

- **Mesas**: Escrever...

- **QR Codes**: Escrever...

- **Integração**: Escrever...

- **Gestão**: Escrever...
