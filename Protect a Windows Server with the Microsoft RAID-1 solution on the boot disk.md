Microsoft Windows offers since Windows 10 a software RAID-1 solution, managed by the OS.
For the solution to work, we need 2 disks -- a primary and a mirror disk, with at least the same capacity as the disk to be mirrored.

The challenge of doing it on the boot disk is not related to the disk itself, but the fact that when a disk breaks, you still want to be able to boot the system, thus, the boot manager and boot loader needs to exist on the replica disk.

This article explores how to do it on an machine running Windows Server, in which a second disk was added to implement RAID-1.

## Pre-Requisites
This article assumes:
- A machine running Windows Server 2025 Standard Edition (Core)
- The machine supports [UEFI](https://uefi.org/specifications)
- A second empty disk was added to the machine 
- You can login with an account that belongs to the local Administrator group
## Understanding the Windows Partition Layout
In order to understand 


## Execution Steps
### Prepare the Replica Disk
### Create the RAID-1 Array
### Replicate the Boot Manager
