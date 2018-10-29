# Private Preview : Removing the Azure Linux Agent

# Step 1 : Create custom image
If the Linux image does not contain agent 2.2.32 ('waagent -version'), then you will need to create a custom image with this baked in.

There are multiple ways to create custom images in the example here, this repo contains Packer and Azure Resource Manager Templates.

For a sample Packer config to bake in the 2.2.32 agent, see this [repo](https://github.com/danielsollondon/azlinuxprov), 
deploy the config:
```bash
 ./packer build /../packerconfig.json
```

# Step 2 : Create the VM from the custom image
When creating a VM, you need to:
* Use the Microsoft.Compute API Version when creating the VM : '2018-06-01'
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

You can use sample Azure Resource Manager deployment and parameters template in this repo.

To deploy the VM, run:
```bash
# template path must be the RAW git path!

declare resourceGroupName=""
declare deploymentName=""
declare templateFilePath="https://raw.githubusercontent.com/danielsollondon/azlinuxprov/master/removeAgt.deploy.json"
declare parametersFilePath="pathToLocalParamsFile.json"

az group deployment create --name $deploymentName --resource-group $resourceGroupName --template-uri $templateFilePath --parameters $parametersFilePath 
```

# Step 3 : Remove the Linux Agent
SSH in, and then simply run:

```bash
sudo yum remove -y WALinuxAgent
```
Note, this will remove the dependency package 'azure-repo-svc.noarch 0:1.0-0.el7.centos', this is only needed for Azure Stack.
