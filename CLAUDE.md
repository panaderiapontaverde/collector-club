# Collector Club — painel financeiro (v2)

Repositório do projeto "The Collector Club" (operação de compra e venda de relógios do João). Este repo guarda o código do site do painel e recebe, via n8n, o snapshot de dados extraído da planilha Google Sheets.

## Mudança de planilha-fonte (2026-09-13)

A partir desta versão, a fonte de dados deixou de ser "Financeiro — The Collector Club" (planilha antiga, com abas Checks/Dashboard/Relógios/Lançamentos/DRE/Fluxo de Caixa) e passou a ser **"Controle Financeiro Simples — The Collector Club"**:

`https://docs.google.com/spreadsheets/d/1Y2lnqYZdzThhyVAoL5O3FCgNPrX1LZcmF4B15M2UMdU`

Essa troca não foi um ajuste de ID só — a estrutura é fundamentalmente diferente (visão acumulada em vez de mensal, sem aba de integridade/checks, tabelas mais enxutas). Por isso o schema do snapshot, o parsing no n8n e o `index.html` foram todos reescritos (v2). A v1 (planilha antiga) fica só como histórico no git.

**Nota de segurança:** essa mesma planilha ("Controle Financeiro Simples") foi acidentalmente commitada neste repositório público em 2026-09-11 por um `git add -A` displicente, e depois expurgada do histórico via `git filter-branch` (ver commit `3003b94`). O `.gitignore` adicionado então já cobre esse tipo de arquivo — não desfazer essa proteção.

## Abas da planilha nova (todas relevantes para o snapshot, exceto Simulador)

| Aba | Linha do header | Colunas | Conteúdo |
|---|---|---|---|
| Dashboard | — (pares label/valor em A/B) | A:B | Posição financeira, DRE acumulada, Indicadores — tudo acumulado, sem quebra mensal |
| Estoque | 4 | A:Q (17 col) | Um item por linha: compra → custos → venda |
| Vendas de Terceiros | 4 | A:L (12 col) | Consignação — relógios de clientes vendidos pela loja |
| Caixa | 4 | A:J (10 col) | Livro-razão simples: uma linha por movimentação, com saldo corrente na coluna J |
| Gerenciador de Anúncios | 1 | A:K (11 col) | Métricas de tráfego pago (Meta Ads) |
| Simulador | — | — | Calculadora, não entra no snapshot |

Não existe mais aba "Checks"/`STATUS GERAL` — **o workflow não tem gate de publicação**, publica direto a cada disparo do trigger. Se quiser reintroduzir uma trava, precisa criar essa aba do zero na planilha nova.

## Schema do snapshot.json (v2 — mudou por completo em relação à v1)

```json
{
  "updatedAt": "ISO 8601",
  "posicaoFinanceira": {
    "caixaDisponivel": 0, "contasReceber": 0, "contasPagar": 0, "estoquePeloCusto": 0,
    "valoresReceberSocio": 0, "ativosOperacionais": 0, "emprestimosRecebidos": 0,
    "emprestimosPagos": 0, "saldoEmprestimosPagar": 0, "patrimonioLiquido": 0
  },
  "dreAcumulada": {
    "vendasBrutas": 0, "taxasVendas": 0, "vendasLiquidas": 0, "custoItensVendidos": 0,
    "despesasOperacionais": 0, "receitaVendasTerceiros": 0, "resultadoLiquido": 0
  },
  "indicadores": { "itensEstoque": 0, "itensVendidos": 0, "margemLiquidaAcumulada": 0 },
  "estoque": [{ "dataCompra","item","origem","valorBase","comissao","iof","frete","imposto",
    "outrosCustos","custoFinal","dataVenda","canalVenda","valorVenda","taxasVenda",
    "valorLiquido","lucro","margem" }],
  "vendasTerceiros": [{ "dataVenda","relogio","proprietario","valorTotalVendido",
    "valorRepassar","comissaoLoja","taxasDespesas","receitaLiquida","recebido",
    "dataRecebimento","formaPagamento","observacoes" }],
  "caixa": [{ "data","descricao","categoria","itemRelacionado","vencimento","realizado",
    "entrada","saida","formaPagamento","saldo" }],
  "anuncios": [{ "data","campanha","alcance","impressoes","cliques","cpc","ctr",
    "valorGasto","resultados","custoPorResultado" }]
}
```

## Pipeline de atualização (n8n → GitHub → site) — v2

Workflow pronto pra importar: `collector-club-n8n-workflow-v2.json`. Quatro nós, sem IF/gate:

