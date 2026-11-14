---
title: Insights Zap — Especificação
status: rascunho v0.1
atualizado: 2026-11-14
---

# Insights Zap — Especificação

> **Uma frase:** tudo o que você e seu primo trocam no WhatsApp que tem valor para a
> empresa (links, documentos, áudios, ideias) é capturado, resumido e organizado por
> agentes de IA numa base de conhecimento pesquisável, sem ninguém precisar sair do
> WhatsApp.

---

## 1. Problema

- Links, PDFs, áudios e ideias importantes chegam no meio da conversa do dia a dia e somem
  na rolagem.
- A solução atual ("mandar para mim mesmo", favoritar com ⭐) não tem busca boa, resumo,
  categoria nem contexto ("por que isso foi mandado?").
- O material é **da empresa**, mas está preso no celular de duas pessoas.
- Ninguém relê. Sem resumo e sem lembrete, o material salvo não vira ação.

## 2. Objetivos

| # | Objetivo | Como medir |
|---|---|---|
| O1 | Nenhum link ou documento da conversa com o primo se perde | 100% dos links/arquivos dos chats permitidos viram item no Insights |
| O2 | Achar qualquer material em segundos | Busca por pergunta ("o que falamos sobre precificação?") devolve a fonte certa no top 3 |
| O3 | Entender um material sem abrir | Todo item tem resumo de 3 a 5 linhas + "por que importa" |
| O4 | Material vira ação | Itens com "próximo passo" viram tarefa no PGBA com um clique |
| O5 | Zero esforço extra no celular | Captura automática; marcar algo como importante = 1 reação (📌) |

### Fora do escopo (v1)

- Ler **todos** os chats do WhatsApp. Só chats explicitamente permitidos (ver §7).
- Grupos (entram numa v2, um a um, com permissão).
- Responder automaticamente ao primo. O bot **não fala** no chat de vocês, exceto
  comandos que você mesmo der (§5.4).
- App mobile próprio. O "app" mobile é o próprio WhatsApp; a tela fica no PGBA (web).

## 3. É possível? — Viabilidade

**Sim.** E a maior parte já existe no `pgba-boilerplate`:

| Precisa de | Já existe no PGBA? | Onde |
|---|---|---|
| Conectar no WhatsApp e receber mensagens | ✅ Evolution API no `docker-compose.prod.yml` (profile `whatsapp`) | `crm/channels.py`, `crm/webhooks.py` |
| Webhook autenticado (apikey/segredo) | ✅ | `crm/webhooks.py#_whatsapp_autentico` |
| Guardar histórico de mensagens | ✅ Evolution grava no Postgres (`evolution_db`) → endpoint `chat/findMessages` | compose |
| Base de conhecimento com busca semântica | ✅ `KnowledgeSource → Document → DocumentChunk` (pgvector) | `ingestion/` |
| Baixar e ler página de um link com segurança | ✅ `safe_http` (anti-SSRF, teto 10 MB) | `ingestion/connectors/safe_http.py` |
| Ler PDF | ✅ `pypdf` | `requirements.txt` |
| Transcrever áudio | ✅ Whisper via Groq/OpenAI | `harness/providers.py#transcribe` |
| Resumir/classificar com IA sem alucinar | ✅ `chat_completion` + `extract_json`/`validate_schema` | `harness/` |
| Mascarar dado pessoal (LGPD) | ✅ `redact_pii` | `core/utils/lgpd.py` |
| Fila de tarefas lentas | ✅ Celery + beat | `config/settings/base.py` |
| Perguntar à base pela tela | ✅ `KnowledgeChat` / `semantic_search` | frontend + `ingestion` |
| Tarefas e quadro Kanban | ✅ `tasks`, padrão Kanban do Design System | frontend |

**O que falta construir:** a vertical `insights` (modelos, pipeline de enriquecimento,
webhook dedicado, telas), o conector `whatsapp` no `ingestion` e o digest semanal.

### Riscos e como lidamos

