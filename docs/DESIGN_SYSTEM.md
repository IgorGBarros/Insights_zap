---
title: Insights Zap — Design System
status: rascunho v0.1
atualizado: 2026-11-14
base: pgba-boilerplate/docs/Designer System.md
---

# Insights Zap — Design System

O Insights Zap é uma vertical do PGBA, então **herda o Design System do PGBA inteiro**
(`pgba-boilerplate/docs/Designer System.md`): tokens, tipografia, espaçamento, `Button`,
`Tabs`, `.panel`, padrão Kanban, faixa de aprovação de agentes, acessibilidade e a lista
"Não fazer". Este documento só acrescenta o que é **específico do Insights**.

> Regra de ouro (herdada): tela nova usa só o que está no DS do PGBA + o que está aqui.
> Precisa de algo novo? Documente aqui primeiro.

---

## 1. Princípios do Insights

1. **O resumo vem primeiro, a fonte está sempre a um clique.** Todo item mostra de onde veio
   (link, arquivo, áudio, quem mandou, quando). Resumo sem fonte não aparece.
2. **Triagem em segundos.** A Caixa de entrada é feita para decidir rápido: manter, arquivar,
   virar tarefa. Teclado primeiro (`j/k`, `e`, `t`).
3. **Contexto da conversa, não a conversa.** Mostramos as mensagens ao redor só para explicar
   o "porquê". Não é um cliente de WhatsApp.
4. **Status real do pipeline.** "Lendo PDF…", "Falhou: página exige login" — nunca um spinner
   genérico nem resumo inventado.
5. **Sem verde de WhatsApp.** Verde no PGBA significa sucesso. Balões de conversa usam
   `secondary`/`elevated`; a identidade do WhatsApp aparece só no ícone do canal.

---

## 2. Tokens novos

Nenhuma cor nova. O Insights mapeia seus conceitos para tokens e etiquetas que já existem.

### 2.1 Categorias → `COR_ETIQUETA` (fixas, não escolhidas pelo usuário)

| Categoria | Cor da etiqueta | Por quê |
|---|---|---|
| Mercado | `sky` | |
| Produto | `violet` | |
| Marketing | `pink` | |
| Finanças | `teal` | evita `green`/`emerald` (= sucesso) |
| Jurídico | `indigo` | |
| Tecnologia | `cyan` | |
| Operações | `orange` | evita `amber` (= atenção) |
| Pessoas | `rose` | evita `red` (= erro) |

Aplicação: `EtiquetaCartao` do PGBA (fundo `color-mix(cor 18%, transparente)` + texto na cor).
Máximo 2 categorias visíveis no cartão; o resto como `+1`.

### 2.2 Tipo de item → ícone (lucide-react, `size-4`, `text-muted-foreground`)

| Tipo | Ícone | Rótulo |
|---|---|---|
| artigo | `Newspaper` | Artigo |
| video | `PlayCircle` | Vídeo |
| documento | `FileText` (PDF) / `FileSpreadsheet` / `File` | Documento |
| audio | `AudioLines` | Áudio |
| ferramenta | `Wrench` | Ferramenta |
| ideia | `Lightbulb` | Ideia |
| noticia | `Rss` | Notícia |
| referencia | `Bookmark` | Referência |
| tarefa | `CheckSquare` | Tarefa |

Ícone nunca colorido: a cor é reservada para categoria e status.

### 2.3 Status do pipeline → tons semânticos

| Status | Tom | Texto | Ícone |
|---|---|---|---|
| recebido / baixando / lendo / resumindo | `muted-foreground` | "Baixando link…", "Lendo PDF (12 pág.)…", "Transcrevendo áudio (2 min)…", "Resumindo…" | `Loader2` com `animate-spin` só aqui |
| pronto | — (não mostra chip) | — | — |
| conteúdo não acessível | `warning` | "Não deu para ler: página exige login" | `Lock` |
| falhou | `destructive` | mensagem real do erro + "Tentar de novo" | `AlertCircle` |

Chip = `StatusChip` do PGBA (`rounded-md px-2 py-0.5 text-xs font-medium text-<tom> bg-<tom>/10`).

### 2.4 Autor

- `AvatarResponsavel` do PGBA (iniciais, cor fixa pelo nome, `size-6`).
- "Eu" para mensagens `fromMe`; apelido do chat permitido para o outro lado ("Primo").
- Nunca mostrar número de telefone; se precisar, mascarado (`•••• 4321`).

### 2.5 Importante (📌)

Ícone `Pin` preenchido em `text-foreground`, no canto do cartão. Não usar amarelo/âmbar
(âmbar = esperando/atenção).

---

## 3. Componentes