1. **Trigger — Caixa atualizado** (`googleSheetsTrigger`, `anyUpdate`, planilha `1Y2lnqYZdzThhyVAoL5O3FCgNPrX1LZcmF4B15M2UMdU`, aba `Caixa`, poll a cada 10 min).
2. **Ler planilha (batchGet)** — um único HTTP Request em `spreadsheets.values:batchGet` trazendo 5 ranges de uma vez, nesta ordem exata (o Code node depende da ordem): `Dashboard!A1:B30`, `Estoque!A4:Q200`, `'Vendas de Terceiros'!A4:L300`, `Caixa!A4:J300`, `'Gerenciador de Anúncios'!A1:K300`.
3. **Montar snapshot** (Code) — monta o `snapshot.json` no schema acima, casando rótulos do Dashboard por chave normalizada (sem acento, caixa alta — mesma lição da v1), devolvendo o conteúdo já em base64.
4. **Buscar sha atual** — `GET /contents/data/snapshot.json`, com `onError: continueRegularOutput` (primeira execução: arquivo ainda não existe, sha vem vazio).
5. **Commit data/snapshot.json** — `PUT /contents/...` na branch `main`, incluindo `sha` só quando ele existir (senão o GitHub cria o arquivo).

### Credenciais necessárias no n8n

| Nó | Tipo de credencial |
|---|---|
| Trigger | `googleSheetsTriggerOAuth2Api` |
| Ler planilha (batchGet) | `googleSheetsOAuth2Api` — registro separado do usado pelo trigger |
| Buscar sha / Commit | `githubApi` (PAT com escopo de escrita em contents deste repo) |

O ID da planilha nova já está preenchido no `documentId` do trigger e na URL do batchGet.

## Site (index.html) — o que mudou

- KPIs trocaram de "9 indicadores mensais" para 9 indicadores da visão **acumulada**: caixa disponível, patrimônio líquido, vendas brutas, resultado líquido, margem líquida acumulada, estoque pelo custo, itens em estoque, saldo de empréstimos a pagar, a receber do sócio.
- O gráfico de "resultado mensal" (que dependia de DRE/Fluxo de Caixa mês a mês, que não existem mais) foi substituído por um gráfico de **evolução do saldo de caixa**, construído a partir da própria coluna "Saldo" (corrente) da aba Caixa — dado que já existe na planilha, sem precisar de agregação.
- As 3 abas de tabela viraram: **Estoque** (era Relógios), **Caixa** (era Lançamentos, agora bem mais simples — 10 colunas em vez de 23), **Vendas de terceiros** (nova) e **Anúncios** (nova, mostra as últimas 30 linhas de tráfego pago).
- Continua: tema claro/escuro automático, fontes Fraunces/Inter/IBM Plex Mono, fetch do `raw.githubusercontent.com` a cada carregamento e a cada 5 min, cache em `localStorage` (chave nova `collector-club-snapshot-cache-v2` — não conflita com cache antigo de visitantes que já tinham a v1 aberta).

## Incidente de 14/09/2026 — painel em branco (ler antes de mexer no workflow)

O workflow publicou um `snapshot.json` com o schema certo e **todos os valores zerados**, e o painel do João ficou em branco no ar. Duas causas somadas, as duas já corrigidas:

**1. Ranges em `queryParameters` se sobrescrevem.** O nó batchGet mandava cinco entradas chamadas `ranges` pelo campo "Query Parameters" do HTTP Request. O n8n monta a query como objeto, então chaves repetidas colapsam e **só a última é enviada**. A API devolvia 1 `valueRange` em vez de 5, e o Code node — que lia por posição fixa (`vr[0]`, `vr[1]`…) com `|| []` de fallback — transformava a ausência em lista vazia, sem erro.

Por isso os ranges agora vão **montados na URL**, nunca em `queryParameters`.

**2. Não havia trava de sanidade.** O Code node não tinha um único `throw`. Publicou o vazio por cima de um snapshot bom sem nenhum sinal. Hoje ele estoura se a aba Caixa vier sem lançamentos, se a Estoque vier sem itens, ou se o Dashboard vier com caixa e vendas ambos em zero. **Não remover essas travas:** execução falhando é visível no n8n; painel apagado em silêncio não é.

### Triggers: Schedule + Manual, não Google Sheets Trigger (14/09/2026)

O Google Sheets Trigger foi removido. Ele falhou três vezes de formas diferentes:

1. Em `rowUpdate` não disparava, porque lançamento novo é linha *acrescentada*, não alterada.
2. Mesmo em `anyUpdate`, uma correção feita enquanto o dado estava ruim ficava esperando a *próxima* edição pra publicar.
3. Depois de importar o workflow, **nunca rodava**: trigger de polling exige o workflow ativado, a primeira sondagem só grava a linha de base, e ainda depende de alguém editar a aba.

