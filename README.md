![llogo.png](/.image/livingston-logo.png)

**Setting up MDT for Imaging and Deployment**

**Version 1.2**

**Revision History**
| **Date** | **Revision \#** | **Editor** | **Description of Change** |
|:-----------:|:-----------:|:-----------:|:-----------:|
| 12/07/2021 | v1.0 | John Fogarty  | Initial Revision |
| 12/17/2021 | v1.1 | John Fogarty  | Additional Documentation |
| 12/23/2021 | v1.2 | John Fogarty  | Expanded Deployment Workbench steps |

----

# Overall Architecture
Multisite MDT is made up of 3 distinct types of deployment servers.  

1. The DFS primary deployment share
2. The DFS read only deployment share
3. The Lab deployment share

You can see in the diagram below how these pieces flow together to keep all MDT sites up to date.

[![dfs-diagram.png](/.image/dfs-diagram.png)](https://lucid.app/lucidchart/9b876d58-992d-4a1d-8ca1-5d85899975e6/edit?invitationId=inv_92c65d1d-caee-4acb-9009-523605ece512)

----
# Configuration
## DFS primary deployment share

## DFS read-only deployment shares


## Lab
The lab setup is the most important part of MDT.  You will build your images in the lab, and test your deployments in the lab before rolling them out to the DFS primary deployment share.

There are two ways to run the lab.  Either as a dedicated MDT server with multiple deployment shares, and a hyper-v node for client machines, or run the entire thing in Hyper-V.  This document will cover the entire process existing in Hyper-V, you will need to make some changes if that is not the way you approach the lab.

Execute the script below, this will create an MDT server, as well as 6 client VMs for various Windows installs.  This assumes your C: drive is where your space is located.  If that is not the case, please update the $VMLOC variable.

### Hyper-V Lab creation script
```powershell
#Path of the VM HDD file stored
$VMLOC = "c:\hyper-v"

#Name of virtual switch which will be used in the VMs
$VMNet = "Default Switch"

#Create the VM's
write-host "MDT Server"
$VM = "labwcinfmdt00"
New-VM -Name $VM -Generation 2 -SwitchName $VMNet
New-VHD -Path "$VMLOC\$VM\$vm.vhdx" -Dynamic -SizeBytes 256GB
ADD-VMHardDiskDrive -VMName $vm -Path "$VMLOC\$VM\$vm.vhdx"
New-VHD -Path "$VMLOC\$VM\$vm-1.vhdx" -Dynamic -SizeBytes 512GB
ADD-VMHardDiskDrive -VMName $vm -Path "$VMLOC\$VM\$vm-1.vhdx"
Add-VMDvdDrive -VMName $vm 
Set-VM $VM -MemoryStartupBytes 8GB -AutomaticCheckpointsEnabled $false

write-host "MDT client vms"
$VMName = 'WIN10-21H2-A','WIN10-21H2-B','WIN10-LTSC21-A','WIN10-LTSC21-B','WIN11-21H2-A','WIN11-21H2-B'
Foreach($vm in $VMName) { 
    New-VM -Name $VM -Generation 2 -SwitchName $VMNet
    New-VHD -Path "$VMLOC\$VM\$vm.vhdx" -Dynamic -SizeBytes 256GB
    ADD-VMHardDiskDrive -VMName $vm -Path "$VMLOC\$VM\$vm.vhdx"
    Set-VM $VM -MemoryStartupBytes 2GB -AutomaticCheckpointsEnabled $false
    Set-VMFirmware -VMName $vm -FirstBootDevice ((Get-VMFirmware -VMName $vm).BootOrder | Where-Object Device -like *Network*).Device
    Checkpoint-VM -Name $vm -SnapshotName BeforeInstall
}
```
After the script has finished, mount your Windows 2019 DVD to the MDT server and boot it.  Install a 2019 server with desktop experience.  Once that is complete, install the following items.

* [The Windows ADK for Windows 10](https://go.microsoft.com/fwlink/?linkid=2086042)
* [The Windows PE add-on for the ADK](https://go.microsoft.com/fwlink/?linkid=2087112)
* [Microsoft Deployment Toolkit](https://www.microsoft.com/en-us/download/confirmation.aspx?id=54259)
* Windows Deployment Services (Powershell below)
```powershell
    Install-WindowsFeature wds-deployment -includemanagementtools
```


Lets create your DFS Read Replica of the deployment share. **TBD**
```powershell
Install-WindowsFeature -Name FS-DFS-Namespace,FS-DFS-Replication –IncludeManagementTools
```
https://docs.microsoft.com/en-us/powershell/module/dfsr/set-dfsrmembership?view=windowsserver2019-ps


Now lets create the Folder structure for your deployment shares.
```powershell
mkdir d:\mdt
robocopy \\tr2wcinfmdt00\lii-deploy$ d:\mdt\lii-deploy /mir /r:2 /w:1
robocopy d:\mdt\lii-deploy d:\mdt\lii-image /mir /r:2 /w:1
robocopy \\tr2wcinfmdt00\d$\mdt\source d:\mdt\source /mir /r:2 /w:1
```

```powershell
$DeploymentShareNTFS = "d:\mdt\lii-deploy"
new-smbshare -Name "lii-deploy$" -path $DeploymentShareNTFS -ChangeAccess "Everyone" -FullAccess "Administrators"
icacls $DeploymentShareNTFS /grant '"Users":(OI)(CI)(RX)'
icacls $DeploymentShareNTFS /grant '"Administrators":(OI)(CI)(F)'
icacls $DeploymentShareNTFS /grant '"SYSTEM":(OI)(CI)(F)'

$ImageDeploymentShareNTFS = "d:\mdt\lii-image"
new-smbshare -Name "lii-image$" -path $ImageDeploymentShareNTFS -ChangeAccess "Everyone" -FullAccess "Administrators"
icacls $ImageDeploymentShareNTFS /grant '"Users":(OI)(CI)(RX)'
icacls $ImageDeploymentShareNTFS /grant '"Administrators":(OI)(CI)(F)'
icacls $ImageDeploymentShareNTFS /grant '"SYSTEM":(OI)(CI)(F)'
icacls "$ImageDeploymentShareNTFS\Captures" /grant '"Administrators":(OI)(CI)(M)'
```

### Deployment Workbench

1. Launch Deployment Workbench
![deployment-workbench.png](/.image/deployment-workbench.png)
2. Right click on Deployment Shares and choose open Deployment Share
![dw-open-deployment-share.png](/.image/dw-open-deployment-share.png)
3. Choose d:\mdt\lii-deploy
![dw-deployment-share-path.png](/.image/dw-deployment-share-path.png)
4. Click Next
5. Click Finish
6. Right click the share and choose properties
![dw-properties.png](/.image/dw-properties.png)
7. Update the Network UNC path with your server name.
![dw-unc-path.png](/.image/dw-unc-path.png)
8. Click the rules tab, and then click Edit Bootstrap.ini
9. Update DeployRoot value
![dw-boot.png](/.image/dw-boot.png)
10. Close and Save file
11. Click Ok
12. Right click the share and choose update Deployment share
![dw-update-share.png](/.image/dw-update-share.png)
13. Click Next
14. Click Next
15. Click Finish

In the Image share, import any new OS from DVD that you need to capture an image from "d:\mdt\source\Operating Systems"

### Windows Deployment Services
Once you have a working image in your Boot folder for either Deploy or Image we need to add the images to the Windows Deployment Service.

```powershell
wdsutil /initialize-server /remInst:"d:\RemoteInstall"
Import-WdsBootImage -Path D:\mdt\lii-deploy\Boot\LiteTouchPE_x64.wim -NewImageName "lii-deploy" -NewDescription "LII Deployment Share" -DisplayOrder "10"
Import-WdsBootImage -Path D:\mdt\lii-image\Boot\LiteTouchPE_x64.wim -NewImageName "lii-image" -NewDescription "LII Image Share" -DisplayOrder "1000"
```

### Configure WDS to respond to any client.
1. Launch the WDS management console
2. Expand Servers, and right click the server you are working on and choose properties.
3. Go to the PXE Response tab and choose Respond to all client computers (known and unknown)
4. Click ok

# Appendix

## At Home MDT

### File Copy
create the Folder structure for your deployment shares.
```powershell
mkdir d:\mdt
robocopy \\tr2wcinfmdt00\lii-deploy$ d:\mdt\lii-deploy /mir /r:2 /w:1
robocopy \\tr2wcinfmdt00\d$\mdt\source d:\mdt\source /mir /r:2 /w:1
```

```powershell
$DeploymentShareNTFS = "d:\mdt\lii-deploy"
new-smbshare -Name "lii-deploy$" -path $DeploymentShareNTFS -ChangeAccess "Everyone" -FullAccess "Administrators"
icacls $DeploymentShareNTFS /grant '"Users":(OI)(CI)(RX)'
icacls $DeploymentShareNTFS /grant '"Administrators":(OI)(CI)(F)'
icacls $DeploymentShareNTFS /grant '"SYSTEM":(OI)(CI)(F)'
```

### Install MDT
Install the following items, if you have run the robocopy you can find these in d:\mdt\mdt_install.

* [The Windows ADK for Windows 10](https://go.microsoft.com/fwlink/?linkid=2086042)
* [The Windows PE add-on for the ADK](https://go.microsoft.com/fwlink/?linkid=2087112)
* [Microsoft Deployment Toolkit](https://www.microsoft.com/en-us/download/confirmation.aspx?id=54259)
* Windows Deployment Services (Powershell below)
```powershell
Install-WindowsFeature wds-deployment -includemanagementtools
```
### Deployment Workbench

1. Launch Deployment Workbench  
![deployment-workbench.png](/.image/deployment-workbench.png)
2. Right click on Deployment Shares and choose open Deployment Share  
![dw-open-deployment-share.png](/.image/dw-open-deployment-share.png)
3. Choose d:\mdt\lii-deploy  
![dw-deployment-share-path.png](/.image/dw-deployment-share-path.png)
4. Click Next
5. Click Finish
6. Right click the share and choose properties  
![dw-properties.png](/.image/dw-properties.png)
7. Update the Network UNC path with your server name.  
![dw-unc-path.png](/.image/dw-unc-path.png)
8. Click the rules tab, and then click Edit Bootstrap.ini
9. Update DeployRoot value  
![dw-boot.png](/.image/dw-boot.png)
10. Close and Save file
11. Click Ok
12. Right click the share and choose update Deployment share  
![dw-update-share.png](/.image/dw-update-share.png)
13. Click Next
14. Click Next
15. Click Finish


### Windows Deployment Services
Once you have a working image in your Boot folder for either Deploy or Image we need to add the images to the Windows Deployment Service.

```powershell
wdsutil /initialize-server /remInst:"d:\RemoteInstall"
Import-WdsBootImage -Path D:\mdt\lii-deploy\Boot\LiteTouchPE_x64.wim -NewImageName "lii-deploy" -NewDescription "LII Deployment Share" -DisplayOrder "10"
```

### Configure WDS to respond to any client.
1. Launch the WDS management console
2. Expand Servers, and right click the server you are working on and choose properties.
3. Go to the PXE Response tab and choose Respond to all client computers (known and unknown)
4. Click ok

## Troubleshooting 

### Re-Configure NTFS Permissions for the MDT Build Lab deployment share if needed
```powershell
$DeploymentShareNTFS = "d:\mdt\lii-deploy"
icacls $DeploymentShareNTFS /grant '"Users":(OI)(CI)(RX)'
icacls $DeploymentShareNTFS /grant '"Administrators":(OI)(CI)(F)'
icacls $DeploymentShareNTFS /grant '"SYSTEM":(OI)(CI)(F)'
icacls "$DeploymentShareNTFS\Captures" /grant '"Administrators":(OI)(CI)(M)'

## Configure Sharing Permissions for the MDT Build Lab deployment share
$DeploymentShare = "lii-Deploy$"
Grant-SmbShareAccess -Name $DeploymentShare -AccountName "EVERYONE" -AccessRight Change -Force
Revoke-SmbShareAccess -Name $DeploymentShare -AccountName "CREATOR OWNER" -Force
```

## Reference Links

Here are all the links I used as reference material as I reverse engineered the previous MDT build as well as planning for a multi-site easily supportable MDT design going forward.

[Windows 10 Deployment with MDT](https://docs.microsoft.com/en-us/windows/deployment/deploy-windows-mdt/prepare-for-windows-deployment-with-mdt)  
[Office 365 as part of an image](https://docs.microsoft.com/en-us/deployoffice/deploy-microsoft-365-apps-operating-system-image)   
[Configure MDT](https://mcpmag.com/articles/2018/12/13/configure-wds-using-powershell.aspx)  
[Hyper-V Lab Setup](https://malwaremily.medium.com/install-ad-ds-dns-and-dhcp-using-powershell-on-windows-server-2016-ac331e5988a7)  