| Risco | Gravidade | Mitigação |
|---|---|---|
| **Evolution API usa WhatsApp não oficial (Baileys/WhatsApp Web).** A Meta pode banir número que se comporta como robô. | Alta | Instância **só lê**. Não envia mensagem em massa; os únicos envios são para você mesmo (digest/comandos). Usar o número pessoal conectado como "aparelho" (igual WhatsApp Web), não um número novo. |
| Ler conversa de outra pessoa (o primo) | Média (LGPD) | Consentimento explícito do primo registrado (`ConsentRecord`), lista de chats permitidos, botão "pausar captura", nada de grupos por padrão. |
| Conteúdo pessoal misturado ("vai almoçar?") | Média | Filtro: só vira item o que tem **link, arquivo, áudio longo ou 📌**. Texto solto não é salvo (§5.2). |
| Link atrás de login (Instagram, LinkedIn, paywall) | Média | Guarda o link + preview do WhatsApp + o texto ao redor; marca "conteúdo não acessível" — nunca inventa resumo. |
| Custo de IA | Baixa | Ollama local por padrão (regra do PGBA); Whisper exige Groq/OpenAI → é decisão explícita (§10). Volume estimado: dezenas de itens/semana. |
| Sessão do WhatsApp cai (celular desconectado) | Média | Alerta no painel + retomada pelo histórico (`findMessages`) quando reconectar. |

## 4. Decisão de arquitetura

**Recomendação: Insights Zap é uma vertical do PGBA, não um sistema separado.**

- O `CLAUDE.md` do PGBA é explícito: *"Nunca fork o repositório para atender um cliente novo:
  adicione uma vertical."*
- Reaproveita tenant, LGPD, harness de IA, busca semântica, aprovações e tarefas.
- Os agentes dos setores (Comercial, Marketing, Desenvolvimento…) passam a **enxergar** o que
  vocês salvaram, com o escopo de setor que já existe.

Este repositório (`Insights_zap`) guarda: o spec, o design system da vertical e,
opcionalmente, o **export em Markdown** do conhecimento (§6.5).

```
                       ┌──────────── PGBA (VPS) ─────────────────────────────────────┐
WhatsApp (seu número)  │                                                              │
   │  aparelho         │  Evolution API ──webhook──▶ insights.webhook                  │
   ▼  conectado        │  (instância "insights")         │ filtra chat permitido      │
Evolution API ─────────┤                                 ▼                            │
                       │                    CapturedMessage (bruto, PII mascarada)    │
                       │                                 │ Celery                     │
                       │                                 ▼                            │
                       │                  Enriquecimento (pipeline §5.3)              │
                       │      link → safe_http · PDF → pypdf · áudio → Whisper        │
                       │                                 │ harness (resumo + JSON)    │
                       │                                 ▼                            │
                       │     Insight ──▶ ingestion.Document (fonte "WhatsApp")        │
                       │        │                  └──▶ DocumentChunk (pgvector)      │
                       │        ├──▶ Tarefa (com aprovação)                           │
                       │        └──▶ Digest semanal ──▶ WhatsApp (só para você)       │
                       │                                                              │
                       │  Telas: Caixa de entrada · Quadro · Pergunte · Digest        │
                       └──────────────────────────────────────────────────────────────┘
```

### Por que uma instância separada da do CRM?

Hoje o CRM usa **uma** `ChannelConfig` WhatsApp por empresa, ligada ao número que atende
leads — e a IA **responde** cada mensagem. O Insights precisa do **seu número pessoal** e
**nunca responde**. Misturar os dois faria a IA responder ao seu primo como se ele fosse lead.
A Evolution aceita várias instâncias no mesmo servidor, então é só criar a instância
`insights` com webhook próprio.

## 5. Funcionalidades

### 5.1 Captura

**Fonte:** webhook `messages.upsert` da instância `insights`, apenas para os chats da lista
permitida (JID do primo; no futuro, grupos escolhidos). Mensagens **dos dois lados**
(`fromMe` true e false) — o que você mandou também é material.

