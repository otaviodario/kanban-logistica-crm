Este repositório apresenta o estudo de caso arquitetural de um ecossistema integrado projetado para faturamento de comércio eletrônico, fluxo fabril (Kanban), automação logística, SAC, inteligência de vendas no WhatsApp e monitoramento de infraestrutura em nuvem. 

A solução unifica e sincroniza planilhas corporativas no Google Sheets, rotinas em Google Apps Script, microsserviços serverless em Cloudflare Workers (com banco de dados no Workers KV), além de uma aplicação web de página única (SPA Frontend) e integrações com APIs de ERP, CRM e Redes de Entrega.

---

## 📌 Visão Geral e Desafios Resolvidos

O projeto foi concebido para mitigar gargalos operacionais comuns em e-commerce de médio e grande porte, tais como:
* **Dessincronização operacional**: Falta de comunicação em tempo real entre vendas (WhatsApp/CRM), fábrica e expedição.
* **Complexidade logística**: Lentidão na cotação de fretes pesados ou compostos por múltiplos volumes.
* **Gargalos de API**: Estouro de limites de cotas de execução diárias das APIs do Google (GAS).
* **Falta de rastreabilidade no SAC**: Ausência de controle unificado sobre trocas, pendências de montagem e reembolsos.

---

## 🏗️ Arquitetura de Integração e Fluxo de Dados

O ecossistema é dividido em três camadas principais que se comunicam de forma assíncrona e otimizada por cache:

```
                     ┌─────────────────────────────┐
                     │   Google Sheets Database    │
                     │    (Faturamento / CRM)      │
                     └──────────────▲──────────────┘
                                    │  (Apps Script Engine)
                                    ▼
┌──────────────────────────────────────────────────────────────────┐
│                 Cloudflare Worker (API Layer)                    │
└──────────────┬────────────────────┬────────────────────┬─────────┘
               │                    │                    │
               ▼                    ▼                    ▼
┌──────────────────────┐   ┌────────────────┐   ┌──────────────────┐
│ SPA Frontend (Kanban)│   │  Workers KV    │   │  APIs Externas   │
│  Tailwind + JS ES6   │   │   (Caching)    │   │ (ERP/CRM/Cloudf.)│
└──────────────────────┘   └────────────────┘   └──────────────────┘

```

1.  Camada de Dados e Regras Físicas (Google Sheets / GAS): Orquestra o banco de
    dados de faturamento em abas divididas por mês/ano, realiza cálculos
    matemáticos de cubagem e gerencia ganchos (webhooks) de saída.
2.  Camada Middleware Serverless (Cloudflare Workers & KV): Atua como gateway de
    API unificado de alta performance. Gerencia sessões com controle de acesso
    baseado em funções (RBAC), consolida estados operacionais em cache estático
    e reduz drasticamente as leituras físicas diretas nas planilhas.
3.  Camada de Apresentação (SPA Kanban): Painel interativo construído em vanilla
    JavaScript para movimentação de cards operacionais, consulta ao catálogo de
    produtos corporativo, monitoramento de SAC e Business Intelligence (BI).

📂 Mapeamento de Componentes e Responsabilidades

Para fins de estudo de engenharia de software, o projeto foi segmentado nos
seguintes módulos e responsabilidades:

1. Camada de Middleware Serverless (/cloudflare-worker)

  - worker.js:
      - Gateways de Roteamento: Concentra as rotas de API para transição de
        estágios operacionais (Workflow), autenticação e sincronizações
        externas.
      - Consolidação em Cache (consolidated_data): Agrupa os dados de vendas,
        SAC, catálogo e telemetria em um único cache JSON de alta velocidade no
        Workers KV.
      - RBAC (Role-Based Access Control): Valida as chaves de permissão do
        operador (Admin, Gestor, Comercial, Expedição) antes de autorizar
        alterações de faturamento na planilha física ou gravação de dados.

2. Camada de Apresentação (/frontend)

  - index.html (Single Page Application):
      - Painel Kanban de Produção: Interface operacional que permite controle
        visual do fluxo fabril (Produção, Montagem, Expedição, Envio de
        Rastreio, Entregues).
      - Filtros Dinâmicos: Pesquisa de cards em tempo real com algoritmo de
        proteção contra carregamento excessivo de dados (Lazy Loading) e
        debouncing de busca.
      - Ficha de Cubagem e Cotação: Formulário flutuante que lê o cadastro
        físico de dimensões por SKU e calcula na tela o volume do envio e peso
        cubado total.
      - Dashboard de Business Intelligence: Compila e plota dados financeiros de
        comissões, vendas líquidas de canais, custos de SAC e telemetria de
        rede.

