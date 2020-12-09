# Self hosting Github action runners
Github provides free managed runners with a good environment. However if you need specialized setup like GPU instances, access to data you may have to create self-hosted runners. 

This recipe provides instructions how to create a self-hosted runner on a VM. Specifically, we create a GPU based Spot instance VM. 
This allows you to leverage GPUs like training / validating an ML model as part of a Github workflow. 

## Trigger
We will trigger a Github Actions workflow when you open a new issue in your repo. 

## Steps
The steps in the workflow are 

* Create a Spot instance of a GPU VM in Azure
* Register the VM instance as a Github action runner
* Execute something on the GPU
* Capture the output as a Github artifact
* Tear down the GPU VM to stop being billed


Github Actions with self-hosted runner is currently recommended only on private repositories as per the [documentation](https://docs.github.com/en/free-pro-team@latest/actions/hosting-your-own-runners/about-self-hosted-runners#self-hosted-runner-security-with-public-repositories), 
though there is discussion to address the issues for public repos also. So for now, please dont try this on public repos or make sure you understand you understand the risks. 

### Setup

1. Create a Personal Access Token for your Github repository where the Github Actions will run. 
You can follow the instruction on the [Github Documentation page](https://docs.github.com/en/free-pro-team@latest/github/authenticating-to-github/creating-a-personal-access-token). 
Temporarily note down the Personal Access Token as you can read it again once you close this screen. You will store it securely in step#3. 


2. Setup a Azure resource group and permission to Github Action
Since we will be creating Azure resources (which may incur charges), we create the Service Principal. We also limit resources to be cerated in a designated Azure Resource group for easier management. 

Run the following commands in the Azure CLI. 

```
az group create --name ghactionrunner -l westus2
az ad sp create-for-rbac --name "ghactionrunner" --sdk-auth --role contributor --scopes /subscriptions/YOUR-SUBSCRIPTION-ID-PLACEHOLDER/resourceGroups/ghactionrunner
```

Set correct value for your Azure Subscription GUID in the command above. The second command will output a JSON document describing the service principal. Copy this JSON into the clipboard. You will store this securely in the next step. 
You can customize the commands above if you want these resources in a different region or want to name the resources differently. 


3. Create a private repo where you want to trigger the Github Actions

4. Once your repo is created, go to the "Settings" and click on "Secrets" in the left side menu. Create two secrets with name AZURE_CREDENTIALS AND PAT. 
Store the Personal Access Token copied above in the secret named PAT and the Service Principal JSON as the secret named AZURE_CREDENTIALS. Here is the screenshot showing where to find the "Secrets" function. Once you store the two secrets your screen will look as shown. 


![Store Secrets](/images/ghsecrets.png)

### Create the Github Actions YAML file. 

Click on the "Actions" tab of your repo and then select 'Setup a workflow yourself". Here you can copy the sample [Actions spec](azure-spot-self-hosted-actions-template.yaml) (YAML) that you can use and adapt.

The sample YAML file is annotated. But basic steps include:
* Create a Azure Spot VM on a Nvidia K80 GPU using Azure CLI available in the default free Github runner environment where this step will execute. In this example we create the VM with a [Data Science VM](http://aka.ms/dsvm) image that has preinstaleld Nvidia drivers, docker, DL frameworks liek Tensorflow, Pytorch etc. You can choose any other OS image and customize the environment to yoru specific needs. 
* Run an Azure [extension script](https://gist.github.com/gopitk/805d7035217dfd2e81d52d0b109c3349/raw/ce581a3472af54ca3293a16016aec55023ed46a7/installgitactrunners.sh) currently a gist, 
which installs the Github Action Runners and installs it as a service. It also registers itself to Github Actions so you will see a runner instance once this script executes on the newly created VM. 
* Then you can run whatever code like training a model on the self hosted runner that is registered in previous step. For simplicity, the command run in this sample is just looking at the GPU status using the "nvidia-smi" command and saving it in a file.
* Next we upload the file as an artifact to the Github Action workflow run. In a real scenario like a model trainign run you can save the output logs, the model file as artifacts. 

### Triggering your Github Actions Workflow
In the sample YAML file, the action is trigger when you create a new issue. You can customize it to trigger on other events liek a pull requests or specific patterns in pull request comments. 

When you create a new issue in your repo, it will run the workflow and you can see the progress. 

As you can see it is quite simpel to create a Github Actions workflow. For me, it needed some trial and error in terms of integrating with Azure CLI, scripts to automatically register the self hosted runner. Hope this recipe will make it a bit easier for you. 

Welcome any feedback and suggestions as pull requests. 



