# Contexto, Correlação e Rastreabilidade

## CorrelationId

Toda requisição HTTP deve receber um identificador único de correlação (`CorrelationId`) que acompanha todos os logs gerados durante seu processamento. Isso permite rastrear a sequência completa de operações de uma única requisição.

### Geração

- O `CorrelationId` deve ser gerado como **GUID v7** (baseado em timestamp, garantindo ordenação cronológica)
- A geração ocorre no `CorrelationIdMiddleware`, no início do pipeline HTTP, via `GuidV7.Create()`
- Se a requisição contiver o header `X-Correlation-Id` com um GUID v7 válido, o ID é reutilizado (rastreabilidade cross-service)
- O valor é injetado no contexto de log via `LogContext.PushProperty`
- O valor é adicionado ao header de resposta `X-Correlation-Id`

### Implementação do Middleware

```csharp
public sealed class CorrelationIdMiddleware(RequestDelegate next, ILogger<CorrelationIdMiddleware> logger)
{
    public const string HeaderName = "X-Correlation-Id";
    internal const string HttpContextItemKey = "CorrelationId";

    public async Task InvokeAsync(HttpContext context)
    {
        logger.LogInformation("[CorrelationIdMiddleware][InvokeAsync] Processar requisição e garantir CorrelationId");

        var correlationId = ResolveCorrelationId(context);

        context.Items[HttpContextItemKey] = correlationId;
        context.Response.Headers[HeaderName] = correlationId.ToString();

        using (LogContext.PushProperty(HttpContextItemKey, correlationId))
        {
            logger.LogInformation(
                "[CorrelationIdMiddleware][InvokeAsync] Prosseguir com CorrelationId enriquecido no contexto. CorrelationId={CorrelationId}",
                correlationId);

            await next(context);

            logger.LogInformation(
                "[CorrelationIdMiddleware][InvokeAsync] Retornar resposta com CorrelationId enriquecido. CorrelationId={CorrelationId}",
                correlationId);
        }
    }

    private static Guid ResolveCorrelationId(HttpContext context)
    {
        if (context.Request.Headers.TryGetValue(HeaderName, out var headerValue)
            && Guid.TryParse(headerValue, out var parsed)
            && GuidV7.IsVersion7(parsed))
        {
            return parsed;
        }

        return GuidV7.Create();
    }
}
```

### Regras

- O `CorrelationId` **não** deve ser gerado manualmente em nenhuma outra camada
- O middleware deve ser registrado **antes** de qualquer outro middleware que gere logs
- O `CorrelationId` deve aparecer em **todos** os logs da requisição automaticamente via template do Serilog

---

## UserName (Contexto de Autenticação)

O nome do usuário autenticado é enriquecido no contexto de log pelo `AuthenticateFilter`:

```csharp
using (LogContext.PushProperty("UserName", userName))
{
    await next(context);
}
```

### Regras

- O `UserName` só é adicionado ao contexto após validação bem-sucedida do token
- Em requisições não autenticadas, o campo `UserName` fica vazio no template de saída
- O `AuthenticateFilter` deve logar explicitamente quando o token é ausente ou inválido

---

## Rastreabilidade por Camada

O padrão storytelling garante rastreabilidade por meio de logs em cada camada da arquitetura. A sequência narrativa de uma requisição típica segue este fluxo:

```
Endpoint → UseCase → ApiClient/CachedApiClient → UseCase → Endpoint
```

### Exemplo de Sequência Completa

```
[14:32:01.123] [CorrelationId-A] [Albert] [WeatherConditionsGetEndpoint][Get] Processar requisição GET /weather-conditions
[14:32:01.124] [CorrelationId-A] [Albert] [WeatherConditionsGetUseCase][ExecuteAsync] Executar caso de uso de condições climáticas
[14:32:01.125] [CorrelationId-A] [Albert] [CachedOpenMeteoApiClient][GetForecastAsync] Verificar cache para condições climáticas
[14:32:01.126] [CorrelationId-A] [Albert] [CachedOpenMeteoApiClient][GetForecastAsync] Cache miss. Consultar API externa
[14:32:01.450] [CorrelationId-A] [Albert] [OpenMeteoApiClient][GetForecastAsync] Executar requisição HTTP ao Open-Meteo
[14:32:01.890] [CorrelationId-A] [Albert] [OpenMeteoApiClient][GetForecastAsync] Retornar resposta da API Open-Meteo
[14:32:01.891] [CorrelationId-A] [Albert] [CachedOpenMeteoApiClient][GetForecastAsync] Armazenar resposta no cache
[14:32:01.892] [CorrelationId-A] [Albert] [WeatherConditionsGetUseCase][ExecuteAsync] Mapear resposta para model da Feature
[14:32:01.893] [CorrelationId-A] [Albert] [WeatherConditionsGetUseCase][ExecuteAsync] Retornar condições climáticas
[14:32:01.894] [CorrelationId-A] [Albert] [WeatherConditionsGetEndpoint][Get] Retornar resposta do endpoint
```

### Filtragem Eficiente

O formato padronizado permite filtros precisos em ferramentas de observabilidade:

- `[WeatherConditionsGetEndpoint]` — todos os logs de um endpoint específico
- `[GetForecastAsync]` — todos os logs de um método específico, em qualquer classe
- `CorrelationId=019580a1-b2c3-7d4e-8f5a-6b7c8d9e0f12` — todos os logs de uma requisição específica

---

## Propagação de Contexto

### Regra Geral

O contexto de log (`CorrelationId`, `UserName`) é propagado automaticamente pelo Serilog via `LogContext`. Nenhuma camada da aplicação precisa passar esses valores explicitamente — eles são injetados uma vez no pipeline e disponibilizados em todos os logs subsequentes.

### Chamadas HTTP Externas

Para chamadas a APIs externas, o `CorrelationId` pode ser propagado via header HTTP para permitir rastreabilidade cross-service:

```csharp
httpClient.DefaultRequestHeaders.Add("X-Correlation-Id", correlationId);
```

Esta propagação é opcional e depende de a API externa suportar o header.
