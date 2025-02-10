- [1. Introdução](#1-introdução)
- [2. Arquitetura](#2-arquitetura)
  - [2.1. Modelo em Camadas](#21-modelo-em-camadas)
  - [2.2. Detalhando a Camada de Integração](#22-detalhando-a-camada-de-integração)
  - [2.3. Organização das Pastas e Packages](#23-organização-das-pastas-e-packages)
- [3. Domínios e Sub-domínios](#3-domínios-e-sub-domínios)
  - [3.1. Domínio Cardápio](#31-domínio-cardápio)
  - [3.2. Domínio Pedidos](#32-domínio-pedidos)
  - [3.3. Domínio Identidade e Acesso](#33-domínio-identidade-e-acesso)
  - [3.4. Domínio Pagamento](#34-domínio-pagamento)
  - [3.5. Domínio Sincronização](#35-domínio-sincronização)
  - [3.6. Domínio Mesas](#36-domínio-mesas)
  - [3.7. Domínio QR Codes](#37-domínio-qr-codes)
  - [3.8. Domínio Integração](#38-domínio-integração)
  - [3.9. Domínio Endereço](#39-domínio-endereço)
  - [3.10. Domínio Gestão](#310-domínio-gestão)
- [4. Internacionalização](#4-internacionalização)
  - [4.1. Tradução por IA](#41-tradução-por-ia)
  - [4.2. Mensagens do Back-end](#42-mensagens-do-back-end)
  - [4.3. Textos Fixos do Cardápio](#43-textos-fixos-do-cardápio)
  - [4.4. Textos Fixos do App Mobile](#44-textos-fixos-do-app-mobile)
- [5. Finalização](#5-finalização)

---

# 1. Introdução

Este documento apresenta uma visão geral do back-end do MENUR, incluindo infraestrutura,
organização de pastas e pacotes, além de domínios e subdomínios da plataforma. A compreensão
detalhada desse cenário permite analisar o modelo atual e guiar o desenvolvimento de uma
futura arquitetura, possivelmente com processamento em borda.

# 2. Arquitetura

Nesta seção, veremos como o MENUR está estruturado em um modelo em camadas e como a aplicação
se organiza internamente. Abordaremos as responsabilidades de cada camada, bem como a forma
como o código-fonte está distribuído.

## 2.1. Modelo em Camadas

A aplicação, desenvolvida em Kotlin com [Quarkus](https://quarkus.io) e compilada com
[GraalVM](https://www.graalvm.org), segue um modelo em camadas dentro de um monolito executado
em contêineres [Docker](https://www.docker.com). As camadas são:

- **Camada de apresentação REST**: Fornece endpoints para o **módulo de cardápio** (cliente do
  estabelecimento), **módulo do app mobile** (garçons, caixa, cozinha e gestores) e **módulo de
  integração** (sistemas parceiros, atualmente só o [Real Software](#38-domínio-integração)).
  Parte das regras de negócio, idealmente na camada de Caso de Uso, acaba aparecendo aqui.

- **Camada de negócio**: Agrupa as regras de negócio bem definidas, que podem ser reutilizadas
  por diferentes partes da camada de apresentação. Um componente nessa camada pode acessar
  outros na mesma camada ou em camadas inferiores, mas nunca a de apresentação.

- **Camada de Integração**: Lida com persistência em banco de dados e comunicação com sistemas
  externos. Aqui ficam os componentes de acesso a dados e as integrações com serviços de
  pagamento ([Stripe Payments](https://stripe.com), Pix, etc.), login, envio de e-mails e
  outros.

## 2.2. Detalhando a Camada de Integração

Esta camada se divide em duas grandes partes:

- **Persistência em Banco de Dados**: Todos os dados do MENUR são gravados em um PostgreSQL 16.
  As tabelas são mapeadas em entidades JPA, acessadas por classes DAO, usando JPQL ou SQL
  nativo em alguns casos. O Hibernate JPA gerencia os objetos propagados por todas as camadas,
  gerando muito acoplamento. Qualquer alteração em um objeto gerenciado dispara um `UPDATE`, e
  se um atributo `LAZY` for acessado, ocorre um novo `SELECT`. Esse acoplamento está nos planos
  de melhoria na futura
  [arquitetura proposta neste link](#2-arquitetura) (exemplo interno).

- **Cliente de outros sistemas**: Abriga integrações diversas, como pagamento (Pix, cartões),
  logins sociais (Apple ID, Google ID) e envio de comunicações (e-mail, SMS, push). Utiliza o
  Resteasy Client, configurado via variáveis de ambiente (facilitando a manutenção na
  [Azure](https://azure.microsoft.com)).

## 2.3. Organização das Pastas e Packages

Talvez não seja óbvio como as camadas se relacionam com a estrutura de pastas do back-end.
Segue um detalhamento para esclarecer essa organização:

**Primeiro Nível (raiz do projeto)**

- `src`: Código-fonte principal, scripts e artefatos de build.
- `doc`: Documentação do projeto, incluindo este documento.
- `target`: Conteúdo gerado pelos builds do Maven (arquivos `.class`, `.jar` etc.).

**Segundo Nível (dentro de `src`)**

- `src/main`: Código-fonte da aplicação e scripts essenciais para seu funcionamento e build.
- `src/test`: Deveria conter testes unitários e de integração, mas não há testes implementados.
- `src/etc`: Artefatos e scripts auxiliares, como modelagens em [pgModeler](https://pgmodeler.io)
  e scripts de migração.

**Terceiro Nível (dentro de `src/main/kotlin`)**

- `core`: Camada de negócios (`business`), persistência (`persistence`), entidades JPA (`entity`)
  e exceções (`exception`).
- `client`: Lógica de integração com sistemas externos, em pastas específicas para cada um.
- `rest`: Camada de apresentação REST, com endpoints versionados. Em `service` ficam os endpoints,
  e em `data` ficam os objetos de transporte de dados.
- `util`: Funcionalidades auxiliares (por exemplo, `Logger`), usadas em múltiplas camadas.
- `sdk`: Funcionalidades transversais, como monitoramento, cache, controle de acesso e
  transações.

Outras pastas não detalhadas aqui são específicas e não agregam muito à visão geral da
arquitetura do MENUR.

---

# 3. Domínios e Sub-domínios

O MENUR em si não apresenta grandes desafios tecnológicos, mas abrange diversas funcionalidades
para otimizar a experiência de clientes e funcionários, garantindo segurança nos processos
e dados.

A seguir, listamos os domínios e subdomínios do back-end em execução na [Azure](https://azure.microsoft.com).

## 3.1. Domínio Cardápio

Sob a ótica do cliente, é por onde tudo começa.

- **Exibição do Cardápio**: Carrega categorias, produtos, fotos, opções, observações, ajustes
  visuais (logo, cor etc.), ocultamento de itens indisponíveis e seleção de idioma. Esses
  endpoints são acessados pelos clientes em seus celulares.

- **Cadastro de Produtos**: Cria e altera produtos, incluindo categorias, fotos, preços e
  códigos de integração com sistemas de gestão (atualmente, só o Real Software). Esses endpoints
  são acessados pelo app mobile MENUR.

- **Ocultamento de Produtos**: Permite ao estabelecimento ocultar produtos em falta ou que
  não devem ser exibidos no cardápio.

## 3.2. Domínio Pedidos

Sob a ótica dos funcionários, é por onde tudo começa.

- **Lançamento de Pedidos**: Pode ser feito pelo cliente (autoatendimento via celular) ou pela
  equipe (app mobile MENUR).

- **Recebimento de Pedidos**: Recebe pedidos do cardápio ou do próprio app MENUR (usado pela
  equipe). Esses endpoints são acessados pelo app mobile MENUR.

- **Status dos Pedidos**: Disponibiliza visualização e alteração do status (desde o recebimento
  até a entrega). Esses endpoints são acessados pelo app mobile MENUR, mas o status também fica
  visível para o cliente no cardápio.

- **Cancelamento de Pedidos**: Processo sensível que afeta produção e caixa. Pedidos recebidos
  não podem ser cancelados pelo cliente; se estiverem em produção, apenas o app MENUR pode
  cancelá-los. Pedidos pagos exigem cancelamento do pagamento. Para mais detalhes, consulte o
  [Domínio Pagamento](#34-domínio-pagamento).

## 3.3. Domínio Identidade e Acesso

Garante a segurança da plataforma, controlando níveis de acesso e confidencialidade.

- **Convites para novos funcionários**: Para usar o app mobile MENUR, o usuário precisa de um
  convite de alguém com nível de acesso igual ou superior, via e-mail com link para baixar o
  app e criar senha.

- **Nível de acesso dos funcionários**: Nem todos podem ver tudo ou acessar todas as
  funcionalidades. Esses níveis são geridos por um administrador (funcionário ou proprietário).
  Não é possível atribuir a si mesmo (ou a outros) um nível superior ao seu. Alguns níveis
  incluem acesso a dados financeiros, cancelamento de pedidos e configurações de pagamento.

- **Login dos funcionários**: Feito por e-mail e senha. Em caso de esquecimento, um e-mail de
  redefinição de senha é enviado. Esses endpoints são acessados pelo app mobile MENUR.

- **Login dos clientes**: Antes de fazer o primeiro pedido via autoatendimento, o cliente
  precisa se identificar. O MENUR integra com Google ID e Apple ID, e esses endpoints são
  acessados diretamente no cardápio.

- **Acesso dos integradores**: O MENUR não faz gestão de estoque, ficha técnica, controle fiscal,
  compras de insumos ou relatórios gerenciais. Disponibiliza, porém, uma API de integração
  (inspirada em [iFood](https://www.ifood.com.br) e
  [Open Delivery Abrasel](https://opendelivery.org.br)) para que sistemas de gestão obtenham
  dados de pedidos e pagamentos. Cada estabelecimento gera uma chave de acesso para o integrador
  que usar essa API.

## 3.4. Domínio Pagamento

O MENUR oferece auto-pagamento via Pix, Google Pay e Apple Pay, com baixa e conciliação
automática. Métodos tradicionais (não integrados) podem ser registrados manualmente.

- **Pagamento com Pix**: O cliente copia o código Pix e paga no app bancário. A confirmação é
  instantânea, tanto no cardápio quanto no app MENUR. O processo passa pelo
  [Banco Central do Brasil](https://www.bcb.gov.br), e, uma vez confirmado, o banco parceiro
  (ex.: Efí Bank) notifica o MENUR.

- **Pagamento com Carteira Digital**: Integração com Google Pay e Apple Pay. O cliente escolhe
  um cartão cadastrado, e a confirmação aparece no cardápio e no app MENUR, mediada pela
  [Stripe Payments](https://stripe.com).

- **Registro Manual de Pagamento**: Se o cliente não optar pelo auto-pagamento, o
  estabelecimento recebe em cartão, dinheiro etc. e registra manualmente no app MENUR. A
  confirmação fica disponível para todos.

## 3.5. Domínio Sincronização

O app mobile MENUR é _local-first_, oferecendo uma UI responsiva. Para isso, todos os dados
necessários são armazenados localmente. O back-end fornece uma API para buscar atualizações
a partir de um _timestamp_. Como há mais de 30 tabelas, as consultas são feitas em paralelo,
e os resultados se unem em uma única resposta. Por segurança, nem todos os dados são expostos,
e alguns têm estrutura diferente da do banco. Cabe ao app interpretar e armazenar tudo localmente.

## 3.6. Domínio Mesas

O cadastro de mesas identifica a origem dos pedidos. Cada mesa tem um QR Code exclusivo. É
possível juntar mesas e definir se o pagamento ocorre antes ou depois do consumo.

- **Cadastro de Mesas**: Pode ser feito no app MENUR ou por meio de QR Codes adquiridos de
  parceiros MENUR.

- **Junção de Mesas**: O app MENUR permite agrupar diversas mesas virtualmente, e também
  desfazer esse agrupamento.

- **Pagamento antecipado**: Alguns estabelecimentos preferem receber antes do preparo, outros
  só ao final. Isso é configurado no app MENUR.

## 3.7. Domínio QR Codes

Os clientes normalmente acessam o cardápio lendo o QR Code das mesas.

- **Impressão dos QR Codes**: O app MENUR gera QR Codes individuais das mesas, resultando em um
  PDF para impressão.

- **Criação de Lotes**: Função reservada aos Parceiros de QR Codes MENUR, que podem criar,
  imprimir e gerenciar lotes de QR Codes virgens vendidos a estabelecimentos.

- **Ativação do QR Code**: Estabelecimentos que adquirem QR Codes virgens podem ativá-los no
  app MENUR, associando-os às mesas.

## 3.8. Domínio Integração

O MENUR não faz controle de estoque, ficha técnica, controle fiscal, compras de insumos ou
relatórios gerenciais. Para isso, estabelecimentos de maior porte costumam usar um sistema
de gestão, enquanto o MENUR disponibiliza uma API para dados de pedidos e pagamentos. A
integração se restringe a parceiros, e atualmente só o Real Software está integrado.

- **Padrão iFood**: O iFood possui uma API bem documentada, comumente adotada por integradores.
  O MENUR aproveita esse padrão para facilitar a curva de aprendizado de novos parceiros.

- **Padrão Open Delivery Abrasel**: A [Abrasel](https://abrasel.com.br) propõe unificar
  integrações entre plataformas de Delivery e sistemas de Gestão. O MENUR ainda não tem
  implementação específica desse padrão, mas acompanha suas evoluções.

> **_OBS:_** O módulo de integração não contempla integrações com plataformas de pagamento ou
> login (essas estão na camada de Integração, mas em outro contexto).

## 3.9. Domínio Endereço

Para cadastrar e pesquisar endereços, o MENUR usa a API do
[Google Maps](https://maps.google.com) em várias partes da plataforma, agilizando a definição
de locais de entrega e áreas restritas.

- **Endereço do estabelecimento**: Fundamental para cálculo de taxas de entrega. Ao cadastrar
  esse endereço, o MENUR pré-carrega dados de Estado, Cidade e Bairro. Esses endpoints são
  acessados pelo app MENUR.

- **Endereço do usuário**: Também necessário para definir a taxa de entrega. O cadastro ativa a
  pré-carga de dados (Estado, Cidade e Bairro) na base do MENUR, e é feito no celular do cliente.

- **Taxas e locais de entrega**: É possível delimitar área de entrega e definir taxas
  específicas. A busca por Estado, Cidade, Bairro ou endereço detalhado usa o Google Maps,
  podendo incluir locais restritos sem cobertura.

## 3.10. Domínio Gestão

Não faz parte do escopo do MENUR fornecer relatórios de gestão detalhados — isso cabe ao
sistema de gestão. Entretanto, o app MENUR oferece algumas funções básicas.

- **Expediente de trabalho**: Deve ser aberto para receber pedidos (autoatendimento ou app). É
  possível fechá-lo quando não se deseja mais aceitar novos pedidos, mas ainda assim receber
  pagamentos pendentes.

- **Dados analíticos**: Em cada expediente, podem-se consultar métricas como tempo médio de
  atendimento, total de vendas, faturamento por meio de pagamento e percentual de atendimento
  por colaborador.

# 4. Internacionalização

Ao longo do tempo, a estrutura do MENUR foi atualizada para oferecer suporte a i18n
(Internationalization). Essa demanda surgiu em testes realizados em Lisboa (Portugal) e pelo
potencial de expansão na Europa. No Brasil, manter o cardápio traduzido para outros idiomas
também é essencial para atender clientes estrangeiros.

## 4.1. Tradução por IA

Todos os dados de produtos cadastrados são traduzidos automaticamente para outros idiomas
usando o [Azure Translator](https://azure.microsoft.com/services/cognitive-services/translator/).
Há um modelo de linguagem especializado no contexto de bares e restaurantes, produzindo
traduções mais adequadas.

A tradução ocorre quando dados do cardápio são criados ou modificados. Após o processo, o
conteúdo traduzido é salvo no banco, junto ao original fornecido pelo usuário.

## 4.2. Mensagens do Back-end

As mensagens de validação e erro do back-end estão traduzidas para alguns idiomas,
principalmente a partir do cabeçalho `Accept-Language`, enviado pelo app MENUR ou pelo
cardápio no celular do cliente. Embora o suporte atual seja restrito a poucos idiomas, a
arquitetura está pronta para expansões futuras.

> **_OBS:_** As traduções estáticas foram produzidas com auxílio do ChatGPT.

## 4.3. Textos Fixos do Cardápio

Todos os textos fixos exibidos no cardápio obedecem ao idioma configurado no celular do
cliente ou à escolha de idioma no próprio cardápio.

Além disso, observações inseridas no momento do pedido são traduzidas automaticamente para
todos os idiomas suportados, garantindo que cozinha e bar entendam facilmente os pedidos de
clientes estrangeiros.

> **_OBS:_** As traduções estáticas foram produzidas com auxílio do ChatGPT.

## 4.4. Textos Fixos do App Mobile

Atualmente, o app MENUR não oferece suporte a i18n por falta de priorização no roadmap de
desenvolvimento, apesar de existir demanda para isso.

# 5. Finalização

Embora o foco deste documento seja o back-end, mencionamos aspectos do cardápio e do app
mobile devido à sua estreita relação com os temas abordados.

Este documento pode ser expandido sempre que surgir a necessidade de um entendimento mais
amplo do projeto, mas não se propõe a ser exaustivo ou definitivo.
