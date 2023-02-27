# DevOps

## Useful Commands

<https://github.com/ArtiomLK/commands>

## Deploy VMSS that could be used as ADO Self Hosted Agents or GH Self Hosted Runners

```bash
# ------------------------------------------------------------------------------------------------
# INIT VARIABLES
# ------------------------------------------------------------------------------------------------
# If using Git Bash avoid C:/Program Files/Git/ being appended to some resources IDs
export MSYS_NO_PATHCONV=1
# ---
# Main Vars
# ---
sub_id='xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx';                          echo $sub_id      # must update
repo_n='devops';                                                        echo $repo_n      # must update
app="ado-self-hosted-agents";                                           echo $app
env="dev";                                                              echo $env
app_rg="rg-$app-$env";                                                  echo $app_rg
l="eastus2";                                                            echo $l
tags="project=bicephub env=$env architecture=$app";                     echo $tags
user_n="artiomlk";                                                      echo $user_n

# ---
# NETWORK TOPOLOGY
# ---
vnet_pre="100.100";                                                     echo $vnet_pre
vnet_n="vnet-$app-$env";                                                echo $vnet_n
vnet_addr="$vnet_pre.0.0/23";                                           echo $vnet_addr

# ---
# Bastion
# ---
snet_n_bas="AzureBastionSubnet";                                        echo $snet_n_bas
snet_addr_bas="$vnet_pre.0.224/27";                                     echo $snet_addr_bas
nsg_n_bastion="nsg-bastion";                                            echo $nsg_n_bastion
bastion_n="bastion-$app-$env";                                          echo $bastion_n
bastion_pip="pip-bastion-$app-$env";                                    echo $bastion_pip

# ---
# SSH Key & KV
# ---
kv_rg="rg-kv-$app-$env";                                                echo $kv_rg
kv_n="kv-ado-runner";                                                   echo $kv_n
ssh_k_n="ssh-$app-$repo_n-$env";                                        echo $ssh_k_n

# ---
# Self Hosted Runners vmss
# ---
vmss_n="vmss-$app-$repo_n-$env";                                        echo $vmss_n
vmss_img="UbuntuLTS";                                                   echo $vmss_img
vmss_sku="Standard_B2s";                                                echo $vmss_sku
vmss_instance_count="1";                                                echo $vmss_instance_count
snet_vmss_n="snet-$app-$repo_n";                                        echo $snet_vmss_n
snet_addr_vmss="$vnet_pre.1.0/24";                                      echo $snet_addr_vmss          # must update
nsg_vmss_n="nsg-$app-$repo_n";                                          echo $nsg_vmss_n

# ------------------------------------------------------------------------------------------------
# DEPLOYMENT CREATE COMPONENTS
# ------------------------------------------------------------------------------------------------

# App RG
az group create \
--subscription $sub_id \
--name $app_rg \
--location $l \
--tags $tags

# Keys RG
az group create \
--subscription $sub_id \
--name $kv_rg \
--location $l \
--tags $tags

# Main vNet
az network vnet create \
--subscription $sub_id \
--name $vnet_n \
--resource-group $app_rg \
--address-prefixes $vnet_addr \
--location $l \
--tags $tags

# Key Vault
az keyvault create \
--subscription $sub_id \
--name $kv_n \
--resource-group $kv_rg \
--location $l \
--tags $tags

# ------------------------------------------------------------------------------------------------
# Create an SSH Keys and KV
# ------------------------------------------------------------------------------------------------
# Generate new ssh pub nad private keys
az sshkey create \
--subscription $sub_id \
--name $ssh_k_n \
--resource-group $kv_rg \
--tags $tags

# Set the private key file
ssh_priv_key_path='C:\Users\artioml\.ssh\1677436842_525029'; echo $ssh_priv_key_path  # must update

# upload private key to the KV
az keyvault secret set --vault-name $kv_n --name $ssh_k_n --value "@$ssh_priv_key_path"

# retrieve secrets from KV
az keyvault secret show --vault-name $kv_n --name $ssh_k_n

# ------------------------------------------------------------------------------------------------
# Create Self Hosted Agents VMSS
# ------------------------------------------------------------------------------------------------
# Self Hosted NSG with Default rules
az network nsg create \
--subscription $sub_id \
--resource-group $app_rg \
--name $nsg_vmss_n \
--location $l \
--tags $tags

# Self Hosted Subnet
az network vnet subnet create \
--subscription $sub_id \
--resource-group $app_rg \
--vnet-name $vnet_n \
--name $snet_vmss_n \
--address-prefixes $snet_addr_vmss \
--network-security-group $nsg_vmss_n

# vm scale set agents
az vmss create \
--subscription $sub_id \
--name $vmss_n \
--resource-group $app_rg \
--image $vmss_img \
--vm-sku $vmss_sku \
--storage-sku StandardSSD_LRS \
--authentication-type SSH \
--ssh-key-values "$ssh_priv_key_path.pub" \
--vnet-name $vnet_n \
--subnet $snet_vmss_n \
--instance-count $vmss_instance_count \
--disable-overprovision \
--upgrade-policy-mode manual \
--single-placement-group false \
--platform-fault-domain-count 1 \
--load-balancer "" \
--assign-identity \
--admin-username $user_n \
--tags $tags

# Verify Manual update policy
az vmss show \
--subscription $sub_id \
--resource-group $app_rg \
--name $vmss_n \
--output table

# ------------------------------------------------------------------------------------------------
# Create vnet peering
# ------------------------------------------------------------------------------------------------
# VNET_1 Variables
vnet1_rg_n=$app_rg;                          echo $vnet1_rg_n
vnet1_n=$vnet_n;                             echo $vnet1_n

# VNET_2 Variables
vnet2_rg_n="rg-name-2";                      echo $vnet2_rg_n
vnet2_n="vnet-name-2";                       echo $vnet2_n

VNET_1_ID=$(az network vnet show --resource-group $vnet1_rg_n --name $vnet1_n --query id --out tsv); echo $VNET_1_ID
VNET_2_ID=$(az network vnet show --resource-group $vnet2_rg_n --name $vnet2_n --query id --out tsv); echo $VNET_2_ID

# VNET PEER from VNET_1 to VNET_2
az network vnet peering create \
--name "peer-$vnet1_n-to-$vnet2_n" \
--resource-group $vnet1_rg_n \
--vnet-name $vnet1_n \
--remote-vnet $VNET_2_ID \
--allow-vnet-access

# VNET PEER from VNET_2 to VNET_1
az network vnet peering create \
--name "peer-$vnet2_n-to-$vnet1_n" \
--resource-group $vnet2_rg_n \
--vnet-name $vnet2_n \
--remote-vnet $VNET_1_ID \
--allow-vnet-access
```

