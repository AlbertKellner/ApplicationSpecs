# Formato e Campos Obrigatórios

## Formato de Prefixo Obrigatório

Todo log da aplicação deve seguir o formato:

```
[NomeDaClasse][NomeDoMétodo] DescriçãoBreveFrase
```

### Exemplos

```csharp
logger.LogInformation("[WeatherConditionsGetEndpoint][Get] Processar requisição GET /weather-conditions");

logger.LogInformation("[OpenMeteoApiClient][GetForecastAsync] Executar requisição HTTP ao Open-Meteo. Latitude={Latitude}, Longitude={Longitude}", input.Latitude, input.Longitude);

logger.LogWarning("[AuthenticateFilter][OnActionExecutionAsync] Retornar 401 - token ausente na requisição");
```

---

## Regras de Escrita de Logs

### 1. Linguagem Imperativa

A descrição após o prefixo deve ser breve, objetiva e no imperativo:

| Correto | Incorreto |
|---------|-----------|
| `Processar requisição` | `A requisição está sendo processada` |
| `Retornar resposta` | `Respondendo com o resultado` |
| `Gerar token JWT` | `Token JWT foi gerado` |

### 2. Log de Entrada do Método

Todo método deve ter um log informando **o que será executado** e registrando os **objetos/parâmetros recebidos**:

```csharp
public async Task<PokemonGetOutput> ExecuteAsync(int id, CancellationToken cancellationToken = default)
{
    logger.LogInformation(
        "[PokemonGetUseCase][ExecuteAsync] Executar caso de uso de consulta de Pokemon. PokemonId={PokemonId}", id);

    // ... lógica do método
}
```

### 3. Log de Saída do Método

Antes de cada `return`, deve haver um log informando **o que está sendo retornado**:

```csharp
    logger.LogInformation(
        "[PokemonGetUseCase][ExecuteAsync] Retornar dados do Pokemon. PokemonId={PokemonId}, PokemonName={PokemonName}",
        output.Id, output.Name);

    return output;
```

### 4. Log Antes e Depois de Loops

Estruturas de iteração (`for`, `foreach`, `while`) devem ter log antes de iniciar e após concluir:

```csharp
logger.LogInformation(
    "[PokemonGetUseCase][ExecuteAsync] Iterar tipos do Pokemon. Count={Count}", result.Types.Count);

foreach (var typeSlot in result.Types)
{
    types.Add(new PokemonGetTypeItem { Slot = typeSlot.Slot, Name = typeSlot.Type.Name });
}

logger.LogInformation("[PokemonGetUseCase][ExecuteAsync] Iteracao de tipos concluida");
```

### 5. Isolamento Visual no Código

Toda instrução `logger.Log*()` deve ter uma **linha em branco acima** e uma **linha em branco abaixo** no código-fonte, garantindo legibilidade:

```csharp
var result = await openMeteoApiClient.GetForecastAsync(input, cancellationToken);

logger.LogInformation(
    "[WeatherConditionsGetUseCase][ExecuteAsync] Mapear resposta da Open-Meteo para model da Feature");

var output = new WeatherConditionsGetOutput { /* ... */ };
```

### 6. Logs em Program.cs

O `Program.cs` segue regras especiais:
- Log inicial informando que a aplicação está sendo iniciada
- Um log por **bloco lógico** (Serilog, DI de infraestrutura, DI de features, DI de segurança, pipeline de middlewares)
- O log é escrito **antes** do conjunto de instruções do bloco
- **Não** criar log por instrução individual dentro do bloco

---

## Template do Console

O template de saída do console é normativo e usa tema ANSI colorido:

```csharp
.WriteTo.Console(
    theme: AnsiConsoleTheme.Code,
    outputTemplate: "[{Timestamp:dd/MM/yyyy HH:mm:ss.fffffff}] [{CorrelationId}] [{UserName}] {Message:lj}{NewLine}{Exception}")
```

### Campos do Template

