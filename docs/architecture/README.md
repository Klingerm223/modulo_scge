# Documentação Arquitetural — SCGE / Módulo Contabilidade

Estrutura de documentação do projeto.

## Strategic Design (DDD)

| # | Documento | Conteúdo |
|---|-----------|----------|
| 01 | [Strategic Design](01-strategic-design.md) | Visão geral, BCs, Context Map, padrões de relacionamento. |
| 02 | [Linguagem Ubíqua](02-ubiquitous-language.md) | Glossário canônico do domínio (com dualidade MCASP/CPC). |
| 03 | [Diagramas C4](03-c4-diagrams.md) | Níveis 1, 2 e zoom hexagonal do BC Contabilidade. |
| 04 | [BC Contabilidade](04-bounded-context-contabilidade.md) | Detalhe do BC alvo deste workspace. |

## Architecture Decision Records (ADRs)

| # | Decisão |
|---|---------|
| 0001 | [Arquitetura Hexagonal + Clean Architecture](adr/0001-arquitetura-hexagonal.md) |
| 0002 | [Monolito Modular com Extração Futura](adr/0002-monolito-modular.md) |
| 0003 | [Plano de Contas Dual MCASP/CPC com Strategy](adr/0003-plano-de-contas-dual.md) |
| 0004 | [Outbox Pattern para Eventos de Domínio](adr/0004-outbox-pattern.md) |

## Próximas Fases

1. ✅ **Strategic Design** — concluída.
2. ⏭ **Modelagem Tática do BC Contabilidade**
   - Agregados detalhados (`LancamentoContabil`, `PlanoDeContas`, `Saldo`).
   - Value Objects.
   - Domain Events com schema JSON.
   - Diagrama UML de classes.
   - Especificação dos Use Cases.
   - Contrato OpenAPI v1.
3. ⏭ **Scaffolding Maven multi-módulo**
   - `pom.xml` parent + módulos hexagonais.
   - Spring Boot 3.3.x + Java 21.
   - Flyway, Testcontainers, ArchUnit, MapStruct.
   - `docker-compose.yml` (Postgres + RabbitMQ + Keycloak).
4. ⏭ **Use Case vertical: `RegistrarLancamentoContabil`**
   - Ponta a ponta: REST → UseCase → Domain → JPA → Outbox → Evento.
   - Testes: unitário, integração com Testcontainers, contrato.

> Avise quando quiser prosseguir para a Fase 2.