## Install CLIs Linux

```bash
# Install Azure CLI
i=1; echo "init i = $i"
while [ $i -le $vmss_instance_count ]
do
   echo Number: $i
   az vmss run-command invoke \
   --resource-group $app_rg \
   --name $vmss_n \
   --instance-id $i \
   --command-id RunShellScript \
   --scripts "curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash"
   ((i++))
done

# Install Bicep
i=0; echo "init i = $i"
while [ $i -le $vmss_instance_count ]
do
   echo Number: $i
   az vmss run-command invoke \
   --resource-group $app_rg \
   --name $vmss_n \
   --instance-id $i \
   --command-id RunShellScript \
   --scripts "runuser -l artiomlk -c 'az bicep install'"
   ((i++))
done
```

## Notes

- You can organize your pipeline into jobs. Every pipeline has at least one job. A job is a series of steps that run sequentially as a unit. In other words, a job is the smallest unit of work that can be scheduled to run. [ref][8]

- For self-hosted parallel jobs, you'll start by deploying our self-hosted agents on your machines. You can register any number of these self-hosted agents in your organization. [ref][8]

- You cannot run Mac agents using **scale sets**. You can only run Windows or Linux agents this way. You should not associate a VMSS to multiple pools. [ref][5]

## Additional Information

- Price
- [Azure | MS | Pricing for Azure DevOps][7]
- [Docs | MS | Configure and pay for parallel jobs][1]
- Jobs
- [Docs | MS | Specify jobs in your pipeline][8]
- [Docs | MS | Deployment jobs][9]
- [Docs | MS | Define container jobs (YAML)][10]
- Self Hosted Agents
- [Docs | MS | Self-hosted Linux agents][2]
- [Docs | MS | Self-hosted macOS agents][3]
- [Docs | MS | Self-hosted Windows agents][4]
- [Docs | MS | Azure virtual machine scale set agents][5]
- [Docs | MS | Run a self-hosted agent in Docker][6]
- [Docs | Learn | Allowed IP addresses and domain URLs][12]
- Vnet Peering
- [Docs | Learn | az network vnet peering create][11]
- Bash
- [Docs | Learn | Unix Operators][13]

[1]: https://docs.microsoft.com/en-us/azure/devops/pipelines/licensing/concurrent-jobs
[2]: https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-linux
[3]: https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-osx
[4]: https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-windows
[5]: https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/scale-set-agents
[6]: https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/docker
[7]: https://azure.microsoft.com/en-us/pricing/details/devops/azure-devops-services/
[8]: https://docs.microsoft.com/en-us/azure/devops/pipelines/process/phases
[9]: https://docs.microsoft.com/en-us/azure/devops/pipelines/process/deployment-jobs
[10]: https://docs.microsoft.com/en-us/azure/devops/pipelines/process/container-phases
[11]: https://learn.microsoft.com/en-us/cli/azure/network/vnet/peering?view=azure-cli-latest
[12]: https://learn.microsoft.com/en-us/azure/devops/organizations/security/allow-list-ip-url?view=azure-devops&tabs=IP-V4#inbound-connections
[13]: https://www.educba.com/unix-operators/
