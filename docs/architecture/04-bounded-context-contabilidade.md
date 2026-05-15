# BC Contabilidade — Visão Detalhada

> Aprofundamento do **Bounded Context Contabilidade**, alvo deste workspace.
> Define escopo, agregados (resumo), invariantes, contratos publicados,
> contratos consumidos e regras de negócio nucleares.

---

## 1. Missão do Bounded Context

> *"Garantir que todo fato econômico seja registrado de forma íntegra,
> balanceada, auditável e em conformidade com o regime contábil da empresa,
> produzindo saldos confiáveis para apuração e demonstrativos."*

---

## 2. Escopo

### 2.1. Dentro do escopo
- Plano de contas (criação, versionamento, vigência).
- Lançamentos contábeis (registro, estorno, consulta).
- Partidas dobradas e validação de balanceamento.
- Apuração de **saldos** por conta/competência/centro de custo.
- Diário, Razão, Balancete (queries).
- Publicação de eventos canônicos.
- Suporte dual MCASP/CPC via *Strategy*.

### 2.2. Fora do escopo (delegado)
- Apuração de resultado e fechamento → **BC Fechamento**.
- Geração de SPED/SICONFI → **BC Fiscal**.
- Cálculo de depreciação → **BC Patrimônio** (publica evento, Contabilidade só registra).
- Conciliação bancária → **BC Financeiro**.
- Trilha de auditoria detalhada → **BC Auditoria** (consumidor dos eventos).
- Identidade e RBAC → **BC Autenticação / Keycloak**.

---

## 3. Agregados (visão estratégica — detalhe tático virá na fase 2)

| Agregado Raiz | Entidades Internas | Value Objects Chave | Invariantes |
|---|---|---|---|
| **LancamentoContabil** | `Partida` | `Valor`, `Natureza`, `Competencia`, `CodigoConta`, `Historico` | Balanceamento; mínimo 2 partidas; mesma competência; conta analítica; período aberto. |
| **PlanoDeContas** | `Conta` (árvore) | `CodigoConta`, `NaturezaPatrimonial`, `Classificacao` | Códigos hierárquicos consistentes; sintética não recebe lançamento; vigência sem sobreposição. |
| **Saldo** | — (projeção) | `Valor`, `Competencia`, `MovimentoDevedor`, `MovimentoCredor` | Saldo final = inicial + débitos − créditos (ou inverso conforme natureza). |
| **HistoricoPadrao** | — | `CodigoHistorico`, `Template` | Placeholders válidos. |

> **Saldo é uma projeção** (read model) materializada por evento
> `LancamentoRegistrado`, não um agregado transacional. CQRS leve.

---

## 4. Invariantes Críticas

### 4.1. Balanceamento (núcleo)
```
∀ Lancamento L:
    SUM(L.partidas.where(Natureza = DEBITO).valor)
  = SUM(L.partidas.where(Natureza = CREDITO).valor)
```
Validação no construtor/factory do agregado. **Lançamento desbalanceado nunca existe.**

### 4.2. Conta Analítica
```
∀ Partida P:
    P.conta.classificacao = ANALITICA
```

### 4.3. Período Aberto
```
∀ Lancamento L:
    PeriodoStatus(L.competencia, L.empresa).estado = ABERTO
```
Validação via *port* `PeriodoStatusPort` (consulta ao BC Fechamento, com cache local
alimentado pelo evento `PeriodoFechado`).

### 4.4. Mesma Competência
```
∀ Partida P em L:
    P.competencia = L.competencia
```

### 4.5. Multitenancy
```
∀ Partida P em L:
    P.conta.tenant = L.tenant
∀ Conta C em Plano:
    C.tenant = Plano.tenant
```

### 4.6. Imutabilidade pós-registro
Após `REGISTRADO`, o agregado **não aceita mutações** — apenas estorno
(que cria novo lançamento neutralizador).

---

## 5. Contratos Publicados (Outbound)

### 5.1. APIs REST (resumo — OpenAPI completo na fase 2)

| Verbo | Path | Caso de Uso |
|-------|------|-------------|
| POST | `/v1/lancamentos` | Registrar lançamento |
| POST | `/v1/lancamentos/{id}/estorno` | Estornar |
| GET | `/v1/lancamentos/{id}` | Consultar |
| GET | `/v1/lancamentos` | Listar (filtros) |
| POST | `/v1/planos-de-contas` | Criar versão |
| POST | `/v1/planos-de-contas/{id}/publicar` | Publicar vigência |
| GET | `/v1/balancetes` | Consultar balancete |
| GET | `/v1/razoes/{conta}` | Razão analítica |

