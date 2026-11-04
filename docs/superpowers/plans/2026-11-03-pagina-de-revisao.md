# Plano: página de revisão como segundo canal do fluxo

Data: 2026-11-03. Estado de partida: branch `main` em 49a5e40, com a etapa 1 já reconstruída
(revisão de requisitos) e a ontologia do Capstone em andamento (21 edições da etapa 2 pendentes).

## 1. Diagnóstico do que existe

O `docs/index.html` tenta ser duas coisas numa página só: o grafo da ontologia (aba Ontologia) e
a tela de decisão (aba Revisão). Olhando a página servida por `akme review` com o estado atual:

| # | Problema | Onde | Efeito para quem usa |
|---|---|---|---|
| 1 | A página só decide edições da etapa 1 (CQs, lacunas, divergências, caso de uso, glossário). Classes, relações, termos, mappings, restrições e precedências aparecem em "Outros (aprovar pelo chat)" como um despejo de chave e valor | `akme/tools/ontology_write.py` `review_edit` linha 444 e `apply_reviewed` linha 481; `docs/review.js` `readonlyCard` | Justamente quando o volume cresce (etapas 2 e 3) o usuário é mandado de volta ao chat. Hoje são 21 itens nessa situação |
| 2 | A Revisão é um único scroll longo com sete grupos e nove tipos de cartão, sem fila nem progresso | `review.js` `render()` | Não dá para saber quantos faltam, por onde começar nem onde parou |
| 3 | Cada cartão mistura leitura e edição: campos de texto, selects, segmentos, chips, evidência, tudo aberto ao mesmo tempo | `pendingCqCard`, `gapCard`, `divCard` | Cartão alto, denso, com até nove controles visíveis antes de qualquer decisão |
| 4 | Rótulos internos vazam para a tela: "Ajuste", "Reservada", `add_relation`, `no_join_reason`, `many_to_one`, `proposta pela IA` em monoespaçada | `review.js` `USES`, `LABELS`, `kvList` | Teste de usuário de 30/10 já apontou "Uso: ajuste / reservada" como incompreensível |
| 5 | A página não diz em que etapa das seis o projeto está nem qual é o próximo passo | não existe | Problema 6 do teste de usuário |
| 6 | A página não abre sozinha e não atualiza sozinha; o agente diz "rode `akme review`" e o usuário precisa recarregar quando o agente propõe algo novo | `SKILL.md` etapa 1 passo 5; `review.js` só carrega no `init` | Quebra o vai e volta chat ↔ página |
| 7 | Desvios do QBDS: cantos arredondados em cartões, botões, abas e segmentos (6 a 10 px; o QBDS é reto), botão de rejeitar vermelho cheio, fonte por Google Fonts, ícone de marca próprio | CSS inline do `index.html` seção "aba Revisao" | A página não parece do mesmo produto que o guia |
| 8 | "Ver tudo" no grafo vira nuvem de rótulos de 9 px; o painel de detalhe é um terceiro bloco de informação disputando a tela | `index.html` script do grafo | Já registrado no teste de usuário; menor prioridade que 1 a 6 |

O que já está bom e deve ser preservado: tokens semânticos do QBDS em `:root` com modo claro e
escuro (`docs/design-tokens.md`), sem biblioteca e sem build, servidor só em `127.0.0.1` com
checagem de Host e Origin, decisões gravadas no bloco `review` do próprio pendente, aplicação em
lote validada em cópia (`batch.py`), e a regra de que nada entra sem nome de quem aprovou.

## 2. Princípio do redesenho

A página passa a ser **a caixa de entrada de decisões do consultor**, em qualquer etapa. O chat
continua sendo onde o agente raciocina e propõe; a página é onde o humano decide, com contexto
visual (evidência, dado observado, grafo). O contrato entre os dois não muda:

