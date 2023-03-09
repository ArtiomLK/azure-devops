# DevOps

## Useful Commands

<https://github.com/ArtiomLK/commands>

## Deploy AZ VMSS that could be used as ADO Self Hosted Agents or GH Self Hosted Runners

```bash
# ------------------------------------------------------------------------------------------------
# INIT VARIABLES
# ------------------------------------------------------------------------------------------------
# If using Git Bash avoid C:/Program Files/Git/ being appended to some resources IDs
export MSYS_NO_PATHCONV=1
# ---
# Main Vars
# ---
sub_id="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx";                                echo $sub_id      # must update
app="ado-self-hosted-agents";                                                 echo $app
env="dev";                                                                    echo $env
l="eastus2";                                                                  echo $l
app_rg="rg-$app-$env-$l";                                                     echo $app_rg
tags="project=bicephub env=$env architecture=$app";                           echo $tags
user_n="artiomlk";                                                            echo $user_n

# ---
# NETWORK TOPOLOGY
# ---
vnet_pre="100.100";                                                           echo $vnet_pre
vnet_n="vnet-$app-$env-$l";                                                   echo $vnet_n
vnet_addr="$vnet_pre.2.0/23";                                                 echo $vnet_addr

# ---
# SSH Key & KV
# ---
kv_rg="rg-kv-$app-$env-$l";                                                   echo $kv_rg
kv_n="kv-hosted-agents";                                                      echo $kv_n
ssh_k_n="ssh-$app-$env-$l";                                                   echo $ssh_k_n

# ---
# Self Hosted Runners vmss (Linux)
# ---
lin_vmss_n="vmss-lin-$app-$env-$l";                                           echo $lin_vmss_n
lin_vmss_img="UbuntuLTS";                                                     echo $lin_vmss_img
lin_vmss_sku="Standard_D2_v5";                                                echo $lin_vmss_sku
lin_vmss_instance_count="1";                                                  echo $lin_vmss_instance_count
lin_snet_vmss_n="snet-lin-$app";                                              echo $lin_snet_vmss_n
lin_snet_addr_vmss="$vnet_pre.2.0/24";                                        echo $lin_snet_addr_vmss          # must update
lin_nsg_vmss_n="nsg-lin-$app";                                                echo $lin_nsg_vmss_n

# ---
# Self Hosted Runners vmss (Windows)
# ---
win_vmss_n="vmss-win-$app-$env-$l";                                           echo $win_vmss_n
win_vmss_img="Win2022Datacenter";                                             echo $win_vmss_img
win_vmss_sku="Standard_D2_v5";                                                echo $win_vmss_sku
win_vmss_instance_count="1";                                                  echo $win_vmss_instance_count
win_snet_vmss_n="snet-win-$app";                                              echo $win_snet_vmss_n
win_snet_addr_vmss="$vnet_pre.3.0/24";                                        echo $win_snet_addr_vmss          # must update
win_nsg_vmss_n="nsg-win-$app";                                                echo $win_nsg_vmss_n

# other images
# ['CentOS', 'Debian', 'Flatcar', 'openSUSE-Leap', 'RHEL', 'SLES', 'UbuntuLTS', 'Win2022Datacenter', 'Win2022AzureEditionCore', 'Win2019Datacenter', 'Win2016Datacenter', 'Win2012R2Datacenter', 'Win2012Datacenter', 'Win2008R2SP1'].

# ------------------------------------------------------------------------------------------------
# ------------------------------------------------------------------------------------------------
# DEPLOYMENT - CREATE COMPONENTS
# ------------------------------------------------------------------------------------------------
# ------------------------------------------------------------------------------------------------

# ------------------------------------------------------------------------------------------------
# RGs
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

# ------------------------------------------------------------------------------------------------
# VNET
# ------------------------------------------------------------------------------------------------
# Main vNet
az network vnet create \
--subscription $sub_id \
--name $vnet_n \
--resource-group $app_rg \
--address-prefixes $vnet_addr \
--location $l \
--tags $tags

# ------------------------------------------------------------------------------------------------
# KV
# ------------------------------------------------------------------------------------------------
# Key Vault
az keyvault create \
--subscription $sub_id \
--name $kv_n \
--resource-group $kv_rg \
--location $l \
--tags $tags

# ------------------------------------------------------------------------------------------------
# Create an SSH Keys
# ------------------------------------------------------------------------------------------------
# Generate new ssh pub nad private keys
az sshkey create \
--subscription $sub_id \
--name $ssh_k_n \
--resource-group $kv_rg \
--tags $tags

# ------------------------------------------------------------------------------------------------
# Create Linux SSH KEYS and store it on the KV
# ------------------------------------------------------------------------------------------------
# Set the private key file
ssh_priv_key_path='C:\Users\artioml\.ssh\1678318438_3874967'; echo $ssh_priv_key_path  # must update

# upload private key to the KV
az keyvault secret set --vault-name $kv_n --name $ssh_k_n --value "@$ssh_priv_key_path"

# retrieve secrets from KV
az keyvault secret show --vault-name $kv_n --name $ssh_k_n

# ------------------------------------------------------------------------------------------------
# Create Linux Self Hosted Agents VMSS
# ------------------------------------------------------------------------------------------------
# Self Hosted NSG with Default rules
az network nsg create \
--subscription $sub_id \
--resource-group $app_rg \
--name $lin_nsg_vmss_n \
--location $l \
--tags $tags

# Self Hosted Subnet
az network vnet subnet create \
--subscription $sub_id \
--resource-group $app_rg \
--vnet-name $vnet_n \
--name $lin_snet_vmss_n \
--address-prefixes $lin_snet_addr_vmss \
--network-security-group $lin_nsg_vmss_n

# vm scale set agents
az vmss create \
--subscription $sub_id \
--name $lin_vmss_n \
--resource-group $app_rg \
--image $lin_vmss_img \
--vm-sku $lin_vmss_sku \
--storage-sku StandardSSD_LRS \
--authentication-type SSH \
--ssh-key-values "$ssh_priv_key_path.pub" \
--vnet-name $vnet_n \
--subnet $lin_snet_vmss_n \
--instance-count $lin_vmss_instance_count \
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
--name $lin_vmss_n \
--output table

# ------------------------------------------------------------------------------------------------
# Store Windows Password on the Azure Key Vault
# ------------------------------------------------------------------------------------------------
temp_pass=$(chars="abcd1234ABCD\!@#$%^&*()"; for i in {1..25}; do echo -n "${chars:RANDOM%${#chars}:1}"; done; echo); echo $temp_pass

az keyvault secret set --vault-name $kv_n --name "win-ado-pass" --value $temp_pass
az keyvault secret show --vault-name $kv_n --name "win-ado-pass" --query "value"

# ------------------------------------------------------------------------------------------------
# Create Windows Self Hosted Agents VMSS
# ------------------------------------------------------------------------------------------------
# Self Hosted NSG with Default rules
az network nsg create \
--subscription $sub_id \
--resource-group $app_rg \
--name $win_nsg_vmss_n \
--location $l \
--tags $tags

# Self Hosted Subnet
az network vnet subnet create \
--subscription $sub_id \
--resource-group $app_rg \
--vnet-name $vnet_n \
--name $win_snet_vmss_n \
--address-prefixes $win_snet_addr_vmss \
--network-security-group $win_nsg_vmss_n

# vm scale set agents
az vmss create \
--subscription $sub_id \
--name $win_vmss_n \
--resource-group $app_rg \
--image $win_vmss_img \
--vm-sku $win_vmss_sku \
--storage-sku StandardSSD_LRS \
--vnet-name $vnet_n \
--subnet $win_snet_vmss_n \
--instance-count $win_vmss_instance_count \
--disable-overprovision \
--upgrade-policy-mode manual \
--single-placement-group false \
--platform-fault-domain-count 1 \
--load-balancer "" \
--assign-identity \
--admin-username $user_n \
--admin-password $temp_pass \
--tags $tags

# Verify Manual update policy
az vmss show \
--subscription $sub_id \
--resource-group $app_rg \
--name $win_vmss_n \
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
while [ $i -le $lin_vmss_instance_count ]
do
   echo Number: $i
   az vmss run-command invoke \
   --resource-group $app_rg \
   --name $lin_vmss_n \
   --instance-id $i \
   --command-id RunShellScript \
   --scripts "curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash"
   ((i++))
done

# Install Bicep
i=0; echo "init i = $i"
while [ $i -le $lin_vmss_instance_count ]
do
   echo Number: $i
   az vmss run-command invoke \
   --resource-group $app_rg \
   --name $lin_vmss_n \
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
