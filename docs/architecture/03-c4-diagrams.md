# SCGE — Diagramas C4

> Modelo C4 (Simon Brown). Níveis 1 (Sistema) e 2 (Containers) renderizados em Mermaid.
> Nível 3 (Componentes) será produzido por BC durante a modelagem tática.

---

## Nível 1 — Diagrama de Contexto do Sistema

```mermaid
C4Context
    title SCGE — Sistema Contábil Gerencial Empresarial (Contexto)

    Person(contador, "Contador / Analista Contábil", "Lança fatos, consulta balancetes, fecha períodos.")
    Person(auditor, "Auditor Interno/Externo", "Consulta trilha de auditoria.")
    Person(gestor, "Gestor", "Consulta demonstrativos.")
    Person(admin, "Administrador", "Gerencia empresas, usuários, planos de contas.")

    System(scge, "SCGE", "Plataforma contábil multiempresa, multiexercício, com suporte a contabilidade pública (MCASP) e privada (CPC/SPED).")

    System_Ext(keycloak, "Keycloak", "Identidade, OIDC, RBAC.")
    System_Ext(siconfi, "SICONFI / SIAFI", "Receita / Tesouro Nacional (público).")
    System_Ext(sped, "SPED (ECD/EFD)", "Receita Federal (privado).")
    System_Ext(erp, "ERPs e Sistemas Financeiros", "Origem de fatos financeiros.")
    System_Ext(banco, "Bancos / Open Finance", "Conciliação bancária.")

    Rel(contador, scge, "Usa", "HTTPS/REST")
    Rel(auditor, scge, "Consulta trilha", "HTTPS/REST")
    Rel(gestor, scge, "Consulta demonstrativos", "HTTPS/REST")
    Rel(admin, scge, "Administra", "HTTPS/REST")

    Rel(scge, keycloak, "Autentica / Valida JWT", "OIDC")
    Rel(scge, siconfi, "Envia matrizes de saldos", "REST/SOAP")
    Rel(scge, sped, "Gera arquivos (ECD/EFD)", "Arquivo .txt")
    Rel(erp, scge, "Solicita lançamentos", "AMQP / REST")
    Rel(banco, scge, "Extratos para conciliação", "Open Finance / OFX")
```

---

## Nível 2 — Diagrama de Containers

```mermaid
C4Container
    title SCGE — Containers (Visão Plataforma)

    Person(usuario, "Usuário", "Contador, Auditor, Gestor.")

    System_Boundary(scge, "SCGE") {
        Container(gateway, "API Gateway", "Spring Cloud Gateway", "Roteamento, rate limit, autenticação JWT.")
        Container(webui, "Web UI", "SPA (futuro)", "Interface do usuário.")

        Container(svc_contabilidade, "Módulo Contabilidade", "Spring Boot 3.3 / Java 21", "Core domain: lançamentos, plano de contas, balancetes.")
        Container(svc_fechamento, "Módulo Fechamento", "Spring Boot 3.3", "Apuração e bloqueio de períodos.")
        Container(svc_patrimonio, "Módulo Patrimônio", "Spring Boot 3.3", "Bens, depreciação.")
        Container(svc_financeiro, "Módulo Financeiro", "Spring Boot 3.3", "AP/AR, tesouraria.")
        Container(svc_fiscal, "Módulo Fiscal", "Spring Boot 3.3", "SPED, SICONFI.")
        Container(svc_auditoria, "Módulo Auditoria", "Spring Boot 3.3", "Event log imutável, queries forenses.")

        ContainerDb(db_contabilidade, "DB Contabilidade", "PostgreSQL 16 (schema scge_contabilidade)", "Plano, lançamentos, saldos, outbox.")
        ContainerDb(db_fechamento, "DB Fechamento", "PostgreSQL 16 (schema scge_fechamento)")
        ContainerDb(db_patrimonio, "DB Patrimônio", "PostgreSQL 16 (schema scge_patrimonio)")
        ContainerDb(db_financeiro, "DB Financeiro", "PostgreSQL 16 (schema scge_financeiro)")
        ContainerDb(db_fiscal, "DB Fiscal", "PostgreSQL 16 (schema scge_fiscal)")
        ContainerDb(db_auditoria, "DB Auditoria", "PostgreSQL 16 (schema scge_auditoria)", "Append-only, particionado.")

        ContainerQueue(rabbit, "RabbitMQ", "AMQP 0.9.1", "Eventos de domínio entre BCs.")
        Container(otel, "OpenTelemetry Collector", "OTLP", "Métricas, traces, logs.")
    }

    System_Ext(keycloak, "Keycloak")
    System_Ext(siconfi, "SICONFI/SIAFI")
    System_Ext(sped, "SPED")

    Rel(usuario, gateway, "HTTPS", "JWT")
    Rel(gateway, svc_contabilidade, "REST")
    Rel(gateway, svc_fechamento, "REST")
    Rel(gateway, svc_patrimonio, "REST")
    Rel(gateway, svc_financeiro, "REST")
    Rel(gateway, svc_fiscal, "REST")
    Rel(gateway, svc_auditoria, "REST")

    Rel(svc_contabilidade, db_contabilidade, "JDBC")
    Rel(svc_fechamento, db_fechamento, "JDBC")
    Rel(svc_patrimonio, db_patrimonio, "JDBC")
    Rel(svc_financeiro, db_financeiro, "JDBC")
    Rel(svc_fiscal, db_fiscal, "JDBC")
    Rel(svc_auditoria, db_auditoria, "JDBC")

    Rel(svc_contabilidade, rabbit, "Publica/Consome", "AMQP")
    Rel(svc_fechamento, rabbit, "Publica/Consome", "AMQP")
    Rel(svc_patrimonio, rabbit, "Publica", "AMQP")
    Rel(svc_financeiro, rabbit, "Publica", "AMQP")
    Rel(svc_fiscal, rabbit, "Consome", "AMQP")
    Rel(svc_auditoria, rabbit, "Consome (todos)", "AMQP")

    Rel(gateway, keycloak, "Valida JWT", "OIDC")
    Rel(svc_fiscal, siconfi, "Envia", "REST")
    Rel(svc_fiscal, sped, "Gera arquivo", "FS")

    Rel(svc_contabilidade, otel, "Telemetria", "OTLP")
```

