# Performance e Controle de Volume

## Princípio Fundamental

Logs devem fornecer observabilidade completa sem comprometer a performance da aplicação. O padrão storytelling gera um volume previsível de logs por requisição, e esse volume deve ser monitorado e controlado.

---

## Volume Esperado por Requisição

O padrão storytelling gera um número previsível de logs por camada:

| Camada | Logs por Chamada | Composição |
|--------|-----------------|------------|
| **Endpoint** | 2 | Entrada + Saída |
| **UseCase** | 4+ | Entrada + chamadas externas + mapeamentos + saída |
| **ApiClient** | 2 | Entrada (parâmetros) + saída (resposta) |
| **CachedApiClient** | 2-3 | Verificação de cache + (hit ou miss + armazenamento) |
| **Iteração (loop)** | 2 por loop | Início (com Count) + conclusão |

### Estimativa Típica

Uma requisição simples (Endpoint → UseCase → ApiClient) gera entre **6 e 10 logs**. Requisições com múltiplas iterações podem gerar mais, mas o volume é sempre proporcional à complexidade do fluxo.

---

## Regras de Performance

### 1. Não Logar Dentro de Loops

Nunca inserir instruções de log **dentro** do corpo de um loop. Os logs devem ser apenas **antes** e **depois** da iteração:

```csharp
// INCORRETO - gera N logs (um por iteração)
foreach (var item in items)
{
    logger.LogInformation("[UseCase][Execute] Processar item. ItemId={ItemId}", item.Id);
    Process(item);
}

// CORRETO - gera exatamente 2 logs
logger.LogInformation("[UseCase][Execute] Iterar itens. Count={Count}", items.Count);

foreach (var item in items)
{
    Process(item);
}

logger.LogInformation("[UseCase][Execute] Iteracao de itens concluida");
```

### 2. Usar Structured Logging (Não Interpolação)

Sempre usar templates estruturados do Serilog, nunca interpolação de strings:

```csharp
// INCORRETO - aloca string mesmo se o nível estiver desabilitado
logger.LogInformation($"[UseCase][Execute] Processar item. ItemId={item.Id}");

// CORRETO - template estruturado, alocação condicional
logger.LogInformation("[UseCase][Execute] Processar item. ItemId={ItemId}", item.Id);
```

### 3. Evitar Serialização em Logs

Não serializar objetos complexos em mensagens de log. Logar apenas campos primitivos relevantes:

```csharp
// INCORRETO - serializa objeto inteiro
logger.LogInformation("[UseCase][Execute] Resultado obtido. Result={Result}", JsonSerializer.Serialize(result));

// CORRETO - loga campos específicos
logger.LogInformation("[UseCase][Execute] Resultado obtido. Id={Id}, Name={Name}", result.Id, result.Name);
```

### 4. Log Assíncrono (Serilog Async Sink)

Em produção, os logs devem ser escritos de forma assíncrona para não bloquear a thread de processamento:

```csharp
.WriteTo.Async(a => a.Console(
    theme: AnsiConsoleTheme.Code,
    outputTemplate: "[{Timestamp:dd/MM/yyyy HH:mm:ss.fffffff}] [{CorrelationId}] [{UserName}] {Message:lj}{NewLine}{Exception}"))
```

---

## Controle de Volume por Ambiente

| Ambiente | Nível Mínimo | Sinks Habilitados |
|----------|-------------|-------------------|
| **Development** | `Debug` | Console |
| **Staging** | `Information` | Console + Sink estruturado (ex: Seq, Elasticsearch) |
| **Production** | `Information` | Sink estruturado (ex: Seq, Elasticsearch, CloudWatch) |

### Override por Namespace

Namespaces de bibliotecas externas devem ter nível mínimo elevado para reduzir ruído:

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "Microsoft.AspNetCore": "Warning",
        "System": "Warning"
      }
    }
  }
}
```

---

## Monitoramento de Volume

- Alertas devem ser configurados para detectar picos anormais de volume de logs
- O custo de armazenamento de logs deve ser revisado periodicamente
- Logs de nível `Debug` e `Trace` **nunca** devem ser habilitados em produção sem justificativa documentada e prazo definido para desabilitação
