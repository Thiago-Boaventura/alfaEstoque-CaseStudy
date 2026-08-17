# 📦 Alfa Estoque — Case Study

### Plataforma SaaS multi-tenant de Gestão de Estoque e PDV para pequeno varejo

[![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat-square&logo=typescript&logoColor=white)](#-stack-tecnológica)
[![NestJS](https://img.shields.io/badge/NestJS-E0234E?style=flat-square&logo=nestjs&logoColor=white)](#-stack-tecnológica)
[![React](https://img.shields.io/badge/React-61DAFB?style=flat-square&logo=react&logoColor=black)](#-stack-tecnológica)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white)](#-stack-tecnológica)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)](#-stack-tecnológica)

---

## 🎯 Visão Geral

O **Alfa Estoque** é uma plataforma SaaS multi-tenant para gestão de pequenos negócios de varejo — controle de estoque, cadastro de produtos e frente de caixa (PDV), operando 100% online e acessada por subdomínio dedicado por loja (`loja.alfaestoque.com.br`).

O projeto nasceu de um problema concreto: pequenos varejistas (roupas, empórios, produtos de limpeza) costumam operar com planilhas soltas ou sistemas desktop desatualizados, sem integração real entre o que é vendido no caixa e o que resta em estoque — e sem nenhum controle sobre fechamento de caixa que iniba fraude ou erro humano na contagem.

Este documento descreve a arquitetura, as decisões técnicas e os desafios de engenharia enfrentados na construção do produto, do zero até produção.

---

## 🧩 O Problema

- **Estoque e venda desconectados**: controle de estoque em uma ferramenta, vendas em outra (ou em papel) — divergência constante entre o que existe fisicamente e o que o sistema informa.
- **Produtos com comportamentos muito diferentes**: uma loja de roupas precisa de grade (cor/tamanho); um empório precisa de lote e validade; uma loja de granéis precisa vender fracionado por peso. Um cadastro de produto rígido não atende os três.
- **Fechamento de caixa sem auditoria real**: sem um processo formal de conferência, é fácil mascarar uma quebra de caixa (sobra ou falta) — ou simplesmente não perceber que ela aconteceu.
- **Custo de entrada alto**: soluções de ERP tradicionais são caras e complexas demais para quem vende em uma loja só, com dois ou três funcionários.

---

## 💡 A Solução

Uma plataforma **SaaS multi-tenant**, com onboarding self-service (trial gratuito → assinatura recorrente via gateway de pagamento), pensada para rodar em qualquer navegador, sem instalação:

- **Backoffice (retaguarda)** para o Gerente da loja: produtos, estoque, relatórios, configurações e gestão de assinatura.
- **PDV (frente de caixa)** para o Operador: venda ágil, leitor de código de barras, múltiplas formas de pagamento, sangria/suprimento e fechamento de caixa às cegas.
- **Painel Master (Super Admin)** para a operação do próprio SaaS: gestão de lojas, planos, métricas de MRR e suporte via impersonation.

### Modelo de produto — comportamento dinâmico (*behavior-driven*)

Em vez de um cadastro de produto único e rígido, cada produto tem **flags de comportamento** que ligam/desligam funcionalidades:

| Flag | Efeito |
|---|---|
| `Controla Grade` | Habilita variações (ex: cor × tamanho), cada uma com seu próprio SKU |
| `Controla Lote/Validade` | Exige lote e data de validade em toda entrada de estoque |
| `Venda Fracionada` | Permite vender por peso/volume (ex: 1,5 kg) em vez de unidades inteiras |

A mesma tabela de produtos atende uma loja de roupas, um empório e uma loja de granéis — sem tabelas paralelas por segmento.

### Fechamento de caixa — método cego validado

Para inibir fraude e erro de contagem, o operador:
1. Declara o valor contado em cada forma de pagamento, sem ver o valor esperado pelo sistema (**fechamento cego**).
2. O sistema cruza o declarado com o registrado e, se houver divergência, mostra um alerta **neutro** — nunca revela qual forma de pagamento está errada nem o valor da diferença (evita "ajustar o número até bater").
3. Se o operador confirmar mesmo assim, o turno fecha como `fechado_com_divergencia`, e a quebra fica visível para auditoria do Gerente.

---

## 🏗️ Arquitetura

Monorepo (pnpm + Turborepo), com separação clara entre aplicações e pacotes compartilhados:

```text
AlfaEstoque/
│
├── apps/
│   ├── api/       # NestJS — API core, arquitetura em camadas por módulo
│   ├── admin/     # React + Vite — Backoffice, PDV e Painel Master
│   └── web/       # React + Vite — Landing page pública + checkout
│
├── packages/
│   ├── ui/        # Design system compartilhado (React)
│   └── config/    # Configuração/env compartilhada entre apps
│
└── docs/
    ├── spec-geral.md        # Especificação de produto
    ├── plano-de-alteracao.md
    └── alteracoes/           # Changelog técnico de cada mudança (~50 entradas)
```

### Backend — Clean Architecture por módulo

Cada módulo do NestJS segue a mesma separação em camadas:

```text
modules/<modulo>/
├── domain/           # Entidades e contratos de repositório — zero dependência de framework
├── application/       # Casos de uso, DTOs e ports (interfaces)
├── infrastructure/    # Implementações concretas — Drizzle ORM, gateways externos
└── presentation/      # Controllers e módulo NestJS
```

Essa separação paga dividendos justamente nos pontos de maior risco do domínio: regras de fechamento de caixa, ciclo de vida de assinatura e isolamento multi-tenant vivem no `domain`/`application`, isolados de detalhes de banco ou de gateway de pagamento — o que tornou possível testar essas regras sem subir banco de dados nem chamar API externa.

### Multi-tenancy

Isolamento por `tenant_id` em nível de banco (PostgreSQL), com identificação da loja por **subdomínio** (`loja.alfaestoque.com.br`) e o `tenantId` do usuário sempre derivado do JWT — nunca de um parâmetro vindo do cliente, mesmo em rotas self-service de gestão de assinatura.

---

## ⚙️ Stack Tecnológica

**Frontend**
`React` `TypeScript` `Vite` `Tailwind CSS` `TanStack Query`

**Backend**
`Node.js` `NestJS` `PostgreSQL` `Drizzle ORM` `JWT / RBAC`

**Infraestrutura**
`Docker Compose` `VPS Linux` `Nginx` `SMTP (Resend)`

**Pagamentos**
`Asaas` (assinaturas recorrentes, PIX, cartão, webhooks)

**Engenharia assistida por IA**
Desenvolvimento guiado por especificação (`docs/spec-geral.md`) e por um protocolo de governança próprio (`AGENTS.md`) que define como agentes de IA devem operar no repositório — carregando o contexto de arquitetura antes de qualquer alteração, documentando plano → execução → mudança real para cada feature (`docs/plano-de-alteracao.md` + `docs/alteracoes/`).

---

## 🚀 Funcionalidades Principais

- **Onboarding self-service**: cadastro com trial gratuito de 5 dias, conversão para assinatura paga via Asaas (PIX/cartão), reativação self-service dentro de janela de retenção.
- **Gestão de assinatura ("Meu Plano")**: troca de plano, antecipação de renovação e cancelamento pelo próprio Gerente, sem depender de suporte.
- **PDV completo**: leitor de código de barras, múltiplos pagamentos por venda, desconto com teto configurável, sangria/suprimento, sessão de caixa obrigatória.
- **Permissões granulares**: Operador de caixa pode ganhar acesso extra a Dashboard/Produtos/Estoque por concessão explícita do Gerente.
- **Relatórios e dashboard**: faturamento, ticket médio, auditoria de caixa por período — calculados sempre no fuso comercial (`America/Sao_Paulo`), independente do fuso do servidor.
- **Painel Master multi-tenant**: métricas de MRR, gestão de planos com sincronização de preço no gateway de pagamento como ação explícita (nunca automática), impersonation para suporte.
- **Geração de etiquetas**: código de barras EAN-13 real, impressão isolada ou em lote.

---

## 🔧 Desafios Técnicos & Decisões de Engenharia

Alguns problemas concretos enfrentados durante o desenvolvimento — e como foram resolvidos:

### Bug de fuso horário mascarando dados de auditoria
O dashboard e o relatório de auditoria de caixa mostravam totais diferentes para o "mesmo dia". Causa raiz: o servidor rodava em UTC, e os limites de dia/mês eram calculados no fuso do processo, não no fuso comercial da loja (`America/Sao_Paulo`, UTC-3). Diagnosticado consultando `current_setting('TIMEZONE')` diretamente no banco de produção e reproduzindo o timestamp exato que cruzava a virada do dia de forma diferente nos dois cálculos. Corrigido centralizando toda lógica de data de negócio em um helper único (`business-timezone.ts`), eliminando o uso do fuso local do processo em qualquer cálculo de relatório.

### Ciclo de vida de assinatura com múltiplos estados de recuperação
Loja suspensa (trial vencido ou problema de pagamento) precisa poder se reativar sozinha; loja bloqueada por decisão administrativa, não. Loja cancelada pode voltar sozinha, mas só dentro de uma janela de retenção (90 dias) — depois disso, só via suporte. Modelado como um pequeno conjunto de estados (`active | suspended | blocked | cancelled`) com as regras de "pode se recuperar sozinho?" vivendo como método na própria entidade `Tenant`, não espalhadas pelos controllers.

### Erros de gateway de pagamento mascarados como "erro interno"
Falhas de rede ou de configuração ao chamar o Asaas viravam `Error` genérico, capturado pelo filtro global de exceções e reescrito como "Erro interno inesperado" — escondendo do usuário (e do suporte) a causa real. Corrigido convertendo toda falha do gateway em um `DomainError` tipado, preservando a mensagem útil para diagnóstico.

### Armadilha do ORM em monorepo
No Drizzle ORM, o schema de um módulo novo só é reconhecido pelas migrations se for reexportado no arquivo barril central (`database/schema.ts`) — esquecer esse passo gera uma migration silenciosamente incompleta, sem erro. Virou item obrigatório de checklist ao criar qualquer módulo novo.

---

## 📊 Estado Atual

- Backend com suíte de testes automatizados cobrindo as regras de negócio críticas (assinatura, fechamento de caixa, permissões).
- Em produção, rodando via Docker Compose em VPS própria.
- ~50 mudanças documentadas incrementalmente em `docs/alteracoes/`, cada uma com plano, execução real e forma de validar — funcionando como um changelog técnico vivo do produto.

## 🗺️ Roadmap

- Módulo fiscal (emissão de NFC-e).
- Módulo financeiro (contas a pagar/receber, DRE).
- Exclusão automática (job) de lojas suspensas/canceladas após o período de retenção.

---

## 🔗 Links

- Repositório: [github.com/Thiago-Boaventura/AlfaEstoque](https://github.com/Thiago-Boaventura/AlfaEstoque)
- Autor: [Thiago Boaventura](https://www.linkedin.com/in/thiago-boaventura-7061421a1/)
