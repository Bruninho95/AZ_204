# Passo a Passo para Criar um Microsserviço Serverless de Validação de CPF

## Requisitos

Antes de iniciar, certifique-se de ter os seguintes itens instalados:

- [.NET SDK](https://dotnet.microsoft.com/download) (versão 6.0 ou superior)
- [Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli)
- [Azure Functions Core Tools](https://docs.microsoft.com/azure/azure-functions/functions-run-local)

## 1. Configurando o Ambiente

1. **Instale o .NET SDK:**
   ```bash
   sudo apt-get update
   sudo apt-get install -y dotnet-sdk-6.0
   ```

2. **Instale o Azure CLI:**
   ```bash
   curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
   ```

3. **Instale o Azure Functions Core Tools:**
   ```bash
   npm install -g azure-functions-core-tools@4 --unsafe-perm true
   ```

## 2. Criando o Projeto Azure Functions

1. **Crie um novo diretório para o projeto:**
   ```bash
   mkdir ValidaCPFFunction
   cd ValidaCPFFunction
   ```

2. **Inicie um novo projeto de Função do Azure:**
   ```bash
   func init --worker-runtime dotnet
   ```

3. **Adicione uma nova função HTTP Trigger:**
   ```bash
   func new --name ValidarCPF --template "HTTP trigger" --authlevel "anonymous"
   ```

## 3. Implementando a Lógica de Validação de CPF

No arquivo `ValidarCPF.cs`, substitua o código pelo seguinte:

```csharp
using System.Linq;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;
using System.IO;

public static class ValidarCPF
{
    [FunctionName("ValidarCPF")]
    public static IActionResult Run(
        [HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = null)] HttpRequest req,
        ILogger log)
    {
        log.LogInformation("Processando solicitação de validação de CPF.");

        string requestBody = new StreamReader(req.Body).ReadToEndAsync().Result;
        dynamic data = JsonConvert.DeserializeObject(requestBody);
        string cpf = data?.cpf;

        if (string.IsNullOrEmpty(cpf) || !IsValidCPF(cpf))
        {
            return new BadRequestObjectResult("CPF inválido.");
        }

        return new OkObjectResult("CPF válido.");
    }

    private static bool IsValidCPF(string cpf)
    {
        cpf = new string(cpf.Where(char.IsDigit).ToArray());
        if (cpf.Length != 11 || cpf.Distinct().Count() == 1) return false;

        int[] multiplicador1 = { 10, 9, 8, 7, 6, 5, 4, 3, 2 };
        int[] multiplicador2 = { 11, 10, 9, 8, 7, 6, 5, 4, 3, 2 };

        string tempCpf = cpf.Substring(0, 9);
        int soma = 0;

        for (int i = 0; i < 9; i++)
            soma += int.Parse(tempCpf[i].ToString()) * multiplicador1[i];

        int resto = soma % 11;
        resto = resto < 2 ? 0 : 11 - resto;
        tempCpf += resto;

        soma = 0;
        for (int i = 0; i < 10; i++)
            soma += int.Parse(tempCpf[i].ToString()) * multiplicador2[i];

        resto = soma % 11;
        resto = resto < 2 ? 0 : 11 - resto;

        return cpf.EndsWith(resto.ToString());
    }
}
```

## 4. Testando Localmente

1. **Execute a função localmente:**
   ```bash
   func start
   ```

2. **Envie uma requisição de teste usando o cURL:**
   ```bash
   curl -X POST http://localhost:7071/api/ValidarCPF -H "Content-Type: application/json" -d '{"cpf":"12345678909"}'
   ```

## 5. Implantando no Azure

1. **Autentique-se no Azure:**
   ```bash
   az login
   ```

2. **Crie um grupo de recursos:**
   ```bash
   az group create --name ValidaCPFResourceGroup --location eastus
   ```

3. **Crie uma conta de Funções do Azure:**
   ```bash
   az functionapp create --resource-group ValidaCPFResourceGroup --consumption-plan-location eastus --runtime dotnet --functions-version 4 --name ValidaCPFApp --storage <NOME_DA_CONTA_DE_ARMAZENAMENTO>
   ```

4. **Implante a função:**
   ```bash
   func azure functionapp publish ValidaCPFApp
   ```

## 6. Testando no Azure

Após a implantação, você receberá uma URL pública. Teste usando o cURL:

```bash
curl -X POST https://<NOME_DA_SUA_FUNCAO>.azurewebsites.net/api/ValidarCPF -H "Content-Type: application/json" -d '{"cpf":"12345678909"}'
```

## Conclusão

Criado com sucesso um microsserviço serverless para validação de CPF usando .NET, Azure CLI e Azure Functions Core Tools. Esse microsserviço pode ser facilmente escalado e integrado a outras aplicações na nuvem.

