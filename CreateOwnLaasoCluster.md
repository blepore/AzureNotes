# Creating a LaaSO cluster in your own subscription
For internal customers/partners only. This will not work for external customers.

## Assumptions
 - Subscription exists and users have access to resources within the subscription via VPN/Public-IP-enabled jumpbox
 - All infrastructure resources (vnet, keyvaults, managed identities) reside in a ‘infra’ rg (designated in config file) 

## Prerequisites
 - Request User be added as Reader to LaaSO image gallery
 - Request User be added as Reader to Geneva certificate Key Vault
 - Request Geneva certificate name from LaaSO team
 - Verify that Images are being replicated to appropriate region
 - Request Quota increase (if necessary)
 	- LaaSO clusters use D32s_v3 VMs. Each VM provides ~3.8TB of raw storage capacity and ~768MB/s of throughput (max). You should request enough quota to build a cluster based on those specs (Rough estimates of required number of VMs are below in the "Create cluster config file" section)

## Procedure
### Create jumpbox

### (Potentially Optional) Create VM - DSv4 (Debian Buster)
<Create VM info/link here>

### Setup SSH forwarding:
To jumpbox ->

	ssh -A -i <path/to/private ssh key> <user>@<publicip>
    
Ssh config file on jumpbox and VM:

.ssh/config file:

	Host *
 	   AddKeysToAgent yes
 	   ForwardAgent yes

### Prepare environment
From VM:

Install git

	sudo apt-get install git

Repo: https://dev.azure.com/msazure/One/_git/Avere-laaso-dev

Repo git clone docs - https://docs.microsoft.com/en-us/azure/devops/repos/git/use-ssh-keys-to-authenticate?view=azure-devops

	git clone git@ssh.dev.azure.com:v3/msazure/One/Avere-laaso-dev

Install python venv

	sudo apt-get install python3-venv

Install python

	sudo apt-get install gcc python3-dev

Install az cli

	curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

Login

	az login

Verify correct subscription

	az account show

Define PYTHONPATH (point to root of repo)

	export PYTHONPATH="/home/brlepore/Avere-laaso-dev/"

Setup venv

	VENV=~/venv_laaso # update this with wherever you want your virtualenv to live
	LAASO_REPO=~/Avere-laaso-dev  # update this with the path to your Avere-laaso-dev sandbox
	rm -rf $VENV
	python3.7 $LAASO_REPO/build/venv_create.py $VENV $LAASO_REPO/laaso/requirements.txt
	source $VENV/bin/activate

LaaSO tools and scripts have been custom-built to create resources in your environment. These tools are highly recommended (if not required) for use in creating LaaSO-ready (aka "LaaSO-ized") resources in your subscription. 

Test base functionality of the LaaSO scripts now that the environment is setup.

	$LAASO_REPO/laaso/resource_group_list.py --subscription_id 1aa4d67b-c6b9-42ac-9e40-7262e38d0342

### Create managed identity in infra rg to allow VM to read KV 
https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/qs-configure-cli-windows-vm#assign-a-user-assigned-managed-identity-to-an-existing-azure-vm

	az identity create -g test-infrastructure-rg -n mylaaso-id
	az vm identity assign -g myvm-rg -n myvm-debian --identities "/subscriptions/1aa4d67b-c6b9-42ac-9e40-7262e38d0342/resourcegroups/test-infrastructure-rg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/mylaaso-id"
	az vm identity show --resource-group myvm-rg --name myvm-debian

### Create KV
A key vault is required to store a few items:
 - SSH public key (used for accessing the cluster nodes if necessary)
 - Geneva certificate (for uploading logs/stats to Geneva for troubleshooting)

For more information about creating Key Vaults in Azure, see this: https://docs.microsoft.com/en-us/azure/key-vault/general/quick-create-portal

Run the following script to create a LaaSO-ized Key Vault:

	$LAASO_REPO/laaso/keyvault_create.py

When creating the Key Vault, most defaults are acceptable

	Choose the following values:
		Resource Group - choose your ‘infra’ resource group 
		Access policy - defaults
		Check all 3: 
			Enable Access to:
				Azure Virtual Machines for deployment
				Azure Resource Manager for template deployment
				Azure Disk Encryption for volume encryption
				Networking - All networks

### Assign Managed Identity to KV
	https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/tutorial-linux-vm-access-nonaad#grant-access
	
	Access Policies -> Add access policy -> ‘Secret Management’ template -> Select Principal = ‘mylaaso-id’ -> Add -> Save
	
	Need GET/LIST on Keys and Certificates as well

### Add SSH public key secret to KV
Stated above, an SSH public key is required in order to provide access to the cluster nodes for troubleshooting/admin purposes. For more information about ssh keys, see here: https://docs.microsoft.com/en-us/azure/virtual-machines/linux/create-ssh-keys-detailed
        
