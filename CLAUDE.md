# Collector Club — painel financeiro

Repositório do projeto "The Collector Club" (operação de compra e venda de relógios do João). Este repo guarda o código do site do painel e vai receber, via n8n, o snapshot de dados extraído da planilha Google Sheets.

## O que já existe

- **Site em produção:** https://collector-club-grupo-cavalcante.vercel.app
  - Publicado na Vercel (projeto `collector-club`, team `grupo-cavalcante`) a partir do `index.html` deste repositório. **O projeto está ligado ao GitHub** (confirmado pelo João em 2026-09-11): push na `main` redeploya sozinho, sem upload manual.
  - **O conector da Vercel disponível ao Claude não enxerga este projeto.** `list_projects` no time `grupo-cavalcante` volta vazio e `get_project` dá 404, embora a URL responda — o conector está autenticado numa conta que não é dona do projeto. Consequência prática: o Claude não consegue ler logs, redeployar nem mexer em Settings. Não tente criar projeto novo pra contornar: `create_git_project` não reconecta projeto existente, criaria um segundo deploy e uma segunda URL.
  - Página estática única (`index.html`), sem build step, sem framework.
  - Busca os dados em `https://raw.githubusercontent.com/panaderiapontaverde/collector-club/main/data/snapshot.json` a cada carregamento e a cada 5 min (`setInterval`).
  - Cache em `localStorage` (`collector-club-snapshot-cache`) como fallback se o GitHub estiver fora do ar — mostra um banner "dados podem estar desatualizados" nesse caso.
  - Mostra: 9 KPIs (faturamento, lucro líquido, saldo de caixa, capital em estoque, peças em estoque, margem líquida, dívida com sócio, patrimônio bruto/líquido), estoque por status, gráfico de resultado mensal (últimos 12 meses), e 3 abas com tabelas pesquisáveis: Relógios, Lançamentos, Fluxo de caixa.
  - Tema claro/escuro automático (`prefers-color-scheme`), fontes Fraunces (display) + Inter (corpo) + IBM Plex Mono (números).

- **Fonte dos dados:** Google Sheets "Financeiro — The Collector Club" (planilha real do João), abas: Instruções, Dashboard, Relógios, Lançamentos, DRE, Fluxo de Caixa, Checks, Listas.

- **`data/snapshot.json`:** existe, semeado vazio (schema completo, arrays em branco). Serve pra dois fins: o site parar de tomar 404 no `raw.githubusercontent`, e a API contents do GitHub ter um `sha` pro n8n sobrescrever. Enquanto o workflow não rodar, o painel mostra estado vazio.

## Schema do snapshot.json

```json
{
  "updatedAt": "ISO 8601",
  "mesAnalisado": "string ou null",
  "kpis": {
    "faturamento": number|null, "lucroLiquido": number|null, "saldoCaixa": number,
    "capitalEstoque": number, "pecasEstoque": number, "margemLiquida": number,
    "dividaSocio": number, "patrimonioBruto": number, "patrimonioLiquido": number
  },
  "estoquePorStatus": [{ "status": string, "qtd": number }],
  "watches": [{ "itemId","marca","modelo","referencia","condicao","status",
    "dataCompra","dataEntrada","dataVenda","canal","precoAnuncio","custoCompleto",
    "vendaBruta","deducoes","lucroLiquido","margem","dias","lote","observacoes" }],
  "lancamentos": [{ "id","dataCompetencia","dataCaixa","itemId","lote","descricao",
    "natureza","grupoDre","categoria","contraparte","conta","forma","moeda",
    "valorOriginal","cambio","valorBRL","status","vencimento","documento",
    "observacoes","centroCusto","tratamento","precisaoData" }],
  "dreMensal": [{ "mes","receitaBruta","deducoes","receitaLiquida","cmv",
    "lucroBruto","despesasOp","resultadoLiquido","margemLiquida" }],
  "fluxoCaixa": [{ "mes","entradas","saidas","saldoMes","saldoAcumulado" }]
}
```

## Pipeline de atualização (n8n → GitHub → site)

Workflow pronto pra importar: [`collector-club-n8n-workflow.json`](collector-club-n8n-workflow.json). Seis nós:

1. **Trigger — Lançamentos atualizado** (`googleSheetsTrigger`, **`anyUpdate`**, poll a cada 10 min). Tem que ser "Row Added or Updated": lançamento novo é linha *acrescentada*, e `rowUpdate` sozinho só enxerga alteração em linha existente — o fluxo não disparava por isso.
2. **Ler planilha (batchGet)** — um único HTTP Request no `spreadsheets.values:batchGet` trazendo as 6 abas (Checks, Dashboard, Relógios, Lançamentos, DRE, Fluxo de Caixa) como arrays de célula crus.
3. **Montar snapshot** (Code) — confere o `STATUS GERAL` da aba Checks e monta o `snapshot.json` no schema acima, devolvendo o conteúdo já em base64.
4. **Publicação pronta?** (IF) — só segue se `STATUS GERAL = OK`.
5. **Buscar sha atual** — `GET /contents/data/snapshot.json`, com `onError: continueRegularOutput`.
6. **Commit data/snapshot.json** — `PUT /contents/...` na branch `main`.

### Por que HTTP Request e não os nós nativos