---

## Nível 2.1 — Zoom no Módulo Contabilidade (Hexagonal)

```mermaid
flowchart LR
    subgraph adapters_in["Adapters de Entrada (Driving)"]
        REST[REST Controller<br/>POST /v1/lancamentos]
        CONSUMER[AMQP Consumer<br/>fila lancamento.solicitado]
        SCHED[Scheduler<br/>Recalcular saldos]
    end

    subgraph application["Application (Use Cases)"]
        UC1[RegistrarLancamentoContabilUseCase]
        UC2[EstornarLancamentoUseCase]
        UC3[ConsultarBalanceteUseCase]
        UC4[PublicarPlanoDeContasUseCase]
    end

    subgraph domain["Domain (puro, sem Spring)"]
        AGG1["Aggregate: LancamentoContabil<br/>(Partidas, invariantes)"]
        AGG2["Aggregate: PlanoDeContas<br/>(Conta sintética/analítica)"]
        AGG3["Aggregate: Saldo<br/>(Movimento por conta/competência)"]
        VO["Value Objects:<br/>Valor, CodigoConta,<br/>Competencia, Natureza"]
        EVT["Domain Events:<br/>LancamentoRegistrado,<br/>LancamentoEstornado,<br/>SaldoAtualizado"]
        PORTS_OUT[/Ports de Saída<br/>LancamentoRepository<br/>PlanoDeContasRepository<br/>SaldoRepository<br/>EventPublisher<br/>PeriodoStatusPort/]
    end

    subgraph adapters_out["Adapters de Saída (Driven)"]
        JPA[JPA Adapter<br/>Hibernate + PostgreSQL]
        OUTBOX[Outbox Adapter<br/>tabela outbox_event]
        RABBIT[RabbitMQ Adapter<br/>via Outbox Relay]
        FECH[HTTP Client<br/>BC Fechamento]
    end

    REST --> UC1
    REST --> UC2
    REST --> UC3
    CONSUMER --> UC1
    SCHED --> UC3

    UC1 --> AGG1
    UC1 --> AGG2
    UC1 --> AGG3
    UC2 --> AGG1
    UC4 --> AGG2

    AGG1 -. emite .-> EVT
    AGG1 --> VO
    AGG2 --> VO
    AGG3 --> VO

    UC1 --> PORTS_OUT
    UC2 --> PORTS_OUT
    UC3 --> PORTS_OUT
    UC4 --> PORTS_OUT

    PORTS_OUT -.implementa.-> JPA
    PORTS_OUT -.implementa.-> OUTBOX
    PORTS_OUT -.implementa.-> FECH
    OUTBOX --> RABBIT

    classDef dom fill:#1f6feb,stroke:#0b3d91,color:#fff
    classDef app fill:#2da44e,stroke:#116329,color:#fff
    classDef adp fill:#9a6700,stroke:#6e4c00,color:#fff
    class AGG1,AGG2,AGG3,VO,EVT,PORTS_OUT dom
    class UC1,UC2,UC3,UC4 app
    class REST,CONSUMER,SCHED,JPA,OUTBOX,RABBIT,FECH adp
```

### Regra de dependência
- `interfaces` → `application` → `domain`
- `infrastructure` → `application`/`domain` (implementa **ports**)
- **`domain` não depende de ninguém**, exceto Java SE.