| Tipo de mensagem Evolution | O que extraímos |
|---|---|
| `conversation` / `extendedTextMessage` | texto + URLs + preview do link (`title`, `description`, `canonicalUrl`) |
| `documentMessage` | arquivo (PDF, DOCX, XLSX…) via `chat/getBase64FromMediaMessage` |
| `audioMessage` (ptt) | áudio → transcrição |
| `imageMessage` / `videoMessage` | legenda; imagem guardada (OCR fica para a v2) |
| `reactionMessage` | 📌 = marca a mensagem reagida como importante; ✅ = "já vi"; 🗑️ = descartar |
| `quotedMessage` (resposta) | liga o comentário à mensagem original (vira "contexto") |

**Histórico (backfill):** ao ligar, importa os últimos N dias via `chat/findMessages`.
Alternativa sem servidor: importar o **"Exportar conversa"** do WhatsApp (.zip com `.txt` + mídia).

### 5.2 Regras do que vira Insight

Nem toda mensagem vira item. Regra padrão (configurável):

1. Tem **link** → vira item.
2. Tem **documento** → vira item.
3. **Áudio** com mais de 45 s → vira item (áudio curto é conversa).
4. Recebeu **📌** de qualquer um dos dois → vira item, seja o que for.
5. Começa com **`#salvar`**, `#ideia` ou `#tarefa` → vira item (e já define o tipo).
6. Mensagens de texto sem nada disso → **não** são salvas como item; ficam só como contexto
   (as 5 mensagens antes e depois de um item, para a IA entender o "porquê").

Mensagens agrupadas: várias mensagens seguidas da mesma pessoa em até 2 min sobre o mesmo
link viram **um** item.

### 5.3 Enriquecimento (pipeline com IA)

Roda no Celery, um passo por vez, cada passo com status visível na tela:

```
recebido → baixando → lendo → resumindo → pronto
                              ↘ falhou (motivo explícito, botão "Tentar de novo")
```

1. **Baixar** — link via `safe_http`; mídia via Evolution.
2. **Ler** — HTML → texto principal; PDF → `pypdf`; áudio → `transcribe`; YouTube → legenda
   quando houver.
3. **Mascarar PII** — `redact_pii` antes de qualquer coisa ir para o `Document`.
4. **Resumir e classificar** — `harness.chat_completion` com saída JSON validada:

```json
{
  "titulo": "string ≤ 90",
  "resumo": "3 a 5 linhas",
  "por_que_importa": "1 frase ligando à empresa / à conversa",
  "tipo": "artigo | video | documento | ferramenta | ideia | noticia | referencia | tarefa",
  "categorias": ["Mercado", "Produto", "Marketing", "Finanças", "Jurídico", "Tecnologia", "Operações", "Pessoas"],
  "tags": ["livre", "até 5"],
  "proximos_passos": ["ação concreta", "..."],
  "setor_sugerido": "nome de um setor existente no PGBA ou null"
}
```

5. **Guardrail** — sem conteúdo legível (paywall, link quebrado), o resumo **não é
   gerado**: o item fica "Conteúdo não acessível" com o preview do WhatsApp e o texto da
   conversa. Nunca resumir a partir só do título.
6. **Indexar** — `Document` na fonte "WhatsApp — Insights" (um `Document` por item, com
   `external_id` = id da mensagem) → chunks → busca semântica.
7. **Deduplicar** — mesmo URL canônico ou mesmo hash de arquivo → não cria item novo;
   adiciona "mandado de novo em 12/11 por Primo" ao item existente.

### 5.4 Comandos pelo WhatsApp

Você manda no chat **"Mensagem para você mesmo"** (o bot nunca escreve no chat do primo):

| Comando | Faz |
|---|---|
| `? <pergunta>` | Responde com base no que foi salvo, citando os itens (ex.: `? o que o primo mandou sobre nota fiscal`) |
| `/semana` | Manda o digest da semana agora |
| `/pendentes` | Lista itens com próximo passo ainda aberto |
| `/pausar` · `/retomar` | Liga/desliga a captura |

### 5.5 Telas no PGBA (detalhe em `DESIGN_SYSTEM.md`)

1. **Caixa de entrada** — feed dos itens novos, triagem rápida (manter, arquivar, virar tarefa).
2. **Quadro** — Kanban: Novo → Para ler → Em ação → Concluído → Arquivo.
3. **Detalhe do item** — resumo, "por que importa", conteúdo extraído, a conversa ao redor,
   próximos passos, fontes.
