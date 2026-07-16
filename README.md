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

```text
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
