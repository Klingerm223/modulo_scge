# SCGE — Linguagem Ubíqua

> Glossário canônico do domínio. **Toda** classe, método, endpoint, tabela,
> coluna, evento e log deve usar estes termos — em português, sem tradução,
> sem abreviações criativas. Quando o termo difere entre contabilidade pública
> e privada, ambos são registrados.

---

## 1. Núcleo Contábil (BC Contabilidade)

| Termo | Definição | Sinônimos proibidos |
|-------|-----------|---------------------|
| **Lançamento Contábil** | Registro de um fato contábil composto por **partidas** (débitos e créditos), sempre balanceado. Agregado raiz. | `transacao`, `registro`, `entry`. |
| **Partida** | Linha de um lançamento. Possui natureza (D/C), conta, valor e histórico. | `linha`, `item`. |
| **Natureza** | Indica se a partida é **Débito** ou **Crédito**. Enum binário. | `tipo`, `sinal`. |
| **Plano de Contas** | Estrutura hierárquica de contas contábeis de uma empresa/exercício. Agregado raiz. | `chart of accounts`, `coa`. |
| **Conta Contábil** | Nó do plano. Possui código, natureza patrimonial, classificação (analítica/sintética). | `account`, `rubrica`. |
| **Conta Analítica** | Conta-folha. **Aceita** lançamento. | `final`, `leaf`. |
| **Conta Sintética** | Conta agrupadora. **Não aceita** lançamento (invariante). | `pai`, `summary`. |
| **Histórico** | Texto descritivo padronizado de uma partida ou lançamento. | `descricao`, `memo`. |
| **Histórico Padrão** | Template reutilizável de histórico com placeholders. | `template`. |
| **Exercício** | Ano contábil (1º jan – 31 dez, salvo exceções). | `ano`, `year`. |
| **Período** | Janela temporal dentro de um exercício (mês, trimestre). | `competencia`. |
| **Competência** | Mês contábil ao qual o fato pertence (regime de competência). | `mes`. |
| **Empresa** | Pessoa jurídica (privada) ou ente (público) titular dos lançamentos. **Tenant**. | `cliente`, `org`. |
| **Centro de Custo** | Dimensão analítica de custo/responsabilidade. | `cc`, `costcenter`. |
| **Origem** | BC que solicitou o lançamento (Financeiro, Patrimônio, manual). | `source`. |
| **Estorno** | Lançamento que neutraliza outro, mantendo histórico de auditoria. | `cancelamento`, `reverse`. |

### 1.1. Estados do Lançamento

```
RASCUNHO → VALIDADO → REGISTRADO → [ ESTORNADO ]
                                  → [ CONSOLIDADO no fechamento ]
```

| Estado | Significado |
|--------|-------------|
| **Rascunho** | Em edição. Não afeta saldos. Não publica evento. |
| **Validado** | Passou em todas as invariantes (balanceamento, conta analítica, período aberto). |
| **Registrado** | Persistido. Afetou saldos. Evento `LancamentoRegistrado` publicado. |
| **Estornado** | Neutralizado por lançamento de estorno. Mantém histórico. |
| **Consolidado** | Pertence a período fechado. Imutável. |

---

## 2. Dualidade Público × Privado

| Conceito | Privada (CPC) | Pública (MCASP) | Modelagem |
|----------|---------------|------------------|-----------|
| Plano de contas | Societário livre | **PCASP** obrigatório | `PerfilContabil` (Strategy) |
| Regime | Competência | Misto (orçamentária por caixa, patrimonial por competência) | `RegimeContabil` (VO) |
| Demonstração principal | DRE, Balanço | BP, DVP, DFC, Balanço Orçamentário | `DemonstrativoStrategy` |
| Encerramento | Lucro/Prejuízo | Superávit/Déficit | `ApuradorResultado` (port) |
| Reporte externo | SPED ECD/ECF | SICONFI/SIAFI | BC Fiscal |

> Decisão: **um único modelo de domínio**, com *Strategies* injetadas conforme
> o `PerfilContabil` da empresa. Ver
> [ADR 0003](adr/0003-plano-de-contas-dual.md).

---

## 3. Saldos e Apuração

| Termo | Definição |
|-------|-----------|
| **Saldo** | Valor acumulado de uma conta em um período. Pode ser inicial, movimento ou final. |
| **Saldo Devedor** | SUM(débitos) > SUM(créditos). |
| **Saldo Credor** | SUM(créditos) > SUM(débitos). |
| **Movimento** | Soma de partidas de uma conta dentro de um período. |
| **Balancete** | Relatório de saldos por conta em um período. |
| **Razão** | Extrato de partidas de uma conta em um período. |
| **Diário** | Listagem cronológica de lançamentos. |

---

## 4. Fechamento (BC Fechamento — referência)

| Termo | Definição |
|-------|-----------|
| **Fechamento Mensal** | Bloqueio de alterações retroativas no período. |
| **Apuração de Resultado** | Transferência de saldos de receitas/despesas para conta de resultado. |
| **Virada de Exercício** | Encerramento anual + abertura do próximo exercício com saldos iniciais. |
| **Período Aberto / Fechado** | Status que governa se o BC Contabilidade aceita lançamento na competência. |

---

## 5. Eventos de Domínio (canônicos do BC Contabilidade)

| Evento | Quando ocorre | Consumidores |
|--------|---------------|--------------|
| `LancamentoRegistrado.v1` | Lançamento sai de Validado → Registrado. | Auditoria, Fiscal, Patrimônio. |
| `LancamentoEstornado.v1` | Estorno é registrado. | Auditoria, Fiscal. |
| `SaldoContaAtualizado.v1` | Saldo de uma conta sofre alteração. | Fiscal, dashboards. |
| `PlanoDeContasPublicado.v1` | Nova versão do plano entra em vigor. | Auditoria. |
| `PeriodoFechado.v1` | Período entra em estado fechado (emitido por Fechamento; **consumido** aqui). | (consumo) |

---

## 6. Termos Proibidos / Antipadrões

| Não use | Use |
|---------|-----|
| `LancamentoDTO` em domínio | `Lancamento` (agregado). DTO só em `application`/`interfaces`. |
| `LancamentoService` genérico | Use Case nominal: `RegistrarLancamentoContabilUseCase`. |
| `tipo` (string mágica) | Enum / VO. |
| `update()` no agregado | Método de intenção: `estornar()`, `consolidar()`. |
| `valor: BigDecimal` solto | VO `Valor` (com moeda). |
| `data: LocalDate` solto | VO `Competencia` ou `DataLancamento`. |
| `delete` de lançamento | Estorno (append-only). |
