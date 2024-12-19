# Deploy-Java-and-.NET-Apps-to-Azure-with-GitHub-Actions
Deploy a Java Spring Boot React app and a .NET app to Azure App Service using GitHub Actions for CI/CD. Containerize apps with Docker, store images in Azure Container Registry, and connect the backend to Azure SQL Database. Create separate pipelines for the backend, frontend, and the .NET app.  

## Project Structure

Two folders were created for the project.   

**Frontend**
The frontend part of the project includes all the JavaScript, CSS, and HTML files.   



**Backend**  The backend part of the project includes all the Java files and configuration files for the Spring Boot application.  

```
frontend/
  src/
    main/
      js/
        api/
        app.js
        client.js
        follow.js
        websocket-listener.js
      resources/
        static/
          built/
          main.css
          templates/
  webpack.config.js
  package.json

backend/
  src/
    main/
      java/
        com/
          contoso/
            payroll/
      resources/
        application.properties
  pom.xml
  mvnw
  mvnw.cmd
  target/
```



### Step 1: Clone the Provided Repositories  
- For the Spring Boot React application:

``git clone https://gitlab.com/cloud-devops-assignments/spring-boot-react-example.git
 ``
- For the .NET application:
  
``git clone https://gitlab.com/blue-harvest-assignments/cloud-assignment.git
  ``
### Step 2: Push the repositories to GitHub:

- Created two separate GitHub repositories:

  ``https://github.com/raphgm/spring-boot-react-example``
  
  ``https://github.com/raphgm/dotnet-app-example``
  
- Push the cloned repositories to their respective GitHub repositories:

  - Spring Boot React Application:
    
    ``cd spring-boot-react-example``
    
    ``git remote set-url --push origin https://github.com/raphgm/spring-boot-react-example.git
      ``
       
    ``git push --mirror``

   - .NET Application:

     ``cd cloud-assignment``
   
     ``git remote add origin https://github.com/raphgm/dotnet-app-example.git``

     ``git branch -M main``
  
     ``git push -u origin main``
  
  

### Step 3: Configure Azure Resources

- **Create a Resource Group for the Project:**

  ``az group create --name app-service-deployment --location eastus``

- **Create Azure Container Registry (ACR):**

  ``az acr create --resource-group app-service-deployment --name myacr1234 --sku Basic``
  
- **Enable Admin Access on ACR:**
  
  ``az acr update --name myacr1234 --admin-enabled true``
  

- **Set Up Azure App Service Plans:**
  - Backend: backend-plan
  -  Frontend: frontend-plan
  -  .NET App: dotnet-plan
 
- Use the following Azure CLI commands:
  
    ``az appservice plan create --name backend-plan --resource-group app-service-deployment --sku B1``
  
  ``az appservice plan create --name frontend-plan --resource-group app-service-deployment --sku B1``
  
  ``az appservice plan create --name dotnet-plan --resource-group app-service-deployment --sku B1``

- **Set Up Azure App Services:** Create App Services for all components:
  - Spring Boot Backend:
    
    ``az webapp create --resource-group app-service-deployment \
  --plan backend-plan \
  --name backend-app \
  --deployment-container-image-name myacr1234.azurecr.io/backend:latest
``

  - React Frontend:

    ``az webapp create --resource-group app-service-deployment \
  --plan frontend-plan \
  --name frontend-app \
  --deployment-container-image-name myacr1234.azurecr.io/frontend:latest
``

  - .NET Application:
    
     ``az webapp create --resource-group app-service-deployment \
  --plan dotnet-plan \
  --name dotnet-app \
  --deployment-container-image-name myacr1234.azurecr.io/dotnet-app:latest
``

- **Set Up Azure SQL Database:**
  - Create the SQL Server and Database:

    ``az sql server create --name sqlserver1234 --resource-group app-service-deployment --admin-user <username> --admin-password <password>``
    
     ``az sql db create --name appdb --server sqlserver1234 --resource-group app-service-deployment --service-objective S0``
    
- **Allow Azure App Services to Connect to SQL Server:**

   ``az sql server firewall-rule create --resource-group app-service-deployment --server <server-name> --name AllowAzureServices --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0``

**Create a Key Vault to Store all Sensitive keys for the deployment**  

``az keyvault create --name deploymentkeyvault --resource-group app-service-deployment --location northeurope``

**Store SQL Server Credentials in Key Vault:**

``az keyvault secret set \ --vault-name deploymentkeyvault \ --name "S<qlUsername>" \ --value "<username value"``

**Store SQL Password:**

``az keyvault secret set \ --vault-name deploymentkeyvault \ --name "<SqlPassword>" \ --value "<password value>"``

**Store SQL Database Credentials in Key Vault:**

``az keyvault secret set \ --vault-name deploymentkeyvault \ --name "<Sqlusername>" \ --value "<username value>"`` 



### Alternative deployment method for automating all Azure resources in the task with Azure Bicep.