4. **Pergunte** — chat sobre a base (reaproveita `KnowledgeChat`), sempre com fontes.
5. **Digest** — resumo semanal: o que entrou, por categoria, o que ficou pendente.
6. **Configurações** — conexão (QR code), chats permitidos, regras de captura, consentimento.

### 5.6 Digest semanal

Toda segunda às 8h40 (Celery beat), gerado pelos itens da semana:

- Quantos itens, de quem, por categoria.
- Os 3 mais relevantes, com uma linha cada.
- Próximos passos em aberto.
- Vai para o "Mensagem para você mesmo" de cada um (opcional) e fica salvo na tela Digest.

### 5.7 Integração com os agentes do PGBA

- A fonte "WhatsApp — Insights" pode ser ligada como `extra_knowledge_sources` de qualquer
  setor → o agente de Marketing passa a citar o artigo que seu primo mandou.
- `proximos_passos` viram **pedido de tarefa** (faixa de aprovação do Design System do PGBA)
  — nunca tarefa criada sem uma pessoa aprovar.
- Função registrada em `orchestration/registry.py`: `insights_por_periodo(tenant_id, de, ate,
  categoria)` para perguntas estruturadas ("quantos links de concorrentes este mês?").

## 6. Modelo de dados (app `insights`)

Todos com `TenantMixin + AuditMixin + SoftDeleteMixin`, conforme o `CLAUDE.md` do PGBA.

```
WhatsAppInstance           1 por pessoa conectada
├─ nome, evolution_instance, status_conexao, ultimo_evento_em
├─ secret_config (apikey, webhook_secret)  ← Fernet
└─ dono (User)

AllowedChat                chats que podem ser lidos
├─ instance → WhatsAppInstance
├─ jid (ex.: 55119...@s.whatsapp.net)  ← criptografado; exibido mascarado
├─ apelido ("Primo"), ativo, capturar_desde
└─ consentimento → core.ConsentRecord

CapturedMessage            mensagem bruta (só de chat permitido)
├─ chat → AllowedChat, message_id (único), autor ("eu" | apelido), enviada_em
├─ tipo (texto|link|documento|audio|imagem|video|reacao)
├─ texto (PII mascarada), urls[], midia (arquivo), respondendo_a
└─ status (ignorada|contexto|virou_insight)

Insight                    o item que a pessoa vê
├─ titulo, resumo, por_que_importa, tipo, categorias[], tags[]
├─ origem_url (canônica), arquivo, transcricao
├─ conteudo_extraido (texto)  ← vai pro Document
├─ status_pipeline (recebido|baixando|lendo|resumindo|pronto|falhou) + erro
├─ etapa_quadro (novo|para_ler|em_acao|concluido|arquivo)
├─ importante (📌), enviado_por, enviado_em, reenvios[]
├─ mensagens → CapturedMessage (M2M: origem + contexto)
├─ document_id  ← ingestion.Document (id solto)
└─ setor_sugerido

NextStep                   próximo passo sugerido
├─ insight → Insight, texto, estado (sugerido|pedido|virou_tarefa|descartado)
└─ task_id (id solto)

Digest
├─ periodo_inicio, periodo_fim, conteudo (markdown), enviado_em
└─ insights (M2M)
```

### 6.5 Export para Markdown / Obsidian (opcional)

Comando `python manage.py exportar_insights --pasta <dir>` gera uma nota por item no padrão
do vault (`title/tipo/tags/setor/responsavel/status/atualizado`), em `Insights/AAAA-MM/`.
Pode ser commitado neste repositório ou ir para o vault da empresa. **Decisão pendente**
(§10): o PGBA hoje só escreve no vault na criação e nas notas de lead.

## 7. Privacidade, LGPD e segurança

1. **Lista de permissão, não de bloqueio.** Só chats cadastrados em `AllowedChat` são lidos;
   o webhook descarta o resto **antes** de gravar (nem mensagem bruta fica).
