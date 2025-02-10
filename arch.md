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

Este documento apresenta uma visão geral do back-end do MENUR, contextualizando sua
infraestrutura, organização de pastas e pacotes, além dos domínios e subdomínios que compõem
a plataforma. A compreensão detalhada desse cenário permite analisar o modelo atual e guiar o
desenvolvimento de uma futura arquitetura com processamento em borda.

# 2. Arquitetura

Nesta seção, veremos como o MENUR está estruturado em um modelo em camadas e como a aplicação
se organiza internamente. Entenderemos as responsabilidades de cada camada, bem como a forma
como o código-fonte está distribuído no projeto.

## 2.1. Modelo em Camadas

A aplicação, desenvolvida em Kotlin com Quarkus e compilada com GraalVM, segue um modelo em
camadas dentro de um monolito executado em contêineres Docker. Essas camadas compreendem:

- **Camada de apresentação REST**: Responsável por fornecer endpoints para o **módulo de
  cardápio** (voltado ao cliente do estabelecimento), para o **módulo do app mobile** (usado por
  garçons, caixa, cozinha e gestores) e para o **módulo de integração** (empregado por sistemas
  parceiros para receber pedidos, atualmente integrado apenas ao Real Software). Parte das
  regras de negócio, que no ideal estariam na camada de Caso de Uso, acabam aparecendo nessa
  camada.

- **Camada de negócio**: Agrupa regras de negócio bem definidas, podendo ser reutilizadas por
  diferentes partes da camada de apresentação. É nela que se concentram os processos mais
  centrais do MENUR. Um componente aqui pode acessar outros na mesma camada ou em camadas
  inferiores, mas nunca pode acessar a camada de apresentação.

- **Camada de Integração**: Lida com a persistência em banco de dados e a comunicação com
  sistemas externos. É aqui que residem os componentes de acesso aos dados e as integrações com
  serviços de pagamento, login, envio de e-mails e outros.

## 2.2. Detalhando a Camada de Integração

A camada de integração divide-se em duas grandes partes:

- **Persistência em Banco de Dados**: Todos os dados do MENUR são gravados em um banco de dados
  PostgreSQL 16. As tabelas estão mapeadas em entidades JPA (classes), e o acesso a essas
  entidades se dá por meio de classes DAO (Data Access Object), usando JPQL ou SQL nativo em
  alguns casos. O Hibernate JPA gerencia os objetos, que são propagados por todas as camadas,
  ocasionando muito acoplamento. Qualquer alteração em um objeto gerenciado desencadeia um
  `UPDATE`, e se um atributo mapeado como `LAZY` for acessado, ocorre um novo `SELECT`. Esse
  acoplamento é algo que se planeja melhorar na futura [arquitetura proposta neste link]().

- **Cliente de outros sistemas**: Abriga integrações diversas, como para processar pagamentos
  (Pix, cartão), efetuar logins sociais (Apple ID, Google ID) e enviar comunicações (e-mail,
  SMS e push). Para essas integrações, utiliza-se Resteasy Client. As configurações, como
  variáveis de ambiente para autenticação, são definidas de forma que facilitem a manutenção
  na Azure.

## 2.3. Organização das Pastas e Packages

Talvez não seja óbvio como as camadas se relacionam com a estrutura de pastas do back-end.
Segue, então, um detalhamento para aclarar essa organização:

**Primeiro Nível (raiz do projeto)**

- `src`: Onde fica o código-fonte principal, scripts e artefatos relacionados ao build.
- `doc`: Documentação do projeto, incluindo este documento.
- `target`: Conteúdo gerado pelos builds do Maven (arquivos `.class`, `.jar`, entre outros).

**Segundo Nível (dentro de `src`)**

- `src/main`: Código-fonte da aplicação e scripts essenciais para seu funcionamento e build.
- `src/test`: Deveria conter testes unitários e de integração, mas no MENUR não há testes
  implementados.
- `src/etc`: Reúne artefatos e scripts complementares ao desenvolvimento, como modelagens em
  pgModeler e scripts de migração.

**Terceiro Nível (dentro de `src/main/kotlin`)**

- `core`: Reúne a camada de negócios (`business`), a camada de persistência (`persistence`), as
  entidades JPA (`entity`) e as exceções (`exception`) da regra de negócio.
- `client`: Toda a lógica de integração com sistemas externos, em pastas específicas para cada
  integração.
