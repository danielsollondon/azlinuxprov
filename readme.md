# Public Preview : Removing the Azure Linux Agent from RHEL 7.6 / CentOS 7.6

# Step 1 : Create custom image
If the Linux image does not contain agent 2.2.32 ('waagent --version'), then you will need to create a custom image with the upgraded version baked in. At the time of this doc update 4th April 2019, RHEL 7.6 (RedHat:Rhel:7-RAW:latest) did not have >=2.2.32.

For more information of version indentification see this [Azure doc](https://docs.microsoft.com/en-us/azure/virtual-machines/extensions/features-linux#agent-updates).

There are multiple ways to create custom images in the example here, you can use the example below, using AZ CLI, or this repo contains Packer and Azure Resource Manager Templates.

For full details of creating Azure images with Packer, see [Azure docs](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/build-image-with-packer). 
Deploying the Packer config:
```bash
 ./packer build /../packerconfig.json
```

If you do not use Packer, then you can use AZ CLI:

0. Create a RHEL or CentOS VM from an Azure MarketPlace Image
```bash
az group create --name <resourceGroupName> --location <location>

az vm create \
--resource-group <resourceGroupName> \
--name <srcVmName> \
--admin-username <userName> \
--image RedHat:Rhel:7-RAW:latest \
--ssh-key-value /../.pub 
```
1. SSH in, and run:
```bash
sudo yum upgrade -y WALinuxAgent
sudo waagent -deprovision+user -force
## ssh out
```
2. Generalize and create image using AZ CLI:
```bash
az vm deallocate --resource-group <resourceGroupName> --name <srcVmName>
az vm generalize --resource-group <resourceGroupName> --name <srcVmName>
az image create --resource-group <resourceGroupName> --name <userChosenImageName> --source <srcVmName>
```

# Step 2 : Create the VM from the custom image
When creating a VM, you need to:
* Use the minimum supported Microsoft.Compute API Version when creating the VM : '2018-06-01'
```json
       "type": "Microsoft.Compute/virtualMachines",
            "apiVersion": "2018-06-01"
```
* Ensure you specify this LinuxConfiguration property (this is not supported in AZ CLI yet)
```json
  "linuxConfiguration": {
    "provisionVMAgent": false
  }
  ```
  This setting will stop any extensions being deployed to the VM.

* Download removeAgt.params.json, update the parameters to your environment, use the <userChosenImageName> from step1.


You can use sample Azure Resource Manager deployment and parameters template in this repo.

To deploy the VM, run:
```bash
# template path must be the RAW git path! ;-)

declare resourceGroupName="<resourceGroupName>"
declare deploymentName="vmNoExtns"
declare templateFilePath="https://raw.githubusercontent.com/danielsollondon/azlinuxprov/master/removeAgt.deploy.json"
declare parametersFilePath="/localPathTo/removeAgt.params.json"

az group deployment create --name $deploymentName --resource-group $resourceGroupName --template-uri $templateFilePath --parameters $parametersFilePath 
```

# Step 3 : Remove the Linux Agent
SSH in, and then simply run:

```bash
sudo yum remove -y WALinuxAgent
```
Note, on CentOS, this will remove the dependency package 'azure-repo-svc.noarch 0:1.0-0.el7.centos', this is only needed for Azure Stack.

# Step 4 : Test deploying an extension to the VM
Try to reset the SSH keys from the Azure Portal, or run an extension from the command line, you should see this error:

```bash
{"code":"DeploymentFailed","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-debug for usage details.","details":[{"code":"Conflict","message":"{\r\n \"error\": {\r\n \"code\": \"OperationNotAllowed\",\r\n \"message\": \"This operation cannot be performed when extension operations are disallowed. To allow, please ensure VM Agent is installed on the VM and the osProfile.allowExtensionOperations property is true.\"\r\n }\r\n}"}]}
```