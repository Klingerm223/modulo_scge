# ADR 0001 — Arquitetura Hexagonal + Clean Architecture

- **Status:** Aceito
- **Data:** 2026-05-15
- **Decisores:** Arquitetura SCGE
- **Escopo:** Todos os bounded contexts do SCGE.

## Contexto

O SCGE possui regras de negócio contábeis complexas e de longa vida útil
(regimes contábeis, MCASP, CPC, SPED). A tecnologia de borda (Spring Boot,
JPA, RabbitMQ, Keycloak, REST) é volátil e tende a ser substituída em ciclos
de 5 a 10 anos. Precisamos proteger o domínio contábil de qualquer mudança
tecnológica.

## Decisão

Adotar **Arquitetura Hexagonal (Ports & Adapters)** combinada com
**Clean Architecture**, com a seguinte estrutura por BC:

```
src/main/java/com/empresa/scge/<bc>/
├── domain/                   ← Java SE puro, sem dependências externas
│   ├── entity/
│   ├── valueobject/
│   ├── aggregate/
│   ├── repository/           ← interfaces (ports de saída)
│   ├── service/              ← domain services
│   ├── event/
│   └── exception/
├── application/              ← orquestração, sem regra de negócio
│   ├── usecase/
│   ├── port/in/              ← contratos dos use cases
│   ├── port/out/             ← contratos infra (publisher, clock, etc.)
│   ├── dto/
│   ├── mapper/
│   └── handler/              ← domain event handlers
├── infrastructure/           ← adapters de saída (driven)
│   ├── persistence/          ← JPA, repositórios, entidades de persistência
│   ├── messaging/            ← RabbitMQ publisher / consumers
│   ├── security/
│   ├── configuration/
│   ├── integration/          ← clients HTTP para outros BCs
│   └── external/
└── interfaces/               ← adapters de entrada (driving)
    ├── rest/                 ← controllers, openapi
    ├── consumer/             ← amqp listeners
    └── scheduler/
```

### Regras de Dependência (enforce com ArchUnit)
1. `domain` **não** depende de Spring, JPA, Jackson, nada externo.
2. `application` depende apenas de `domain`.
3. `infrastructure` e `interfaces` dependem de `application` e `domain`.
4. Nenhuma classe de `infrastructure`/`interfaces` é referenciada por `domain`/`application`.
5. Entidades JPA **não** são entidades de domínio — vivem em `infrastructure/persistence`.

## Alternativas Consideradas

| Alternativa | Por que não |
|-------------|-------------|
| Camadas tradicionais (Controller → Service → Repository com `@Entity` JPA no domínio) | Mistura domínio e infra; testes lentos; impossível trocar JPA sem reescrever regras. |
| Onion Architecture | Praticamente equivalente; escolhemos Hexagonal pela nomenclatura *ports/adapters* mais explícita. |
| Vertical Slice por feature | Difícil garantir invariantes do agregado entre slices; reavaliar quando o BC crescer. |

## Consequências

### Positivas
- Domínio testável sem Spring/banco.
- Troca de tecnologia de persistência ou broker = mudar adapter.
- Onboarding mais claro (camadas com responsabilidade óbvia).

### Negativas
- Mais código de *mapping* (Domain ↔ JPA Entity ↔ DTO).
- Curva de aprendizado para devs vindos de arquitetura em camadas.

### Mitigações
- MapStruct para mapeamentos repetitivos.
- ArchUnit no pipeline CI para travar violações.
- Testes de domínio rápidos compensam o boilerplate.