| Componente | Base no PGBA | Especificação |
|---|---|---|
| **`InsightCard`** | `CartaoQuadro` | Ordem fixa: **etiquetas (categorias) → título → resumo (2 linhas, `line-clamp-2`, `text-xs text-muted-foreground`) → rodapé**. Rodapé: ícone do tipo + domínio/nome do arquivo (`font-mono text-[11px]`) à esquerda; 📌, contagem de próximos passos, avatar de quem mandou e data curta à direita. Status de pipeline substitui o resumo enquanto não está pronto. |
| **`InsightRow`** | — (novo) | Versão em linha para a Caixa de entrada: `h-16`, ícone do tipo · título + resumo de 1 linha · categorias · autor · data. Linha selecionada: `bg-elevated` + `ring-1 ring-ring`. Lida/não lida: título `font-semibold` quando novo, `font-medium` depois. |
| **`FonteDoItem`** | — (novo) | Bloco com a origem: favicon + domínio + "Abrir original ↗" (link), ou arquivo com tamanho e "Baixar", ou player de áudio nativo com duração. |
| **`ContextoConversa`** | — (novo) | Até 5 mensagens antes/depois, em balões **neutros**: as minhas à direita `bg-elevated`, as do outro à esquerda `bg-secondary`, `rounded-xl px-3 py-2 text-sm`, hora em `text-[11px] text-muted-foreground`. A mensagem que originou o item tem `ring-1 ring-ring`. Colapsado por padrão ("Ver conversa"). |
| **`ProximosPassos`** | faixa de aprovação | Lista com checkbox. Cada passo tem "Virar tarefa" → cria **pedido** (faixa `warning/5` do PGBA), nunca tarefa direta. |
| **`ResumoIA`** | — (novo) | `panel p-4`: "Resumo" (`text-sm font-semibold`) + texto; "Por que importa" em `text-sm` com borda esquerda `border-l-2 border-border pl-3`. Rodapé `text-[11px] text-muted-foreground`: "Resumo gerado por IA a partir de <fonte> · modelo · Reprocessar". |
| **`FiltrosInsights`** | `Tabs` segmentado + `Select` | Abas de etapa (`Novos 12 · Para ler 5 · Em ação 3 · Tudo`), filtros de categoria, tipo, autor e período; busca por palavra. |
| **`PerguntaBox`** | `KnowledgeChat` | Campo grande "Pergunte sobre o que vocês salvaram…" + resposta com **fontes como `InsightCard` compacto**. Sem fonte = "Não encontrei nada salvo sobre isso." |
| **`ConexaoWhatsApp`** | `panel` | Estado da instância: Conectado (`success`), Aguardando QR (`warning`, mostra QR `size-56` com borda), Desconectado (`destructive` + "Reconectar"). Último evento recebido em texto relativo ("há 3 min"). |
| **`ChatPermitidoRow`** | — | Apelido, número mascarado, "Capturando desde 10/11", estado do consentimento (chip), botões Pausar e "Apagar dados deste chat" (`destructive` + `ConfirmDialog`). |

---

## 4. Telas

Todas dentro de `ModuloLayout` do PGBA (grupo → tela → seção), módulo **Insights** com estas
telas: Caixa de entrada · Quadro · Pergunte · Digest · Configurações.

### 4.1 Caixa de entrada (tela padrão)

```
┌ Insights ─────────────────────────────────────────────────────────────────────┐
│ [Novos 12|Para ler 5|Em ação 3|Tudo]   Categoria ▾  Tipo ▾  Quem ▾  [Buscar…] │
├──────────────────────────────────────┬────────────────────────────────────────┤
│ ● 📰 Como precificar SaaS B2B…        │ [Mercado] [Finanças]               📌  │
│     Artigo sobre ancoragem e…  PR 2h │ Como precificar SaaS B2B no Brasil      │
│ ● 📄 Contrato_modelo_v3.pdf           │ ┌ Fonte ─────────────────────────────┐ │
│     Modelo de contrato de pr… EU 5h  │ │ 🌐 exemplo.com.br · Abrir original ↗│ │
│   🎧 Áudio 2:14 — ideia de onboard…   │ └─────────────────────────────────────┘ │
│     Transcrevendo áudio…       PR 1d │ Resumo ·········                       │
│   ⚠ linkedin.com/posts/…              │ Por que importa ·········              │
│     Não deu para ler: exige login    │ Próximos passos  ☐ … [Virar tarefa]    │
│                                      │ ▸ Ver conversa (4 mensagens)            │
│                                      │ [Arquivar] [Mover para…]  [Para ler ▸] │
└──────────────────────────────────────┴────────────────────────────────────────┘
```

- Lista (`InsightRow`) à esquerda, detalhe à direita (≥1024px). Abaixo disso: lista em tela
  cheia e detalhe em página/modal.
- Teclado: `j/k` navega, `Enter` abre, `e` arquiva, `l` "para ler", `t` virar tarefa, `p` 📌,
  `/` busca. Atalhos mostrados no `title` dos botões.
- Ação principal da área (único `Button default`): mover para a próxima etapa.

### 4.2 Quadro