### 5.2. Eventos (Published Language)
- `contabilidade.lancamento.registrado.v1`
- `contabilidade.lancamento.estornado.v1`
- `contabilidade.saldo.atualizado.v1`
- `contabilidade.plano-de-contas.publicado.v1`

---

## 6. Contratos Consumidos (Inbound)

### 6.1. Eventos consumidos
| Evento | Origem | Ação |
|--------|--------|------|
| `financeiro.pagamento.efetuado.v1` | Financeiro | Registrar lançamento via ACL. |
| `financeiro.recebimento.efetuado.v1` | Financeiro | idem. |
| `patrimonio.depreciacao.apurada.v1` | Patrimônio | idem. |
| `patrimonio.bem.baixado.v1` | Patrimônio | idem. |
| `fechamento.periodo.fechado.v1` | Fechamento | Atualizar cache local `PeriodoStatus`. |
| `fechamento.periodo.reaberto.v1` | Fechamento | idem. |

### 6.2. Anti-Corruption Layer
Para cada evento externo, há um **tradutor** em
`infrastructure/messaging/acl/` que converte payload externo → comando interno
(`RegistrarLancamentoCommand`). Domínio nunca conhece o formato externo.

---

## 7. Regras de Negócio Nucleares (resumo executivo)

1. Lançamento desbalanceado é rejeitado (HTTP 422 / DLQ no consumer).
2. Conta sintética em partida é rejeitada (HTTP 422).
3. Lançamento em período fechado é rejeitado (HTTP 409).
4. Mesma `correlationId` processada duas vezes retorna o mesmo lançamento
   (idempotência via tabela `processed_message`).
5. Estorno cria novo lançamento com partidas invertidas; não apaga o original.
6. Plano de contas tem **vigência por exercício**; mudança gera nova versão,
   nunca alteração em versão publicada.
7. Saldo é projeção: rebuild possível a qualquer momento a partir do log de eventos.
8. Toda partida carrega `tenant_id`, `exercicio`, `competencia`, `centro_custo` (opcional).

---

## 8. Estratégia de Persistência (visão executiva)

| Tabela | Particionamento | Índices Chave |
|--------|-----------------|---------------|
| `lancamento` | RANGE por `exercicio` | `(tenant_id, competencia)`, `(correlation_id)` UNIQUE |
| `partida` | RANGE por `exercicio` (herda) | `(conta_id, competencia)`, `(lancamento_id)` |
| `conta` | — | `(tenant_id, codigo)` UNIQUE, GIST em `path` (ltree) |
| `saldo_mensal` | RANGE por `exercicio` | `(tenant_id, conta_id, competencia)` PK |
| `outbox_event` | RANGE por `created_at` (mensal) | `(status, created_at)` parcial em `status='PENDING'` |
| `processed_message` | RANGE por `created_at` | `(correlation_id, event_type)` UNIQUE |

Detalhamento completo + DDL na fase 2.

---

## 9. Riscos Técnicos

| Risco | Mitigação |
|-------|-----------|
| Volume de partidas (milhões/ano) | Particionamento por exercício + BRIN em `competencia`. |
| Rebuild de saldos | Snapshot mensal + replay incremental. |
| Concorrência em saldo | Atualização via *upsert* idempotente por `(conta, competencia, lancamento_id)`. |
| Acoplamento com BC Fechamento | Cache local de `PeriodoStatus` + evento; chamada síncrona só no caminho de exceção. |
| Drift entre evento e estado | **Outbox Pattern** (mesma transação). |
| Latência cross-BC | Eventual consistency assumida; UI mostra "processando". |

---

## 10. Melhorias Futuras

- **Event Sourcing puro** para `LancamentoContabil` (hoje é state-based + outbox).
- **CQRS pleno**: read model em projeção materializada separada.
- Migração de RabbitMQ → Kafka para *event log* da Auditoria.
- *Schema-per-tenant* para grandes corporações.
- Integração nativa com OpenSearch para razão analítica.
- **Dual Write Verifier** (job de reconciliação outbox ↔ broker).