- **Google Sheets:** o nó nativo (v4.5) consome a primeira linha do range como cabeçalho e devolve objetos, não arrays de célula. No Dashboard isso comeria justamente a linha de KPIs. O `batchGet` devolve a matriz crua. De quebra, uma request no lugar de cinco elimina o fan-in: cinco nós ligados na mesma entrada de um Code node fazem ele executar cinco vezes, o que geraria cinco commits idênticos por rodada.
- **GitHub:** o nó nativo codifica o conteúdo em base64 sozinho — mandar base64 pra ele gera base64 duplo e um arquivo ilegível. E sua operação `file:edit` estoura 404 quando o arquivo não existe, sem caminho de fallback. Com a API crua dá pra fazer `GET sha → PUT`, tratando o 404 como "arquivo novo".

### Credenciais necessárias no n8n

| Nó | Tipo de credencial |
|---|---|
| Trigger | `googleSheetsTriggerOAuth2Api` |
| Ler planilha (batchGet) | `googleSheetsOAuth2Api` — **é um registro separado do usado pelo trigger** |
| Buscar sha / Commit | `githubApi` (PAT com escopo de escrita em `contents` deste repo) |

O ID da planilha (`1GN251Lcp8mP8fqITpxXLpXL-864Lvio8WYwEkgdEvvs`) já está preenchido no `documentId` do trigger e na URL do batchGet.

### Layout do Dashboard (não estreitar o range)

O Dashboard tem **12 colunas**. Os KPIs vivem em três blocos de células mescladas de 3 colunas: A-C, E-G e I-K. Um range `A1:H50` corta fora o terceiro bloco inteiro — `SALDO DE CAIXA`, `MARGEM LÍQUIDA` e `PATRIMÔNIO LÍQUIDO` viram zero em silêncio, sem erro nenhum. Por isso o range é `Dashboard!A1:L50`.

Os rótulos são casados por chave normalizada (sem acento, caixa alta), não por texto literal, porque o Sheets pode devolver a forma decomposta do acento ou espaço não-quebrável. O `resumo` do nó traz `rotulosDashboard` justamente pra diagnosticar isso quando um KPI vier zerado.

`faturamento` e `lucroLiquido` como `null` são **corretos** quando a planilha traz `-` nessas células — não é falha de parsing.

### Acentuação: não há problema (não "consertar")

O snapshot sempre saiu em UTF-8 correto — `Em trânsito` grava os bytes `C3 A2`, e a
forma duplo-codificada nunca apareceu no arquivo, nem no primeiro commit.

Se alguma inspeção sugerir mojibake, suspeite primeiro da ferramenta: num terminal
Windows, `curl ... | python -c "json.load(sys.stdin)"` lê o stdin com a codificação do
console (cp1252) e **produz** o mojibake na leitura. Para inspecionar de verdade, baixe
para arquivo e abra com `encoding='utf-8'` explícito, ou compare os bytes crus.

O pipeline rodou ponta a ponta contra a planilha real em 2026-09-10 (commit `0df92f3`): 8 relógios, 43 lançamentos, 24 meses de DRE e 24 de fluxo, todos corretos. Essa primeira execução expôs o range estreito do Dashboard, já corrigido.

## Decisões já tomadas (não reabrir sem necessidade)

- **Sem backend/serverless:** os dados moram como arquivo estático no GitHub, lidos via `raw.githubusercontent.com`. Não há função serverless nenhuma.
  - A justificativa antiga ("a Vercel não tem tooling para env vars/storage") estava errada: o projeto `pegae-machine` do João usa exatamente isso — uma rota `/api/ingest` na Vercel com `INGEST_SECRET` em env var. A decisão de ficar sem backend continua válida por simplicidade, mas não por impossibilidade.
- **Sem autenticação de aplicação:** o painel não tem login próprio (decisão explícita do João). **Atenção:** hoje a URL não está pública — o projeto na Vercel está com Deployment Protection ligada e devolve 302 para o SSO da Vercel. Precisa desligar em Settings → Deployment Protection pra o João conseguir abrir.
- **Atualização periódica, não real-time:** o site dá refresh a cada 5 min enquanto aberto; a atualização "de verdade" depende do n8n rodar (manual por enquanto — agendamento automático só depois do fluxo estar validado ponta a ponta).
- **n8n é o responsável por escrever no GitHub**, não o Claude — evita o Claude precisar de credenciais de escrita no repo.
- **Repositório público** (`panaderiapontaverde/collector-club`), confirmado pelo João.

## Próximos passos

- [x] Importar o workflow, preencher o ID da planilha e ligar as 3 credenciais.
- [x] Primeira execução ponta a ponta, com commit em `data/snapshot.json`.
- [x] Reimportar o workflow corrigido e rodar: os 9 KPIs vieram preenchidos (`saldoCaixa` R$ 3.714,21, `patrimonioLiquido` R$ 5.193,63, `mesAnalisado` setembro/2026).
- [ ] Desligar a Deployment Protection na Vercel — sem isso o painel não abre pro João.
- [ ] Ativar o workflow (`active: true`).
- [ ] Avaliar troca do Google Sheets Trigger por Schedule Trigger + comparação de conteúdo: some o furo da correção que fica esperando a próxima edição, e evita commit sem mudança.
