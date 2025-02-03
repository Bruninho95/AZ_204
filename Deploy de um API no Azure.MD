# Passo a Passo para Deploy de APIs no Azure Usando Git, Azure DevOps, OCR e Webhook

## Requisitos

Antes de iniciar, certifique-se de ter os seguintes itens:

- Conta no [Azure](https://azure.microsoft.com/)
- Conta no [Azure DevOps](https://dev.azure.com/)
- Git instalado na sua máquina ([Download Git](https://git-scm.com/downloads))
- OCR configurado se necessário para automação

## 1. Configurando o Repositório no Git

1. **Clone o repositório existente ou crie um novo:**
   ```bash
   git clone https://github.com/seu-usuario/seu-repositorio.git
   cd seu-repositorio
   ```

2. **Adicione o código da sua API:**
   ```bash
   git add .
   git commit -m "Adicionando código da API"
   git push origin main
   ```

## 2. Configurando o Azure DevOps

1. **Acesse o [Azure DevOps](https://dev.azure.com/):**

2. **Crie um novo projeto:**
   - Clique em **New Project**
   - Defina o nome e configurações desejadas

3. **Configure o Pipeline de CI/CD:**
   - Acesse **Pipelines > Create Pipeline**
   - Selecione **GitHub** ou **Azure Repos Git**
   - Escolha o repositório onde está o código da API

4. **Defina o arquivo YAML do pipeline:**
   Crie um arquivo `azure-pipelines.yml` no seu repositório:

   ```yaml
   trigger:
     - main

   pool:
     vmImage: 'ubuntu-latest'

   steps:
   - task: UseDotNet@2
     inputs:
       packageType: 'sdk'
       version: '6.x'

   - script: dotnet build
     displayName: 'Compilar a API'

   - script: dotnet test
     displayName: 'Executar Testes'

   - task: AzureWebApp@1
     inputs:
       azureSubscription: '<Nome-da-Subscrição>'
       appType: 'webApp'
       appName: '<Nome-da-API-no-Azure>'
       package: '$(System.DefaultWorkingDirectory)/**/*.zip'
   ```

5. **Salve e execute o pipeline:**
   - Clique em **Run Pipeline** para testar o deploy

## 3. Integrando OCR (Reconhecimento Óptico de Caracteres)

Se o deploy envolver OCR, por exemplo, para processar documentos:

1. **Adicione dependências de OCR ao seu código:**
   - Pode ser o [Azure Cognitive Services](https://azure.microsoft.com/en-us/services/cognitive-services/computer-vision/)

2. **Configure o OCR no Azure:**
   - Crie um recurso de **Computer Vision** no Azure Portal
   - Copie a chave e URL de endpoint

3. **Integre o OCR ao código da API:**
   ```csharp
   var client = new ComputerVisionClient(new ApiKeyServiceClientCredentials("<sua-chave>"))
   {
       Endpoint = "<sua-url-de-endpoint>"
   };
   ```

## 4. Configurando o Webhook para Deploy Automatizado

1. **Acesse o Azure Portal:**
   - Encontre o recurso da sua API
   - Clique em **Deployment Center**

2. **Adicione um Webhook:**
   - Clique em **Webhook +**
   - Insira a URL do webhook do Azure DevOps (encontrada em **Service Hooks** do Azure DevOps)

3. **Teste o Webhook:**
   - Faça um commit no repositório:
     ```bash
     git commit -m "Trigger webhook para deploy"
     git push origin main
     ```

4. **Verifique o deploy no Azure DevOps:**
   - Acesse **Pipelines** e acompanhe a execução

## 5. Monitoramento e Logs

- No Azure Portal, acesse **App Service > Logs** para verificar o status do deploy.
- No Azure DevOps, verifique o histórico de execução dos pipelines.

## Conclusão

Você configurou com sucesso o deploy automatizado de APIs no Azure usando Git, Azure DevOps, OCR e Webhook. Esse fluxo permite uma entrega contínua mais eficiente e monitorada.

