# Collector Club — painel financeiro

Repositório do projeto "The Collector Club" (operação de compra e venda de relógios do João). Este repo guarda o código do site do painel e vai receber, via n8n, o snapshot de dados extraído da planilha Google Sheets.

## O que já existe

- **Site em produção:** https://collector-club-grupo-cavalcante.vercel.app
  - Publicado direto na Vercel (projeto `collector-club`, team `grupo-cavalcante`) a partir do `index.html` deste repositório, via upload de arquivo (não está linkado ao GitHub na Vercel ainda — ver "Próximos passos").
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

1. **Trigger — Lançamentos atualizado** (`googleSheetsTrigger`, `rowUpdate`, poll a cada 10 min).
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

Trocar `COLOQUE_AQUI_O_ID_DA_PLANILHA` pelo ID real da planilha em dois lugares: no `documentId` do trigger e na URL do batchGet.

O parsing (moeda BRL, negativo entre parênteses, percentual com vírgula, linhas raggeds do batchGet) foi testado contra um fixture representativo — KPIs, relógios, lançamentos, DRE e fluxo saem corretos. O que **não** foi testado ainda é o workflow rodando contra a planilha real.

## Decisões já tomadas (não reabrir sem necessidade)

- **Sem backend/serverless:** os dados moram como arquivo estático no GitHub, lidos via `raw.githubusercontent.com`. Não há função serverless nenhuma.
  - A justificativa antiga ("a Vercel não tem tooling para env vars/storage") estava errada: o projeto `pegae-machine` do João usa exatamente isso — uma rota `/api/ingest` na Vercel com `INGEST_SECRET` em env var. A decisão de ficar sem backend continua válida por simplicidade, mas não por impossibilidade.
- **Sem autenticação de aplicação:** o painel não tem login próprio (decisão explícita do João). **Atenção:** hoje a URL não está pública — o projeto na Vercel está com Deployment Protection ligada e devolve 302 para o SSO da Vercel. Precisa desligar em Settings → Deployment Protection pra o João conseguir abrir.
- **Atualização periódica, não real-time:** o site dá refresh a cada 5 min enquanto aberto; a atualização "de verdade" depende do n8n rodar (manual por enquanto — agendamento automático só depois do fluxo estar validado ponta a ponta).
- **n8n é o responsável por escrever no GitHub**, não o Claude — evita o Claude precisar de credenciais de escrita no repo.
- **Repositório público** (`panaderiapontaverde/collector-club`), confirmado pelo João.

## Próximos passos

- [ ] Importar `collector-club-n8n-workflow.json` no n8n, preencher o ID da planilha (2 lugares) e ligar as 3 credenciais.
- [ ] Rodar uma vez com o botão de teste e conferir o `resumo` do nó "Montar snapshot" (contagem de relógios/lançamentos/meses) antes de deixar comitar.
- [ ] Desligar a Deployment Protection na Vercel — sem isso o painel não abre pro João.
- [ ] Ativar o workflow (`active: true`) depois de validado ponta a ponta.
- [ ] Opcional: linkar o projeto da Vercel a este repositório GitHub, pra o `index.html` fazer deploy automático a cada push em vez de upload manual.
