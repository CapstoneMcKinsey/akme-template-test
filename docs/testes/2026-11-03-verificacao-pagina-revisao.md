# Verificação da página de revisão como caixa de entrada de decisões

- **Data:** 2026-11-03
- **Plano:** `docs/superpowers/plans/2026-11-03-pagina-de-revisao.md`
- **Como:** projeto de teste descartável (dados da demo em `examples/local_demo`, ontologia do
  template, seed aplicado) com nove propostas pendentes de tipos diferentes, servido por
  `akme review --background --port 8766`, e a ontologia do Capstone (21 propostas da etapa 2)
  servida na porta 8765 só para leitura visual. Browser do Claude desktop, 1440 px, tema escuro
  e claro.

## O que foi conferido

| # | Cenário | Resultado |
|---|---|---|
| 1 | Página carrega sem erros de console nas duas portas; sidebar com stepper das seis etapas, fila e tipos; cabeçalho com "Etapa X de 6" | ok |
| 2 | Fila lista os nove tipos (CQ, lacuna, glossário, classe, relação, termo, mapping) com título legível e subtítulo | ok |
| 3 | Detalhe de relação mostra rótulo, comentário, multiplicidade em português, inversa, junção ou motivo de não haver, evidência com "conferir nas fontes" e, com junção, "o que o dado mostra" com alerta de fanout | ok |
| 4 | Aceitar uma CQ abre prioridade e uso com rótulos "Usar na construção" e "Guardar para o teste" e texto de ajuda | ok |
| 5 | Lacuna: escolha em lista, indicação "entra no pedido de dados", motivo obrigatório | ok |
| 6 | Atalhos `↑ ↓ a r` percorrem e decidem; item decidido sai de "A decidir" e entra em "Decididas"; barra fixa conta decididas e não salvas | ok |
| 7 | Salvar grava o bloco `review` em nove pendentes com revisor e hora; a etapa no cabeçalho passa de 1 para 2 | ok |
| 8 | Aplicar abre diálogo com resumo e nome; o lote roda em cópia com validate; resultado mostra "9 propostas entraram", tipos aplicados e a etapa seguinte; a fila esvazia e "Aplicadas" sobe para 17 | ok (cerca de 10 s) |
| 9 | Vista Ontologia depois da aplicação: 10 classes, grafo com a classe nova, cantos retos, badges em vez de chips | ok |
| 10 | Tema escuro: contraste de botões, badges e segmento pressionado (corrigida a precedência de hover sobre pressionado) | ok |
| 11 | Projeto sem `docs/index.html` (criado com `akme init`): `akme export` copia a página de `akme/static/` | ok |

## Problemas encontrados e corrigidos durante a verificação

1. Projeto criado fora do template não tinha a página. A página passou a viver em `akme/static/`
   e `akme export` copia para `docs/`.
2. Segmento pressionado perdia a cor no hover (seletor `:hover` com especificidade maior).
3. `local_folder.path` relativo em `akme.yml` é resolvido a partir do diretório de trabalho, não da
   raiz do projeto; o projeto de teste usa caminho absoluto. Não alterado.

## O que não foi verificado

- Largura de celular e navegação só por teclado nos formulários de CQ nova, alteração e exclusão
  de itens aplicados.
- Atualização automática com o agente propondo em paralelo (testada só pelo endpoint
  `/api/events` nos testes de unidade).
- Teste de usuário com uma pessoa nova no fluxo. As seis perguntas de
  `docs/testes/2026-10-30-teste-usuario-acme.md` têm resposta no desenho, mas falta a rodada.

## Ajustes depois da primeira rodada com o usuário (mesmo dia)

- O usuário achou a divisão lista + detalhe feia e difícil de usar e pediu uma proposta por vez.
  A Revisão virou um cartão por vez com barra de progresso, Anterior e Próxima; a lista lateral
  com scroll interno saiu.
- Textos repetidos foram cortados (faixa da etapa, introdução, "A decidir" em cada linha, selos
  de etapa, legendas e ajudas antes da escolha). O campo Revisor foi para a barra de ação.
- Barras de rolagem passaram a seguir o tema (`color-scheme`) com cor dos tokens.
- Evidências gravadas pelo agente como vários objetos JSON numa string passaram a ser lidas.
- O grafo foi mantido exatamente como na versão anterior, a pedido do usuário.