- `rest`: Camada de apresentação REST, com endpoints versionados. Em `service` ficam os endpoints,
  e em `data` ficam os objetos de transporte de dados.
- `util`: Funcionalidades auxiliares, como `Logger`, usadas em múltiplas camadas.
- `sdk`: Implementações de funcionalidades transversais, como monitoramento, cache, controle de
  acesso e transações.

Outras pastas não detalhadas aqui são específicas e não agregam significativamente à visão geral
da arquitetura do MENUR.

---

# 3. Domínios e Sub-domínios

A plataforma MENUR não apresenta grandes desafios em termos de tecnologia, mas reúne uma série
de funcionalidades e detalhes que visam melhorar ao máximo a experiência de uso — tanto para
clientes quanto para funcionários — além de garantir segurança nos processos e dados.

A seguir, listamos os domínios e subdomínios que compõem o back-end atualmente em execução na
Azure.

## 3.1. Domínio Cardápio

Sob a ótica do cliente, é por onde tudo começa.

- **Exibição do Cardápio**: Carrega categorias, produtos, fotos, opções, observações, ajustes de
  layout (logotipo, cor etc.), ocultamento de itens em falta e seleção de idioma. Esses endpoints
  são acessados pelos clientes em seus celulares.

- **Cadastro de Produtos**: Cria e altera produtos, inclusive categorias, fotos, preços e códigos
  de integração com sistemas de gestão (atualmente, só o Real Software). Esses endpoints são
  acessados pelo app mobile MENUR.

- **Ocultamento de Produtos**: Permite ao estabelecimento ocultar produtos indisponíveis ou que
  não devem mais ser exibidos no cardápio.

## 3.2. Domínio Pedidos

Sob a ótica dos funcionários do estabelecimento, é por onde tudo começa.

- **Lançamento de Pedidos**: Pode ser feito pelo cliente (autoatendimento, via celular) ou pela
  equipe (via app mobile MENUR).

- **Recebimento de Pedidos**: Recebe pedidos vindos do cardápio ou do próprio app MENUR (usado
  pela equipe). Esses endpoints são acessados pelo app mobile MENUR.

- **Status dos Pedidos**: Disponibiliza a visualização e alteração do status, do recebimento
  até a entrega. Esses endpoints são acessados pelo app mobile MENUR, mas o status também fica
  visível para o cliente no cardápio.

- **Cancelamento de Pedidos**: Um processo sensível, que afeta produção e controle de caixa.
  Pedidos recebidos não podem ser cancelados pelo cliente, e se já estiverem em produção,
  apenas o app mobile MENUR pode cancelá-los. Pedidos pagos exigem cancelamento do pagamento.
  Para mais detalhes, consulte o Domínio Pagamentos.

## 3.3. Domínio Identidade e Acesso

Garante a segurança da plataforma, controlando níveis de acesso e confidencialidade.

- **Convites para novos funcionários**: Para utilizar o app mobile MENUR, é preciso receber um
  convite de alguém cujo nível de acesso seja igual ou maior. O convidado recebe um e-mail com
  link para baixar o app e criar senha.

- **Nível de acesso dos funcionários**: Nem todos podem ver todas as informações ou acessar todas
  as funcionalidades. Esses níveis são geridos por um administrador (funcionário ou proprietário).
  Um funcionário não pode atribuir a si mesmo (ou a outros) um nível superior ao seu. Alguns níveis
  incluem acesso a dados financeiros, cancelamento de pedidos e configurações de pagamento.

- **Login dos funcionários**: Feito por e-mail e senha. Em caso de esquecimento, um e-mail com
  instruções para redefinir a senha é enviado. Esses endpoints são acessados pelo app mobile MENUR.

- **Login dos clientes**: Antes de fazer o primeiro pedido via autoatendimento, o cliente precisa
  se identificar. O MENUR integra com Google ID e Apple ID, e esses endpoints são acessados
  diretamente pelo cardápio.

- **Acesso dos integradores**: O MENUR não faz gestão de estoque, ficha técnica, controle fiscal,
  compras de insumos ou relatórios gerenciais. Em vez disso, disponibiliza uma API de integração
  (inspirada em iFood e Open Delivery Abrasel) para que sistemas de gestão obtenham dados de
  pedidos e pagamentos. Cada estabelecimento gera uma chave de acesso para o integrador que
  deseje usar essa API.

