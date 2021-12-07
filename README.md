![llogo.png](/.image/livingston-logo.png)

**Setting up MDT for Imaging and Deployment**

**Version 1.0**

**Revision History**
| **Date** | **Revision \#** | **Editor** | **Description of Change** |
|:-----------:|:-----------:|:-----------:|:-----------:|
| 12/07/2021 | v1.0 | John Fogarty  | Initial Revision |


## Overall Architecture
Multisite MDT is made up of 3 distinct types of deployment servers.  

1. The DFS primary deployment share
2. The DFS read only deployment share
3. The Lab deployment share

You can see in the diagram below how these pieces flow together to keep all MDT sites up to date.

![dfs-diagram.png](https://johnafogarty4.github.io/mdt/.image/dfs-diagram.png)

# Check for elevation
Write-Host "Checking for elevation"

If (-NOT ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole(`
    [Security.Principal.WindowsBuiltInRole] "Administrator"))
{
    Write-Warning "Oupps, you need to run this script from an elevated PowerShell prompt!`nPlease start the PowerShell prompt as an Administrator and re-run the script."
    Write-Warning "Aborting script..."
    Break
}

# Configure NTFS Permissions for the MDT Build Lab deployment share
$DeploymentShareNTFS = "d:\mdt\lii-deploy"
icacls $DeploymentShareNTFS /grant '"Users":(OI)(CI)(RX)'
icacls $DeploymentShareNTFS /grant '"Administrators":(OI)(CI)(F)'
icacls $DeploymentShareNTFS /grant '"SYSTEM":(OI)(CI)(F)'
icacls "$DeploymentShareNTFS\Captures" /grant '"Administrators":(OI)(CI)(M)'

# Configure Sharing Permissions for the MDT Build Lab deployment share
$DeploymentShare = "lii-Deploy$"
Grant-SmbShareAccess -Name $DeploymentShare -AccountName "EVERYONE" -AccessRight Change -Force
Revoke-SmbShareAccess -Name $DeploymentShare -AccountName "CREATOR OWNER" -Force

# Generation 1 - only win10-21h2-a
$vm = 'jafwcinfmdt01'
#Path of the VM HDD file stored
$VMLOC = "c:\hyper-v"

#Name of virtual switch which will be used in the VMs
$VMNet = "Lab"

New-VM -Name $VM -Generation 2 -SwitchName $VMNet
New-VHD -Path "$VMLOC\$VM\$vm.vhdx" -Dynamic -SizeBytes 256GB
ADD-VMHardDiskDrive -VMName $vm -Path "$VMLOC\$VM\$vm.vhdx"
Set-VM $VM -MemoryStartupBytes 8GB -AutomaticCheckpointsEnabled $false
Copy-Item '\\192.168.70.10\mp-image$\boot\LiteTouchPE_x64.iso' C:\hyper-v\
Add-VMDvdDrive -VMName $vm -Path c:\hyper-v\LiteTouchPE_x64.iso
$dvd = Get-VMDVDDrive -VMName $vm
Set-VMFirmware -VMName $vm -FirstBootDevice $dvd

Checkpoint-VM -Name $vm -SnapshotName BeforeInstall


# Generation 2 - all the rest
$VMName = 'WIN10-21H2-A','WIN10-21H2-B','WIN10-LTSC21-A','WIN10-LTSC21-B','WIN11-21H2-A','WIN11-21H2-B','WIN11-21H2-C'
#Path of the VM HDD file stored
$VMLOC = "c:\hyper-v"

#Name of virtual switch which will be used in the VMs
$VMNet = "External - Ethernet"

#Create the VM's
Foreach($vm in $VMName) { New-VM -Name $VM -Generation 2 -SwitchName $VMNet
New-VHD -Path "$VMLOC\$VM\$vm.vhdx" -Dynamic -SizeBytes 256GB
ADD-VMHardDiskDrive -VMName $vm -Path "$VMLOC\$VM\$vm.vhdx"
Set-VM $VM -MemoryStartupBytes 8GB -AutomaticCheckpointsEnabled $false

Set-VMFirmware -VMName $vm -FirstBootDevice ((Get-VMFirmware -VMName $vm).BootOrder | Where-Object Device -like *Network*).Device

Checkpoint-VM -Name $vm -SnapshotName BeforeInstall

}
