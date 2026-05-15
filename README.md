# modulo_scge

Módulo de Contabilidade do **SCGE** (Sistema Contábil Gerencial Empresarial).

Bounded Context **Contabilidade** — núcleo do sistema. Suporte dual a
contabilidade pública (MCASP/PCASP) e privada (CPC/SPED).

## Stack
- Java 21 LTS · Spring Boot 3.3.x · Maven
- PostgreSQL 16 · Flyway · JPA/Hibernate
- RabbitMQ · Keycloak (OIDC/JWT)
- Arquitetura Hexagonal + Clean Architecture + DDD

## Documentação Arquitetural
Veja [docs/architecture/README.md](docs/architecture/README.md) para
Strategic Design, Context Map, Linguagem Ubíqua, diagramas C4 e ADRs.

## Status
Fase atual: **Strategic Design concluído**. Próxima: modelagem tática.