## 3.4. Domínio Pagamento

O MENUR oferece auto-pagamento via Pix, Google Pay e Apple Pay, com baixa e conciliação
automática. Também se pode usar métodos tradicionais (não integrados) e registrar o pagamento
manualmente.

- **Pagamento com Pix**: O cliente copia o código Copia&Cola e paga no app bancário. A
  confirmação é instantânea, tanto no cardápio quanto no app MENUR. O processo passa pelo
  Banco Central e, uma vez confirmado, o banco parceiro (Efí Bank) notifica o MENUR.

- **Pagamento com Carteira Digital**: Integração com Google Pay e Apple Pay. O cliente inicia
  o pagamento no cardápio e escolhe um cartão cadastrado. A confirmação surge no cardápio e no
  app MENUR, intermediada pela Stripe Payments.

- **Registro Manual de Pagamento**: Se o cliente não optar pelo auto-pagamento, o estabelecimento
  pode receber em cartão, dinheiro etc. e registrar manualmente no app mobile MENUR. A
  confirmação fica disponível para todos.

## 3.5. Domínio Sincronização

O app mobile MENUR é _local-first_, com uma UI responsiva de forma instantânea. Para isso,
todos os dados necessários são armazenados localmente no app. O back-end oferece uma API
para buscar dados atualizados a partir de um _timestamp_. Para manter o desempenho, as mais
de 30 tabelas são consultadas em paralelo, e os resultados são reunidos em uma única resposta.
Por segurança, nem todos os dados são expostos, e alguns têm estrutura diferente da do banco.
Cabe ao app interpretar e armazenar essas informações localmente.

## 3.6. Domínio Mesas

O cadastro de mesas é relevante para identificar a origem dos pedidos. Cada mesa tem um QR Code
único. É possível cadastrar, juntar mesas e definir se o pagamento ocorre antes ou depois do
consumo.

- **Cadastro de Mesas**: Pode ser feito digitando as informações no app MENUR ou ativando um lote
  de QR Codes obtido junto a um parceiro MENUR.

- **Junção de Mesas**: O app MENUR possibilita juntar várias mesas virtualmente, tornando-as uma
  só para fins de controle. A separação também é feita pelo app.

- **Pagamento antecipado**: Alguns estabelecimentos preferem receber antes do preparo, enquanto
  outros cobram ao final. Essa definição é configurada no app MENUR.

## 3.7. Domínio QR Codes

Os clientes normalmente acessam o cardápio lendo o QR Code das mesas em seus celulares.

- **Impressão dos QR Codes**: É possível gerar QR Codes individuais das mesas por meio do app
  MENUR, resultando em um PDF com instruções para impressão.

- **Criação de Lotes**: Essa função é reservada aos Parceiros de QR Codes MENUR e permite criar,
  imprimir e gerenciar lotes de QR Codes virgens, vendidos a estabelecimentos para posterior
  ativação.

- **Ativação do QR Code**: Estabelecimentos que adquirem QR Codes virgens ativam-nos diretamente
  no app MENUR, associando-os às mesas.

## 3.8. Domínio Integração

Como o MENUR não faz controle de estoque, ficha técnica, controle fiscal, compras de insumos
ou relatórios gerenciais, os estabelecimentos que não são MEI contratam um sistema de gestão.
Para que tais sistemas acessem os dados gerados no MENUR — como pedidos e pagamentos — há uma
API de integração restrita a parceiros. Atualmente, o único sistema de gestão integrado é o
Real Software. A estrutura de dados se baseia em padrões de mercado, mas com um nível de
detalhamento maior no status de cada item do pedido.

- **Padrão iFood**: O iFood fornece uma API bem documentada, muito adotada pelos seus
  integradores parceiros. O MENUR aproveita esse mesmo padrão para minimizar a curva de
  aprendizado de novos parceiros.

- **Padrão Open Delivery Abrasel**: A Abrasel — Associação Brasileira de Bares e Restaurantes —
  propôs um padrão para unificar integrações entre sistemas de Delivery e de Gestão, embora
  ele ainda não seja amplamente aceito no mercado. O MENUR não tem implementação específica de
  Open Delivery, mas acompanha as evoluções desse padrão.

> **_OBS:_** O módulo de integração não contempla integrações com plataformas de pagamento,
> nem com Google ou Apple para autenticação, nem outras integrações realizadas pelo MENUR.

