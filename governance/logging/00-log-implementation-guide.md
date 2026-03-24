# Guia de Implantação — Logging com Serilog e Datadog

## Visão Geral

Este guia apresenta o plano de implantação passo a passo para tornar projetos aderentes à governança de Logging descrita nesta seção. Ele deve ser seguido na ordem apresentada, com commits intermediários a cada etapa relevante.

---

## Etapas de Implantação

### Etapa 1 — Diagnóstico da Solução de Logging Atual

Verifique se o projeto já utiliza uma solução baseada em **Serilog**.

- Inspecione o `Program.cs`, `Startup.cs` e os arquivos `.csproj` em busca de referências a pacotes de logging (ex: `Serilog`, `NLog`, `log4net`, `Microsoft.Extensions.Logging` puro).
- Identifique sinks configurados (Console, File, Seq, Datadog, etc.).
- Verifique se há configurações de logging em `appsettings.json`, `appsettings.*.json` ou variáveis de ambiente.

Se o projeto **já utiliza Serilog** de forma aderente à governança, avance para a **Etapa 4**.

---

### Etapa 2 — Remoção da Solução de Logging Não Aderente

Caso o projeto utilize uma solução de logging que **não** seja aderente às definições de governança:

1. **Inventariar** todas as variáveis de ambiente, configurações e secrets relacionados ao logging existente. Registre essas informações na janela de contexto para reuso na adequação à governança.
2. **Remover** pacotes, configurações e código da solução de logging não aderente.
3. **Realizar commit** com a remoção.

> **Importante:** Não prossiga sem antes capturar todas as variáveis de ambiente e configurações do logging anterior — elas podem conter informações necessárias para a integração com o Datadog.

---

### Etapa 3 — Instalação da Infraestrutura Serilog + Datadog

Instale os pacotes NuGet necessários e configure a infraestrutura de logging com Serilog:

#### Pacotes Requeridos

```xml
<PackageReference Include="Serilog" Version="..." />
<PackageReference Include="Serilog.AspNetCore" Version="..." />
<PackageReference Include="Serilog.Sinks.Console" Version="..." />
<PackageReference Include="Serilog.Enrichers.Environment" Version="..." />
<PackageReference Include="Serilog.Settings.Configuration" Version="..." />
```

#### Configuração do Serilog no Program.cs

Configure o Serilog seguindo o padrão normativo da governança:

- Template de console com tema ANSI colorido e campos obrigatórios (`Timestamp`, `CorrelationId`, `UserName`, `Message`, `Exception`) conforme @governance/logging/02-log-structure.md
- Sink assíncrono para console conforme @governance/logging/05-log-performance.md
- `DatadogHttpSink` condicional conforme @governance/logging/09-log-datadog-integration.md
- Override de namespaces de bibliotecas externas (`Microsoft`, `System`) para nível `Warning`
- Enriquecimento via `LogContext` habilitado (`Enrich.FromLogContext()`)

#### Implementação do DatadogHttpSink

Implemente o `DatadogHttpSink` com processamento assíncrono em batch via `System.Threading.Channels`, conforme especificado em @governance/logging/09-log-datadog-integration.md.

**Realizar commit** com a infraestrutura instalada.

---

### Etapa 4 — Configuração de Secrets e Parâmetros do Datadog

Atualize os arquivos de configuração JSON e variáveis de ambiente com os dados necessários para a integração com o Datadog:

#### Arquivos de Configuração

- `appsettings.json` — configuração base com `Datadog:DirectLogs` e `Serilog:MinimumLevel`
- `appsettings.*.json` — overrides por ambiente (Development, Staging, Production)
- `docker-compose.yml` — configuração do Datadog Agent e labels de Autodiscovery

#### Informações Necessárias

| Informação | Origem | Exemplo |
|------------|--------|---------|
| `DD_API_KEY` | Secret do ambiente / `.env` | `abc123...` |
| `DD_SITE` | Região do Datadog | `datadoghq.com` |
| `DD_ENV` | Contexto de execução | `local`, `ci`, `build` |
| `DD_HOSTNAME` | Identificação do host | `meu-servico-local` |
| Nome do serviço | Padrão do projeto | `meu-servico-api` |

> **Importante:** Caso essas informações não estejam disponíveis no próprio projeto ou no `docker-compose.yml`, **solicite os dados ao usuário e não prossiga** até que todas as informações necessárias tenham sido fornecidas.

**Realizar commit** com as configurações atualizadas.

---

