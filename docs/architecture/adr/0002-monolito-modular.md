# ADR 0002 — Monolito Modular com Extração Futura para Microsserviços

- **Status:** Aceito
- **Data:** 2026-05-15

## Contexto

O SCGE será dividido em 7 bounded contexts. Há tensão entre:
- **Microsserviços desde o início** — alinhamento perfeito com DDD, mas
  custo operacional alto (7 deploys, 7 pipelines, 7 bancos, observabilidade,
  *service mesh*, mensageria desde o dia zero, *distributed tracing*).
- **Monolito clássico** — rápido para começar, mas costuma colapsar nos
  limites de domínio (acoplamento por banco compartilhado, repos genéricos).

A equipe inicial é pequena; o domínio ainda está sendo descoberto.

## Decisão

Construir um **Monolito Modular** com **fronteiras de bounded context
rigorosamente respeitadas**, preparado para extração de microsserviços
quando justificado por:

- Volume (ex.: BC Auditoria com 100M eventos/mês).
- Cadência de deploy independente.
- Time dedicado por BC.
- Requisitos diferentes de SLA/escala.

### Regras do Monolito Modular
1. **Um módulo Maven por BC** dentro do mesmo *parent*.
2. **Um schema PostgreSQL por BC**, mesmo cluster físico inicialmente.
3. **Sem joins cross-schema**, sem `@JoinColumn` cross-BC.
4. Comunicação entre BCs **apenas** por:
   - Evento de domínio (RabbitMQ in-process via Spring Events para começar; broker real desde o dia 1 para Auditoria).
   - API REST do BC alvo (via *port* + adapter HTTP).
5. **Shared kernel mínimo**: módulo `shared-kernel` apenas para envelope de
   evento, `Money`, `Cnpj`, `TenantId`, `Periodo`. Sem regras de negócio.
6. ArchUnit garante que pacotes de um BC não importem pacotes internos de outro.

### Caminho de Extração
Quando um BC justificar microsserviço:
1. Trocar adapter de evento in-process por broker real (já é o caso).
2. Trocar adapter HTTP por client REST de fato.
3. Mover schema para cluster próprio.
4. Mover módulo Maven para repositório próprio + pipeline próprio.
5. **Domínio não muda.**

## Alternativas Consideradas

| Alternativa | Por que não agora |
|-------------|-------------------|
| Microsserviços desde o dia 1 | Custo operacional + descoberta de domínio ainda em curso. |
| Monolito clássico | Falha em respeitar fronteiras → *big ball of mud*. |
| Modulith (Spring Modulith) | Avaliar — atrativo, mas queremos controle explícito do isolamento de schema, que Modulith não impõe. |

## Consequências

### Positivas
- Deploy único inicial; observabilidade simples.
- Refactor entre BCs ainda é seguro (mesma JVM + tipos).
- Custos de infra baixos.

### Negativas
- Risco de "vazamento" entre BCs por preguiça. **Mitigado** com ArchUnit.
- Escalabilidade vertical inicial (mesma JVM).

### Limites Disparadores de Extração
- Throughput sustentado >500 req/s por BC.
- Necessidade de release independente.
- Time dedicado >5 devs por BC.