## 3.9. Domínio Endereço

Para cadastrar e pesquisar endereços, o MENUR usa a API do Google Maps em diversas partes da
plataforma, otimizando a experiência ao definir locais de entrega e áreas restritas.

- **Endereço do estabelecimento**: Fundamental para o cálculo de taxas de entrega. Ao cadastrar
  esse endereço, o MENUR pré-carrega dados de Estado, Cidade e Bairro. Esses endpoints são
  acessados pelo app mobile MENUR.

- **Endereço do usuário**: Também necessário para calcular a taxa de entrega. O cadastro
  dispara a pré-carga de dados (Estado, Cidade e Bairro) na base do MENUR, sendo acessível
  pelo próprio celular do cliente.

- **Taxas e locais de entrega**: É possível delimitar a área de entrega e definir taxas
  específicas. A busca por Estado, Cidade, Bairro, Rua ou endereço detalhado usa o Google Maps,
  podendo incluir locais restritos onde não há cobertura.

## 3.10. Domínio Gestão

Não faz parte do escopo do MENUR fornecer relatórios de gestão detalhados — esta tarefa é feita
pelo sistema de gestão integrado. Contudo, o app mobile MENUR dispõe de alguns recursos básicos.

- **Expediente de trabalho**: Deve ser aberto para receber pedidos (seja no autoatendimento ou
  no app). É possível fechá-lo quando não se quer mais aceitar pedidos, mas ainda assim receber
  pagamentos pendentes.

- **Dados analíticos**: Em cada expediente, podem-se consultar métricas como tempo médio de
  atendimento, total de vendas, faturamento por meio de pagamento e percentual de atendimento
  por colaborador, entre outras.

# 4. Internacionalização

Ao longo do tempo, a estrutura interna do MENUR foi atualizada para oferecer suporte a i18n
(Internationalization). Essa preocupação surgiu principalmente pelos testes realizados em
Lisboa (Portugal) e pelo potencial de expansão para outros países da Europa. Além disso, mesmo
no Brasil, torna-se relevante manter o cardápio traduzido para outros idiomas, visando atender
a clientes estrangeiros de forma mais completa.

## 4.1. Tradução por IA

Todos os dados dos produtos cadastrados pelos estabelecimentos são traduzidos automaticamente
para outros idiomas utilizando o Azure Translator. Contamos com um modelo de linguagem (LLM)
especializado no contexto de bares e restaurantes, garantindo traduções mais adequadas a este
segmento.

A tradução ocorre no momento em que os dados do cardápio são cadastrados ou modificados. Após
o processo, o conteúdo traduzido é salvo no banco de dados junto ao texto original fornecido
pelo usuário.

## 4.2. Mensagens do Back-end

As mensagens de validação e de erro do back-end estão traduzidas para alguns idiomas, definidos
principalmente com base no cabeçalho `Accept-Language`, enviado pelo app mobile ou pelo cardápio
no celular do cliente. Embora atualmente haja suporte para poucos idiomas, a arquitetura está
preparada para a adição de outros, sem grandes impactos na estrutura.

> **_OBS:_** As traduções estáticas foram produzidas com o auxílio do ChatGPT.

## 4.3. Textos Fixos do Cardápio

Todos os textos fixos exibidos no cardápio são determinados conforme o idioma configurado no
celular do cliente, ou de acordo com a escolha direta do idioma no próprio cardápio.

Além disso, observações inseridas durante o momento do pedido são traduzidas automaticamente
para todos os idiomas suportados, garantindo que cozinha e bar entendam o pedido de clientes
estrangeiros sem dificuldades.

> **_OBS:_** As traduções estáticas foram produzidas com o auxílio do ChatGPT.

## 4.4. Textos Fixos do App Mobile

Até o momento, o app mobile do MENUR não oferece suporte a i18n. Isso não ocorre por falta de
necessidade, mas sim por não ter sido priorizado em seu roadmap de desenvolvimento.

# 5. Finalização

Apesar de o foco deste documento ser o back-end, alguns aspectos do cardápio e do app mobile
foram mencionados devido à estreita relação com os tópicos abordados.

Este documento pode ser expandido sempre que houver necessidade de uma compreensão mais ampla
do projeto. No entanto, não tem a pretensão de ser completo ou exaustivo.