1. O agente chama `propose_edit`; o item aparece na página sem recarregar.
2. O humano decide na página (aceitar, aceitar com edição, rejeitar com motivo). Isso grava
   `review` no pendente, como hoje.
3. A aplicação continua exigindo nome de quem aprova e validação em cópia. Ela pode ser disparada
   pelo chat ("aplica") ou por um botão na página que chama o mesmo `apply_reviewed` (equivalente
   ao `akme apply --all-reviewed --approved-by NOME` que a CLI já oferece ao humano).

## 3. Decisões que precisam de confirmação antes de codar

| Decisão | Recomendação | Alternativa |
|---|---|---|
| A. A página decide edições das etapas 2 e 3 (classes, relações, termos, mappings, restrições, precedências)? | Sim. É o ganho principal: 21 dos 21 pendentes de hoje | Manter só etapa 1 e melhorar apenas o visual |
| B. Botão "Aplicar decisões" na página, com nome do aprovador e confirmação? | Sim, com diálogo que mostra o resumo (N aceitas, M rejeitadas) e o resultado do validate em cópia antes de promover | Página só grava; aplicação só pelo chat ou CLI |
| C. Grafo e Revisão continuam na mesma página? | Sim, uma página, mas a Revisão vira a tela inicial e o grafo vira apoio: ao selecionar uma classe ou relação pendente, o painel mostra a vizinhança dela no grafo | Duas páginas (`index.html` e `review.html`) |
| D. Manter sem build e sem framework? | Sim. HTML, CSS e JS modulares em `docs/`, D3 continua só para o layout do grafo | Migrar para o Next.js do guia (muito mais custo, e a página precisa funcionar aberta do disco) |
| E. Fontes | Inter e Roboto Mono servidas localmente em `docs/fonts/` (os `.woff2` já existem no repositório do guia, licença OFL) | Continuar com Google Fonts |

## 4. Fases

Cada fase fecha em um PR com testes. A ordem segue o valor para o usuário: primeiro o que o
teste de usuário e a situação atual mais pedem (fases 1 a 3), depois acabamento (4 e 5).

### Fase 1. Servidor e modelo: decisão em qualquer etapa

Arquivos: `akme/tools/ontology_write.py`, `akme/tools/batch.py`, `akme/review_server.py`,
`akme/models.py`, `tests/test_review.py`, `tests/test_review_server.py`.

- `review_edit` aceita todos os `EditType`. Para tipos da etapa 2 e 3, `edits` só pode tocar
  campos editáveis sem quebrar o contrato (label, comment, definition, synonyms, rationale);
  mudar domain, range, multiplicity ou join exige nova proposta, porque o `observed` foi
  calculado para a proposta original.
- `apply_reviewed` ganha `stage: "requirements" | "ontology" | "all"`; o portão da etapa 1
  continua valendo (edições da etapa 2 só aplicam com requisitos fechados).
- `/api/pending` devolve, para cada item, um bloco `view` já pronto para a tela: título legível,
  subtítulo, lista de campos com rótulo em português, evidência normalizada, `observed` (nulos,
  órfãos, multiplicidade observada) e os conflitos (`validate` em cópia só desse item, cacheado).
  Assim o JS não precisa conhecer os dezesseis tipos.
- Novos endpoints: `POST /api/apply` (body `{approved_by, stage}`, roda `apply_reviewed` e
  devolve o resumo), `GET /api/stage` (etapa atual das seis, o que falta para fechá-la e o próximo
  passo, derivado de `requirements_status`, contagem de classes, mappings e `validation.json`),
  `GET /api/events` (long-polling simples sobre o mtime de `ontology/pending/` e `content.yml`,
  para a página atualizar sozinha).
- `akme review` passa a aceitar `--background` (sobe o servidor desacoplado e imprime a URL) para
  o agente poder chamá-lo.

### Fase 2. Fundação visual QBDS

