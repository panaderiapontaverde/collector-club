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

- **`data/snapshot.json`:** ainda não existe neste repo. Vai ser criado/atualizado pelo workflow do n8n descrito abaixo. Até isso rodar, o site mostra estado vazio.

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

Desenho completo (nó a nó + código JS) está no guia publicado: pedir pro Claude reabrir ou consultar o histórico da conversa em claude.ai/code/session_011JNSMZpF7PEpiaSaUitdHE. Resumo:

1. Google Sheets Trigger (rowUpdate na aba Lançamentos).
2. Lê a aba **Checks** e confere a linha `STATUS GERAL` — só segue se `= OK` (gate de "pronto para publicar", reaproveitando a verificação de integridade que já existe na planilha, em vez de um checkbox manual novo).
3. Lê Dashboard, Relógios, Lançamentos, DRE, Fluxo de Caixa (raw, arrays de células).
4. Code node monta o `snapshot.json` seguindo o schema acima (parsing de moeda BRL, porcentagem, datas).
5. Nó do GitHub (File → Edit) comita `data/snapshot.json` na branch `main` deste repo.

Esse pipeline foi validado nesta sessão com um script Python equivalente (`build_snapshot.py`) rodando sobre uma leitura real da planilha — os números de KPIs, 8 relógios, 43 lançamentos, 24 meses de DRE e 24 meses de fluxo de caixa saíram corretos.

## Decisões já tomadas (não reabrir sem necessidade)

- **Sem backend/serverless:** Vercel não tem tooling neste projeto para provisionar storage (Blob/KV) nem env vars — por isso os dados moram como arquivo estático no GitHub, lido via `raw.githubusercontent.com`, e não há função serverless nenhuma.
- **Sem autenticação:** acesso é só para o João, painel fica público na URL mas sem login (decisão explícita dele).
- **Atualização periódica, não real-time:** o site dá refresh a cada 5 min enquanto aberto; a atualização "de verdade" depende do n8n rodar (manual por enquanto — agendamento automático só depois do fluxo estar validado ponta a ponta).
- **n8n é o responsável por escrever no GitHub**, não o Claude — evita o Claude precisar de credenciais de escrita no repo.
- **Repositório público** (`panaderiapontaverde/collector-club`), confirmado pelo João.

## Próximos passos

- [ ] Montar o workflow no n8n (guia já entregue).
- [ ] Rodar o workflow uma vez, confirmar o commit de `data/snapshot.json` e ver o site popular.
- [ ] Opcional: linkar o projeto na Vercel a este repositório GitHub (via `create_git_project`) para deploy automático do `index.html` a cada push, em vez de upload manual.
- [ ] Decidir junto com o João se/como agendar a execução automática do trigger do n8n.
