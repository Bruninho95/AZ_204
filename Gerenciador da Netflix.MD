# Passo a Passo para Criar um Gerenciador de Catálogos da Netflix

## Tecnologias Utilizadas

- **SQL Database**: Para armazenar metadados do catálogo.
- **Azure Storage Account**: Para armazenar arquivos de mídia e imagens.
- **CosmosDB**: Para armazenar e consultar registros de forma escalável.
- **Azure Functions**: Para operações de backend serverless (salvar, filtrar e listar registros).
- **Redis Cache**: Para otimizar consultas frequentes.

## 1. Configuração do Banco de Dados SQL

1. **Crie o SQL Database no Azure:**
   - Acesse o [Azure Portal](https://portal.azure.com/)
   - Clique em **Create a resource** > **SQL Database**
   - Configure o servidor e defina credenciais

2. **Crie a Tabela de Metadados:**
   ```sql
   CREATE TABLE CatalogoNetflix (
       Id INT PRIMARY KEY IDENTITY(1,1),
       Titulo NVARCHAR(255),
       Genero NVARCHAR(100),
       AnoLancamento INT,
       Descricao NVARCHAR(MAX),
       UrlImagem NVARCHAR(500)
   );
   ```

## 2. Configuração do Azure Storage Account

1. **Crie uma Storage Account:**
   - No Azure Portal, clique em **Create a resource** > **Storage Account**
   - Defina o nome e configurações

2. **Crie um Container:**
   - Acesse a Storage Account > **Containers** > **+ Container**
   - Defina permissões conforme necessidade (privado ou público)

## 3. Configuração do CosmosDB

1. **Crie uma Instância do CosmosDB:**
   - No Azure Portal, clique em **Create a resource** > **Azure Cosmos DB**
   - Selecione a API do Core (SQL)

2. **Configure o Banco e a Coleção:**
   - Crie um novo banco de dados e uma coleção chamada `Catalogo`

## 4. Criação de Azure Functions

### a. **Salvar Arquivos no Storage Account**

1. **Crie uma Function App:**
   ```bash
   func init CatalogManager --worker-runtime dotnet
   cd CatalogManager
   func new --name UploadArquivo --template "HTTP trigger" --authlevel "anonymous"
   ```

2. **Implementação:**
   ```csharp
   [FunctionName("UploadArquivo")]
   public static async Task<IActionResult> Run(
       [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequest req,
       ILogger log)
   {
       var file = req.Form.Files[0];
       var connectionString = Environment.GetEnvironmentVariable("AzureWebJobsStorage");
       var blobClient = new BlobContainerClient(connectionString, "catalogo-media");
       await blobClient.CreateIfNotExistsAsync();
       var blob = blobClient.GetBlobClient(file.FileName);
       await blob.UploadAsync(file.OpenReadStream());

       return new OkObjectResult($"Arquivo salvo em: {blob.Uri}");
   }
   ```

### b. **Filtrar e Listar Registros no CosmosDB**

1. **Nova Função:**
   ```bash
   func new --name ListarCatalogo --template "HTTP trigger" --authlevel "anonymous"
   ```

2. **Implementação:**
   ```csharp
   [FunctionName("ListarCatalogo")]
   public static async Task<IActionResult> Run(
       [HttpTrigger(AuthorizationLevel.Function, "get")] HttpRequest req,
       [CosmosDB(
           databaseName: "NetflixDB",
           containerName: "Catalogo",
           Connection = "CosmosDBConnection")] CosmosClient client,
       ILogger log)
   {
       var container = client.GetContainer("NetflixDB", "Catalogo");
       var query = new QueryDefinition("SELECT * FROM c WHERE c.Genero = @genero")
           .WithParameter("@genero", req.Query["genero"]);

       var iterator = container.GetItemQueryIterator<dynamic>(query);
       var resultados = new List<dynamic>();

       while (iterator.HasMoreResults)
       {
           foreach (var item in await iterator.ReadNextAsync())
           {
               resultados.Add(item);
           }
       }

       return new OkObjectResult(resultados);
   }
   ```

## 5. Configuração do Redis Cache

1. **Crie uma Instância do Redis Cache:**
   - No Azure Portal, clique em **Create a resource** > **Azure Cache for Redis**

2. **Integração com Azure Functions:**
   ```csharp
   private static readonly ConnectionMultiplexer redis =
       ConnectionMultiplexer.Connect(Environment.GetEnvironmentVariable("RedisConnection"));

   [FunctionName("GetCatalogoCache")]
   public static async Task<IActionResult> Run(
       [HttpTrigger(AuthorizationLevel.Function, "get")] HttpRequest req,
       ILogger log)
   {
       var db = redis.GetDatabase();
       var cacheKey = "catalogo_completo";
       var cacheData = await db.StringGetAsync(cacheKey);

       if (cacheData.IsNullOrEmpty)
       {
           // Consulta no CosmosDB caso o cache esteja vazio
           var resultados = await ConsultarCosmosDB();
           await db.StringSetAsync(cacheKey, JsonConvert.SerializeObject(resultados), TimeSpan.FromMinutes(10));
           return new OkObjectResult(resultados);
       }

       return new OkObjectResult(JsonConvert.DeserializeObject<List<dynamic>>(cacheData));
   }
   ```

## 6. Testes e Validação

- **Testar Upload de Arquivos:** via Postman ou cURL
- **Consultar o Catálogo:** utilizando parâmetros de filtro
- **Verificar Desempenho do Cache:** observando respostas mais rápidas após a primeira consulta

## Conclusão

Configurado um gerenciador de catálogos da Netflix usando Azure SQL Database, Storage Account, CosmosDB, Azure Functions e Redis Cache. Esse ambiente é escalável, otimizado para alta performance e fácil de manter.