Arquivos: novo `docs/akme.css` (extraído do `<style>` inline), `docs/fonts/`, `docs/index.html`,
`docs/design-tokens.md`, `tests/test_docs_page.py`.

- Tokens: manter os de `design-tokens.md`, acrescentar `--rad-*` em 0 e `--radius-round`, as
  elevações 0 a 2 e as camadas de estado (hover 8 %, pressed 16 %).
- Componentes em CSS puro, com os mesmos nomes do guia: `btn` (default, secondary, outline,
  ghost; tamanhos sm e default; sem raio), `badge` (high-emphasis, alternative, error, warning,
  success, outline), `tag`, `callout` (nota, atenção, exemplo), `stepper` vertical (incomplete,
  active, completed, error), `table`, `input` e `select` com anel de foco de 2 px, `card` com
  inset de 28 px e elevação 0.
- Ícones: subconjunto de Material Symbols Sharp em `docs/fonts/` gerado como no guia
  (`scripts/material-symbols-subset.mjs`), ou SVG inline só dos doze ícones usados. Decidir pelo
  tamanho do arquivo.
- Cabeçalho de 56 px com o símbolo Akme do guia (`public/brand/akme-symbol-*.svg`), sidebar de
  16rem, coluna de leitura de 52rem para a Revisão e largura cheia só para o grafo.

### Fase 3. A Revisão como fila de decisões

Arquivos: `docs/review.js` (reescrito em módulos: `review/state.js`, `review/cards.js`,
`review/queue.js`, `review/apply.js`), `docs/index.html`.

Layout em três colunas, do mais geral para o mais específico:

1. **Sidebar: etapa e fila.** No topo o stepper das seis etapas (LOT), com a atual marcada e, em
   uma linha, o que falta para fechar. Abaixo, a fila agrupada por etapa e tipo, com contagem de
   pendentes sem decisão: "Requisitos · CQs 0", "Ontologia · classes 6 · relações 4 · termos 11".
   Filtros: sem decisão, decididas e não aplicadas, aplicadas.
