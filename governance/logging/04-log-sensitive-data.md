# Dados Sensíveis e Conformidade

## Princípio Fundamental

Logs **nunca** devem conter dados sensíveis que possam comprometer a privacidade de usuários ou a segurança da aplicação. Todo dado logado deve ser avaliado quanto ao risco de exposição.

---

## Dados Proibidos em Logs

Os seguintes tipos de dados **nunca** devem aparecer em mensagens de log:

| Categoria | Exemplos | Motivo |
|-----------|----------|--------|
| **Credenciais** | Senhas, tokens de acesso, API keys, secrets | Risco de comprometimento de segurança |
| **Dados pessoais identificáveis (PII)** | CPF, RG, número de passaporte, endereço completo | Conformidade com LGPD/GDPR |
| **Dados financeiros** | Número de cartão de crédito, conta bancária, CVV | Conformidade com PCI-DSS |
| **Dados de saúde** | Diagnósticos, prontuários, CID | Conformidade com regulações de saúde |
| **Conteúdo de autenticação** | JWT payload completo, refresh tokens, session IDs | Risco de session hijacking |

---

## Dados Permitidos em Logs

| Categoria | Exemplos | Regra |
|-----------|----------|-------|
| **Identificadores de entidade** | `PokemonId=25`, `UserId=42` | IDs numéricos ou UUIDs são seguros |
| **Nomes públicos** | `PokemonName=Pikachu`, `UserName=Albert` | Apenas nomes de exibição, nunca nomes completos reais |
| **Parâmetros de rota/query** | `Latitude=-23.5475`, `Longitude=-46.6361` | Dados públicos da requisição |
| **Metadados de operação** | `CacheKey=...`, `StatusCode=200`, `Count=5` | Informações técnicas de fluxo |
| **Timestamps e durações** | `DurationSeconds=10`, `Timezone=America/Sao_Paulo` | Dados técnicos não sensíveis |

---

## Regras de Conformidade

### 1. Mascaramento de Dados

Quando for necessário referenciar um dado sensível para diagnóstico, usar mascaramento:

```csharp
// INCORRETO - expõe o token completo
logger.LogInformation("[AuthFilter][Validate] Token recebido. Token={Token}", token);

// CORRETO - mascara o token
logger.LogInformation("[AuthFilter][Validate] Token recebido. TokenPrefix={TokenPrefix}", token[..8] + "***");
```

### 2. Nunca Logar Body de Requisição/Resposta Completo

O body de requisições e respostas HTTP pode conter dados sensíveis. Logar apenas campos específicos e seguros:

```csharp
// INCORRETO - loga o body inteiro
logger.LogInformation("[ApiClient][Post] Enviar requisição. Body={Body}", JsonSerializer.Serialize(request));

// CORRETO - loga apenas campos seguros
logger.LogInformation("[ApiClient][Post] Enviar requisição. UserId={UserId}, Action={Action}", request.UserId, request.Action);
```

### 3. Exceções e Stack Traces

Stack traces são permitidos em logs de `Error` e `Critical`, pois geralmente não contêm dados sensíveis. Porém, se a exceção contiver dados sensíveis na mensagem, o log deve ser sanitizado.

### 4. Variáveis de Ambiente e Configuração

Nunca logar valores de variáveis de ambiente ou configurações que contenham secrets:

```csharp
// INCORRETO
logger.LogInformation("[Startup] ConnectionString={ConnectionString}", connectionString);

// CORRETO
logger.LogInformation("[Startup] Conexão com banco de dados configurada. Database={Database}", databaseName);
```

---

## Revisão de Logs

Todo pull request deve incluir revisão dos logs adicionados, verificando:

- [ ] Nenhum dado sensível é logado diretamente
- [ ] Tokens e credenciais não aparecem em mensagens de log
- [ ] Dados pessoais são mascarados ou omitidos
- [ ] Bodies de requisição/resposta não são logados integralmente
