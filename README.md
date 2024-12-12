# Deploy-Java-and-.NET-Apps-to-Azure-with-GitHub-Actions
Deploy a Java Spring Boot React app and a .NET app to Azure App Service using GitHub Actions for CI/CD. Containerize apps with Docker, store images in Azure Container Registry, and connect the backend to Azure SQL Database. Create separate pipelines for the backend, frontend, and the .NET app.  
### Step 1: Clone the Provided Repositories  
- For the Spring Boot React application:

``git clone https://gitlab.com/cloud-devops-assignments/spring-boot-react-example.git
 ``
- For the .NET application:
  
``git clone https://gitlab.com/blue-harvest-assignments/cloud-assignment.git
  ``
### Step 2: Push the repositories to your GitHub account:
- Create two separate GitHub repositories:

  ``spring-boot-react-example``
  
  ``dotnet-app-example``
- Push the cloned repositories to their respective GitHub repositories:

  - Spring Boot React Application:
    
    ``cd spring-boot-react-example``
    
    ``git remote add origin https://github.com/<your-username>/spring-boot-react-example.git``
    
    ``git branch -M main``
     
    ``git push -u origin main``

   - .NET Application:

    ``cd cloud-assignment``
   
    ``git remote add origin https://github.com/<your-username>/dotnet-app-example.git``

    ``git branch -M main``
    ``git push -u origin main``
  



### Step 3: Configure Azure Resources

- Create a Resource Group for the Project:

  ``az group create --name app-service-deployment --location eastus``

- Create Azure Container Registry (ACR):

  ``az acr create --resource-group app-service-deployment --name <your-acr-name> --sku Basic``
  
- Enable Admin Access on ACR:
  
  ``az acr update --name <your-acr-name> --admin-enabled true``
  

- Set Up Azure App Service Plans:
  - Backend: backend-plan
  -  Frontend: frontend-plan
  -  .NET App: dotnet-plan
 
- Use the following Azure CLI commands:
  
    ``az appservice plan create --name backend-plan --resource-group app-service-deployment --sku B1``
  
  ``az appservice plan create --name frontend-plan --resource-group app-service-deployment --sku B1``
  
  ``az appservice plan create --name dotnet-plan --resource-group app-service-deployment --sku B1``

- **Set Up Azure App Services:** Create App Services for all components:
  - Spring Boot Backend:
    
    ``az webapp create --resource-group app-service-deployment --plan backend-plan --name backend-app --deployment-container-image-name <your-acr- 
   name>.azurecr.io/backend:latest``

  - React Frontend:

    ``az webapp create --resource-group app-service-deployment --plan frontend-plan --name frontend-app --deployment-container-image-name <your-acr- 
  name>.azurecr.io/frontend:latest``

  - .NET Application:
    
     ``az webapp create --resource-group app-service-deployment --plan dotnet-plan --name dotnet-app --deployment-container-image-name <your-acr- 
   name>.azurecr.io/dotnet-app:latest``

- **Set Up Azure SQL Database:**
  - Create the SQL Server and Database:

    ``az sql server create --name <server-name> --resource-group app-service-deployment --admin-user <username> --admin-password <password>``
    
     ``az sql db create --name <database-name> --server <server-name> --resource-group app-service-deployment --service-objective S0``
    
- **Allow Azure App Services to Connect to SQL Server:**

   ``az sql server firewall-rule create --resource-group app-service-deployment --server <server-name> --name AllowAzureServices --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0``

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
docker build -t <your-acr-name>.azurecr.io/dotnet-app:latest.
docker push <your-acr-name>.azurecr.io/dotnet-app:latest

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
          docker build -t <your-acr-name>.azurecr.io/backend:latest ./backend
          docker push <your-acr-name>.azurecr.io/backend:latest
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
          docker build -t <your-acr-name>.azurecr.io/frontend:latest ./frontend
          docker push <your-acr-name>.azurecr.io/frontend:latest ```

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
          docker build -t <your-acr-name>.azurecr.io/dotnet-app:latest .
          docker push <your-acr-name>.azurecr.io/dotnet-app:latest   
 ```




  


                                                    

  





    



   




  
 



  



      




    
    
    

    
    
   

    

    


  
