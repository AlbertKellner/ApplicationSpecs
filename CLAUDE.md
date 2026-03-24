# Governance — Application Specs

Este repositório contém políticas de governança genéricas e reutilizáveis para o Claude Code.
Ele deve ser importado por repositórios de aplicação como referência de padronização.

> **Natureza:** Somente leitura. Este repositório não documenta decisões arquiteturais
> de repositórios específicos nem registra definições particulares de implementação.

---

## Políticas Disponíveis

### Logging

Diretrizes para padronização de logs em qualquer aplicação.

- @governance/logging/00-log-implementation-guide.md — Guia de implantação (Serilog + Datadog)
- @governance/logging/01-log-levels.md — Níveis de log e critérios de uso
- @governance/logging/02-log-structure.md — Formato e campos obrigatórios
- @governance/logging/03-log-context.md — Contexto, correlação e rastreabilidade
- @governance/logging/04-log-sensitive-data.md — Dados sensíveis e conformidade
- @governance/logging/05-log-performance.md — Performance e controle de volume
- @governance/logging/06-log-error-handling.md — Registro de erros e exceções
- @governance/logging/07-log-context-enrichment.md — Enriquecimento de contexto nos logs
- @governance/logging/08-log-request-example.md — Exemplo de requisição completa com fluxo de logs
- @governance/logging/09-log-datadog-integration.md — Integração com Datadog (Agent Docker e HTTP Sink)
- @governance/logging/10-log-testing.md — Testes de logging com FakeLogger

---

## Como Importar

Para aplicar esta governança em outro repositório, adicione no `CLAUDE.md` do repositório alvo:

```
@path/to/ApplicationSpecs/CLAUDE.md
```

Ou importe políticas específicas:

```
@path/to/ApplicationSpecs/governance/logging/01-log-levels.md
```
