# Assumptions:
	Subscription exists and users have access to resources within the subscription via VPN/Public-IP-enabled jumpbox
	All infrastructure resources (vnet, keyvaults, managed identities) reside in a ‘infra’ rg (designated in config file) 

# Prerequisites:
	Add user as a Reader to our image gallery


# Procedure
## Create geneva certificate
	From CloudShell ->

	$Env = "dev0"
	$Location = "eastus"
	$BaseSubId = "1aa4d67b-c6b9-42ac-9e40-7262e38d0342"
	$BaseVaultName = "mylaaso-kv"
	$Tenant = "72f988bf-86f1-41af-91ab-2d7cd011db47"
	Set-AzureRmContext -Subscription $BaseSubId
	$CertificateName = "brlepore-geneva-${Env}-${Location}"
	$genevacert = @{
	dnsNames = "brlepore-laaso-${Env}-${Location}.geneva.keyvault.hpccache.azure.com";
	distinguishedName = "CN=brlepore-laaso-${Env}-${Location}.geneva.keyvault.hpccache.azure.com";
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








```
Id                        : https://mylaaso-kv.vault.azure.net/certificates/brlepore-geneva-dev0-eastus/pending
Status                    : inProgress
StatusDetails             : Pending certificate created. Certificate request is in progress. This may take some time based on the issuer provider. Please check again later.
RequestId                 : 1205b0680d2a4782be95129197c159f8
Target                    :
Issuer                    : OneCert
CancellationRequested     : False
CertificateSigningRequest : MIIFJjCCAw4CAQAwSDFGMEQGA1UEAxM9YnJsZXBvcmUtbGFhc28tZGV2MC1lYXN0dXMuZ2VuZXZhLmtleXZhdWx0LmhwY2NhY2hlLmF6dXJlLmNvbTCCAiIwDQYJKoZIhvcNAQEBBQADggIPADCCAgoCggIBAO44D7tA9r2tcX0lFYPq71vyq6/J823JJJZ47CIiar5f+XRjAttEfxPNIXg5OR4l/TtYoRybjJpZNdJOBK4yjxXD1QDCnZX27
                            X0lqe25QX1dhJTJzytMkJ6cUBxizyOy7bcmmmz5QM4FCR3W9VkKLxdjLenAXXnwJYJK7HlNkjmzgy6FNUC4Jy2iWBXQhoMMNQb1zX2omU8aPKvczZv4C5bmChSal6XEDW7K9r1b6Xmi3wzWUhlUgljPlDSSd2mYHG43QHGAx1MLTdrD9puYZM1EjcMFzWUomdLCj3l77Fsdzdlfo9sxaNZpj9knBg5lpkPG8mPFJ5qU84ohueldhcruAAv0eD
                            YOKLwUEha0DsGUC+1solQqMRpzQAm6TSVswL/Z9U23xUT/j3m1v+NkGuQFS8mvLwEPgs9QLi/eNusWH+04ipnLzhb62YcSC+A6ovDaBCd2rBjBdxps0d8OiCYQhT94EcZLUvgFy4RmWaBkUgaVDH3MWrKxVxKCwX3m4utUhNo9/Fqg8eOL/hMMKD/BXOOXqZalCUMzdVxaJ6rYD0LucmPcYA1PjLMBaWRCMjlYdvuetNq8aTRQAxJARU72G4p
                            gqoBZ8ZGLPA8Mnqb7/1bQiSUUjAaGYejqtvV7yBoVzyRv/SSa/cDm4bo1SSt5t7oZkV5KM6eJhmQU2MElAgMBAAGggZgwgZUGCSqGSIb3DQEJDjGBhzCBhDAOBgNVHQ8BAf8EBAMCBaAwHQYDVR0lBBYwFAYIKwYBBQUHAwEGCCsGAQUFBwMCMEgGA1UdEQRBMD+CPWJybGVwb3JlLWxhYXNvLWRldjAtZWFzdHVzLmdlbmV2YS5rZXl2YXVs
                            dC5ocGNjYWNoZS5henVyZS5jb20wCQYDVR0TBAIwADANBgkqhkiG9w0BAQsFAAOCAgEAJY9PcQJ9YnqUj+rpcgRTBjRStozc/lP8wJoO0PaJo4FdkUaucNkWOco7KBY6LFkM1BfdmvJbjIER/fzK5RrHIiWCHeJwYEQupophQ26/0VXfD+iW4kCzs3ShNrizeaD+aZaTLDr+/Rw9gOmb1QfC3F8RN0zcNsXI830OKWKaT3peplvXwX94BcqgX
                            xMaJXSRH3OP7Khm3IO0ZHllhc/vHYoNSzb3MDT3U2ZBXUlDcgcac4KkKSwAopsKvib4/I+hp5c81nCy5rw6fd+gqlWxzbQCqQsKGPwCwDsG2XkQ6MbjI5ArzBJ8nG4TdsJLldmuWNpisnDNhpQwYSI+UXCy0uyFEtEi4k8LJkyCEAliKU9jEti9ZZCilLQrR44Xp4qMt5GvT/mWm+xSamr7GjmskeQj99Dfi7gIU2IPkUIYsZpsc2SvU5nzJJ
                            PH16j4HjF0co7Xi9w8lPl7hEgxzOJD2AJVjVNiCUnsVHXoAssU7DtGH75Tqn50TnvEhhqcvXQie4TN6fuO+lR+QAEHFQ4Kj0bL1HcI7aCUqlJf4zcHamWSznIn0l3JaVzvN5SFE7Y/tBL0ABJw5HSRa91bCczfELdFUVDTk2y9UVRtKJBybHK6t+DGahUk+8LKPsI8pJCrEpAnUXkEiOeAFNM8TitnLzl+KzcI+w6cqOcPHLoqMuk=
ErrorCode                 :
ErrorMessage              :
Name                      :
VaultName                 :
```





	Check portal -> Key Vault -> Certificates to see status









## Create jumpbox
## (Potentially Optional) Create VM - DSv4 (Debian Buster)

## Setup SSH forwarding:
    To jumpbox -> ssh -A -i .ssh/laaso_id_rsa brlepore@52.249.219.237
    
    Ssh config file on jumpbox and VM:

cat .ssh/config
Host *
 AddKeysToAgent yes
 ForwardAgent yes

     
## Prepare environment
From VM:

Install git
	```sudo apt-get install git```


Repo: https://dev.azure.com/msazure/One/_git/Avere-laaso-dev

Repo git clone docs - https://docs.microsoft.com/en-us/azure/devops/repos/git/use-ssh-keys-to-authenticate?view=azure-devops

	git clone git@ssh.dev.azure.com:v3/msazure/One/Avere-laaso-dev

Install python venv
	```sudo apt-get install python3-venv```

Install python
	```sudo apt-get install gcc python3-dev```

Install az cli
	```curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash```

Login
	```az login```

Verify correct subscription
	```az account show```


Define PYTHONPATH (point to root of repo)
	```export PYTHONPATH="/home/brlepore/Avere-laaso-dev/"```

Setup venv
	```VENV=~/venv_laaso # update this with wherever you want your virtualenv to live
	LAASO_REPO=~/Avere-laaso-dev  # update this with the path to your Avere-laaso-dev sandbox
	rm -rf $VENV
	python3.7 $LAASO_REPO/build/venv_create.py $VENV $LAASO_REPO/laaso/requirements.txt
	source $VENV/bin/activate```


Test base functionality of LaaSO scripts
	```$LAASO_REPO/laaso/resource_group_list.py --subscription_id 1aa4d67b-c6b9-42ac-9e40-7262e38d0342```


## Create managed identity in infra rg to allow VM to read KV 
	https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/qs-configure-cli-windows-vm#assign-a-user-assigned-managed-identity-to-an-existing-azure-vm
	```az identity create -g test-infrastructure-rg -n mylaaso-id
	az vm identity assign -g myvm-rg -n myvm-debian --identities "/subscriptions/1aa4d67b-c6b9-42ac-9e40-7262e38d0342/resourcegroups/test-infrastructure-rg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/mylaaso-id"
        az vm identity show --resource-group myvm-rg --name myvm-debian```


## Create KV
	Put in ‘infra’ rg
        Access policy - defaults
	Check all 3: 
		Enable Access to:
			Azure Virtual Machines for deployment
			Azure Resource Manager for template deployment
			Azure Disk Encryption for volume encryption
	Networking - All networks


## Add secret to KV

        SSH pub key location: 
	~/.ssh/id_rsa.pub

	create secret ->
	Manual
	Name - pubkey-<owner>
	Value - ssh public key
	enabled - yes



## Assign Managed Identity to KV
	https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/tutorial-linux-vm-access-nonaad#grant-access
	Access Policies -> Add access policy -> ‘Secret Management’ template -> Select Principal = ‘mylaaso-id’ -> Add -> Save


## Create storage account for lustre logs
	Location should be in same region as cluster (for perf)
	Other defaults are ok
	mylaaso-sa-rg / mylaasosa	


## Create container for controller create logs
     ```laaso/container_create.py 1aa4d67b-c6b9-42ac-9e40-7262e38d0342:mylaasosa/vm-create```


## Create NSG for controller VM:
	```laaso/nsg_create.py test-infrastructure-rg standupvm-nsg standup --location eastus```

## Create controller/shepherd VM:
	```laaso/vm_create.py laaso-vm-rg laaso-vm devel --owner brlepore --location eastus```
