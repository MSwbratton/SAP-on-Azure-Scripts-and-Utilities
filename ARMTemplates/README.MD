# **Overview**

The scripts and templates in this folder structure can be used to deploy the infrastructure for a SAP on Azure deployment

## **The folder structure**

The folder structure in the repo is designed to support an organisation that has a large number of SAP deployments

### **/baseInfrastructure**

This folder contains the templates for the core infrastructure artifacts

#### **asg.json**

This is the template for the Application Security Group deployment

#### **network.json**

This is the template for the a sample network deployment

#### **ppgavset.json**

This is the template for Proximity Placement Group and Availability Sets deployment

### **/deploymentScripts**

This folder contains the deployment scripts for the templates

#### **deployInfrastructure.ps1**

This is the deployment script for deploying the base infrastructure assets (asg.json)

#### **deployLandscape.ps1**

This is the placeholder deployment script for deploying the landscape. Do not run this, it will be updated when using the createNewLandscape.ps1 script.

#### **deployNetwork.ps1**

This is the  deployment script for deploying the landscapesample network configuration

### **/landScapes**

This is the root folder for all the landscapes. Each SAP deployment (SID) should be represented by a folder under this folder, for example /landScapes/S10, /landScapes/S11

#### **createNewEnvironment.ps1**

This is the script you can use to create the deployment artifacts for a new landscape.

### **/serverTemplates**

This is the  folder containing all the server templates. After cloning the repository edit the template files to reflect your requirements (naming conventions etc)

```json
        "sapSystemIdLower": "[toLower(parameters('sapSystemId'))]",
        "locationName": "[if(empty(parameters('location')) , resourceGroup().location,parameters('location'))]",
        "sapMachineName": "[concat(variables('sapSystemIdLower'),'-',parameters('virtualMachineName'))]",
        "networkInterfaceName": "[concat(variables('sapMachineName'),'-nic')]",
        "publicIpAddressName": "[concat(variables('sapMachineName'),'-pip')]",
        "avSetName": "[if(empty(parameters('availabilitySetName')), concat(variables('sapSystemIdLower'),'-app-avset'),parameters('availabilitySetName'))]",
        "ppgName": "[if(empty(parameters('proximityPlacementGroupName')), concat(variables('sapSystemIdLower'),'-ppg'),parameters('proximityPlacementGroupName'))]",

```

### **/serverTemplates/parameterFiles**

This is the  folder containing all the server template parameter files. After cloning the repository edit the parameter files to reflect your environment (Application Security Group Names, Resource Group name for Virtual Network, Virtual Network name etc.)

```json
    "applicationSecurityGroupName": {
      "value": "sap-app-asg"
    },
    "virtualNetworkResourceGroupName": {
      "value": "demo-vnet-rg"
    },
    "virtualNetworkName": {
      "value": "demo-vnet"
    },
    "subnetName": {
      "value": "app-snet"
    },
    "vmTagValues": {
      "value": { "System": "SAP", "$perfOptimizationLevel": "1" }
    },
```

## **Server Templates**

### **ASCSVM.json**

This is the template for an ASCS virtual machine

### **APPVM.json**

This is the template for an application server virtual machine

### **anyDBProdVM.json**

This is the template for a database virtual machine

### **HanaProdVM.json**

This is the template for a database virtual machine designed to host the Hana Server

### **HanaDevVM.json**

This is the template for a development database virtual machine designed to host the Hana Server

## **Creating a new deployment of a new SAP environment on Azure**

Navigate to the /landscapes folder. Modify the values in the **createNewEnvironment.ps1** file
Change the [SUBSCRIPTIONID], [IMAGEGALLERYRGNAME],[VAULTNAME],[IMAGENAME] to reflect your environment

```PowerShell
$SID = "S40"
$region = "westeurope"

$subscriptionID = "[SUBSCRIPTIONID]"

$KeyVaultID = "/subscriptions/[SUBSCRIPTIONID]/resourceGroups/sharedservices/providers/Microsoft.KeyVault/vaults/[VAULTNAME]"

#If you want to use a marketplace image $xxxxxImageID needs to be an empty string
#Custom image ID
$AppServerImageID = "/subscriptions/[SUBSCRIPTIONID]/resourceGroups/[IMAGEGALLERYRGNAME]/providers/Microsoft.Compute/galleries/CorpImageGallery/images/[IMAGENAME]/versions/0.24089.43435"
$AppServerImageID = ""
$ASCSServerImageID = ""
$HanaServerImageID = ""

Add-Type -TypeDefinition @"
   public enum DBType
   {
      AnyDB,
      HanaDev,
      HanaProd
   }
"@

[DBType]$Database = [DBType]::HanaDev

#How many ASCS Servers are needed
$NumberOfASCSServers = 2
#Marketplace Template Information for the ASCS Server
#If ImageID is provided then these fields will be ignored
$ASCSPublisher = "suse"
$ASCSOffer = "sles-15-sp1"
$ASCSSKU = "gen1"
$ASCSSKUVersion = "latest"
#VM Size for the ASCS server
$ASCSVMSize = "Standard_D2s_v3"

#How many Application Servers are needed
$NumberOfAppServers = 2
#Marketplace Template Information for the Application Server
#If ImageID is provided then these fields will be ignored
$AppPublisher = "suse"
$AppOffer = "sles-15-sp1"
$AppSKU = "gen1"
$AppSKUVersion = "latest"
#VM Size for the application server
$AppVMSize = "Standard_D4s_v3"

#How many DB Servers are needed
$NumberOfDatabaseServers = 2
#Marketplace Template Information for the Database Server
#If ImageID is provided then these fields will be ignored
$DBPublisher = "suse"
$DBOffer = "sles-15-sp1"
$DBSKU = "gen1"
$DBSKUVersion = "latest"
#VM Size for the database server
$DBVMSize = "Standard_E16s_v3"

#Is High Availability required
$SAPHA = $true
```