Padrão Kanban do PGBA, sem mudanças: listas **Novo · Para ler · Em ação · Concluído ·
Arquivo** com `InsightCard`. Mover é otimista. Cabeçalho da lista mostra contagem (não
soma em R$). Ponto da lista: tons neutros (`muted-foreground`), exceto Concluído (`success`).

### 4.3 Detalhe do item

Mesmo conteúdo do painel direito da Caixa de entrada, em modal (`ModalShell`) quando aberto
pelo Quadro. Seções na ordem: Cabeçalho (categorias, título, 📌, autor, data) → `FonteDoItem`
→ `ResumoIA` → `ProximosPassos` → `ContextoConversa` → "Conteúdo extraído" (colapsado,
`font-body text-sm`, `max-h-96 overflow-y-auto`) → Tags editáveis → Histórico (reenvios).

### 4.4 Pergunte

Uma coluna, `max-w-3xl` centralizada. `PerguntaBox` no topo; sugestões de pergunta feitas a
partir das categorias mais frequentes ("O que salvamos sobre Jurídico este mês?"). Resposta
em `panel p-4` com as fontes abaixo.

### 4.5 Digest

Lista de digests por semana à esquerda; conteúdo à direita: `Metric` do PGBA no topo (itens
na semana, de quem, pendentes), depois "Destaques" (3 `InsightCard`), "Por categoria"
(barras horizontais, skill `dataviz`), "Próximos passos em aberto". Botão "Enviar no
WhatsApp" (`outline`).

### 4.6 Configurações

Seções: Conexão (`ConexaoWhatsApp`) · Chats permitidos (`ChatPermitidoRow` + "Adicionar
chat" com aviso de consentimento) · Regras de captura (toggles: links, documentos, áudio
>45 s, 📌, hashtags; campo de duração mínima do áudio) · Digest (dia, hora, enviar no
WhatsApp sim/não) · IA (provedor de resumo e de transcrição, mostrando onde roda — local ou
nuvem) · Privacidade (exportar, apagar tudo).

---

## 5. Estados

| Situação | O que mostrar |
|---|---|
| Carregando lista | Texto "Carregando itens…" centralizado (padrão PGBA, sem skeleton decorativo) |
| Caixa vazia (nada novo) | "Nada novo. Tudo que vocês mandarem com link, arquivo ou 📌 aparece aqui." |
| Sem conexão com o WhatsApp | Faixa `destructive/5` no topo: "WhatsApp desconectado desde 13/11 14:20. Mensagens novas não estão chegando." + "Reconectar" |
| Nenhum chat permitido | Vazio com ação: "Escolha qual conversa o Insights pode ler" → Configurações |
| Busca sem resultado | "Nada com essa busca." |
| Erro da API | Mensagem real + "Tentar de novo" |
| Item falhou | Chip `destructive` com motivo + "Tentar de novo" no item |

---

## 6. Texto e tom

- Português direto, voz ativa: "Virar tarefa", "Abrir original", "Ver conversa", "Apagar
  dados deste chat".
- Datas no cartão: relativas até 24 h ("há 2 h"), depois curtas ("12 de nov."); completas no
  detalhe ("12 de novembro de 2026, 14:32").
- Duração de áudio `m:ss`. Tamanho de arquivo `1,2 MB`.
- Nunca "IA mágica", "inteligente", ✨. A IA é identificada com texto pequeno ("Resumo gerado
  por IA a partir de…").

---

## 7. Responsivo

- **≥1280px:** lista + detalhe lado a lado (lista `w-[420px]`).
- **768–1279px:** lista + detalhe, lista `w-[340px]`, resumo da linha escondido.
- **<768px (≈390px):** só a lista; detalhe em tela cheia com "← Voltar". Quadro rola na
  horizontal com listas de `w-[272px]` (padrão PGBA). Margem lateral 16px.

---

## 8. Acessibilidade (além do PGBA)

- Lista da Caixa de entrada com `role="listbox"`, itens `role="option"` + `aria-selected`.
- Status do pipeline em `aria-live="polite"` no detalhe (muda de "Lendo PDF" para pronto).
- Player de áudio nativo, com a transcrição logo abaixo (alternativa em texto).
- QR code com texto alternativo e instrução em texto ("WhatsApp → Aparelhos conectados →
  Conectar aparelho").

---

## 9. Checklist de tela do Insights

- [ ] Checklist de tela nova do PGBA cumprido (tokens, estados, dois temas, 1440px e 390px).
- [ ] Todo resumo mostra a fonte e a indicação de que foi gerado por IA.
- [ ] Nenhum telefone em claro; autor aparece como "Eu" ou apelido.
- [ ] Categorias com as cores fixas da §2.1; ícones de tipo sem cor.
- [ ] Status de pipeline real (texto do passo/erro), nunca spinner genérico.
- [ ] Ação de agente (próximo passo → tarefa) aparece como pedido com aprovação.