```
rovider "azurerm" {
  features {}
  client_id       = var.client_id
  client_secret   = var.client_secret
  tenant_id       = var.tenant_id
  subscription_id = var.subscription_id
}

# Resource Group
resource "azurerm_resource_group" "rg" {
  name     = "my-app-rg"
  location = "East US"
}

# Log Analytics Workspace
resource "azurerm_log_analytics_workspace" "log_workspace" {
  name                = "my-log-analytics-workspace"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = "PerGB2018"
}

# App Service Plan
resource "azurerm_service_plan" "app_service_plan" {
  name                = "my-app-service-plan"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  os_type             = "Linux"
  sku_name            = "B1"
}

# Key Vault
resource "azurerm_key_vault" "kv" {
  name                = "myAppKeyVault"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  tenant_id           = var.tenant_id
  sku_name            = "standard"
}

# Key Vault Secret for Database Password
resource "azurerm_key_vault_secret" "db_password" {
  name         = "db-password"
  value        = var.db_password
  key_vault_id = azurerm_key_vault.kv.id
}

# Frontend App Service
resource "azurerm_linux_web_app" "frontend" {
  name                = "frontend-app"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  service_plan_id     = azurerm_service_plan.app_service_plan.id
  identity {
    type = "SystemAssigned"
  }
  app_settings = {
    WEBSITES_ENABLE_APP_SERVICE_STORAGE = "false"
    WEBSITE_RUN_FROM_PACKAGE            = "https://<frontend-package-url>.zip"
    KEYVAULT_URL                        = azurerm_key_vault.kv.vault_uri
  }
}

# Backend App Service
resource "azurerm_linux_web_app" "backend" {
  name                = "backend-app"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  service_plan_id     = azurerm_service_plan.app_service_plan.id
  identity {
    type = "SystemAssigned"
  }
  app_settings = {
    DATABASE_URL                       = azurerm_mssql_server.sql_server.fully_qualified_domain_name
    DATABASE_USERNAME                  = azurerm_mssql_server.sql_server.administrator_login
    DATABASE_PASSWORD                  = azurerm_key_vault_secret.db_password.value
    WEBSITE_RUN_FROM_PACKAGE           = "https://<backend-package-url>.zip"
  }
}

# SQL Server
resource "azurerm_mssql_server" "sql_server" {
  name                         = "sql-server-myapp"
  resource_group_name          = azurerm_resource_group.rg.name
  location                     = azurerm_resource_group.rg.location
  version                      = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = azurerm_key_vault_secret.db_password.value
}

# SQL Database
resource "azurerm_mssql_database" "sql_database" {
  name                = "myappdb"
  server_id           = azurerm_mssql_server.sql_server.id
  sku_name            = "S1"
  max_size_gb         = 5
}

# Outputs
output "frontend_url" {
  value = azurerm_linux_web_app.frontend.default_hostname
}

output "backend_url" {
  value = azurerm_linux_web_app.backend.default_hostname
}

output "database_connection" {
  value = "Server=${azurerm_mssql_server.sql_server.fully_qualified_domain_name};Database=${azurerm_mssql_database.sql_database.name};User Id=${azurerm_mssql_server.sql_server.administrator_login};Password=${azurerm_key_vault_secret.db_password.value};"
}
```
## Service Principal

``az ad sp create-for-rbac --name "terraform-sp" --role Contributor --scopes /subscriptions/3b22ccd0-4a9a-458a-ba05-4349d7dac8fb
``

``
{
  "appId": "2fa69971-a672-4511-ad88-e938ba2dc338",  
  
  "displayName": "terraform-sp",  
  
  "password": "************",  
  
  "tenant": "******" 
}
``

## Deploy the Bicep file

``az deployment group create \
  --resource-group <resource-group-name> \
  --template-file main.bicep \
  --parameters environment=prod sqlAdminPassword=YourSecurePassword
``

  ### Step 4: Create a Dockerfile for .NET application
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
COPY . .
RUN dotnet restore
RUN dotnet build -c Release -o /app/build

FROM build AS publish
RUN dotnet publish -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish.
ENTRYPOINT ["dotnet"]
```
- Build and push Dockerfile:
  
 ```bash
docker build -t myacr1234.azurecr.io/dotnet-app:latest.
docker push myacr1234.azurecr.io/dotnet-app:latest

```

### Step 5:Set Up CI/CD with GitHub Actions ##

**Add the following secrets to GitHub for each repository:**
- ACR_USERNAME
- ACR_PASSWORD
- DB_USERNAME
- DB_PASSWORD

**Create GitHub Action workflows:**  

- Backend Workflow (.github/workflows/backend.yml):
  
```yaml
name: Backend CI/CD

on:
  push:
    paths:
      - "backend/**"

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Set up JDK
        uses: actions/setup-java@v3
        with:
          java-version: 17

      - name: Build Backend
        run: mvn clean package -DskipTests -f ./backend/pom.xml

      - name: Login to Azure Container Registry
        uses: azure/docker-login@v1
        with:
          login-server: myacr1234.azurecr.io
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}

      - name: Build and Push Docker Image
        run: |
          docker build -t myacr1234.azurecr.io/backend:latest ./backend
          docker push myacr1234.azurecr.io/backend:latest
```



 - Frontend Workflow (.github/workflows/frontend.yml):

``` yaml
name: Frontend CI/CD

on:
  push:
    paths:
      - "frontend/**"

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 16

      - name: Install Dependencies and Build
        run: |
          cd frontend
          npm install
          npm run build

      - name: Login to Azure Container Registry
        uses: azure/docker-login@v1
        with:
          login-server: myacr1234.azurecr.io
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}

      - name: Build and Push Docker Image
        run: |
          docker build -t myacr1234.azurecr.io/frontend:latest ./frontend
          docker push myacr1234.azurecr.io/frontend:latest 

```
 - NET Workflow (.github/workflows/dotnet.yml):

 ```yaml
   name: .NET CI/CD

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Set up .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: 6.0

      - name: Restore, Build, and Publish
        run: |
          dotnet restore
          dotnet build
          dotnet publish -c Release -o ./publish

      - name: Login to Azure Container Registry
        uses: azure/docker-login@v1
        with:
          login-server: myacr1234.azurecr.io
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}

      - name: Build and Push Docker Image
        run: |
          docker build -t myacr1234.azurecr.io/dotnet-app:latest .
          docker push myacr1234.azurecr.io/dotnet-app:latest   
 ```




  


                                                    

  





    



   




  
 



  



      




    
    
    

    
    
   

    

    


  
