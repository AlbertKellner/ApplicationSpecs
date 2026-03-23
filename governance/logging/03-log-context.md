# Contexto, Correlação e Rastreabilidade

## CorrelationId

Toda requisição HTTP deve receber um identificador único de correlação (`CorrelationId`) que acompanha todos os logs gerados durante seu processamento. Isso permite rastrear a sequência completa de operações de uma única requisição.

### Geração

- O `CorrelationId` deve ser gerado como **GUID v7** (baseado em timestamp, garantindo ordenação cronológica)
- A geração ocorre no `CorrelationIdMiddleware`, no início do pipeline HTTP
- O valor é injetado no contexto de log via `LogContext.PushProperty`

### Implementação do Middleware

```csharp
public sealed class CorrelationIdMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext context)
    {
        var correlationId = Guid.CreateVersion7().ToString();
        context.Items["CorrelationId"] = correlationId;

        using (LogContext.PushProperty("CorrelationId", correlationId))
        {
            await next(context);
        }
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
