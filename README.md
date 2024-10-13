# Azure Functions
Executar código sem servidor controlado por eventos com uma experiência de desenvolvimento de ponta a ponta.

## Definição:
- **Serverless**: Execute código sem precisar gerenciar infraestrutura;
- **Escalabilidade**: Escala automaticamente com base na demanda;
- **Event-driven**: Executa em resposta a eventos (HTTP requests, mensagens em filas, timers, etc.);

## Modelos de Execução:
- **HTTP Trigger**: Executa em resposta a requisições HTTP;
- **Timer Trigger**: Executa com base em um cronograma;
- **Blob Trigger**: Executa quando arquivos são criados/alterados no Azure Blob Storage;
- **Queue Trigger**: Executa quando uma nova mensagem é adicionada a uma fila do Azure Storage Queue;
- **Event Grid/Service Bus Trigger**: Responde a eventos/mensagens em serviços de mensagens;

## Ferramentas:

### Azure Functions Core Tools
Ferramenta de linha de comando (CLI) para desenvolver, testar e implantar funções localmente antes de fazer deploy para a Azure. Com Core Tools, você pode criar uma função localmente e rodar o ambiente serverless no seu próprio sistema.

**Comandos**:
- Criar nova Azure Function (_AzFc_):
```bash
func init <NomeDaApp> --worker-runtime python
func new
```
- Rodar Localmente:
```bash
func start
```
- Publicar/deploy para Azure:
```bash
func azure functionapp publish <NomeDoApp>
```

