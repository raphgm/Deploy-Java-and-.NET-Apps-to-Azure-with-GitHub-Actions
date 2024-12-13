# Deploy-Java-and-.NET-Apps-to-Azure-with-GitHub-Actions
Deploy a Java Spring Boot React app and a .NET app to Azure App Service using GitHub Actions for CI/CD. Containerize apps with Docker, store images in Azure Container Registry, and connect the backend to Azure SQL Database. Create separate pipelines for the backend, frontend, and the .NET app.  
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

  Create a Key Vault to Store all Sensitive keys for the deployment
az keyvault create --name deploymentkeyvault --resource-group app-service-deployment --location northeurope

Store SQL Server Credentials in Key Vault:

az keyvault secret set \ --vault-name deploymentkeyvault \ --name "S<qlUsername>" \ --value "<username value'" 

Store SQL Password:

az keyvault secret set \ --vault-name deploymentkeyvault \ --name "<SqlPassword>" \ --value "<password value>"

Store SQL Database Credentials in Key Vault:

az keyvault secret set \ --vault-name deploymentkeyvault \ --name "<Sqlusername>" \ --value "<username value>" 



### Alternative deployment method for automating all Azure resources in the task with Azure Bicep.
```yaml
// Parameters
param location string = 'northeurope'
param resourceGroupName string = 'app-service-deployment'
param appServicePlanSku string = 'B1'
param backendAppName string = 'java-backend-app'
param frontendAppName string = 'react-frontend-app'
param dotnetAppName string = 'dotnet-web-app'
param sqlServerName string = 'sqlserver1234'
param sqlDatabaseName string = 'appdb'
param acrName string = 'myacr1234'
param keyVaultName string = 'deploymentkeyvault'
param logAnalyticsWorkspaceName string = 'app-monitoring-law'
param applicationInsightsName string = 'app-insights'
param monitoringSolutionName string = 'MonitoringSolution'
param solutionType string = 'ContainerInsights'
@secure()
param sqlAdminPassword string
@secure()
param sqlAdminUsername string

// Resource Group
resource resourceGroup 'Microsoft.Resources/resourceGroups@2021-04-01' = {
  name: resourceGroupName
  location: location
}

// Azure Key Vault
resource keyVault 'Microsoft.KeyVault/vaults@2023-02-01' = {
  name: keyVaultName
  location: location
  properties: {
    tenantId: subscription().tenantId
    sku: {
      family: 'A'
      name: 'standard'
    }
    accessPolicies: [
      {
        tenantId: subscription().tenantId
        objectId: subscription().subscriptionId
        permissions: {
          secrets: [ 'get', 'set' ]
        }
      }
    ]
  }
}

// Store SQL credentials in Key Vault
resource sqlAdminUsernameSecret 'Microsoft.KeyVault/vaults/secrets@2021-06-01-preview' = {
  name: '${keyVaultName}/sqlAdminUsername'
  properties: {
    value: sqlAdminUsername
  }
  dependsOn: [keyVault]
}

resource sqlAdminPasswordSecret 'Microsoft.KeyVault/vaults/secrets@2021-06-01-preview' = {
  name: '${keyVaultName}/sqlAdminPassword'
  properties: {
    value: sqlAdminPassword
  }
  dependsOn: [keyVault]
}

// Azure Container Registry
resource acr 'Microsoft.ContainerRegistry/registries@2023-01-01-preview' = {
  name: acrName
  location: location
  sku: {
    name: 'Basic'
  }
  properties: {
    adminUserEnabled: true
  }
}

// App Service Plan
resource appServicePlan 'Microsoft.Web/serverfarms@2022-09-01' = {
  name: 'appServicePlan'
  location: location
  sku: {
    name: appServicePlanSku
    tier: 'Basic'
  }
}

// Backend App Service
resource backendApp 'Microsoft.Web/sites@2022-09-01' = {
  name: backendAppName
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      appSettings: [
        {
          name: 'DOCKER_REGISTRY_SERVER_URL'
          value: 'https://${acrName}.azurecr.io'
        }
        {
          name: 'DOCKER_REGISTRY_SERVER_USERNAME'
          value: acr.properties.adminUserEnabled ? acr.listCredentials().username : ''
        }
        {
          name: 'DOCKER_REGISTRY_SERVER_PASSWORD'
          value: acr.properties.adminUserEnabled ? acr.listCredentials().passwords[0].value : ''
        }
        {
          name: 'DATABASE_URL'
          value: 'jdbc:sqlserver://${sqlServerName}.database.windows.net:1433;database=${sqlDatabaseName}'
        }
        {
          name: 'DATABASE_USERNAME'
          value: '@Microsoft.KeyVault(SecretUri=https://${keyVaultName}.vault.azure.net/secrets/sqlAdminUsername)'
        }
        {
          name: 'DATABASE_PASSWORD'
          value: '@Microsoft.KeyVault(SecretUri=https://${keyVaultName}.vault.azure.net/secrets/sqlAdminPassword)'
        }
      ]
      linuxFxVersion: 'DOCKER|${acrName}.azurecr.io/backend:latest'
    }
  }
}

// Frontend App Service
resource frontendApp 'Microsoft.Web/sites@2022-09-01' = {
  name: frontendAppName
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      appSettings: [
        {
          name: 'DOCKER_REGISTRY_SERVER_URL'
          value: 'https://${acrName}.azurecr.io'
        }
        {
          name: 'DOCKER_REGISTRY_SERVER_USERNAME'
          value: acr.properties.adminUserEnabled ? acr.listCredentials().username : ''
        }
        {
          name: 'DOCKER_REGISTRY_SERVER_PASSWORD'
          value: acr.properties.adminUserEnabled ? acr.listCredentials().passwords[0].value : ''
        }
      ]
      linuxFxVersion: 'DOCKER|${acrName}.azurecr.io/frontend:latest'
    }
  }
}

// .NET App Service
resource dotnetApp 'Microsoft.Web/sites@2022-09-01' = {
  name: dotnetAppName
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      appSettings: [
        {
          name: 'DOCKER_REGISTRY_SERVER_URL'
          value: 'https://${acrName}.azurecr.io'
        }
        {
          name: 'DOCKER_REGISTRY_SERVER_USERNAME'
          value: acr.properties.adminUserEnabled ? acr.listCredentials().username : ''
        }
        {
          name: 'DOCKER_REGISTRY_SERVER_PASSWORD'
          value: acr.properties.adminUserEnabled ? acr.listCredentials().passwords[0].value : ''
        }
      ]
      linuxFxVersion: 'DOCKER|${acrName}.azurecr.io/dotnet-app:latest'
    }
  }
}

// SQL Server
resource sqlServer 'Microsoft.Sql/servers@2022-02-01-preview' = {
  name: sqlServerName
  location: location
  properties: {
    administratorLogin: sqlAdminUsername
    administratorLoginPassword: sqlAdminPassword
  }
}

// SQL Database
resource sqlDatabase 'Microsoft.Sql/servers/databases@2022-02-01-preview' = {
  name: sqlDatabaseName
  parent: sqlServer
  properties: {
    edition: 'Basic'
    maxSizeBytes: 2147483648
  }
}

// Firewall Rule for SQL Server
resource sqlFirewallRule 'Microsoft.Sql/servers/firewallRules@2022-02-01-preview' = {
  name: 'AllowAzureServices'
  parent: sqlServer
  properties: {
    startIpAddress: '0.0.0.0'
    endIpAddress: '0.0.0.0'
  }
}

// Azure Monitor Log Analytics Workspace
resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2021-06-01' = {
  name: logAnalyticsWorkspaceName
  location: location
  properties: {
    sku: {
      name: 'PerGB2018'
    }
  }
}

// Application Insights
resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: applicationInsightsName
  location: location
  properties: {
    Application_Type: 'web'
    WorkspaceResourceId: logAnalytics.id
  }
}

// Monitoring Solution
resource monitoringSolution 'Microsoft.OperationsManagement/solutions@2015-11-01-preview' = {
  name: '${monitoringSolutionName}(workspace-${logAnalytics.name})'
  location: location
  properties: {
    workspaceResourceId: logAnalytics.id
    solutionType: solutionType
  }
  plan: {
    name: monitoringSolutionName
    product: 'OMSGallery/ContainerInsights'
    promotionCode: ''
    publisher: 'Microsoft'
  }
}


```
### Create a Key Vault to Store all Sensitive keys for the deployment

