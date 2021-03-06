# How to prepare and upload a Windows VHD file for use in an Azure VM 

## How to prepare
### 1. Get a windows image in VHD format

      This could be any windows builds.
      Eg: You can get it straight from a place like ...\release\rs4_release\17134.1.180410-1804\amd64fre\vhd\vhd_server_serverdatacenter_en-us
     
      Copy it to a local directory. eg c:\temp, let say it's named WindowsRS4.vhd
      
### 2. Make it a Fixed format VHD and make it bootable 
   
     Make sure you are running the following steps on a machine with Hyper-v feature enabled
   - copy unattended.xml and MakeWindowsVHDBootable.ps1 from https://github.com/soccerGB/Tools/tree/master/Azure/prepAzureVHD repro to your local directory (eg c:\temp)  

   - In an elevated powershell windows, run the following script to generate a bootable VHD:
     PS D:\github\Tools\Azure\prepAzureVHD> ./MakeWindowsVHDBootable.ps1 WindowsRS3.vhd

     This script expands the dynamic VHD to 24 GB size and converts it to fixed VHD format before creating a VM ("vmname") and boot into Windows unattendedly
     It also creates an administrator account with the following credential:
     
     - Username: administrator
     - Password: replacepassword1234$
  
### 3. Prep Windows VHD image for Azure
  
   - Connect to the "vmname" VM from the Hyper-V Manager
   - Enable remote desktop connection for the machine
   
      You can do it in powershell:
      
            Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server" -Name "fdenyTSXonnections" -Value 0
            Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
            
     Or
      
       From UI in Control Panel\System and Security\System\Remote settings if it's for a build with UI
   
   - Enable Hyper-V and Containers feature
   
            Enable-WindowsOptionalFeature -Online -FeatureName containers –All
            Enable-WindowsOptionalFeature -Online -FeatureName:Microsoft-Hyper-V -All
            
            ps: only reboot after activating the second feature.

   - [Install Docker EE](https://docs.docker.com/engine/installation/windows/docker-ee/#install-docker-ee)
      - Install-Module DockerProvider -Force
      - Install-Package Docker -ProviderName DockerProvider -Force
      
      ( you might need to add an external network interface to enable access to internet for downloading files)

   - Preload matching Docker servercore Docker container image to the VM
   
      eg:
      
               on RS3: 
                  docker pull microsoft/nanoserver:1709
                  docker pull microsoft/windowsservercore:1709

               on RS4
                  docker pull microsoft/nanoserver:1803 
                  docker pull microsoft/windowsservercore:1803      
      
      ps. For an internal build, you can find its servercore image in \amd64fre\containerbaseospkgs\cbaseospkg_serverdatacentercore_en-us
   
   - Optional step: [Install the VM Agent](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/agent-user-guide)
     downalod VM Agent binary package before installing with the following command 
     
     msiexec.exe /i WindowsAzureVmAgent.2.7.1198.802.rd_art_stable.170327-1033.fre /quiet
     
   - Generalize the image using sysprep tool
      Run "c:\windows\system32\sysprep\sysprep.exe /generalize /oobe /shutdown" to [sysprep] (https://docs.microsoft.com/en-us/azure/virtual-machines/windows/classic/createupload-vhd) a VHD  
      
   - You can delete the VM("vmname") from the Hyper-V Manager 

      The output image will be named Azure+"WindowsRS3.vhd" in the current directory
   
      If you decide to skip the image generating process from step 1-3, you can get a copy of what I used in [here](     
      https://ms.portal.azure.com/#blade/Microsoft_Azure_Storage/ContainersBlade/storageAccountId/%2Fsubscriptions%2Fe5839dfd-61f0-4b2f-b06f-de7fc47b5998%2FresourceGroups%2Fsoccerl-storage%2Fproviders%2FMicrosoft.Storage%2FstorageAccounts%2Fsoccerlstorage)
      
## How Upload to the Azure 

   Install [Azure CLI2.0] (https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
   
   Assuming you have an existing Azure subscription, the following information is what I used in my example on Azure CLI 2.0
   - resource group name: soccerl-storage 
   - storage account name:soccerlstorage
   - storage container name:rs4container 
   - blob name:AzureWindowsRS4.vhd
   - disk name:rs4disk

      - az login
      - az group create --name soccerl-storage --location westus2
      - az storage account create --resource-group soccerl-storage --location westus2 --name soccerlstorage --kind Storage --sku Standard_LRS
      
          Get the value of the account-key (key1) from the following for use in the later commands        
      - az storage account keys list --resource-group soccerl-storage --account-name soccerlstorage
      
            PS D:\github\tools\Azure\prepAzureVHD> az storage account keys list --resource-group soccerl-storage --account-name soccerlstorage
            [
              {
                "keyName": "key1",
                "permissions": "Full",
                "value": "CSNUTol7N8rlbI8nafJeuuSdhy8YD5k5SoiLwtHDUnWGcNkUzPT101uMpfD9Wx3UCm40YsRBE4k5TicqrG6TIg=="
              },
              {
                "keyName": "key2",
                "permissions": "Full",
                "value": "RST63FdGsimA6hY5j/agDv+cS0/lprTYVqT11DaGWSoYc+cXXqxK/zcwhioJL5cJGXAVmGgERLT7aG7lQjEa5A=="
              }
            ]
      
      - az storage container create -n rs4container --account-name soccerlstorage --account-key "CSNUTol7N8rlbI8nafJeuuSdhy8YD5k5SoiLwtHDUnWGcNkUzPT101uMpfD9Wx3UCm40YsRBE4k5TicqrG6TIg=="

      - az storage blob upload --account-name soccerlstorage --account-key "CSNUTol7N8rlbI8nafJeuuSdhy8YD5k5SoiLwtHDUnWGcNkUzPT101uMpfD9Wx3UCm40YsRBE4k5TicqrG6TIg==" --container-name rs4container --type page --file ./AzureWindowsRS4.vhd --name AzureWindowsRS4.vhd

      - az storage blob url    --account-name soccerlstorage --account-key "CSNUTol7N8rlbI8nafJeuuSdhy8YD5k5SoiLwtHDUnWGcNkUzPT101uMpfD9Wx3UCm40YsRBE4k5TicqrG6TIg==" --container-name rs4container --name AzureWindowsRS4.vhd

      - az disk create --resource-group soccerl-storage --name rs4disk --source "https://soccerlstorage.blob.core.windows.net/rs4container/AzureWindowsRS4.vhd"

      - az vm create --resource-group soccerl-storage  --location westus2 --name wrs4vm --os-type windows --attach-os-disk rs4disk --size Standard_D2s_v3

After the the VM is successfully created, you would need to wait a few minutes for the Windows to fully boot before you could go to the Azure portal to remote-desktop to it successfully (eg  mstsc.exe /v:52.219.2.2 )