### **Retrieving IDs, etc for the supporting services**

#### Get the Image SKUs available in a region

```PowerShell
$Region = "westeurope"
Get-AzVMImageOffer -PublisherName Oracle -Location $Region

Get-AzVMImageOffer -PublisherName Suse -Location $Region

#Get the SKUs
Get-AzVMImageSku -PublisherName Oracle -Location $Region -Offer Oracle-Linux

Get-AzVMImageOffer -PublisherName Suse -Location $Region -Offer sles-15-sp1

#Get the Image details
Get-AzVMImage -PublisherName Oracle -Location $Region -Offer Oracle-Linux -Skus 77

Get-AzVMImage -PublisherName Suse -Location $Region -Offer sles-15-sp1 -Skus gen1

```

#### Get the VM SKU sizes available in a region

```PowerShell
$Region = "westeurope"
Get-AzVMSize -Location westeurope

#List all 8 core SKUs

Get-AzVMSize -Location westeurope | Where-Object {$_.NumberOfCores -eq 8}

```

Validate the SKU size using [SAP Note 1928533](https://launchpad.support.sap.com/#/notes/1928533)

#### Get the Azure Key Vault ID

Open the key vault in the portal and choose Resource ID from the properties tab for your keyvault

![Azure Keyvault ID](./AzKeyVaultID.png)

```PowerShell
$ResourceGroup = "sharedserviced"
$VaultName = "corp-vault"

$id = (Get-AzResource -Name $VaultName -ResourceGroupName $ResourceGroup -ResourceType Microsoft.KeyVault/vaults).ResourceId

Write-Host "Key Vault ID" $id
```

#### Get the Image template ID

```PowerShell
$ResourceGroupName = "SharedImagesUS"
$ImageName = "nwImage20200111-01"

$id = (Get-AzResource -Name $ImageName -ResourceGroupName $ResourceGroupName -ResourceType Microsoft.Compute/galleries/images).ResourceId

Write-Host "Image ID" $id

```

### **Creating the folder structure**

Execute the *createNewEnvironment.ps1* script

### **Update the server parameter files SID.xxxx.paramater.json and change the following parameters:**

Navigate to the new folder, this folder will contain a deployment script and the parameter files required to deploy the availability sets and the proximity placement group as well as parameter files (ASCS, application and database)

- availabilitySetName, for example: "availabilitySetName": {"value": "S04-app-avset"}
- availabilitySetNameDB, for example: "availabilitySetNameDB": {"value": "S04-db-avset"}
- proximityPlacementGroupName, for example: "proximityPlacementGroupName": {"value": "S04-ppg"}
- virtualMachineSize, for example: "virtualMachineSize": {"value": "Standard_E4s_v3"}
- PrivateIP, for example: "PrivateIP": {"value":"111.222.333.444"}
- HasPublicIP, for example: "HasPublicIP": {"value": false}
- virtualNetworkResourceGroupName, for example "virtualNetworkResourceGroupName": { "value": "VNetRG" }
- virtualNetworkName, for example "virtualNetworkName": { "value": "prodvnet" }
- subnetName, for example "subnetName": { "value": "apps" }
- imageReference, for example: "imageReference": {"value": {"publisher": "suse","offer": "SLES-SAP","sku": "12-sp4","version": "latest"}}
- imageID, for example: "imageID": {"value": "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxxx/resourceGroups/SharedImagesUS/providers/Microsoft.Compute/galleries/CorpImageGallery/images/nwImage20200111-01/versions/0.24089.43435"} see above
- Tags, for example "vmTagValues" : {"value" : "{ System": "SAP", "$perfOptimizationLevel": "1" }"

and any other parameter values unique for the deployment.

Validate that the parameters in the *deployLandscape.ps1* script reflect your environment.

```PowerShell
$ResourceGroupName = "SAP-S22"
$location = "northeurope"
$subscriptionID = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
$SID = "S22"
```

Deploy the environment using the *deployLandscape.ps1* file.  

**Note** The templates expect that the virtual network, the subnets and the application security groups already exist
