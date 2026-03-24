# Guia de Implantação — Logging com Serilog e Datadog

## Visão Geral

Este guia apresenta o processo de implantação para tornar aderentes à governança de Logs apenas os projetos de Lambda encontrados no repositório, conforme definido na seção de Logging.

Não adicione novo código de Logging nos projetos Core. A adequação deve ser aplicada exclusivamente aos projetos de Lambda, preservando o acoplamento já existente com os projetos Core em todos os demais aspectos e promovendo desacoplamento somente na camada de logs. Para atender à nova governança, os projetos de Lambda não podem receber novos acoplamentos; cada projeto deve implementar sua própria solução de Logging, de forma independente, sem criar dependências adicionais entre Lambdas, Core ou componentes compartilhados voltados a logs.

---

## Etapas de Implantação

### Etapa 1 — Identificação dos Projetos de Lambda Elegíveis

Identifique no repositório quais projetos são de Lambda e estão elegíveis à nova definição de governança de logs. Nessa etapa, considere que a implementação de Logging deve permanecer dentro de cada própria Lambda, sem uso de helpers de log compartilhados e sem delegar a responsabilidade de logging para outras Lambdas. Remova helpers de log e não utilize Lambdas terceiras para logar. A própria Lambda deve ser responsável por sua estratégia de Logging.

---

### Etapa 2 — Diagnóstico da Solução de Logging Atual

Verifique se cada projeto de Lambda já utiliza uma solução baseada em Serilog.

---

### Etapa 3 — Remoção da Solução de Logging Não Aderente

Caso o projeto utilize uma solução de Logging não aderente às definições de governança, identifique-a e, antes de removê-la, envie para sua janela de contexto todas as variáveis de ambiente relacionadas a esse Logging não aderente, para uso futuro na etapa de adequação ao plano de governança. Nesse processo, respeite a configuração específica de Datadog de cada projeto e não padronize indevidamente parâmetros, Secrets, índices ou quaisquer outras definições próprias da aplicação. Em seguida, remova a solução de logs não aderente e realize o commit da alteração.

---

### Etapa 4 — Instalação da Infraestrutura Serilog + Datadog

Após a remoção ou, caso o projeto não possua nenhuma solução de Logs, implemente no próprio projeto de Lambda a solução de infraestrutura necessária para registrar logs com Serilog diretamente no Datadog, incluindo os direcionamentos de sink, sem introduzir novos acoplamentos com projetos Core, com outros projetos de Lambda, com helpers de log compartilhados ou com Lambdas terceiras, e realize o commit.

---

### Etapa 5 — Configuração de Secrets e Parâmetros do Datadog

Em seguida, atualize os arquivos JSON com os Secrets do Datadog e com os parâmetros necessários para que a aplicação registre os logs nos índices corretos. Se já existir um appsettings.json no projeto, atualize esse arquivo com as configurações necessárias, preservando a estrutura já adotada pela aplicação. Como cada projeto possui sua própria configuração de Datadog, essa particularidade deve ser mantida integralmente durante a adequação. Caso essas informações não estejam disponíveis no próprio projeto ou no docker compose, solicite os dados ao usuário e não prossiga até que todas as informações necessárias tenham sido fornecidas.

---

### Etapa 6 — Verificação da Integração com o Datadog

Após implantar a conexão com o Datadog, verifique se a integração foi estabelecida corretamente e se a aplicação consegue enviar logs para a plataforma de forma adequada.

---

### Etapa 7 — Implementação Completa do Logging Estruturado

Após os testes de conexão com o Datadog, implemente em cada projeto de Lambda a solução completa de Logging descrita nos arquivos de governança, respeitando a autonomia de cada projeto e evitando qualquer novo acoplamento estrutural na camada de logs. Nessa implementação, a própria Lambda deve saber logar, sem dependência de helpers de log ou de outras Lambdas para essa responsabilidade, e realize o commit.

---

### Etapa 8 — Validação Final (Build, Testes e Run)

Depois de implantar a solução de Logging e concluir uma segunda verificação, faça o build do projeto e da solution alterada, execute os testes unitários e faça o run da aplicação. Nos testes unitários, realize ajustes somente quando eles já cobrirem Logging. Caso contrário, não crie novos testes unitários nem adicione validações de logs onde elas ainda não existirem.

---

### Etapa 9 — Correção de Falhas

Caso qualquer uma dessas etapas de compilação, teste ou execução falhe, corrija o problema e repita o processo até que as três etapas sejam concluídas com sucesso.

---

### Etapa 10 — Pull Request

Em seguida, crie um Pull Request com as alterações realizadas.