| Campo | Formato | Origem |
|-------|---------|--------|
| `Timestamp` | `dd/MM/yyyy HH:mm:ss.fffffff` | Gerado automaticamente pelo Serilog |
| `CorrelationId` | GUID v7 | Enriquecido pelo `CorrelationIdMiddleware` via `LogContext.PushProperty` |
| `UserName` | String | Enriquecido pelo `AuthenticateFilter` via `LogContext.PushProperty` |
| `Message` | Template estruturado | Definido em cada instrução `logger.Log*()` |
| `Exception` | Stack trace completo | Incluído automaticamente quando há exceção |

### Exemplo de Saída no Console

```
[23/03/2026 14:32:01.1234567] [019580a1-b2c3-7d4e-8f5a-6b7c8d9e0f12] [Albert] [WeatherConditionsGetEndpoint][Get] Processar requisição GET /weather-conditions
[23/03/2026 14:32:01.1235000] [019580a1-b2c3-7d4e-8f5a-6b7c8d9e0f12] [Albert] [WeatherConditionsGetUseCase][ExecuteAsync] Executar caso de uso de condições climáticas de São Paulo
[23/03/2026 14:32:01.1236000] [019580a1-b2c3-7d4e-8f5a-6b7c8d9e0f12] [Albert] [CachedOpenMeteoApiClient][GetForecastAsync] Verificar cache para condições climáticas
[23/03/2026 14:32:01.1237000] [019580a1-b2c3-7d4e-8f5a-6b7c8d9e0f12] [Albert] [CachedOpenMeteoApiClient][GetForecastAsync] Cache miss. Consultar API externa. CacheKey=OpenMeteo:WeatherConditionsGet:123
[23/03/2026 14:32:01.4500000] [019580a1-b2c3-7d4e-8f5a-6b7c8d9e0f12] [Albert] [OpenMeteoApiClient][GetForecastAsync] Executar requisição HTTP ao Open-Meteo. Latitude=-23.5475, Longitude=-46.6361
[23/03/2026 14:32:01.8900000] [019580a1-b2c3-7d4e-8f5a-6b7c8d9e0f12] [Albert] [OpenMeteoApiClient][GetForecastAsync] Retornar resposta da API Open-Meteo. Timezone=America/Sao_Paulo
[23/03/2026 14:32:01.8910000] [019580a1-b2c3-7d4e-8f5a-6b7c8d9e0f12] [Albert] [CachedOpenMeteoApiClient][GetForecastAsync] Armazenar resposta no cache. CacheKey=OpenMeteo:WeatherConditionsGet:123, DurationSeconds=10
[23/03/2026 14:32:01.8920000] [019580a1-b2c3-7d4e-8f5a-6b7c8d9e0f12] [Albert] [WeatherConditionsGetUseCase][ExecuteAsync] Mapear resposta da Open-Meteo para model da Feature
[23/03/2026 14:32:01.8930000] [019580a1-b2c3-7d4e-8f5a-6b7c8d9e0f12] [Albert] [WeatherConditionsGetUseCase][ExecuteAsync] Retornar condições climáticas obtidas da Open-Meteo. Timezone=America/Sao_Paulo
[23/03/2026 14:32:01.8940000] [019580a1-b2c3-7d4e-8f5a-6b7c8d9e0f12] [Albert] [WeatherConditionsGetEndpoint][Get] Retornar resposta do endpoint com condições climáticas de São Paulo
```

---

## Responsabilidade de Logging por Camada

Cada camada da arquitetura tem um papel específico nos logs:

| Camada | Responsabilidade de Logging |
|--------|----------------------------|
| **Endpoint (Controller)** | Início e fim da requisição HTTP; parâmetros da rota/query; resultado retornado |
| **UseCase** | Início do caso de uso; decisões de fluxo; mapeamentos; resultado da orquestração |
| **ApiClient** | Parâmetros da chamada HTTP externa; resposta recebida da API; loops de paginação |
| **CachedApiClient** | Cache hit vs. cache miss; chave de cache; duração do cache |
| **Middleware** | Geração de CorrelationId; enriquecimento de contexto |
| **AuthenticateFilter** | Validação de token; 401 por token ausente/inválido; enriquecimento de UserId/UserName |