2. **Lista.** Uma linha por item: ícone do tipo, título legível ("Relação matriculadoEm:
   Estudante → Curso", "Termo data_da_materia em MateriaNaMidia"), origem (IA ou nome), estado
   (sem decisão, aceita, rejeitada, aplicada) e alerta quando o `observed` contradiz a proposta.
   Setas do teclado percorrem; `A` aceita, `R` rejeita, `E` edita.
3. **Detalhe.** Cartão de leitura, não de edição: título, definição ou comentário em corpo 16 px,
   campos em lista rótulo e valor com nomes em português ("Multiplicidade: muitos para um",
   "Junção: pedidos.id_cliente → clientes.id_cliente", "Sem junção física porque ..."), evidência
   com caminho, página e trecho e botão "Conferir nas fontes", bloco "O que o dado mostra"
   (nulos, órfãos, chaves repetidas, com exemplos) e, para classe ou relação, miniatura da
   vizinhança no grafo. No rodapé do cartão, só dois botões: **Aceitar** (default) e **Rejeitar**
   (outline). "Editar antes de aceitar" abre os campos no lugar; "Rejeitar" abre o motivo.
   Para CQs, os controles de prioridade e uso ficam dentro de "Aceitar", com rótulos
   autoexplicativos: "Usar na construção (vira exemplo para o agente)" e "Guardar para o teste
   (exige a resposta esperada)".
4. **Barra de ação fixa.** "12 de 21 decididas · 9 sem decisão" e dois botões: **Salvar
   decisões** (grava `review` nos pendentes, como hoje) e **Aplicar decididas** (fase 1, item B),
   que pede o nome, mostra o resumo e o resultado do validate em cópia, e só então promove.
   Depois de aplicar, um callout diz o próximo passo ("Etapa 2 de 6 fechada. No chat, peça os
   mappings") e a etapa avança no stepper.

Comportamentos:

- Atualização automática por `/api/events`: quando o agente propõe algo, o item entra na fila
  com um marcador "novo"; quando o agente aplica pelo chat, os itens somem da fila e vão para
  "aplicadas". Rascunhos locais não se perdem.
- Modo leitura (arquivo aberto do disco ou `akme export` sem servidor): mesma tela, botões
  desabilitados com o motivo visível, callout "abra com `akme review` para decidir".
- Estados vazios com instrução: "Nada a decidir. No chat, peça ao agente para propor as classes
  da etapa 2".

### Fase 4. Integração com o fluxo do agente

Arquivos: `SKILL.md`, `.claude/skills/akme/SKILL.md`, `akme/cli.py` (`demo`), `README.md`,
`docs/guia-do-fluxo.md`, `tests/test_skill_text.py`.

- Em toda parada "Pare e pergunte" das etapas 1, 2 e 3, o agente: roda `export`, garante que o
  servidor está de pé (`akme review --background`), e escreve uma linha fixa: "Revise em
  http://127.0.0.1:8765 (N itens). Quando terminar, diga `revisei` ou aplique pela página".
- Ao retomar, o agente lê `list_pending` e trata os três casos: decididas e não aplicadas (pede
  "aplica" ou constata que foi aplicado pela página), aplicadas pela página (lê `changelog.yml` e
  segue), nada decidido (pergunta se prefere decidir no chat).
- Fim de etapa: toda resposta termina com "Etapa X de 6 concluída. Próximo passo: ..." (mesma
  frase que a página mostra, vinda de `/api/stage`).
- `akme demo` e README apontam para a URL, nunca para o arquivo.

### Fase 5. Grafo como apoio

Arquivos: `docs/graph.js` (extraído do inline), `docs/index.html`.

- A vizinhança continua o padrão. "Ver tudo" passa a agrupar por raiz (Entidade, Métrica,
  Regra, Relação) com rótulos a partir de 12 px e zoom mínimo que não corte texto.
- O painel de detalhe do grafo usa o mesmo cartão de leitura da Revisão (uma relação por linha:
  nome em cima, "de X para Y" embaixo).
- Classes e relações ainda pendentes aparecem no grafo tracejadas, com um selo "proposta", e o
  clique leva ao item na fila.

### Fase 6. Verificação

- Testes de unidade dos endpoints novos e do `review_edit` para cada `EditType`.
- `tests/test_docs_page.py` passa a validar a presença dos módulos e dos componentes QBDS.
- Repetir o teste de usuário de 30/10 com os dados do Capstone: as seis perguntas da tabela
  "Problemas de uso" precisam sair resolvidas; registrar em `docs/testes/`.
- Conferir em claro e escuro, em 1280 px e em largura de celular, e com teclado só.

## 5. Ordem e esforço

| Fase | Entrega | Depende de | Esforço |
|---|---|---|---|
| 1 | Servidor decide qualquer etapa, `/api/apply`, `/api/stage`, `/api/events`, `--background` | decisões A e B | médio |
| 2 | `akme.css` com tokens e componentes QBDS, fontes locais | decisão E | pequeno |
| 3 | Revisão como fila (sidebar, lista, detalhe, barra de ação) | 1 e 2 | grande |
| 4 | SKILL, CLI, README e guia alinhados ao novo vai e volta | 1 e 3 | pequeno |
| 5 | Grafo reorganizado e ligado à fila | 2 e 3 | médio |
| 6 | Testes e teste de usuário | todas | pequeno |

Fases 1 e 2 podem andar em paralelo. Fase 3 é o coração e deve ser revisada com um protótipo
estático (HTML com dados do Capstone) antes de ligar na API.

## 6. Fora de escopo deste plano

Avaliação pelo resultado, cobertura de CQs no validate, Databricks como metric view, edição e
remoção de classes já aplicadas (continuam na lista de `docs/guia-do-fluxo.md`, seção 6).