``az keyvault create --name deploymentkeyvault --resource-group app-service-deployment --location northeurope``  

- **Store SQL Server Credentials in Key Vault:**
  
  ``az keyvault secret set \
  --vault-name deploymentkeyvault \
  --name "SqlUsername" \
  --value "sqlserver1234'"
  ``
  
- **Store SQL Password:**
  
  ``
  az keyvault secret set \
  --vault-name deploymentkeyvault \
  --name "SqlPassword" \
  --value "P@ssw0rd1234"
  ``
  
- **Store SQL Database  Credentials in Key Vault:**
    
    ``az keyvault secret set \
  --vault-name deploymentkeyvault \
  --name "SqladminUsername" \
  --value "sqladminuser"
  ``

### Create a Service principal that will be used for the deployment  
<!-- Create a service principal with the name dsvp and scope it to the app-service-deployment resource group -->



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
ENTRYPOINT ["dotnet", "YourApp.dll"]
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
          login-server: <your-acr-name>.azurecr.io
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
          login-server: <your-acr-name>.azurecr.io
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
          login-server: <your-acr-name>.azurecr.io
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}

      - name: Build and Push Docker Image
        run: |
          docker build -t myacr1234.azurecr.io/dotnet-app:latest .
          docker push myacr1234.azurecr.io/dotnet-app:latest   
 ```




  


                                                    

  





    



   




  
 



  



      




    
    
    

    
    
   

    

    


  