---

## Exemplos Completos

### Endpoint com parâmetro de rota

```csharp
[ApiController]
[Route("pokemon")]
[Authenticate]
public sealed class PokemonGetEndpoint(
    PokemonGetUseCase useCase,
    ILogger<PokemonGetEndpoint> logger) : ControllerBase
{
    [HttpGet("{id:int}")]
    public async Task<IActionResult> Get([FromRoute] int id, CancellationToken cancellationToken)
    {
        logger.LogInformation("[PokemonGetEndpoint][Get] Processar requisicao GET /pokemon/{PokemonId}", id);

        var result = await useCase.ExecuteAsync(id, cancellationToken);

        logger.LogInformation(
            "[PokemonGetEndpoint][Get] Retornar resposta do endpoint com dados do Pokemon. PokemonId={PokemonId}, PokemonName={PokemonName}",
            id, result.Name);

        return Ok(result);
    }
}
```

### Endpoint com parâmetros de query

```csharp
[ApiController]
[Route("weather-conditions")]
[Authenticate]
public sealed class WeatherConditionsGetEndpoint(
    WeatherConditionsGetUseCase useCase,
    ILogger<WeatherConditionsGetEndpoint> logger) : ControllerBase
{
    [HttpGet]
    public async Task<IActionResult> Get(
        [FromQuery] double latitude,
        [FromQuery] double longitude,
        CancellationToken cancellationToken)
    {
        logger.LogInformation(
            "[WeatherConditionsGetEndpoint][Get] Processar requisição GET /weather-conditions. Latitude={Latitude}, Longitude={Longitude}",
            latitude, longitude);

        var input = new WeatherConditionsGetInput { Latitude = latitude, Longitude = longitude };
        var result = await useCase.ExecuteAsync(input, cancellationToken);

        logger.LogInformation(
            "[WeatherConditionsGetEndpoint][Get] Retornar resposta do endpoint com condições climáticas. Latitude={Latitude}, Longitude={Longitude}",
            latitude, longitude);

        return Ok(result);
    }
}
```

### UseCase com iterações

```csharp
public sealed class PokemonGetUseCase(
    IPokemonApiClient pokemonApiClient,
    ILogger<PokemonGetUseCase> logger)
{
    public async Task<PokemonGetOutput> ExecuteAsync(int id, CancellationToken cancellationToken = default)
    {
        logger.LogInformation(
            "[PokemonGetUseCase][ExecuteAsync] Executar caso de uso de consulta de Pokemon. PokemonId={PokemonId}", id);

        logger.LogInformation(
            "[PokemonGetUseCase][ExecuteAsync] Consultar PokeAPI. PokemonId={PokemonId}", id);

        var result = await pokemonApiClient.GetByIdAsync(id, cancellationToken);

        logger.LogInformation(
            "[PokemonGetUseCase][ExecuteAsync] Mapear resposta da PokeAPI para model da Feature");

        var types = new List<PokemonGetTypeItem>();

        logger.LogInformation(
            "[PokemonGetUseCase][ExecuteAsync] Iterar tipos do Pokemon. Count={Count}", result.Types.Count);

        foreach (var typeSlot in result.Types)
        {
            types.Add(new PokemonGetTypeItem { Slot = typeSlot.Slot, Name = typeSlot.Type.Name });
        }

        logger.LogInformation("[PokemonGetUseCase][ExecuteAsync] Iteracao de tipos concluida");

        var output = new PokemonGetOutput
        {
            Id = result.Id,
            Name = result.Name,
            Types = types
        };

        logger.LogInformation(
            "[PokemonGetUseCase][ExecuteAsync] Retornar dados do Pokemon. PokemonId={PokemonId}, PokemonName={PokemonName}",
            output.Id, output.Name);

        return output;
    }
}
```
