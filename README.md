![llogo.png](/.image/livingston-logo.png)

**Setting up MDT for Imaging and Deployment**

**Version 1.0**

**Revision History**
| **Date** | **Revision \#** | **Editor** | **Description of Change** |
|:-----------:|:-----------:|:-----------:|:-----------:|
| 12/07/2021 | v1.0 | John Fogarty  | Initial Revision |

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
After the script has finished, mount your Windows 2019 DVD to the MDT server and boot it.  Install a full blown 2019 server with desktop experience.  Once that is complete, install the following items.

* [The Windows ADK for Windows 10](https://go.microsoft.com/fwlink/?linkid=2086042)
* [The Windows PE add-on for the ADK](https://go.microsoft.com/fwlink/?linkid=2087112)
* [Microsoft Deployment Toolkit](https://www.microsoft.com/en-us/download/confirmation.aspx?id=54259)
* Windows Deployment Services (Powershell below)
```powershell
    Install-WindowsFeature wds-deployment -includemanagementtools
```
Now lets create the Folder structure for your deployment shares.

```shell
mkdir d:\mdt
mkdir d:\source
mkdir d:\mdt\lii-deploy
mkdir d:\mdt\lii-image
robocopy \\tr2wcinfmdt00\lii-deploy$ d:\mdt\lii-deploy /mir /r:2 /w:1
robocopy \\tr2wcinfmdt00\d$\mdt\source d:\mdt\source /mir /r:2 /w:1
```

```powershell
$DeploymentShareNTFS = "d:\mdt\lii-deploy"
new-smbshare -Name "lii-deploy$" -path $DeploymentShareNTFS -ChangeAccess "Everyone" -FullAccess "Administrators"
icacls $DeploymentShareNTFS /grant '"Users":(OI)(CI)(RX)'
icacls $DeploymentShareNTFS /grant '"Administrators":(OI)(CI)(F)'
icacls $DeploymentShareNTFS /grant '"SYSTEM":(OI)(CI)(F)'

$ImageDeploymentShareNTFS = "d:\mdt\lii-image"
new-smbshare -Name "lii-deploy$" -path $ImageDeploymentShareNTFS -ChangeAccess "Everyone" -FullAccess "Administrators"
icacls $ImageDeploymentShareNTFS /grant '"Users":(OI)(CI)(RX)'
icacls $ImageDeploymentShareNTFS /grant '"Administrators":(OI)(CI)(F)'
icacls $ImageDeploymentShareNTFS /grant '"SYSTEM":(OI)(CI)(F)'
icacls "$ImageDeploymentShareNTFS\Captures" /grant '"Administrators":(OI)(CI)(M)'
```

Now you should be able to open both Deployment Shares in Deployment Workbench.



## Deployment Toolbox


## Windows Deployment Services

# Appendix
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
