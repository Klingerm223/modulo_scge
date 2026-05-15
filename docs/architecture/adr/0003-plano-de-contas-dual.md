# ADR 0003 — Plano de Contas Dual: MCASP e CPC com Strategy

- **Status:** Aceito
- **Data:** 2026-05-15
- **Escopo:** BC Contabilidade.

## Contexto

O SCGE deve atender, na mesma plataforma, empresas privadas (regime CPC,
SPED ECD/ECF) e órgãos públicos (regime MCASP, plano PCASP obrigatório,
SICONFI/SIAFI). As diferenças não são cosméticas:

| Aspecto | Privado (CPC) | Público (MCASP) |
|---------|---------------|------------------|
| Plano de contas | Livre, definido pela empresa | PCASP obrigatório, hierarquia federal |
| Regime | Competência | Misto (orçamentária por caixa, patrimonial por competência) |
| Apuração | Lucro/Prejuízo | Superávit/Déficit, execução orçamentária |
| Demonstrações | DRE, BP, DFC, DMPL, DVA | BP, DVP, DFC, Balanço Orçamentário, DCASP |
| Reporte | SPED ECD/ECF | SICONFI matriz de saldos, SIAFI |

## Decisão

**Um único modelo de domínio**, com variações encapsuladas em **Strategies**
selecionadas pelo VO `PerfilContabil` da `Empresa`:

```java
public enum PerfilContabil { PRIVADO_CPC, PUBLICO_MCASP }
```

### Pontos de variação como *ports* no domínio

| Port | Implementação Privado | Implementação Público |
|------|----------------------|----------------------|
| `ValidadorPlanoDeContas` | Permite qualquer hierarquia coerente | Valida aderência ao PCASP |
| `ClassificadorConta` | Por natureza patrimonial CPC | Por classe PCASP (1–8) |
| `ApuradorResultado` | Receitas/Despesas → Resultado do Exercício | Receitas/Despesas correntes/capital → Superávit/Déficit |
| `GeradorMatrizSaldos` | Saída para SPED ECD | Saída para SICONFI |
| `RegrasFechamento` | Anual + intermediário | Bimestre, quadrimestre, anual |

### Modelagem
- O agregado `PlanoDeContas` é **único**, com `perfilContabil` no header.
- `Conta` carrega `classificacao` e atributos opcionais por perfil
  (`naturezaInformacao`, `naturezaOrcamentaria` — só preenchidos no público).
- Empresa não pode trocar perfil após existir lançamento (regra invariante).

### Identificação no Runtime
```java
@Component
class PerfilContabilFactory {
  ValidadorPlanoDeContas validador(PerfilContabil perfil) {
    return switch (perfil) {
      case PRIVADO_CPC   -> new ValidadorPlanoCPC();
      case PUBLICO_MCASP -> new ValidadorPlanoPCASP();
    };
  }
}
```

Injetado via *Specification* / Strategy nos Use Cases.

## Alternativas Consideradas

| Alternativa | Por que não |
|-------------|-------------|
| Dois sistemas separados | Duplicação massiva (lançamento, partida, saldo, balancete são idênticos). |
| Herança de agregado (`PlanoCPC extends PlanoDeContas`) | Quebra encapsulamento e LSP; muda o agregado raiz. |
| Tabelas separadas por perfil | Inviabiliza queries unificadas multitenancy. |
| Feature flag por método | Espalha *ifs* pelo domínio (anti-Strategy). |

## Consequências

### Positivas
- Um único modelo, dois comportamentos.
- Adicionar novo perfil futuro (ex.: terceiro setor, ITG 2002) = nova Strategy.
- Núcleo (lançamento, partida, balanceamento) é 100% comum.

### Negativas
- Risco de Strategies "fugirem" e implementarem regras divergentes.
  **Mitigação:** testes de contrato por Strategy + comparação de demonstrativos
  em dataset de referência.
- Plano de contas tem campos opcionais por perfil (atributos públicos ausentes
  no privado e vice-versa).
  **Mitigação:** Value Objects opcionais (`Optional<NaturezaOrcamentaria>`),
  não nulos sem semântica.
