# Níveis de Log — Critérios de Uso

## Padrão Adotado: Logging Estruturado com Storytelling

O projeto adota o padrão de **Logging Estruturado com Storytelling**, que permite reconstruir a narrativa completa de qualquer requisição a partir dos logs, sem necessidade de debugger. Todo log segue um formato padronizado que identifica precisamente a origem (classe e método) e descreve a ação em linguagem imperativa.

---

## Níveis de Log

| Nível | Uso | Exemplo |
|-------|-----|---------|
| **Trace** | Detalhes internos de algoritmos, dados intermediários de cálculos. Nunca habilitado em produção. | `logger.LogTrace("[MathEngine][Calculate] Valor intermediário. Step={Step}, Value={Value}", step, value);` |
| **Debug** | Informações úteis para diagnóstico durante desenvolvimento — valores de variáveis, caminhos de decisão tomados. | `logger.LogDebug("[PokemonGetUseCase][ExecuteAsync] Tipo encontrado. Slot={Slot}, Name={Name}", slot, name);` |
| **Information** | Fluxo normal de execução — entrada e saída de métodos, chamadas a serviços externos, mapeamentos, início e fim de iterações. Este é o nível padrão do storytelling. | `logger.LogInformation("[PokemonGetEndpoint][Get] Processar requisição GET /pokemon/{PokemonId}", id);` |
| **Warning** | Situação inesperada mas recuperável — cache miss, fallback acionado, token ausente, timeout com retry bem-sucedido. | `logger.LogWarning("[AuthenticateFilter][OnActionExecutionAsync] Retornar 401 - token ausente na requisição");` |
| **Error** | Falha que impede a conclusão da operação corrente, mas não compromete a aplicação — exceção em chamada HTTP, falha de parsing, violação de regra de negócio. | `logger.LogError(ex, "[OpenMeteoApiClient][GetForecastAsync] Falha ao executar requisição HTTP. StatusCode={StatusCode}", response.StatusCode);` |
| **Critical** | Falha que compromete a estabilidade da aplicação — falha de startup, perda de conexão com banco de dados, esgotamento de recursos. | `logger.LogCritical(ex, "[Program][Main] Falha crítica ao iniciar a aplicação");` |

---

## Regras de Uso

### 1. Nível Padrão: Information

O padrão storytelling opera predominantemente no nível `Information`. Todo log de entrada de método, saída de método, chamada externa, mapeamento e iteração deve usar `LogInformation`.

### 2. Warning para Situações Recuperáveis

Use `Warning` quando a aplicação contorna a situação sem falha, mas o comportamento merece atenção:

```csharp
logger.LogWarning(
    "[CachedOpenMeteoApiClient][GetForecastAsync] Cache miss. Consultar API externa. CacheKey={CacheKey}",
    cacheKey);
```

### 3. Error com Exceção Obrigatória

Todo `LogError` deve incluir o objeto `Exception` como primeiro parâmetro:

```csharp
logger.LogError(ex,
    "[OpenMeteoApiClient][GetForecastAsync] Falha ao executar requisição HTTP. Url={Url}",
    requestUrl);
```

### 4. Nunca Usar Trace/Debug em Produção

Os níveis `Trace` e `Debug` devem ser configurados apenas para ambientes de desenvolvimento. A configuração de nível mínimo por ambiente deve ser:

| Ambiente | Nível Mínimo |
|----------|-------------|
| Development | `Debug` |
| Staging | `Information` |
| Production | `Information` |

### 5. Critical Reservado para Falhas de Infraestrutura

O nível `Critical` é reservado para situações que comprometem o funcionamento da aplicação como um todo. Não usar para erros de negócio ou falhas de requisições individuais.