Once you've generated your SSH key pair, you'll need to place the public key into the key vault. Copy the public key (typically resides in the home directory and named something similar to ~/.ssh/id_rsa.pub)

Place the key as a 'Secret' in the key vault. To create a 'Secret', reference this: https://docs.microsoft.com/en-us/azure/key-vault/secrets/quick-create-portal#add-a-secret-to-key-vault

NOTE: it is important to name the key as described below (with the 'pubkey-' prefix plus username)

	Choose the following values: 
		Upload Options - Manual
		Name - pubkey-<owner>
	   	    Example: pubkey-brlepore
		Value - <ssh public key>
		Enabled - yes

### Copy Geneva certificate from LaaSO sub to KV
	
	$LAASO_REPO/laaso/keyvault_certificate_clone.py test-infrastructure-rg mylaaso-kv partner-laaso-dev0-eastus /subscriptions/751411ed-8325-4f6a-902a-b5ce4eb3dd14/resourceGroups/partner-kv-rg/providers/Microsoft.KeyVault/vaults/partner-eastus-kv

### Create storage account for lustre logs
For debugging purposes, a storage account is needed to store logs locally in the subscription. 

For more information about creating Storage Accounts in Azure see this: https://docs.microsoft.com/en-us/azure/storage/common/storage-account-create?tabs=azure-portal

	Choose the following values:
		Location - same region as cluster (for performance reasons)	

### Create container for controller create logs
	$LAASO_REPO/laaso/container_create.py 1aa4d67b-c6b9-42ac-9e40-7262e38d0342:mylaasosa/vm-create

### Create container for sub setup (may not be necessary)
	
	$LAASO_REPO/laaso/container_create.py 1aa4d67b-c6b9-42ac-9e40-7262e38d0342:mylaasosa/subscription-setup
	
### Create container for deploy-cluster

	$LAASO_REPO/laaso/container_create.py 1aa4d67b-c6b9-42ac-9e40-7262e38d0342:mylaasosa/deploy-cluster

### Create NSG for controller VM:
	
	$LAASO_REPO/laaso/nsg_create.py test-infrastructure-rg standupvm-nsg standup --location eastus

### Create Log Analytics workspace for OMS extensions
Create a Log Analytics workspace for OMS extensions. For more on how to create a workspace, see this: https://docs.microsoft.com/en-us/azure/azure-monitor/learn/quick-create-workspace

For ease of use, it is recommended that you put the workspace in the 'infra' resource group.

### Update Configuration File

	$LAASO_REPO/src/config/testing_subscriptions.yaml

### Create controller/shepherd VM:
	
	$LAASO_REPO/laaso/vm_create.py mylaaso-vm-rg mylaaso-vm devel --owner brlepore --location eastus

### SCP config file to new VM (save sample config off as 'orig' for reference)
	
	mv src/config/testing_subscriptions.yaml src/config/testing_subscriptions.yaml.orig; scp 172.24.0.6:/home/brlepore/Avere-laaso-dev/src/config/testing_subscriptions.yaml /home/brlepore/Avere-laaso-dev/src/config/testing_subscriptions.yaml

### Log into new controller VM and prepare to create cluster

	git clone git@ssh.dev.azure.com:v3/msazure/One/Avere-laaso-dev
	export PYTHONPATH="/home/brlepore/Avere-laaso-dev/"
	VENV=~/venv_laaso # update this with wherever you want your virtualenv to live
	LAASO_REPO=~/Avere-laaso-dev  # update this with the path to your Avere-laaso-dev sandbox
	rm -rf $VENV
	python3.7 $LAASO_REPO/build/venv_create.py $VENV $LAASO_REPO/laaso/requirements.txt
	source $VENV/bin/activate

Login

	az login

Verify correct subscription

	az account show

### Create cluster config file
The deploy scripts accept yaml-formatted config files as input. The config file specifies:

 - Region (location)
 - Vnet and subnet to use	
	- virtualNetworkRG - name of resource group in which the vnet resides
	- virtualNetworkId - name of vnet in subscription
	- serverSubnet - name of the subnet in which the server will be deployed
	- clientSubnet (optional) - if you'd like to deploy clients, this is the subnet in which the clients will deploy
 - User information - used primarily to access the key vault and Geneva certificates
 	- genevaKeyVaultRG
	- genevaIdentity
	- genevaCertificate
 - Hydration information - used to copy metadata/data from Blob into cluster
 	- hydrationKeyVault
 	- hydrationKeyVaultRG
 	- hydrationKeyVaultIdentity 
 - Cluster capacity and throughput - Noted in the "Prerequisites" section above, LaaSO clusters use D32s_v3 VMs. Each VM provides ~3.8TB of raw storage capacity and ~768MB/s of throughput (max). You should request enough quota to build a cluster based on those specs.
 	- Examples:
		- 2 node cluster - provides ~7.6TB of capacity and 1536 MB/s of max throughput
		- 53 node cluster - provides ~200TB of capacity and 40GB/s of max throughput
		- 131 node cluster - provides ~500TB of capacity and 100GB/s of max throughput