### VScode plugins:
- Azure Functions ([link](https://learn.microsoft.com/en-us/azure/azure-functions/functions-develop-vs-code?tabs=node-v4%2Cpython-v2%2Cisolated-process%2Cquick-create&pivots=programming-language-python));
- Azure Resources ([link](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azureresourcegroups));
- Azurite ([link](https://marketplace.visualstudio.com/items?itemName=Azurite.azurite));
- Azure CLI Tools ([link](https://marketplace.visualstudio.com/items?itemName=Azurite.azurite));

Comandos:
- `CTRL` + `SHIFT` + `p`: Abrir Command Pallet;
- `Azure Functions: Create New Project`;
- Escolher **Python** como runtime da função;
- Após criar o projeto, você pode testar localmente clicando em "Run" ou rodando o comando `func start`;
- Fazer deploy é fácil, basta clicar com o botão direito no projeto e selecionar "Deploy to Function App";

### Azure CLI ([link](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-linux))
```bash
az --version
```
Exemplo de uso:
- Criação de um novo Function App:
```bash
az functionapp create --resource-group <NomeDoGrupo> --consumption-plan-location <Localização> --runtime python --runtime-version 3.9 --functions-version 3 --name <NomeDoApp> --storage-account <NomeDoStorage>
```
- Fazer Deploy:
```bash
az functionapp deployment source config-zip --resource-group <NomeDoGrupo> --name <NomeDoApp> --src <ArquivoZip>
```
## File System
```bash
|-- home\
    |-- site\
        |-- wwwroot\
            |-- host.json               # Configurações de runtime da Function App
            |-- <NomeDaFunção1>\
                |-- __init__.py          # Código da função Python
                |-- function.json        # Configurações de triggers e bindings da função
            |-- <NomeDaFunção2>\
                |-- __init__.py          # Código da função Python
                |-- function.json        # Configurações de triggers e bindings da função
    |-- LogFiles\                        # Logs temporários da aplicação
    |-- local\
        |-- Temp\                        # Diretório para arquivos temporários
    |-- data\                            # Dados persistidos temporariamente (logs, diagnósticos)
    |-- .python_packages\                # Dependências Python instaladas
```

## Estrutura Básica da Função (Python):

### HTTP Trigger:
```python
import logging
import azure.functions as func

def main(req: func.HttpRequest) -> func.HttpResponse:
    logging.info('Python HTTP trigger function processed a request.')
    
    name = req.params.get('name')
    if not name:
        try:
            req_body = req.get_json()
        except ValueError:
            pass
        else:
            name = req_body.get('name')

    if name:
        return func.HttpResponse(f"Hello, {name}!")
    else:
        return func.HttpResponse(
             "Please pass a name in the query string or in the request body",
             status_code=400
        )
```
- O HTTP Trigger é ativado por uma solicitação HTTP (GET ou POST);
- O exemplo acima captura um parâmetro name da URL ou do corpo da requisição. Se encontrado, retorna uma mensagem de boas-vindas; caso contrário, retorna um erro 400;
- Útil para criar APIs ou expor endpoints para integração com outros serviços;

#### Bindings para HTTP Trigger (`function.json`):
```json
{
  "bindings": [
    {
      "authLevel": "function",
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "methods": ["get", "post"]
    },
    {
      "type": "http",
      "direction": "out",
      "name": "$return"
    }
  ]
}
```
- `authLevel`: Define o nível de autenticação. Pode ser:
    - `function`: Requer uma chave de função para acessar.
    - `anonymous`: Acesso sem restrição.
    - `admin`: Requer chave de administrador.
- `methods`: Define os métodos HTTP aceitos, como GET e POST.
- `name`: Nome do parâmetro que receberá a solicitação HTTP (neste caso, `req`).
- `$return`: Define que o valor retornado será enviado como resposta HTTP.

I/O:
- Input Binding: Solicitação HTTP que aciona a função.
- Output Binding: Resposta HTTP retornada.

### Timer Trigger:
```python
import datetime
import logging

def main(mytimer: func.TimerRequest) -> None:
    utc_timestamp = datetime.datetime.utcnow().replace(
        tzinfo=datetime.timezone.utc).isoformat()

    if mytimer.past_due:
        logging.info('The timer is past due!')

    logging.info(f'Python timer trigger function ran at {utc_timestamp}')
```
- O **Timer Trigger** é ativado de acordo com um cronograma definido (pode ser diário, semanal, ou personalizado);
- No exemplo, ele verifica se o timer está atrasado (`past_due`) e registra a execução com o timestamp atual;
- Esse tipo de trigger é ótimo para tarefas agendadas, como limpeza de dados ou geração de relatórios;

#### Bindings para Timer Trigger (`function.json`):
```json
{
  "bindings": [
    {
      "name": "mytimer",
      "type": "timerTrigger",
      "direction": "in",
      "schedule": "0 */5 * * * *"
    }
  ]
}
```
- `schedule`: Define a frequência do trigger usando uma expressão CRON:
    - Exemplo: `"0 */5 * * * *"` ativa a função a cada 5 minutos;
    - Formato: `"segundos minutos horas diasDoMês mês diasDaSemana"`;
- `name`: Nome do parâmetro de entrada, neste caso `mytimer`;

### Blob Trigger:
```python
import logging

def main(blob: func.InputStream):
    logging.info(f'Python Blob trigger function processed blob \n'
                 f'Name: {blob.name}\n'
                 f'Blob Size: {blob.length} bytes')
```
- O **Blob Trigger** é ativado quando um arquivo (blob) é criado ou alterado em um **Azure Blob Storage**;
- No exemplo, ele processa o arquivo, registra seu nome e tamanho;
- Muito útil para processamento de dados, como análise de imagens ou arquivos enviados para um storage;

#### Bindings para Blob Trigger (`function.json`):
```json
{
  "bindings": [
    {
      "type": "blobTrigger",
      "direction": "in",
      "name": "inputBlob",
      "path": "samples-workitems/{name}",
      "connection": "AzureWebJobsStorage"
    },
    {
      "type": "blob",
      "direction": "out",
      "name": "outputBlob",
      "path": "samples-workitems-output/{name}",
      "connection": "AzureWebJobsStorage"
    }
  ]
}
```
- `path`: Define o caminho do blob no Azure Storage. O `{name}` é um placeholder para capturar o nome do arquivo.
- `connection`: Nome da string de conexão com o **Azure Blob Storage**, configurado no **App Settings**.
I/O:
- Input Binding: Captura um blob quando ele é criado/alterado;
- Output Binding: Grava um blob processado no storage;

### Queue Trigger:
```python
import logging

def main(msg: func.QueueMessage) -> None:
    logging.info(f'Python Queue trigger function processed message: {msg.get_body().decode("utf-8")}')
```
- O **Queue Trigger** é ativado quando uma mensagem é adicionada à fila;
- A mensagem da fila é passada como um argumento para a função `main`, e seu corpo pode ser acessado com `msg.get_body()`;

#### Bindings para Queue Trigger (`function.json`):
```json
{
  "bindings": [
    {
      "type": "queueTrigger",
      "direction": "in",
      "name": "myQueueItem",
      "queueName": "input-queue",
      "connection": "AzureWebJobsStorage"
    },
    {
      "type": "queue",
      "direction": "out",
      "name": "outputQueueItem",
      "queueName": "output-queue",
      "connection": "AzureWebJobsStorage"
    }
  ]
}
```
- `queueTrigger`: Indica que a função será disparada por uma mensagem na fila;
- `queueName`: Nome da fila no Azure Storage;
I/O:
- Input Binding: Captura uma mensagem de uma fila no Queue Storage;
- Output Binding: Envia uma nova mensagem para outra fila;

### Event Grid Trigger:
```python
import logging
import json

def main(event: func.EventGridEvent):
    logging.info(f'Python EventGrid trigger function received an event: {event.id}')
    logging.info(f'Event data: {json.dumps(event.get_json(), indent=2)}')
```
- O **Event Grid Trigger** é acionado quando um evento é publicado em um **Event Grid**;
- O evento recebido contém um **JSON** com informações como o tipo de evento, o ID e os dados do evento em si. Estes podem ser acessados com `event.get_json()`;

#### Bindings para Event Grid Trigger (`function.json`):
```json
{
  "bindings": [
    {
      "type": "eventGridTrigger",
      "name": "event",
      "direction": "in"
    }
  ]
}
```
- `eventGridTrigger`: Dispara a função quando um evento é recebido do Event Grid;
- `name`: Nome do parâmetro que a função vai usar para acessar o evento;

### Service Bus Trigger:
```python
import logging
import json

def main(event: func.EventGridEvent):
    logging.info(f'Python EventGrid trigger function received an event: {event.id}')
    logging.info(f'Event data: {json.dumps(event.get_json(), indent=2)}')
```
- O **Service Bus Trigger** é acionado por uma nova mensagem na fila do **Azure Service Bus**;
- A mensagem é passada como `ServiceBusMessage`, e o corpo pode ser acessado e processado;

#### Bindings para Service Bus Trigger (`function.json`):
```json
{
  "bindings": [
    {
      "type": "serviceBusTrigger",
      "name": "msg",
      "queueName": "myqueue",
      "connection": "AzureWebJobsServiceBus"
    }
  ]
}
```
- `serviceBusTrigger`: Define o tipo de trigger para o Service Bus;
- `queueName`: Nome da fila do Service Bus que vai disparar a função;
- `connection`: String de conexão ao Service Bus (definida no App Settings);

## Variáveis de Ambiente (App Settings)
Variáveis de ambiente que armazenam configurações (conexão ao banco de dados, configurações de sistema, chaves API, etc).

### Configurações:
- Localmente: Editar arquivo `local.settings.json`;
- Global: Via Azure CLI ou no Azure Portal;

**Azure CLI**:
- Adicionar App Setting:
```bash
az functionapp config appsettings set --name <NomeDoApp> --resource-group <NomeDoGrupo> --settings "<NomeDaConfig>=<Valor>"
```
- Listar App Settings:
```bash
az functionapp config appsettings list --name <NomeDoApp> --resource-group <NomeDoGrupo>
```
- Remover um App Setting:
```bash
az functionapp config appsettings delete --name <NomeDoApp> --resource-group <NomeDoGrupo> --setting-names <NomeDaConfig>
```
**Local Settings**
Estrutura:
```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "python",
    "AzureWebJobsServiceBus": "Endpoint=sb://<ServiceBusName>.servicebus.windows.net/;SharedAccessKeyName=<KeyName>;SharedAccessKey=<Key>"
  }
}
```
`"AzureWebJobsStorage"`: String de conexão para o **Azure Blob Storage** e **Queue Storage**;
`"AzureWebJobsServiceBus"`: String de conexão para o **Azure Service Bus**;
`"FUNCTIONS_WORKER_RUNTIME"`: Define a linguagem do worker (neste caso, python);

**Variáveis Comuns (Commons)**
- `AzureWebJobsStorage`: String de conexão para o Azure Blob Storage e Queue Storage;
- `AzureWebJobsServiceBus`: String de conexão para o Azure Service Bus;
- `APPINSIGHTS_INSTRUMENTATIONKEY`: Chave para monitoramento com Application Insights;
- `DB_CONNECTION_STRING`: String de conexão para banco de dados;

## Bindings
Bindings são configurações que permite a integração da AzFc com outros serviços,como Storage, Service Bus, Cosmo DB, HTTP sem a necessidade de IaC (Infra as Code). Bindings são divididas em:
- **Input Bindings**: Trazer dados externos para sua função (ex: dados de uma tabela, fila, blob);
- **Output Bindings**: Enviar dados processados pela função para outros serviços (ex: gravar em uma tabela, enviar uma mensagem para uma fila, armazenar um blob);

Uma função pode ter múltiplos **bindings**, e cada um é configurado dentro do arquivo `function.json` ou diretamente no código, dependendo da abordagem.

### Configurando Bindings
Exemplo de Binding - HTTP trigger com output para o Blob Storage (`function.json`):
```json
{
  "bindings": [
    {
      "authLevel": "function",
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "methods": ["get", "post"]
    },
    {
      "type": "blob",
      "direction": "out",
      "name": "outputBlob",
      "path": "samples-workitems/{rand-guid}",
      "connection": "AzureWebJobsStorage"
    }
  ]
}
```
- `type`: Define o tipo de binding (HTTP, Blob, Queue, etc.).
- `direction`: Define se é um binding de entrada (in) ou de saída (out).
- `name`: Nome do parâmetro que vai receber ou enviar dados.
- `path`: O caminho no serviço, como a localização do blob ou fila.
- `connection`: O nome da string de conexão, que pode ser configurada nos App Settings.

### Criando Bindings com Python
Bindings em Python são acessadas como parametros de um função. Para criar, defina os parametros correspondentes, assim, o Azure Functions Runtime gerenciará o fluxo de dados.
Exemplo - HTTP Trigger com Blob Output:
```python
import logging
import azure.functions as func

def main(req: func.HttpRequest, outputBlob: func.Out[str]) -> func.HttpResponse:
    logging.info('Python HTTP trigger function processed a request.')

    name = req.params.get('name')
    if not name:
        try:
            req_body = req.get_json()
        except ValueError:
            pass
        else:
            name = req_body.get('name')

    if name:
        outputBlob.set(f"Hello, {name}!")  # Grava o dado no blob
        return func.HttpResponse(f"Hello, {name}!")
    else:
        return func.HttpResponse(
            "Please pass a name in the query string or in the request body",
            status_code=400
        )
```
- `req`: Solicitação HTTP que aciona a função;
- `outputBlob`: Um output binding para gravar dados no Blob Storage;


## Durable Functions
Permitem workflows complexos, como orquestração de várias funções.
```python
import azure.functions as func
import azure.durable_functions as df

def orchestrator_function(context: df.DurableOrchestrationContext):
    result1 = yield context.call_activity('Hello', "Tokyo")
    result2 = yield context.call_activity('Hello', "Seattle")
    result3 = yield context.call_activity('Hello', "London")
    return [result1, result2, result3]

main = df.Orchestrator.create(orchestrator_function)
```

## Deployment
- CI/CD: Git Actions ou Azure DevOps para automação;
- Azure CLI (Azure Functions Core Tools): Para a criação, testes e deploy localmente;

CLI Deployment:
```bash
func azure functionapp publish <FunctionAppName>
```

## Boas Práticas
- **Mantenha as funções pequenas e modulares**: Evite funções complexas. Cada função deve fazer uma única tarefa bem;
- **Use Bindings para minimizar código de integração**: Conecte a serviços de armazenamento, filas e bancos de dados sem reinventar a roda;
- **Melhorar latência**: Coloque a função na mesma região dos serviços que ela consome;