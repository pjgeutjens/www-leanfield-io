---
timeToRead: 0
authors: []
title: Mastering Azure through Cloud Katas
excerpt: Through executing a series of short exercises or *katas* revolving around
  VM creation, monitoring and ARM templating, our goal is for the reader to become
  increasingly familiar with and skilled using the Azure Cloud. Meanwhile we track
  the execution time of the tasks to measure progress towards mastery.
date: 2020-03-06T23:00:00+00:00
hero: "/images/kata.png"

---
## Introduction <a name="introduction"></a>

In his best-selling work on habit creation *Atomic Habits*, author James Clear states at one point

> You don't want to merely be planning, you want to be practicing. If you want to master a habit, the key is to start with repetition, not perfection. You don't need to map out every feature of a new habit. You just need to practice it.

If you manage to work the practice of a task into a daily routine, you will get better and better at it over time, to the point even of allowing you to execute the tasks with hardly any conscious thought. It has become *second nature* to you.

There is one major problem with endless repetition though. There's no two ways about it, repeating some identical task day in day out will get.. well.. *boring* after some time. While it may be true that the ability to persevere with the boring stuff can be a great help in mastering a skill or field, there may be a better approach.

Research on teaching methodologies has long searched for the optimal way to teach skills to humans and non-humans alike. Results show that one of the determining factors in learning speed and motivation is the difficulty of the task, or more specifically the level of challenge the student experiences while training. This has led to the introduction of the term 'Goldilocks zone' of task difficulty for optimal motivation and learning.<sup>**1, 2**</sup> The idea is to continuously move along a learning path towards mastering a field where each task is right at the edge of the trainee's current ability. Not too easy or too hard but just right. Combining this approach with an accurate measure of progress tends to lead to the best results.

In this blog post we will apply these insights to learning about and getting hands-on with the Azure Cloud. Through executing a series of short exercises or *katas*<sup>**3**</sup> revolving around VM creation, monitoring and ARM templating, our goal is for the reader to become increasingly familiar with and skilled using the Azure Cloud. Meanwhile we track the execution time of the tasks to measure progress towards mastery.


## Before you Start <a name="before-you-start"></a>

Some initial notes before starting the exercises


- You will need an Azure Subscription to execute the katas.
- At the start of each kata you will find an indication of the total time to run the exercise and the distribution of time spent in the portal, PowerShell and a bash shell. The latter also includes time running commands using the *azure cli*.
- Try timing your runs to collect your own results as you repeat the katas, you'll get a good view on your progress.
- The first three katas produce the same results but show different activity distributions. We do not however want to favor one method or tool over another, as this choice is most often determined by previous experience and existing environments.
- If you're planning on taking any of the Microsoft Associate Level exams, these katas are an excellent preparation for familiarising yourself with some of the required skills that will be tested!


> **_REMARK:_**  When you finish a set of katas, don't forget to remove the resource groups you created! The resources we create do represent a real cost to your Azure Subscription.

## Kata Datasheet <a name="datasheet"></a>

In the following paragraphs we will present five starter katas. The table below shows an overview to give you an idea of the topics, time requirements and tools used for each exercise. The timings are only an indication to use as a baseline. If you have a limited time to run kata exercises, they may help you select which one(s) you want to practice. Your results may vary depending on typing speed, familiarity with the tools, and repetition.

| # | kata title                                  |   duration    | uses   |
|---|---------------------------------------------|---------------| -------|
| 1 | Provisioning a Linux VM with nginx in the Azure Portal | 8 -12 minutes | portal, bash
| 2 | Provisioning a Linux VM with nginx using Azure CLI     | 6 - 8 minutes | bash, az-cli
| 3 | Provisioning a Linux VM with nginx using PowerShell    | 10 - 12 minutes | bash, PowerShell
| 4 | Setting up an Azure Monitor Metrics Alert using Azure CLI | 12 - 15 minutes | bash, az-cli
| 5 | Building and Deploying an ARM template using VS Code | 12 - 15 minutes | vscode, PowerShell

## Kata 1: Provisioning a Linux VM in the Azure Portal <a name="kata-1"></a>

total time: 8-12 minutes