No lugar entram dois triggers ligados no mesmo fluxo:

- **Executar agora** (Manual) — para rodar na hora, logo depois de importar.
- **A cada 15 min** (Schedule) — não depende de detectar edição nenhuma.

Para o Schedule não gerar ~100 commits idênticos por dia, o nó **Mudou?** compara o snapshot recém-montado com o que já está no GitHub (o `GET /contents` devolve o conteúdo junto com o `sha`) e o IF seguinte só deixa passar quando diferem. O campo `updatedAt` é excluído da comparação de propósito: ele muda a cada execução e faria todo snapshot parecer diferente.

### Como o workflow lê a planilha agora

Nada depende do nome nem da ordem das abas:

1. **Descobrir abas** — `GET /spreadsheets/{id}?fields=sheets.properties.title` pergunta à planilha quais abas existem.
2. **Ler planilha (batchGet)** — a URL é montada por expressão a partir desses títulos; range sem A1 significa a aba inteira.
3. **Montar snapshot** — cada aba é reconhecida pela **linha de cabeçalho** (Estoque tem "Data da compra"+"Item"+"Custo final", Caixa tem "Realizado?"+"Saldo", Anúncios tem "Campanha"+"CTR", Dashboard tem "Posição financeira"), e cada coluna é lida **pelo rótulo**, não pelo índice. Aba renomeada, reordenada ou com coluna inserida no meio não quebra mais o fluxo.

O `resumo` na saída do Code node traz as contagens e o campo `saldoConfere`, que compara o saldo da última linha do Caixa com o "Caixa disponível" do Dashboard.

### Conferência do parser (14/09/2026)

Validado contra o conteúdo real da planilha, com as abas passadas em ordem embaralhada de propósito. Todos os cruzamentos fecham com o próprio Dashboard:

| Derivado das linhas | Dashboard |
|---|---|
| soma do custo dos itens sem venda | `estoquePeloCusto` R$ 9.799,47 |
| soma das vendas dos itens vendidos | `vendasBrutas` R$ 22.700,00 |
| soma do custo dos itens vendidos | `custoItensVendidos` R$ 17.519,27 |
| saldo da última linha do Caixa | `caixaDisponivel` R$ 26.714,21 |
| 8 vendidos / 2 em estoque | `indicadores` |

### Revisão de 17/09/2026 contra a planilha

A planilha **renomeou** a coluna `Custo final` da aba Estoque para **`Custo registrado`** e acrescentou duas colunas: `Custo final projetado` e `Custos ainda estimados`. Por isso o Code node agora aceita **rótulos alternativos** — cada item de uma assinatura pode ser uma lista, e o leitor de célula tenta os nomes em ordem. Assinatura presa a um nome único faz o fluxo inteiro estourar quando alguém renomeia uma coluna.

O painel usa a projeção da planilha quando ela chega no snapshot (`custoProjetado` / `custosEstimados`) e só cai numa estimativa própria — pela média histórica dos custos já lançados — enquanto esses campos não vierem. O rodapé da aba Estoque diz qual das duas está valendo. A estimativa local chega perto mas não acerta em cheio: a taxa real da planilha é outra (o Simulador usa 20% / 5% / 5,7% / 18,3% / 0%).

Conferido item a item: com as colunas novas no snapshot, o painel reproduz exatamente os R$ 25.615,95 de estoque projetado e os R$ 3.627,64 de custo ainda estimado.

## Decisões já tomadas (não reabrir sem necessidade)

- Sem backend/serverless, sem autenticação de aplicação, repositório público, atualização periódica (não real-time) — mesmas decisões da v1, continuam válidas.
- **Sem gate de publicação** nesta v2 — decisão tomada por não existir mais aba de integridade na planilha nova. Se o João quiser um botão "só publica se eu confirmar", precisa criar essa aba/checkbox na planilha nova primeiro.
- Site linkado à Vercel via GitHub (push na main redeploya sozinho) — projeto `collector-club`, time `grupo-cavalcante`. Deployment Protection ainda pode estar ligada (ver pendência abaixo).

## Próximos passos

1. Importar `collector-club-n8n-workflow-v2.json` no n8n (substitui ou convive com o workflow v1 — desativar o v1 pra não gerar dois commits concorrentes).
2. Preencher as 2 credenciais Google Sheets + 1 GitHub.
3. Rodar manualmente uma vez, conferir o commit em `data/snapshot.json` e os 9 KPIs no site.
4. Confirmar se a Deployment Protection da Vercel está desligada (pendência que já vinha da v1).
5. Ativar o workflow (`active: true`) quando a primeira execução estiver validada.
