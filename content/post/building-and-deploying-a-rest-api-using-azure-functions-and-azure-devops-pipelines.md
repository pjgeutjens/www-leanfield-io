---
timeToRead: 0
authors:
- Pieter Jan Geutjens
title: Building and Deploying a REST API using Azure Functions and Azure DevOps Pipelines.
excerpt: Representational State Transfer (REST) API's are everywhere. A REST API is
  the frontend to a data source, it provides create, retrieve, update and delete access
  to the data items. In a typical 3-tier application it sits between the UI where
  end-users can consult and modify the data, and the database where the data is stored.
date: 2020-04-17T06:00:00+00:00
hero: "/images/azfunction.png"

---
# Introduction

Representational State Transfer (REST) API's are everywhere. A REST API is the frontend to a data source, it provides *create*, *retrieve*, *update* and *delete* access to the data items. In a typical 3-tier application it sits between the UI where end-users can consult and modify the data, and the database where the data is stored.

For Developer teams an API is often a *gateway to the world*. Ever since the 2006 acmqueue interview with Werner Vogels, CTO of Amazon, where he uttered the phrase *"You build it, you run it."*, a movement has taken shape driving development teams to not only build and design their own API's, but to take end-to-end responsibility for them. This includes database design and operations, infrastructure management and monitoring. The teams essentially make the statement *"This is our API, use it and we will make sure it's available to you at all times."* 

With the arrival of public clouds these teams have gained a new tool for their toolbox, allowing them to focus on writing their code, and less on managing infrastructure.

Serverless Computing (or Functions as a Service) is hot these days. It's the top rung of the Iaas-PaaS-SaaS-FaaS ladder of compute and infrastructure, completely eliminating the need for infrastructure management, and allowing total focus on application code. This allows teams to cycle faster in their delivery of value, putting the burden of infrastructure provisioning, management and scaling in the hands of the service providers.

The servers are still there of course, but they are essentially invisible to the developer. The developer only has to deliver his functions, indicate what environment he wants them to run in, and consume their output.

Besides the added value Functions as a Service can provide in the development cycle, there's also a cost-benefit to them. Instead of running a dedicated always-on server to host the API, leveraging Azure Functions allows you to limit your spending to only cover the memory usage and execution time of your code. Add to that the fact that scaling concerns can be completely handed off to Microsoft, they will make sure your API always has enough resources available, and this makes for a very compelling use-case for doing away with the classic infrastructure and moving to Serverless Computing.

## What we'll be Building

We will build a simple Q&A system that holds a number of common questions for users to consult. A question will have a subject and an answer, and may contain a number of hyperlinks with further information.

Users can request a list of the available questions from the REST API. New questions can be added and existing questions can be updated or deleted.

We will base our architecture on the Azure Architecture Center's reference architecture for a Serverless Web Application as described [here](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/serverless/web-app). Our solution will contain a Cosmos DB instance to store question documents and a Function App to allow Create, Read, Update and Delete (CRUD) operations. 

For the sake of simplicity we will omit the use of a Content Delivery Network (CDN) and API Management layer. While they are an integral part of real-life implementations of this type of architecture, implementing them will take us too far from the topic of this blog post today. We will also not be spending time in this article building out a web frontend for our application. Instead we will use Postman. Postman is an API development tool that allows us to quickly and easily send REST requests to our API. It also supports storing these requests in collections for easy access and rapid, automated testing.

## Prerequisites