distribution:

![distribution kata 1](https://dev-to-uploads.s3.amazonaws.com/i/nec1w42ieae4b1nniidk.png)

The goal of this exercise is to execute the Azure Quickstart exercise you can find at https://docs.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-portal

You will provision a Linux VM, open up network access to port 80, and install the nginx web server to host a static website. Our results indicated a roughly equal amount of time spent in the portal, the bash shell, and waiting for the resources to provision.

Continue running through this exercise until you feel like you've reached a basic level of comfort. Your execution time should also reflect this.

## Katas 2 and 3: Provisioning a Linux VM using the Azure CLI or Powershell <a name="kata-2-3"></a>

total time: 6-8 minutes for CLI, 10-12 minutes for PowerShell

distribution - CLI:

![distribution kata 1](https://dev-to-uploads.s3.amazonaws.com/i/a32bhzhy3n5uhbezc0fd.png)

distribution - PowerShell:

![distribution kata 1](https://dev-to-uploads.s3.amazonaws.com/i/n4np3fmbcoitbzw1sr18.png)

Like kata 1, this exercise will have you provision a Linux VM running an nginx web server. This time however you will achieve this using either Azure CLI commands or PowerShell cmdlets from the Azure Cloud Shell.

You can get an instance of the Cloud Shell by pointing your browser to https://shell.azure.com or by opening a shell from the Azure Portal

For this kata, have a go at one of these two Quickstart exercises
* https://docs.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-cli (azure cli) 
* https://docs.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-powershell (PowerShell)

A quick tip for the PowerShell version: you can run VS Code either on your machine or in the cloud shell to quickly edit the copied code blocks. Try running the different commands one by one in the PowerShell Cloud Shell so you always have the required variables available for the next steps.

Notice how both exercises produce the same end result through different procedures. Try them both and see how you improve through repetition. Does one feel more familiar to you than the other, are you progressing quicker? We also noticed that the actual provisioning times of the resources were significantly lower here than in kata 1 where we used the Portal.

## Kata 4 - Setting up an Azure Monitor Metrics Alert using Azure CLI <a name="kata-4"></a>

total time: 12-15 minutes

distribution:

![distribution kata 4](https://dev-to-uploads.s3.amazonaws.com/i/o1m72is6nw6o00pzgc3a.png)

In our fourth kata we will have a look at Azure Monitor. Collection of metrics and generating alerts based on their values are two core functionalities of the Azure platform. Knowing your way around the Azure Monitoring features is also a key skill when preparing for the various Azure certifications.

In this exercise you will use the Azure CLI to create (and test) a metric alert rule on an Azure Virtual Machine that will trigger when the machine's average CPU load exceeds 85%. In order to complete this kata you should have a running Linux VM available in your subscription. You could run through one of the previous katas to quickly create one.

Log in to the Cloud Shell from either the Azure Portal or from https://shell.azure.com. You can use either a bash or PowerShell instance, as the az executable is available in both.

You can start the exercise by executing the `--help` option of the `az monitor` command to list available commands for interacting with metric alerts.

``` bash
az monitor metrics alert --help
```

Make a note of the target VM's resource ID. You can quickly get a list of Ids for a resource group by running this command

```bash
az vm list --query "[0:1].id" -g <group name> -o tsv
```

Our goal is to send out an email when the alert is triggered. In order to set this up you'll first create an action group defining the target email address.

```bash
az monitor action-group create --action email <receiver name> <receiver email> --name EmailActionGroup --resource-group <group name>
```

Next, create a simple metrics alert that will trigger when the average CPU load of the VM exceeds 85% for 5 minutes.
```bash
az monitor metrics alert create --name csKataCPU85 -g csKata1 --scopes <VM resource ID> --condition "avg Percentage CPU > 85" --description "avg CPU > 85%"
```

And finally update the Metric Alert and add the action to it.

```bash
az monitor metrics alert update --resource-group csKata1 --name csKataCPU85 --add-action EmailActionGroup
``` 

### Testing the Alert Rule <a name="kata-4-testing"></a>

In order to test the alert you will install the `stress` tool on your Linux VM. Connect to the machine via SSH.

```bash
ssh azureuser@<ip address>
```

Install the stress tool.

```bash
sudo apt-get -y install stress
```
And start a 10 minute test in the background.

```bash
stress --cpu 2 --timeout 600 &
```
You can monitor the test using the `htop` command which will show you CPU load as well as the runtime for the *stress* processes. After a couple of minutes you should receive the alert mail from Azure

![Alert Email](https://dev-to-uploads.s3.amazonaws.com/i/wk9gopyjrppq2cgyljty.png)

## Kata 5 - Building an ARM template for an Azure Storage Account in VS Code <a name="kata-5"></a>

preparation (one-time): 5 - 10 minutes

total time: 10 - 15 minutes

distribution:

![distribution kata 5](https://dev-to-uploads.s3.amazonaws.com/i/3rlndkcf4ep5j2v62m02.png)


In the final kata of this first series you will set up Visual Studio code to work with Azure Resource Manager (ARM) templates, and use it to run through an exercise building a template for Storage Account, 

### Preparation: Setting up Visual Studio Code to author ARM templates <a name="kata-5-preparation"></a>

If you don't have Visual Studio Code installed yet, head over to [https://code.visualstudio.com/](https://code.visualstudio.com/) and install the version for your operating system. Launch the editor and head over to the *Extensions* tab on the left sidebar. We highly recommend installing the following list of extensions for working with ARM templates:

- Azure Resource Manager (ARM) Tools
- Azure Resource Manager Snippets
- ARM Params Generator
- ARM Template Viewer

Once these are installed create a working folder for your kata, and create a file named `azuredeploy.json`

### Authoring the ARM template <a name="kata-5-arm"></a>

Open the `azuredeploy.json` file and type the word `arm`. This should trigger the ARM Snippets extension and show you a list of available templates. Select the empty *resource group template* which should give you the following starting point

```javascript
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {},
    "variables": {},
    "resources": [],
    "outputs": {}
}
```
Next, move your cursor into the `"resources"` array and, once again typing `arm`, select the `arm-storage` template. this will result in the following `azuredeploy.json` file contents

```javascript
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {},
    "variables": {},
    "resources": [
        {
            "name": "storageaccount1",
            "type": "Microsoft.Storage/storageAccounts",
            "apiVersion": "2015-06-15",
            "location": "[resourceGroup().location]",
            "tags": {
                "displayName": "storageaccount1"
            },
            "properties": {
                "accountType": "Standard_LRS"
            }
        }
    ],
    "outputs": {},
    "functions": []
}
```
While the injected snippet is perfectly valid, at the time of this writing the Visual Studio Code extension uses an API version that is almost 5 years out of date. Therefore, replace the code with the following changed resource definition

```javascript
 "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {},
    "variables": {},
    "resources": [
        {
            "type": "Microsoft.Storage/storageAccounts",
            "apiVersion": "2019-04-01",
            "name": "storageAccount1",
            "location": "[resourceGroup().location]",
            "sku": {
                "name": "Standard_LRS"
            },
            "kind": "StorageV2",
            "properties": {
            }
        }
    ],
    "outputs": {},
    "functions": []
```

As the idea behind using an ARM template is to have a re-usable artifact for multiple deployments, you will now introduce 2 parameters to hold the per-deployment values of the storage account name and redundancy tier or sku.

To do this move your cursor to the `"parameters"` section of the template and, again using the snippets extension, introduce 2 parameters like this

```javascript
"parameters": {
    "storageName": {
        "type": "string",
        "metadata": {
            "description": "Storage Account Name"
        }
    },
    "storageSKU": {
        "type": "string",
        "defaultValue": "Standard_LRS",
        "allowedValues": [
            "Standard_LRS",
            "Standard_GRS",
            "Standard_RAGRS",
            "Standard_ZRS",
            "Premium_LRS",
            "Premium_ZRS",
            "Standard_GZRS",
            "Standard_RAGZRS"
        ]
    }
}
```

To finish off we incorporate the parameters in the template, you'll also add an output object specifying we want the storage account's endpoints listed upon completion of the deployment. This can once again be done through the ARM snippets extension. Move the cursor into the `"outputs"` object and add and tweak an `arm-output` snippet. This leaves you with the following final contents of `azuredeploy.json`

```javascript
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "storageName": {
            "type": "string",
            "metadata": {
                "description": "Storage Account Name"
            }
        },
        "storageSKU": {
            "type": "string",
            "defaultValue": "Standard_LRS",
            "allowedValues": [
                "Standard_LRS",
                "Standard_GRS",
                "Standard_RAGRS",
                "Standard_ZRS",
                "Premium_LRS",
                "Premium_ZRS",
                "Standard_GZRS",
                "Standard_RAGZRS"
            ]
        }
    },
    "variables": {},
    "resources": [
        {
            "type": "Microsoft.Storage/storageAccounts",
            "apiVersion": "2019-04-01",
            "name": "[parameters('storageName')]",
            "location": "[resourceGroup().location]",
            "sku": {
                "name": "[parameters('storageSKU')]"
            },
            "kind": "StorageV2",
            "properties": {
            }
        }
    ],
    "outputs": {
        "storageEndpoint": {
            "type": "object",
            "value": "[reference(parameters('storageName')).primaryEndpoints]"
        }
    },
    "functions": [
    ]
}
```

### Deploying the ARM Template to a Resource Group <a name="kata-5-deploying"></a>


Before deploying the template to Azure, let's first leverage the *ARM Params Generator* extension to extract the parameters file for the deployment. Launch the command palette in Visual Studio Code by pressing Ctrl-Shift-P (Cmd-Shift-P on a Mac) and select `Azure ARM: Generate parameters file`. 

![generate parameters](https://dev-to-uploads.s3.amazonaws.com/i/bcq0qmmpgndpdcxfolma.png)

This will create a new `azuredeploy.parameters.json` file. Open it and edit as follows

```javascript
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "storageName": {
            "value": "cskata20200225"
        },
        "storageSKU": {
            "value": "Standard_LRS"
        }
    }
}
```

Next, open up an Azure Cloud Shell and select PowerShell as the environment. Once it's running cd into your working directory.

```bash
cd /home/pieter
```

Use the upload/download file button at the top of the terminal to upload the `azuredeploy.json` and `azuredeploy.parameters.json` files your created locally.

Finally deploying the template is a two-step process. First you'll define some variables and create the target resource group, and finally you launch the deployment.

```powershell
$rgName = "csKata5"
$location = "westeurope"
$template = "./azuredeploy.json"
$params = "./azuredeploy.parameters.json"

New-AzResourceGroup -Location $location -Name $rgName

New-AzResourceGroupDeployment -Name "csKata5StorageAccount" -ResourceGroupName $rgName -TemplateFile $template -TemplateParameterFile $params
```
Just a short while later the deployment should complete and you get output listing all the URL's where your storage account lives.

![deployment success](https://dev-to-uploads.s3.amazonaws.com/i/h1kvbexjb41yzkiksepl.png)

> **_NOTE:_**  While this first kata exercise in authoring and deploying templates was relatively basic, the possibilities for extending this kata are endless. Using more complex resource combinations and/or more features of ARM template syntax, you can make this kata as short or as long, as easy or as complex as you like.

## Conclusion <a name="conclusion"></a>

Hopefully running through this first set of katas has convinced you that repeating these kinds of small exercises until the become familiar and you get them into you *muscle memory* is a great way to hone your cloud skills. Especially with the inclusion these days of hands-on labs in many of the Microsoft Azure Certification exams, diving in and getting your hands dirty is key in preparing to take the tests.

If you enjoyed this way of familiarising yourself with the Azure Cloud environment, its tools and practices, be sure to expand on it! Think of new scenario's that will support your study efforts, try them out, and repeat to the point of mastery.

## Footnotes <a name="footnotes"></a>

1: https://www.nature.com/articles/s41467-019-12552-4 "The Eighty Five Percent Rule for optimal learning"

2: https://jamesclear.com/goldilocks-rule "Goldilocks rule" from *Atomic Habits*

3: Kata is a term from Japanese martial arts referring to a set of fundamental movements or routines that allow a student, through repetition, to improve their skill level.