### Etapa 5 — Verificação da Integração com o Datadog

Verifique se a integração foi estabelecida corretamente:

1. **Build** da aplicação para garantir que não há erros de compilação.
2. **Run** da aplicação (ou dos containers via `docker-compose up`).
3. Verifique nos logs de console que o Serilog inicializou corretamente.
4. Se o `DatadogHttpSink` estiver ativado, confirme o log `[Program][Main] Datadog HTTP Sink ativado`.
5. Se usando Docker Agent, verifique que o container `dd-agent` está coletando logs (`docker logs dd-agent`).

Corrija quaisquer problemas de conectividade antes de prosseguir.

---

### Etapa 6 — Implementação Completa do Logging Estruturado

Implemente a solução completa de logging seguindo todos os arquivos de governança:

#### 6.1 — Middlewares e Filtros

- `CorrelationIdMiddleware` com geração de GUID v7 e enriquecimento via `LogContext` (conforme @governance/logging/03-log-context.md e @governance/logging/07-log-context-enrichment.md)
- `AuthenticateFilter` com enriquecimento de `UserId` e `UserName` (conforme @governance/logging/07-log-context-enrichment.md)
- Registro do middleware na pipeline **antes** de `UseExceptionHandler()`

#### 6.2 — Logging por Camada

Implemente logs em todas as camadas seguindo o padrão de Storytelling:

| Camada | Referência |
|--------|-----------|
| Endpoints | @governance/logging/02-log-structure.md — seção "Exemplos Completos" |
| UseCases | @governance/logging/02-log-structure.md — seção "UseCase com iterações" |
| ApiClients | @governance/logging/02-log-structure.md — seção "Responsabilidade de Logging por Camada" |
| CachedApiClients | @governance/logging/08-log-request-example.md — seção "Cache Hit vs. Cache Miss" |
| Program.cs | @governance/logging/02-log-structure.md — seção "Logs em Program.cs" |

#### 6.3 — Regras Obrigatórias

- Prefixo `[Classe][Metodo]` em toda mensagem de log (conforme @governance/logging/02-log-structure.md)
- Linguagem imperativa na descrição (conforme @governance/logging/02-log-structure.md)
- Log de entrada e saída de todo metodo (conforme @governance/logging/02-log-structure.md)
- Logs antes e depois de loops, nunca dentro (conforme @governance/logging/05-log-performance.md)
- Isolamento visual no codigo-fonte — linha em branco acima e abaixo de cada `logger.Log*()` (conforme @governance/logging/02-log-structure.md)
- Structured logging com templates, nunca interpolacao (conforme @governance/logging/05-log-performance.md)
- Dados sensiveis nunca logados diretamente (conforme @governance/logging/04-log-sensitive-data.md)
- `LogError` sempre com objeto `Exception` como primeiro parametro (conforme @governance/logging/06-log-error-handling.md)
- Niveis de log corretos por tipo de evento (conforme @governance/logging/01-log-levels.md)

#### 6.4 — Testes de Logging

Implemente testes unitarios com `FakeLogger<T>` conforme @governance/logging/10-log-testing.md:

- Verificar prefixo da classe em todos os logs
- Verificar log de entrada e saida dos metodos
- Verificar nivel correto (`Information`, `Warning`, `Error`)
- Usar `Assert.Contains` com termo-chave, nunca comparacao exata de mensagem

**Realizar commit** com a implementacao completa do logging.

---

### Etapa 7 — Validacao Final (Build, Testes e Run)

Execute as tres etapas de validacao em sequencia:

1. **Build** — compile o projeto e a solution alterada. Todos os projetos devem compilar sem erros.
2. **Testes** — execute todos os testes unitarios (incluindo os novos testes de logging). Todos devem passar.
3. **Run** — execute a aplicacao e verifique que os logs sao emitidos corretamente no console e/ou no Datadog.

---

### Etapa 8 — Correcao de Falhas

Caso qualquer uma das etapas de validacao (build, testes ou run) falhe:

1. Identifique a causa raiz do erro.
2. Corrija o problema.
3. Repita o processo de validacao (Etapa 7) ate que as tres etapas sejam concluidas com sucesso.

---

### Etapa 9 — Pull Request

Apos todas as validacoes bem-sucedidas, crie um Pull Request com as alteracoes realizadas:

- Titulo conciso descrevendo a implantacao de logging
- Descricao com resumo das alteracoes por etapa
- Checklist de validacao (build, testes, run) confirmada