- an Azure Subscription
- Basic knowledge of C# and the concept of Dependency Injection. For more information on using Dependency Injection in Azure Functions, check out [this](https://docs.microsoft.com/en-us/azure/azure-functions/functions-dotnet-dependency-injection) link.
- basic knowledge of the principles of REST API's, you can get an overview from the [Microsoft docs](https://docs.microsoft.com/en-us/rest/api/azure/).
- a text editor or IDE, we will be using [Visual Studio Code](https://code.visualstudio.com/)
- Postman, for installation instructions check out the [project homepage](https://www.postman.com/)
- access to an Azure Devops organisation. you can start free at https://azure.microsoft.com/en-us/services/devops/
- clone the reference repository for this article from the CloudSkills github page [here](https://github.com/CloudSkills/rest-csQA)

## Provisioning the Infrastructure using ARM Templates

Now, while we *could* start this article with an exhaustive explanation of how to manually provision all the components of our solution in the Azure portal, in the spirit of Agile development and DevOps in general, we will leverage resource group deployments using ARM templates. The Reference Architecture provided a good starting point for the ARM template. As mentioned we have prepared an ARM template for you as a starting point.

```bash
mkdir csQA
cd csQA
git clone https://github.com/CloudSkills/rest-csQA
cd rest-CSQA
```

The ARM templates can be found in the folder `Iac/functionapp`.
Start by editing some of the values in the `azuredeploy.parameters.json` file to make sure the the resources that require globally unique names don't clash

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "appName": {
      "value": "<UNIQUE APP NAME>"
    },
    "storageAccountType": {
      "value": "Standard_LRS"
    },
    "cosmosDatabaseName": {
      "value": "<UNIQUE COSMOS DB INSTANCE NAME>"
    },
    "cosmosDatabaseCollection": {
      "value": "questions"
    },
    "cosmosCollectionPartitionKey": {
      "value": "/partitionKey"
    },
    "cosmosDatabaseSharedThroughput": {
      "value": 400
    },
    "defaultConsistencyLevel": {
      "value": "Session"
    },
    "multipleWriteLocations": {
      "value": false
    },
    "automaticFailover": {
      "value": false
    }
  }
}
```

To deploy it, open a CloudShell instance, upload the `azuredeploy.json` and `azuredeploy.parameters.json`  files from the IaC/functionapp folder and run the following commands in CloudShell.

```bash
mkdir csQA
cp azuredeploy.json ./csQA
cp azuredeploy.parameters.json ./csQA
cd csQA
az group create -n csQA -l westeurope
az deployment group create -n csQA -g csQA --template-file ./azuredeploy.json --parameters ./azuredeploy.parameters.json --no-wait 
```

This deployment sets up the following resources:
- a storage account
- a Cosmos DB account
- a Cosmos DB database
- a Collection in the database called 'questions'
- an App Service to host our Azure Functions
- an Application Insights instance

It takes about 10 minutes to run. Check out the csQA resource group in the Azure Portal once the deployment is completed. You should see something like this (your resource names will differ).

![csQA Resource Group](https://dev-to-uploads.s3.amazonaws.com/i/vasmwk4jvk40saufrlnd.png)

Opening up the details for the Cosmos DB instance you'll find the questions collection was provisioned also.

![Cosmos collection](https://dev-to-uploads.s3.amazonaws.com/i/0r428r0d462usmqflafc.png)

This is about all the infrastructure we'll need to start building our REST API. Let's start by looking at the Azure Functions.

# The Azure Functions App

You can find the code for the questions API in the folder `AzureFunctions`. As we will be using Visual Studio Code in this blog post, we will first prepare the editor to work with Azure Functions.

## Setting up VS Code to work with Azure Functions

There are a few tools we'll need to make developing Azure Functions in VS Code an enjoyable experience. 

The first tool is the `Azure Functions for Visual Studio Code` extension. Navigate to the *Extensions* tab using the sidebar on the left side of the editor and install it.

![Functions Extension](https://dev-to-uploads.s3.amazonaws.com/i/g5z3707ynrly0lo9exbv.png)

A Second extension is the `C#` one. Again look for it and install in Visual Studio code.

![Functions Extension](https://dev-to-uploads.s3.amazonaws.com/i/raf1gi5q6osrbp1p599b.png)


Next there's the Azure Functions Core Tools which allow you to develop and test your functions on your local computer from the command prompt or terminal. You can find more information on how to install the toolset [here](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=windows%2Ccsharp%2Cbash). Make sure you install `version 3` of the Core Tools runtime, as that's what the sample code uses.

Finally if you're using a Windows system you should install the Azure Cosmos Emulator. This tool provides a local instance of the Cosmos DB Service for you to use while developing. For more information on how to install and use this tool check out [this](https://docs.microsoft.com/en-us/azure/cosmos-db/local-emulator) link on the Microsoft Docs site. When you're done installing the emulator, don't forget to add a database called `csQA` with a collection called `questions` like we did when we provisioned our Azure resources. You can access the Emulator's admin page by browsing to https://localhost:8081.

## The Azure Functions for a REST API

It's time now to dive in to the code. Open the `AzureFunctions` folder of the repo. If VS Code prompts you to set up a `.vscode` folder, accept this, it will set up the project for debugging. You'll find the Azure Functions in a subfolder `Functions`.

![tree](https://dev-to-uploads.s3.amazonaws.com/i/hj7uph8xjq7aprk7s60s.png)

While I will not blindly copy-paste all the functions into this post, let's highlight some of the key points.

First of all, this code uses dependency injection. The Azure Functions runtime will register a CosmosClient instance as a service when the function app starts, allowing us to use a single instance of this service throughout our code. You can see this registration in the `Startup.cs` file of the project.

```csharp
    builder.Services.AddSingleton((s) => {
                string endpoint = configuration["COSMOS_DB_DATABASE_URL"];
                if (string.IsNullOrEmpty(endpoint))
                {
                    throw new ArgumentNullException("Please specify a valid endpoint in the local.settings.json file or your Azure Functions Settings.");
                }

                string authKey = configuration["COSMOS_DB_DATABASE_KEY"];
                if (string.IsNullOrEmpty(authKey) || string.Equals(authKey, "Super secret key"))
                {
                    throw new ArgumentException("Please specify a valid AuthorizationKey in the local.settings.json file or your Azure Functions Settings.");
                }

                CosmosClientBuilder configurationBuilder = new CosmosClientBuilder(endpoint, authKey);
                return configurationBuilder
                    .Build();
            });

```

You may notice that this code references some configuration settings for the Cosmos DB database URL and secret key. In order to provide these, create a file called `local.settings.json` in the root of the project and add the following json

```javascript
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "dotnet",
    "COSMOS_DB_DATABASE_URL": "https://localhost:8081/",
    "COSMOS_DB_DATABASE_KEY": "<Key From Cosmos DB Emulator>",
    "Settings:CosmosDbDatabaseName": "csQA",
    "Settings:CosmosDbContainerName": "questions"
  }
}
```

> **_REMARK:_**  If you are not using the Cosmos DB Emulator, replace the *COSMOS_DB_DATABASE_URL* and *COSMOS_DB_DATABASE_KEY* with the values for the Cosmos DB instance you created earlier.

Notice how there's 2 types of settings defined here? The ones in ALL_CAPS are used in the Startup function to initialize the CosmosClient. The ones prefixed by *Settings:* will be used in the actual Azure Functions code to target the correct database and collection. 

Let's have a look at some of the actual REST related functions. Open up the `ListQuestions.cs` file which contains the code for the READ functionality of the API. You can see how the local `_cosmosClient` variable gets initialized in the constructor using dependency injection (this will be a recurring pattern in all the functions). A `FeedIterator` is used to get all the items from the collection.

`GetQuestionById.cs` implements the second type of READ functionality for getting individual documents. In this case we're passing the Id of the question we want to get using the route.

```csharp
public IActionResult Run(
            [HttpTrigger(AuthorizationLevel.Function, "get", Route = "questions/{questionId}")] HttpRequest req, string questionId)
```

 In this function you'll also find an example of using Linq queries with `_container`. This is a different technique for retrieving documents.

```csharp
Question question = _container.GetItemLinqQueryable<Question>(true)
                .Where(q => q.Id == questionId)
                .AsEnumerable().FirstOrDefault();
```

The `CreateQuestion.cs` file contains code for creating a new question document in the database. It assumes the data for the question is passed to our API using the body of an HTTP POST request. The code parses this data into an instance of the `Question` DTO class for storage. Notice how we're using a static PartitionKey of "QUESTION" for the Cosmos DB documents. This is viable in this case since we really have no other property that is a good candidate PartitionKey, and we'll only be storing and querying a single set of documents.

`UpdateQuestion.cs` and `DeleteQuestion.cs` complete the code for our API, giving us all the operations to work with the questions stored in the database. Check them out, it should be clear to you how they do their work based on what we saw in previous functions.

In the next section we will use the Postman tool to test the endpoints of our API.

## Testing the code using Postman

In this section we will test the API with Postman. If you are unsure how to use Postman, check out this 10 minute [video tutorial](https://www.youtube.com/watch?v=t5n07Ybz7yI) on YouTube.

First we have to make sure the API is running. In VS Code go to the Run tab and select *Attach to .NET Functions* at the top. This will launch a debug session for the azure functions core tools. You'll see the core tools launch in the terminal and after some time it shows you the available HTTP endpoints. These endpoints adhere to the principles of REST, combining different HTTP verbs (GET, POST, PUT, DELETE) with different endpoints to represent the CRUD operations we can perform on the data.

![debugging](https://dev-to-uploads.s3.amazonaws.com/i/ho8ciqcnh6lvwdc74l0t.png)

In order to get up and running with Postman without delay, we have provided a collection of requests. You can access this collection using [this](https://www.getpostman.com/collections/3d321121b4250e5602f8) link. Download the .json file to your machine and in Postman use the File -> Import menu to load the collection. The collection will show up in the left sidebar and you're ready to start testing.

Let's start by creating a question. Open the `CreateQuestion` request and notice how it will send a POST request to `http://localhost:7071/api/questions` with a json payload in the body like this:

```json
{
    "title": "Who is Mike Pfeiffer?",
    "answer": "Mike Pfeiffer is a twenty year IT industry veteran, published author, and international conference speaker. He's a former architect for Amazon Web Services and engineer for Microsoft. Today Pfeiffer serves as Chief Technologist for CloudSkills.io, a consulting and training firm specializing in cloud computing.",
    "links": [
        {
            "url": "https://mvp.microsoft.com/en-us/PublicProfile/4033603?fullName=Mike%20Pfeiffer",
            "description": "Mike's MVP Profile"
        }
    ]
}
```

Executing this request you should see a *200* return code from the API with the details of the question document. Try running this request again modifying the different fields in the json to add another question. 

If we now run the `ListQuestions` request in Postman our API will return an array containing the data for the questions you entered in the last step. Try putting a breakpoint in one of the functions in Visual Studio Code by clicking to the left of a line number where you want to track execution and executing the request in Postman that triggers the function you're interested in.

If we now run the `ListQuestions` request in Postman our API will return an array containing the data for the questions you entered in the last step. Try putting a breakpoint in one of the functions in Visual Studio Code by clicking to the left of a line number where you want to track execution and executing the request in Postman that triggers the function you're interested in.

![api response](https://dev-to-uploads.s3.amazonaws.com/i/lwpxyu884o4jmfk7pjkn.png)

Check out the other sample requests in the Postman collection. Use `GetQuestionById` to fetch the document for a specific question. Replace the {id} in the target URL with one of the values in the "id" field of the response you got earlier. `UpdateQuestion` allows you to modify an existing question, `DeleteQuestion` removes a question from the database by Id.

We have now verified that our API is functional and returning data. In the next section we will move beyond local testing and publish our Azure Functions API to the cloud.

# Deploying the Functions App to Azure from Visual Studio Code

Visual Studio code with the Azure Functions extension makes it very easy to get our code running in Azure. All we have to do is press CTRL-SHIFT-P (CMD-SHIFT-P on Mac) and find the `Azure Functions: Deploy to Function App` action. This will launch a wizard where you select your subscription in Azure, and select the target Function App we created using ARM templates earlier. After publishing you should see an option to `Copy Settings` which will allow you to copy the variables from the `local.settings.json` file to the Function App configuration. When this is done, open the Azure Portal and check out the Functions app. You'll see that it contains our Questions REST API. Clicking on the *Configuration* link under Configured Features takes you to the page where you can update the `COSMOS_DB_DATABASE_URL` and `COSMOS_DB_DATABASE_KEY` with the values from the Cosmos DB instance in the resource group.

![Azure Functions Configuration](https://dev-to-uploads.s3.amazonaws.com/i/wkmgrf1s4dsfaluhbou4.png)


# Deploying Using Azure DevOps Pipelines

Let's bring this story to its logical conclusion and take an extra step. In this section, using Azure DevOps, we will set up a pipeline to build our function app, provision the required infrastructure for it in Azure, and deploy the solution.

## Setting up Azure DevOps and creating a Service Connection

Browse to your Azure DevOps organisation page and start by creating a new project. This project will host the files required for the release in an Azure Repo. You can directly fork the repo we provided or create your own local copy and push it to Azure DevOps. If you're looking for some guidance on how to get started with this, check out this excellent [post](https://cloudskills.io/blog/git-azure-devops) by Nicole Stevens on Cloudskills.

Once the repo is ready, navigate to the *Project Settings* page via the link on the bottom of the sidebar and click the *Service Connections** link. This is where you set up the connection between your Azure DevOps project and your Azure Subscription, allowing pipelines to create resources in Azure. Create a new Service connection using the button on the top right of the page. Pick *Azure Resource Manager* as the connection type and use the recommended *Service principal (automatic)* authentication method.

In the next screen choose a Scope Level of subscription and select your subscription from the dropdown box. Make sure the *grant access permissions to all pipelines* option is enabled and click save. Make a note of the Service Connection name as we will need it in our pipeline definition later.

> **_REMARK:_**  Creating a service connection this way will give your azure pipelines Contributor access to all resources in your subscription. In real-life scenarios you'll want to limit the scope of the connection to a specific pre-provisioned resource group, adhering to the principle of least privilege. If you want even more granular access control you can manually create a service principal and assign the required access to it or use Managed Identities.

## Getting ARM deployment outputs into Azure Pipeline variables

Our goal for the Azure DevOps Pipeline is to build the Functions App and deploy it into Azure. In order to provision the resources we will use the ARM templates from the previous sections. This leaves us with one challenge though: how do we get the output variables of the ARM provisioning into pipeline variables for subsequent deployment steps?

Luckily Adam Bertram recently wrote a [blog post](https://adamtheautomator.com/arm-output-variables-in-azure-pipelines-powershell/) covering exactly this topic. It talks about adding a Powershell script to a release pipeline that will parse the output variable of an ARM deployment task into pipeline variables. You'll find the script from the blog post in the `Scripts` folder of the project repo.

## The Azure DevOps Pipeline

Next, move on to the Pipelines page for your Azure DevOps project. Click on the button to create a new pipeline and when asked where your code is select *Azure Repos Git [YAML]*

![code location](https://dev-to-uploads.s3.amazonaws.com/i/s6h6gtscyc7ngc9uyuaz.png)

On the next screen, select your project repository and choose to start from an existing Azure Pipelines YAML file.

![existing yaml](https://dev-to-uploads.s3.amazonaws.com/i/boe342uorr5mwe1ob3e2.png)

The root folder of the repo for this project contains an `azure-pipelines.yml`. Open it up in Visual Studio code and you'll see the file describes a 2-stage pipeline with a Build and a Deploy phase.

The Build phase will
- build the Functions App using the `dotnet` cli.
- compress the build files into a .zip archive.
- copy the ARM templates for provisioning our infrastructure.
- copy the script to deal with ARM deployment outputs.
- publish the build, the ARM template files and the outputs script as Build Artifacts.

The Deploy phase
- executes the ARM deployment of our resources, creating what we need to host the Azure Function.
- runs the script to populate pipeline variables from the ARM deployment output.
- publishes our Azure Functions App to the runtime created in the previous step.

The pipeline definition uses these variables
- `vmImageName`: the name of the build host image to use.
- `azureSubscription`: enter the name of the service connection we created earlier between single quotes
- `resourceGroupName`: the name of the target resource group for infrastructure deployment
- `resourceGroupLocation`: resource group location
- `questionsFunctionAppName`: a placeholder variable for the Azure Functions app name

Look through the pipeline definition. The 2 stages and the different build steps for each stage should be easily spotted. Notice the 3 instances of `PublishBuildArtifacts@1` tasks publishing our function app build, the ARM templates and the variables powershell script.

```yaml
# azure-pipelines.yml build stage (truncated for brevity)

trigger:
- master

variables:
  vmImageName: 'vs2017-win2016'
  azureSubscription: '<your service connection name>'
  resourceGroupName: 'csQADevOps'
  resourceGroupLocation: 'West Europe'
  questionsFunctionAppName: ''

stages:
- stage: Build
  displayName: Build stage

  jobs:
  - job: Build
    displayName: Build
    pool:
      vmImage: $(vmImageName)

    steps:
    ...

    - task: PublishBuildArtifacts@1
      displayName: 'Publish Artifact: src'
      inputs:
        PathtoPublish: '$(Build.ArtifactStagingDirectory)/src'
        ArtifactName: src

    - task: PublishBuildArtifacts@1
      displayName: 'Publish Artifact: arm'
      inputs:
        PathtoPublish: '$(Build.ArtifactStagingDirectory)/arm'
        ArtifactName: arm

    - task: PublishBuildArtifacts@1
      displayName: 'Publish Artifact: scripts'
      inputs:
        PathtoPublish: '$(Build.ArtifactStagingDirectory)/scripts'
        ArtifactName: scripts

    ...
```

In the Deploy stage we call the powershell script to set up varables based on ARM deployment output and the `AzureFunctionApp@1` task using these variables to deploy our code into Azure Functions.

```yaml
# Deployment stage (truncated)
- stage: Deploy
  displayName: Deploy stage
  dependsOn: Build
  condition: succeeded()

  jobs:
  - deployment: Deploy
    displayName: Deploy
    environment: 'development'
    pool:
      vmImage: $(vmImageName)

    ...
    
          - task: PowerShell@2
            displayName: 'Parse ARM Deployment outputs'
            inputs:
              targetType: filePath
              filePath: '$(Pipeline.Workspace)/scripts/parse_arm_deployment_output.ps1'
              arguments: '-ArmOutputString ''$(deploymentOutputs)'' -ErrorAction Stop'
          - task: AzureFunctionApp@1
            displayName: 'Azure Function App Deploy: $(questionsFunctionAppName)'
            inputs:
              azureSubscription: '$(azureSubscription)'
              appType: functionApp
              appName: '$(questionsFunctionAppName)'
              package: '$(Pipeline.Workspace)/src/**/*.zip'
              deploymentMethod: zipDeploy
```

Once you've modified the pipeline file to include your Service Connection, try running the pipeline and after about 5-10 minutes you should have a functional REST API running in Azure again, all with a single click of a button!

![pipeline success](https://dev-to-uploads.s3.amazonaws.com/i/u3hpkrbr6tkl0sdk4gqq.png)

# Conclusion

There we have it. A fully automated deployment of a Q&A REST API into Azure using Azure Functions and an Azure DevOps pipeline for automated deployment. Open up the target Resource Group for your pipeline deployment and you'll see everything neatly provisioned and waiting to be tested.

![pipeline resource group](https://dev-to-uploads.s3.amazonaws.com/i/slwgsn5ej37hx20jinwz.jpg)

As mentioned at the start of this post, we could further optimise this architecture, including things like Azure API Management, image uploads to Blob storage, CDNs for faster access to resources. This will be the stuff for future blog posts.

Hope to see you there!

