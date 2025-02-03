# Movie Catalog Manager With Azure Function.

This project aims to create an movie catalog manager with Azure Function and CosmosDB. We'll show step by step of this process.


## ⚙️ Implementation Steps:


These are the steps we will follow to structure this video catalog manager. Next, we will break down each of these steps in detail.


- Create the Cloud Infrastructure;

- Create an Azure Function to save files to the storage account;

- Create an Azure Function to save to CosmosDB;

- Create an Azure Function to filter records in CosmosDB;

- Create an Azure Function to list records in CosmosDB.


### Create the Cloud Infrastructure:


The first Microsoft Azure role we will create is an API Manager. After creating this manager, we need to create the CosmosDB NoSQL database and a Storage Account.

Finally, we need to create a Front End for the application. In this step, we will use Streamlit.io just to have some visualization.

These creation steps are simple, just follow what Microsoft Azure asks for. So we won't go into details at this stage. I believe a relevant topic to understand here is the format in which records are made in CosmosDB.

```json

{
  "id" : <Database ID>,
  "title" : <Video Title>,
  "year" : <Movie Year>,
  "video" : <Video Path>,
  "thub" : <Template Image Path>
}

```


### Create an Azure Function to save files to the storage account:


Before going into the details of the code behind the function to save files to the storage account, we need to make a small modification to the configuration file of this function. The following modification must be in the Program.cs folder so that the function is able to accept large files (in this case 100 MB), since videos tend to be larger in size.

```csharp

builder.Services.Configure<KestrelServerOptions>(options =>
{
    options.Limits.MaxRequestBodySize = 104857600; // 1024 * 1024 * 100, // 100 MB
});


```

Remembering that we also need to add the connection string key to the function settings, so that it can communicate with the storage account. We also have to create two containers in the storage account, one for the videos and one for the template images.


Now we can show the code used for the main functionality of the function.

```csharp

using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.Logging;
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;
using System;
using System.Threading.Tasks;

namespace fnPostDataStorage
{
    public class Function1
    {
        private readonly ILogger<Function1> _logger;

        public Function1(ILogger<Function1> logger)
        {
            _logger = logger;
        }

        [Function("dataStorage")]
        public async Task<IActionResult> Run([HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequest req)
        {
            _logger.LogInformation("Processando a Imagem no Storage");

            try
            {
                if (!req.Headers.TryGetValue("file-type", out var fileTypeHeader))
                {
                    return new BadRequestObjectResult("O cabeçalho 'file-type' é obrigatório");
                }

                var fileType = fileTypeHeader.ToString();
                var form = await req.ReadFormAsync();
                var file = form.Files["file"];

                if (file == null || file.Length == 0)
                {
                    return new BadRequestObjectResult("O arquivo não foi enviado");
                }

                string connectionString = Environment.GetEnvironmentVariable("AzureWebJobsStorage");
                string containerName = fileType;

                BlobContainerClient containerClient = new BlobContainerClient(connectionString, containerName);
                await containerClient.CreateIfNotExistsAsync();
                await containerClient.SetAccessPolicyAsync(PublicAccessType.BlobContainer);

                string blobName = file.FileName;
                var blob = containerClient.GetBlobClient(blobName);

                using (var stream = file.OpenReadStream())
                {
                    await blob.UploadAsync(stream, true);
                }

                _logger.LogInformation($"Arquivo {file.FileName} armazenado com sucesso");

                return new OkObjectResult(new
                {
                    Message = "Arquivo armazenado com sucesso",
                    BlobUri = blob.Uri
                });
            }
            catch (Exception ex)
            {
                _logger.LogError($"Erro ao processar a requisição: {ex.Message}");
                return new StatusCodeResult(StatusCodes.Status500InternalServerError);
            }
        }
    }
}

```


### Create an Azure Function to save to CosmosDB:


Now that we have the function that saves the images and videos in the storage account, we need a function to save this record in CosmosDB.

For the application to have connectivity with the database, it is necessary to add some parameters in the local configuration file: localsettings.json

```json

{
  "CosmosDBConnection" : <Primary Connection String>,
  "DatabaseName" : <Data Base Name>,
  "ContainerName" : <Container Name>
}

```


Before we delve into the code that will be used to perform the main task of this function, we must create the file that will dictate the format of a record in MovieRequest.cs in the format that was previously shown.

```csharp

using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace fnPostDatabase
{
    internal class MovieRequest
    {
        public string id { get { return Guid.NewGuid().ToString(); } }
        public string title { get; set; }
        public string year { get; set; }
        public string video { get; set; }
        public string thub { get; set; }
    }
}


```


Now that we have made all the configuration changes, we can move on to the main code of the function.

```csharp

using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.Logging;
using System.IO;
using System.Threading.Tasks;
using Newtonsoft.Json;

namespace fnPostDatabase
{
    public class Function1
    {
        private readonly ILogger<Function1> _logger;

        public Function1(ILogger<Function1> logger)
        {
            _logger = logger;
        }

        [Function("movie")]
        [CosmosDBOutput("%DatabaseName%", "%ContainerName%", Connection = "CosmoDBConnection", CreateIfNotExists = true, PartionKey = "id")]
        public async Task<object> Run([HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequest req)
        {
            _logger.LogInformation("C# HTTP trigger function processed a request.");

            MovieRequest movie = null;

            var content = await new StreamReader(req.Body).ReadToEndAsync();

            try
            {
                movie = JsonConvert.DeserializeObject<MovieRequest>(content);
            }
            catch (Exception ex)
            {
                return new BadRequestObjectResult("Erro ao desserializar o objeto: " + ex.Message);
            }

            return JsonConvert.SerializeObject(movie);
        }
    }
}


```