2. **Consentimento do primo** registrado (`ConsentRecord`, finalidade "base de conhecimento
   da empresa"), com data. Sem consentimento ativo, o chat não é capturado.
3. **PII mascarada** (`redact_pii`) em tudo que vira `Document` — é o que alimenta a IA.
4. **Webhook autenticado** como no CRM (`apikey` no corpo ou `?token=`), mais rate limit e
   registro em `seguranca` quando falhar.
5. **Segredos** só em `secret_config` (Fernet); a tela mostra `••••1234`.
6. **Direito de apagar:** botão "Apagar tudo deste chat" (exclusão física, rotina de expurgo LGPD).
7. **Auditoria:** toda chamada de IA em `QueryLog`.
8. Mídia guardada no storage do PGBA com acesso só autenticado; nunca URL pública.

## 8. API (rascunho)

```
POST /api/v1/insights/webhook/<instance_public_id>/      Evolution → PGBA (sem login, autenticado)

GET  /api/v1/insights/items/?etapa=&categoria=&q=&de=&ate=
GET  /api/v1/insights/items/{id}/
PATCH /api/v1/insights/items/{id}/                        etapa, importante, tags, categorias
POST /api/v1/insights/items/{id}/reprocessar/
POST /api/v1/insights/items/{id}/proximos-passos/{n}/pedir-tarefa/

POST /api/v1/insights/perguntar/                          {pergunta} → {resposta, fontes[]}
GET  /api/v1/insights/digests/  ·  POST /api/v1/insights/digests/gerar/

GET/POST /api/v1/insights/instancias/                     cria instância na Evolution, devolve QR
GET  /api/v1/insights/instancias/{id}/status/
GET/POST/DELETE /api/v1/insights/chats-permitidos/
POST /api/v1/insights/importar-exportacao/                 .zip do "Exportar conversa"
```

## 9. Plano de entrega

| Fase | Entrega | Pronto quando |
|---|---|---|
| **0 — Prova** (1–2 dias) | Instância `insights` na Evolution, webhook gravando `CapturedMessage` do chat do primo | Link mandado no WhatsApp aparece no admin do Django |
| **1 — MVP** (1 semana) | Pipeline de links + PDF, resumo/classificação, `Document` indexado, tela Caixa de entrada + Detalhe | Pergunta no `KnowledgeChat` acha o link com fonte |
| **2 — Uso diário** | 📌 por reação, áudio → transcrição, deduplicação, Quadro, backfill do histórico | Os dois usam por 2 semanas sem abrir o admin |
| **3 — Ação** | Próximos passos → tarefa com aprovação, digest semanal, comandos `?` e `/semana` | Digest de segunda chega e é lido |
| **4 — Expansão** | Grupos permitidos, OCR de imagem, export para Obsidian/repo, ligar a setores | — |

## 10. Decisões em aberto

| # | Pergunta | Recomendação |
|---|---|---|
| D1 | Vertical no PGBA ou sistema separado? | **Vertical no PGBA** (§4) |
| D2 | Qual número conectar: seu pessoal, o do primo, ou os dois? | Começar com **o seu** (vê os dois lados da conversa). Os dois só se quiserem capturar chats do primo com terceiros. |
| D3 | Transcrição de áudio na nuvem (Groq) ou local? | Groq (barato, rápido) com decisão registrada; local (faster-whisper) se houver GPU. |
| D4 | Bot pode mandar mensagem (digest, respostas a `?`)? | Sim, **só** para "Mensagem para você mesmo". |
| D5 | Export do conhecimento: vault Obsidian, este repo, ou só PGBA? | Só PGBA no MVP; export na fase 4. |
| D6 | Quem vê o quê: você e o primo veem tudo? | Sim — mesma empresa (tenant), mesmo acesso. |
| D7 | Primo também usa a tela do PGBA? | Sim, usuário próprio no tenant. |

## 11. Glossário

- **Item / Insight** — uma unidade de conhecimento salva (link, documento, áudio, ideia).
- **Captura** — receber a mensagem do WhatsApp.
- **Enriquecimento** — baixar, ler, resumir, classificar.
- **Digest** — resumo periódico.
- **Chat permitido** — conversa que o sistema tem autorização para ler.
- **Evolution API** — servidor que conecta o WhatsApp ao PGBA (como WhatsApp Web).
