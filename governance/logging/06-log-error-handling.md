# Registro de Erros e Exceções

## Princípio Fundamental

Todo erro e exceção deve ser logado de forma que permita diagnóstico completo sem necessidade de reproduzir o cenário. O log de erro deve conter contexto suficiente para identificar **o que** falhou, **onde** falhou e **por que** falhou.

---

## Regras de Logging de Erros

### 1. Sempre Incluir a Exceção como Primeiro Parâmetro

O objeto `Exception` deve ser passado como primeiro parâmetro do `LogError` ou `LogCritical`, garantindo que o stack trace completo seja registrado:

```csharp
// CORRETO
logger.LogError(ex,
    "[OpenMeteoApiClient][GetForecastAsync] Falha ao executar requisição HTTP. Url={Url}, StatusCode={StatusCode}",
    requestUrl, response.StatusCode);

// INCORRETO - perde o stack trace
logger.LogError(
    "[OpenMeteoApiClient][GetForecastAsync] Falha ao executar requisição HTTP. Error={Error}",
    ex.Message);
```

### 2. Manter o Prefixo Padrão

Logs de erro seguem o mesmo formato de prefixo `[Classe][Método]`:

```csharp
logger.LogError(ex,
    "[PokemonGetUseCase][ExecuteAsync] Falha ao consultar PokeAPI. PokemonId={PokemonId}",
    id);
```

### 3. Incluir Contexto de Diagnóstico

O log de erro deve incluir os parâmetros e dados que permitem reproduzir o cenário:

```csharp
// CORRETO - inclui contexto para diagnóstico
logger.LogError(ex,
    "[WeatherConditionsGetUseCase][ExecuteAsync] Falha ao obter condições climáticas. Latitude={Latitude}, Longitude={Longitude}",
    input.Latitude, input.Longitude);

// INCORRETO - sem contexto, dificulta diagnóstico
logger.LogError(ex, "[WeatherConditionsGetUseCase][ExecuteAsync] Falha ao obter condições climáticas");
```

### 4. Logar no Ponto de Captura

O erro deve ser logado **onde a exceção é capturada** (`catch`), não onde é lançada (`throw`):

```csharp
try
{
    var result = await httpClient.GetAsync(url, cancellationToken);
    result.EnsureSuccessStatusCode();
}
catch (HttpRequestException ex)
{
    logger.LogError(ex,
        "[OpenMeteoApiClient][GetForecastAsync] Falha na requisição HTTP. Url={Url}",
        url);

    throw; // re-throw preserva o stack trace original
}
```

### 5. Não Logar e Engolir Exceções

Toda exceção logada deve ser re-lançada (`throw`) ou tratada de forma que o chamador saiba que houve falha. Nunca logar uma exceção e silenciosamente continuar:

```csharp
// INCORRETO - engole a exceção
catch (Exception ex)
{
    logger.LogError(ex, "[UseCase][Execute] Erro inesperado");
    return null; // chamador não sabe que houve falha
}

// CORRETO - loga e re-lança
catch (Exception ex)
{
    logger.LogError(ex, "[UseCase][Execute] Erro inesperado");
    throw;
}
```

---

## Tratamento Centralizado de Exceções

### Global Exception Handler

A aplicação deve ter um handler global de exceções que captura erros não tratados e gera um log `Error` ou `Critical`:

```csharp
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var exception = context.Features.Get<IExceptionHandlerFeature>()?.Error;
        var logger = context.RequestServices.GetRequiredService<ILogger<Program>>();

        logger.LogError(exception,
            "[ExceptionHandler][Handle] Erro não tratado na requisição. Path={Path}, Method={Method}",
            context.Request.Path, context.Request.Method);

        context.Response.StatusCode = StatusCodes.Status500InternalServerError;
        await context.Response.WriteAsJsonAsync(new { error = "Erro interno do servidor" });
    });
});
```

---

## Níveis de Log para Diferentes Tipos de Erro

| Tipo de Erro | Nível | Exemplo |
|-------------|-------|---------|
| Erro de validação de entrada | `Warning` | Parâmetro fora do range, formato inválido |
| Falha em chamada HTTP externa | `Error` | Timeout, 5xx da API externa, connection refused |
| Exceção de negócio esperada | `Warning` | Recurso não encontrado, operação não permitida |
| Exceção inesperada | `Error` | NullReferenceException, InvalidOperationException |
| Falha de infraestrutura | `Critical` | Banco de dados indisponível, falha de startup |

---

## Isolamento Visual

Assim como todos os outros logs, os logs de erro devem respeitar o isolamento visual com linha em branco acima e abaixo:

```csharp
var response = await httpClient.GetAsync(url, cancellationToken);

logger.LogError(ex,
    "[ApiClient][GetAsync] Falha na requisição HTTP. Url={Url}, StatusCode={StatusCode}",
    url, response.StatusCode);

throw new ApiClientException("Falha na chamada externa", ex);
```