Sample config file (2nodeD32_2clients.yaml):

	cluster:

	  clientCount: 2
	  ossCount: 2

	  clientSubnet: "default"
	  serverSubnet: "default"
	  virtualNetworkId: "test-vnet"
	  virtualNetworkRG: "test-infrastructure-rg"
	  location: "eastus"

	hydration:
	  hydrationKeyVault: mylaaso-kv
	  hydrationKeyVaultRG: test-infrastructure-rg
	  hydrationKeyVaultIdentity: /subscriptions/1aa4d67b-c6b9-42ac-9e40-7262e38d0342/resourceGroups/test-infrastructure-rg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/mylaaso-id
	  #hydrationKvSecret: laasoimporttestdata-key
	  #hydrationStorageAccount: laasoimporttestdata
	  #hydrationContainer: import
	  #loggingContainer: amlfilesystem-logging

	user:
	  genevaKeyVaultRG: test-infrastructure-rg
	  genevaIdentity: mylaaso-id
	  genevaCertificate: partner-laaso-dev0-eastus
	  
## Deploy cluster
Deploy cluster and clients using the following command. 'nohup' is used here for convenience and is not necessary. Output is logged to 'nohup.out' as a result.

	nohup laaso/deploy_cluster.py ~/yamlfiles/2nodeD32_2clients.yaml --cleanup=0 --log_level=debug --debug=2 &
	
## Log into clients 

By defaults, the scripts configure 'azureuser' as the user using the public ssh key in the key vault. 


### Hydration
Assign managed identity to 'Storage Blob Data Owner' role in SA/Blob

## LaaSO Team Prerequisite Checklist/Procedures (* if you're a Partner, stop here)
 - Add user as a Reader to our image gallery
 - Add user as a Reader to partner key vault
 - Verify that Images are being replicated to appropriate region and add region to replication list if not
 	- This happens at the per-version level
 - Create partner-specific geneva certificate

From CloudShell (for ease of use, CertificateName, dnsNames, distinguishedName prefix should all match)

	$Env = "dev0"
	$Location = "eastus"
	$BaseSubId = "751411ed-8325-4f6a-902a-b5ce4eb3dd14"
	$BaseVaultName = "partner-eastus-kv"
	$Tenant = "72f988bf-86f1-41af-91ab-2d7cd011db47"
	Set-AzureRmContext -Subscription $BaseSubId
	$CertificateName = "partner-laaso-${Env}-${Location}"
	$genevacert = @{
	dnsNames = "partner-laaso-${Env}-${Location}.geneva.keyvault.hpccache.azure.com";
	distinguishedName = "CN=partner-laaso-${Env}-${Location}.geneva.keyvault.hpccache.azure.com";
	certificateName = $CertificateName
	}
	Write-Host "certificateName: $CertificateName"
	$validity = 24  # in months; for test environments we rotate real fast, every 1 month so we can blow up fast
	$renewLife = 4  # 1/24 = .041666, round down
    	# Set Cert Issuer to OneCert as Certificates created must be signed by OneCert.
    	$policy = New-AzureKeyVaultCertificatePolicy -IssuerName 'OneCert' -SubjectName $genevacert['distinguishedName'] -SecretContentType 'application/x-pkcs12' `
   	-Ekus "1.3.6.1.5.5.7.3.1", "1.3.6.1.5.5.7.3.2" -ValidityInMonths $validity -KeyType 'RSA'  -KeySize 4096 `
    	-RenewAtPercentageLifetime $renewLife -DnsNames $genevacert['dnsNames']
    	Set-AzureKeyVaultCertificateIssuer -VaultName $BaseVaultName -Name 'OneCert' -IssuerProvider 'OneCert'
    	Add-AzureKeyVaultCertificate -VaultName $BaseVaultName -Name $CertificateName -CertificatePolicy $policy	

Check portal -> Key Vault -> Certificates to see status



### Upload Certificate to Geneva 
Preview.jarvis-int.dc.ad.msft.net
configurations -> 1.3v0 namespace 

Jarvis -> manage -> logs -> User roles -> 'AZSClogs_MaCommunication' -> manage certificates view/add -> key vault managed certificates -> upload certificate



Logs account - azsclogs 
Logs endpoint - test 