3. Camada de Regras de Negócio e Triggers (/google-apps-script)

  - kanban_bling.gs:
      - Integra-se à API Bling V3 para atualizar o peso bruto, quantidade de
        volumes, etiquetas de envio e dados da transportadora faturada, com
        tratamento para equivalência de nomes (alias de-para) e suporte a
        múltiplas contas do ERP (LTDA e RP).
  - kanban_metricas.gs:
      - Consulta periodicamente a API GraphQL da Cloudflare para extrair
        métricas de requisições, visitantes únicos, economia de banda, ameaças
        bloqueadas pelo WAF e invocações de scripts serverless, enviando esses
        dados ao Worker para construção do painel de telemetria.
  - kanban_config.gs:
      - Responsável pelo motor de sincronização em lote das planilhas locais
        para o Cloudflare KV. Utiliza hashes MD5 para detectar mudanças nas abas
        e disparar a atualização apenas se houverem novos dados salvos.
  - analise_cot_mktsender.gs:
      - Algoritmo de processamento que lê conversas ativas no WhatsApp de
        cotação de fretes. Identifica o número físico da linha do faturamento,
        extrai os valores oferecidos pelas transportadoras participantes e
        atribui o menor preço (vencedor) e o respectivo código de cotação de
        forma direta nas células da planilha.
  - mkt_cot.gs:
      - Otimiza a busca diária de tickets abertos no WhatsApp CRM com tags
        específicas, estruturando logs visuais limpos e detectando valores
        financeiros na conversa.
  - mkt_despachados.gs:
      - Monitora o status "DESPACHADO" na planilha e dispara o link de
        rastreamento de transporte adequado de forma automática ao celular do
        cliente via WhatsApp.
  - mkt_entregues.gs:
      - Cria contatos dinamicamente na API do CRM, localiza tickets ativos por
        UUID e gerencia a resposta rápida de entrega concluída com envio de
        pesquisa NPS (Google Maps).
  - mktsender.gs:
      - Centraliza rotinas administrativas de autenticação (JWT Login),
        validação de expiração de credenciais e sincronização de feriados
        nacionais na BrasilAPI para cálculo de prazos úteis de fabricação.
  - despachados_aba.gs:
      - Consolidador assíncrono que extrai históricos de despachados em abas
        protegidas por mês/ano e organiza os registros por data decrescente de
        envio.
  - entregues_lista.gs:
      - Gera lotes consolidados de saídas finalizadas e exporta para painéis
        visuais ou sistemas automáticos de pós-venda.
  - entregues_rastreio.gs:
      - Varredura periódica em lote que cruza os dados das transportadoras
        externas, altera as planilhas locais para "ENTREGUE" e ativa o envio de
        mensagens de conclusão de forma instantânea.

⚙️ Principais Workflows do Ecossistema

A. Fluxo de Vida do Pedido e Automação de Rastreio

1.  Entrada: Pedido cai na planilha mãe e é registrado no Kanban como "Produção"
    com prazo calculado em dias úteis (descontando finais de semana e feriados
    via BrasilAPI).
2.  Workflow: Operadores avançam o card para "Montagem" e depois "Expedição".
3.  Cubagem: A expedição confere as dimensões físicas dos volumes baseando-se no
    catálogo de SKUs integrado. O faturamento gera a NF-e.
4.  Faturamento ERP: Os dados são integrados diretamente ao Bling ERP.
5.  Automação de Rastreio: Ao marcar como "Despachado", o sistema dispara a
    mensagem com o link de rastreamento correspondente à transportadora de forma
    automatizada ao cliente pelo WhatsApp.
6.  Alerta de Entrega: Ao constar como entregue na transportadora, o robô
    reverte o status na planilha e dispara uma mensagem amigável solicitando
    avaliação do produto.

B. Minimização de Gargalos por Cache Centralizado

```

[Alteração Local na Planilha]
             │
             ▼
[Apps Script calcula Hashes MD5]
             │
             ▼
      Hash Divergente?
       ├── (Não) ──> [Sincronização Ignorada]
       └── (Sim) ──> [Envio de Payload ao Cloudflare Worker]
                           │
                           ▼
                [Workers atualiza o KV]
                           │
                           ▼
                [State é consolidado]
                           │
                           ▼
                [Injeta Logs no Kanban]

```

🔒 Segurança e Práticas de Engenharia Aplicadas

  - Arquitetura Zero-Trust: Chaves de APIs, segredos de faturamento e chaves JWT
    administrativas são mantidos exclusivamente em variáveis de ambiente
    criptografadas (Secrets na Cloudflare) ou propriedades privadas do script do
    Apps Script, prevenindo qualquer exposição indesejada.
  - Mecanismos de Antiduplicidade: Lógica de identificação matemática de chaves
    unindo "Nome da Aba" + "Linha Física" para garantir que a importação de
    históricos logísticos nunca sobrescreva ou duplique dados legítimos.
  - Debouncing & Sanitização: Prevenção de concorrência e injeções de caracteres
    nas pontas das APIs do Bling e CRM, garantindo consistência total no banco
    de dados distribuído.

