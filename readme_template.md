## Create a resource in Azure using Terraform 

This repository will build an Azure SQL Database Instance for the project you are working within. 

This consists of two main components:

The Azure SQL Database
The SQL Server logical environment where the database resides

In addition, to support the database, a storage account will also be created.

When you run this build, the *.auto.tfvars that you will need to supply include the following:


- SQLServerName - the name of the SQL Server used in your environment - will complete the URL <SQLServerName>.database.windows.net
- DataBaseName - the name of the SQL Database that you are building - where the data will live

#### Note: #### 
``` *.auto.tfvars file is required to send along the values that will populate variables used when terraform executes. Without them, no build will happen.
Also - names and subscription info have been changed to protect the guilty``` 

### Prerequisites
To build this environment you will need access to the following:
Terraform version 0.13 - [available here](https://dereks.info/get-tf) for Windows, Mac, and a whole mess of Linux.
AzureRM provider at least 2.20 - more info about the [Azure RM Provider](#azure-rm-provider) can be found below.
Azure CLI - [available here](https://dereks.info/get-az-cli)

#### Remote-State
Terraform uses state to keep track of infrastructure it has provisioned and help it to be idempotent (meaning only things that require change will be affected). To work with this across Coterie, we are leveraging terraform cloud to manage terraform state. This prevents state for specific resources from existing on someone's laptop or requiring management within a storage account or on a virtual machine serving as the terraform build server. You can run terraform files without the code below, but they will be initialized and have state stored on your laptop/computer. Which may not be completely desireable.

##### the-remote-state-config
```
terraform {
  backend "remote" {
    organization = "donald-ducks-farm"

    workspaces {
      name = "derek_workspace_name"
    }
  }
}
```

If you have build local state and wish to move it to terraform cloud to be remotely managed you certainly can. Doing so requires the code above to be added to your configuration and your project to be reinitialized by saving the changes and running terraform init again.

#### Azure-RM-Provider
```
provider "azurerm" {
    version = "~> 2.27.0"

    features {}

    #playground subscription
    subscription_id = "99#######-###-####-####-#########d8"
    }
```
When this code is included in a .tf file it will ensure that at least version 2.27 of the Azure RM provider is downloaded for use. If there are newer versions available, they will be downloaded and used for this project run. 

#### Note: 
``` 
 For builds in the playground subscription, where code and infra is being tested, using at least 2.27 is encouraged. 
 
 For other subscriptions - a specific version should be used to avoid breaking changes in the infrastructure build.
```
You will notics that there is an empty features() block included. This is new as of terraform version 12 and is required - I do not know yet what it is used for. You need it.  Make sure it exists.

The subscription_id section tells terraform where your infrastructure will be built. Documentation like this will call out Playground, but changing this to the correct subscription should be done when you are ready to push your code somewhere real that will be around a while.

When creating providers and state for projects, because many of the projects built by coterie will have similar remote state and Azure provider information, it is recommended to keep these items in a file called terraform.tf.

This way, when the project is initialized, the terraform.tf file will contain nothing more than state and azure provider information. Other things specific to terraform or providers you may use within your builds can also be placed in the terraform.tf file.

### Data Sources
For items that already exist in Azure or items that your specific terraform will use but is not managing state for [see earlier mention of State](#remote-state) for more info on why state is super important - we can use a terraform datasource to reference other resources.  For example - if I already have a virtual network in Azure that other Azure components are using, I might want to reference it and avoid it being destroyed or recreated by any terraform. In this case, I would create a data source to reference the existing item.

The same thing goes for resources that are managed by terraform, but will be managed outside the current project. I can use a data source to reference them and not need terraform to worry about the management of state for these resources.

##### Note:
``` 
If you are creating the resource group to use with your terraform infrastructure - it should be managed by your terraform project (created as a resource).
This way, if you need to mix and match some ARM template stuff that doesnt have native terraform, when terraform tries to destroy these resources, should that become necessary, it is unable to ensure that resources created with ARM templates are removed.
When using ARM based items, be sure that the resource they are in is managed by terraform.
Once a resource group is destroyed by Terraform, similarly to ARM based deployments, all of the resources within the resource group are destroyed.
```

A data source is just a representation of another terraform resource - allowing reference to an exising or unmanaged item, but no need for terraform to manage the state of the resource.

### For any existing things that you did not create, use a data source to ensure that you do not cause trouble with other resources :)

### What to execute to get moving

Terraform cloud will use a service principal to execute commands within Azure. To replicate this in a local environment plesse see the following documentation about creating a [Service Principal using the Azure CLI](https://dereks.info/az-cli-sp).

If you are going to be using a managed service identity with the resource(s) that you are creating with Terraform, [click here for additional info] (https://dereks.info/tf-az-msi) from Terraform about Azure MSI.

Once you have the service principal or MSI that you need created and configured in Azure you should be able to start working in terraform.

If the terraform code you will be working with is stored in github, clone the repository to your local machine.

#### Note:
``` 
Make sure to keep each resource set of related resources separate for now. For example - if you are working with an app service and app insights for your build, those can be referenced together as long as they are explicitly related. For any shared components - a Vnet for example, there will be a speparate terraform project for these resources.
```
The reason for this is to ensure that an app service cannot destroy anything that might be used by other things. If that happens, terraform will likely knock down things other resources (and developers) are using.

Each related set of resources should live in its own folder/set of folders for local development:

tf_app_service_BOP2.0

Anything explicitly BOP 2.0 would be placed within this project folder and referenced from there as things are built.  As we improve our use of terraform, the resources in Azure will be made even more generic for terraform - a module for an app service will be called for any app service(s) that need to be provisioned - the information supplied during those builds and the main.tf file would be for a specific project - like BOP 2.0.


#### Using Terraform
To use any terraform code you simply need to create five files within your project folder:

- Readme.md - a copy of this file with the info at the top changed for the resource you're building
- terraform.tf - the terraform file that defines the location of your state file and any providers you need
- main.tf - the root module for your terraform project. This is where all the resources will get made
- variables.tf - the file where the variables are defined
- *.auto.tfvars - the variable values for your project

Once these files exist you can run terraform init from the project folder at the command line. Doing so will ensure any downloads happen for providers and state items.

Terraform Init looks similar to the image below:
![terraform Init](https://github.com/derekschauland/documentation/blob/master/media/gh-terraform-init.png)


[The terraform documentation](https://dereks.info/tf-docs) is your friend - refer to it often and dont be affraid to copy the terraform for things you have done before into new projects

Following intialization, running terraform plan will compile your TF code and tell you what errors it has found. If you are working with local state (on your laptop) make sure you have logged into azure with az login - you will get an object ID error during planning if terraform cannot login.

You should run a plan every time you make a change in your terraform and wish to see what happens. You should also run a plan before any apply.

Planning shows you what terraform intends to do - any resources it will add, any resources it will modify in place, and any resources it will destroy. Some actions require existing resources to be destroyed and replaced by new resources - this is something to be careful with **ESPECIALLY IN PROD!!**

A plan will look similar to the image below (image truncated to reduce size/scrolling):
![Terraform Plan](https://github.com/derekschauland/documentation/blob/master/media/gh-terraform-plan.png)

To build the planned resources, executing terraform apply will regenerate the plan to ensure nothing has changed and prompt you to be sure you want to create the resources according to the plan. The apply is shown below.  Type yes to start the build process.

![Terraform Apply](https://github.com/derekschauland/documentation/blob/master/media/gh-terraform-apply.png)

When terraform builds your resources, it will build as much as it can in parallel to speed up the creation process. Things that have dependency (an app service and app service plan for example) will be built in the correct order.

Congratulations - you've just built some infrastructure with Terraform.


#### Cleaning Up 
If your resources are for testing terraform or the resources themselves, please knock down the resoruces when your testing has completed to help keep our azure spend under control. If the resources are in ** Playground ** they should be cleaned up at the end of the day.  In other lower environments, please clean them up when you know they are no longer needed.

To handle this cleanup, from the command line where your resources were created (in the same folder where you ran terraform init etc) simply run terraform destroy and enter yes when prompted. This will instruct Terraform to delete any resources managed by the state for this project.