Remembering that there is a need to create a container in cosmosDB and the partition keys so that errors do not occur when saving records.


### Create an Azure Function to filter records in CosmosDB:


Now that we have saved the data to the database, we need to create the functions that consume this data. Let's start with the function that filters the records from CosmosDB.

Before we present the main code of the function, let's create the MovieResulta class that represents the search result.

```csharp

using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace fnGetMovieDetail
{
    internal class MovieResult
    {
        public string id { get; set; }
        public string title { get; set; }
        public string year { get; set; }
        public string video { get; set; }
        public string thumb { get; set; }
    }
}

```


Remembering that there is a need to add the cosmosDB connectivity key to the local configuration file (localsettings.json), this step has already been demonstrated previously.

In order to be able to run local tests, we also need to add CORS in the same local configuration file.

```json

"Host" : {
  "CORS" : "*"
}

```


To avoid having to perform repetitive instance processes in cosmosDB, we need to make a change to the Program.cs configuration file. We can see the contents of the file after the following modifications.

```csharp

using Microsoft.Azure.Cosmos;
using Microsoft.Azure.Functions.Worker.Builder;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var builder = FunctionsApplication.CreateBuilder(args);

builder.ConfigureFunctionsWebApplication();

// Application Insights isn't enabled by default. See https://aka.ms/AAt8mw4.
// builder.Services
//     .AddApplicationInsightsTelemetryWorkerService()
//     .ConfigureFunctionsApplicationInsights();

builder.Services.AddSingleton(s =>
{
    string connectionString = Environment.GetEnvironmentVariable("CosmoDBConnection");
    return new CosmosClient(connectionString);
});

builder.Build().Run();


```


Now that all the necessary configurations have been made, we can present the main code of the Azure function for filtering CosmosDB records.

```csharp

using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.Logging;

namespace fnGetMovieDetail
{
    public class Function1
    {
        private readonly ILogger<Function1> _logger;
        private readonly CosmosClient _cosmosClient;

        public Function1(ILogger<Function1> logger, CosmosClient cosmosClient)
        {
            _logger = logger;
            _cosmosClient = cosmosClient;
        }

        [Function("detail")]
        public async Task<HttpResponseData> Run([HttpTrigger(AuthorizationLevel.Function, "get")] HttpRequestData req)
        {
            _logger.LogInformation("C# HTTP trigger function processed a request.");
            var container = _cosmosClient.GetContainer("%DatabaseName%", "%ContainerName%");
            var id = req.Query["id"];
            var query = "SELECT * FROM c WHERE c.id = @id";
            var queryDefinition = new QueryDefinition(query).WithParameter("@id", id);
            var result = container.GetItemQueryIterator<MovieResult>(queryDefinition);
            var results = new List<MovieResult>();
            
            while (result.HasMoreResults)
            {
                foreach (var item in await result.ReadNextAsync())
                {
                    results.Add(item);
                }
            }
            
            var responseMessage = req.CreateResponse(System.Net.HttpStatusCode.OK);
            await responseMessage.WriteAsJsonAsync(results.FirstOrDefault());
            
            return responseMessage;
        }
    }
}


```


### Create an Azure Function to list records in CosmosDB:

In the same way as in the previous item, it is necessary to create a MovieResults class and make the modifications in Program.cs to automate the instantiation in CosmosDB. Remember that there is a need to recreate these files even though they have already been created previously for another microservice, since these microservices are independent and we must maintain the minimum dependency between them.

Now that all the necessary configurations are done, we can present the main code of the Azure Function to List CosmosDB Records.

```csharp

using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.Logging;

namespace fnGetMovieDetail
{
    public class Function1
    {
        private readonly ILogger<Function1> _logger;
        private readonly CosmosClient _cosmosClient;

        public Function1(ILogger<Function1> logger, CosmosClient cosmosClient)
        {
            _logger = logger;
            _cosmosClient = cosmosClient;
        }

        [Function("all")]
        public async Task<HttpResponseData> Run([HttpTrigger(AuthorizationLevel.Function, "get")] HttpRequestData req)
        {
            _logger.LogInformation("C# HTTP trigger function processed a request.");
            var container = _cosmosClient.GetContainer("DioFlixDB", "movies");
            var query = "SELECT * FROM c";
            var queryDefinition = new QueryDefinition(query);
            var result = container.GetItemQueryIterator<MovieResult>(queryDefinition);
            var results = new List<MovieResult>();
            
            while (result.HasMoreResults)
            {
                foreach (var item in await result.ReadNextAsync())
                {
                    results.Add(item);
                }
            }
            
            var responseMessage = req.CreateResponse(System.Net.HttpStatusCode.OK);
            await responseMessage.WriteAsJsonAsync(results);
            
            return responseMessage;
        }
    }
}


```

## ✳️ Final Considerations

With the creation and implementation of all these microservices, the last step would be to create a front end to visualize how they work. This step was not implemented, but it would be possible to perform this test with AI-generated code.

