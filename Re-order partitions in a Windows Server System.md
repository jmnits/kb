This script explains how to re-order partitions in a Windows Server disk.

## Environment
In the current script, we are using a Windows Server 2025 Core with 2 disks of 1 TB, as described by the [[Using Microsoft RAID 1 to Mirror the Windows Server Boot Drive]] article.

Since we have one 1 TB empty disk, we'll use this one to backup the relevant partitions, before move them.
![[part-table-changes.png]]

The following considerations must be taken:
- The *PAGEFILE* partition only contains the page file; this can be re-created on another disk, and requires no backup.
- Both, *Recovery* and *CACHE* partitions should be backed so we can restore their contents
- The *CACHE* directory might be accessed by the applications in the server; to avoid this partition to be in use, the partition backup should be done in *safe mode*

This leave us with the following action plan:

| Action                       | *Recovery*                              | *PAGEFILE*                              | *CACHE*                                        |
| ---------------------------- | --------------------------------------- | --------------------------------------- | ---------------------------------------------- |
| Stop using the partitions    | Stop recovery agent (use `reagentc`)    | Move pagefile to another location       | Reboot in safe mode (minimal) -- use `bcdedit` |
| Backup partition data        | Use `dism` utility                      | --                                      | Use `dism` utility                             |
| Redo the partitions          | Use `diskpart` utility                  | Use `diskpart` utility                  | Use `diskpart` utility                         |
| Restore partition data       | Use `dism` utility                      | --                                      | Use `dism` utility                             |
| Restart using the partitions | Restart recovery agent (use `reagentc`) | Move pagefile to the PAGEFILE partition | Reboot in normal mode -- use `bcdedit`         |
 
As the current system contains an empty disk of 1 TB, we'll use it to backup the *Recovery* and *CACHE* partitions.
## Log Book

> For all the actions made below, you need to access the server, with a local administrator account.

### Prepare System
1. Check the current disk layout, partitions and volumes
```
PS C:\> 'list disk' | diskpart

Microsoft DiskPart version 10.0.26100.1150

Copyright (C) Microsoft Corporation.
On computer: SERVER-01

DISKPART>
  Disk ###  Status         Size     Free     Dyn  Gpt
  --------  -------------  -------  -------  ---  ---
  Disk 0    Online         1024 GB      0 B        *
  Disk 1    Online         1024 GB  1024 GB

DISKPART>
PS C:\> 'sel disk 0;list par' -split ´;´ | diskpart

Microsoft DiskPart version 10.0.26100.1150

Copyright (C) Microsoft Corporation.
On computer: SERVER-01

DISKPART>
Disk 0 is now the selected disk.

DISKPART>
  Partition ###  Type              Size     Offset
  -------------  ----------------  -------  -------
  Partition 1    System             100 MB  1024 KB
  Partition 2    Reserved            16 MB   101 MB
  Partition 3    Primary             69 GB   117 MB
  Partition 4    Recovery           674 MB    69 GB
  Partition 5    Primary             16 GB    70 GB
  Partition 6    Primary            937 GB    86 GB

DISKPART>
PS C:\> 'sel disk 0;sel par 4;det par' -split ´;´ | diskpart

Microsoft DiskPart version 10.0.26100.1150

Copyright (C) Microsoft Corporation.
On computer: SERVER-01

DISKPART>
Disk 0 is now the selected disk.

DISKPART>
Partition 4 is now the selected partition.

DISKPART>
Partition 4
Type    : de94bba4-06d1-4d40-a16a-bfd50179d6ac
Hidden  : Yes
Required: Yes
Attrib  : 0X8000000000000001
Offset in Bytes: 74576822272

  Volume ###  Ltr  Label        Fs     Type        Size     Status     Info
  ----------  ---  -----------  -----  ----------  -------  ---------  --------
* Volume 5                      NTFS   Partition    674 MB  Healthy    Hidden

DISKPART>
PS C:\> 'list vol' | diskpart

Microsoft DiskPart version 10.0.26100.1150

Copyright (C) Microsoft Corporation.
On computer: SERVER-01

DISKPART>
  Volume ###  Ltr  Label        Fs     Type        Size     Status     Info
  ----------  ---  -----------  -----  ----------  -------  ---------  --------
  Volume 0     D   VBox_GAs_7.  CDFS   CD-ROM        52 MB  Healthy
  Volume 1     C                NTFS   Partition     69 GB  Healthy    Boot
  Volume 2         PAGEFILE     NTFS   Partition     16 GB  Healthy
    C:\VMEM\
  Volume 3     K   CACHE        NTFS   Partition    937 GB  Healthy
  Volume 4                      FAT32  Partition    100 MB  Healthy    System
  Volume 5                      NTFS   Partition    674 MB  Healthy    Hidden
  
DISKPART>
```
2. Check the page file configuration
```
PS C:\> gwmi Win32_ComputerSystem | select AutomaticManagedPagefile

AutomaticManagedPagefile
------------------------
                   False


PS C:\> gwmi Win32_PageFileSetting -EnableAllPrivileges

MaximumSize Name                 Caption
----------- ----                 -------
          0 C:\VMEM\pagefile.sys C:\ 'VMEM\pagefile.sys'
```
3. Prepare the backup area using disk 1
```
PS C:\> 'sel disk 1;clean;create par pri;format fs=ntfs quick label=BACKUP;assign letter x' -split ';' | diskpart

Microsoft DiskPart version 10.0.26100.1150

Copyright (C) Microsoft Corporation.
On computer: SERVER-01

DISKPART>
Disk 1 is now the selected disk.

DISKPART>
DiskPart succeeded in cleaning the disk.

DISKPART>
DiskPart succeeded in creating the specified partition.

DISKPART>
  100 percent completed

DiskPart successfully formatted the volume.

DISKPART>
DiskPart successfully assigned the drive letter or mount point.

DISKPART>
```
4. Move the pagefile to X:\ (only effective after the next reboot)
```
PS C:\> gwmi Win32_PageFileSetting -EnableAllPrivileges

MaximumSize Name                 Caption
----------- ----                 -------
          0 C:\VMEM\pagefile.sys C:\ 'VMEM\pagefile.sys'
          
PS C:\> Set-WmiInstance -Class Win32_PageFileSetting -Arguments @{Name='X:\pagefile.sys'; InitialSize=0; MaximumSize=0 } | Out-Null
PS C:\> gwmi Win32_PageFileSetting -EnableAllPrivileges

MaximumSize Name                 Caption
----------- ----                 -------
          0 C:\VMEM\pagefile.sys C:\ 'VMEM\pagefile.sys'
          0 X:\pagefile.sys      X:\ 'pagefile.sys'


PS C:\> (gwmi Win32_PageFileSetting -EnableAllPrivileges | ?{ $_.Name -match 'C:' }).Delete()
PS C:\> gwmi Win32_PageFileSetting -EnableAllPrivileges

MaximumSize Name            Caption
----------- ----            -------
          0 X:\pagefile.sys X:\ 'pagefile.sys'
```
4. Disable Recovery agent
```
PS C:\> ReAgentc.exe /info
Windows Recovery Environment (Windows RE) and system reset configuration
Information:

    Windows RE status:         Enabled
    Windows RE location:       \\?\GLOBALROOT\device\harddisk0\partition4\Recovery\WindowsRE
    Boot Configuration Data (BCD) identifier: 8e73986c-289e-11f0-bf53-a4d06eada308
    Recovery image location:
    Recovery image index:      0
    Custom image location:
    Custom image index:        0

REAGENTC.EXE: Operation Successful.
PS C:\> ReAgentc.exe /disable
REAGENTC.EXE: Operation Successful.
PS C:\> ReAgentc.exe /info
Windows Recovery Environment (Windows RE) and system reset configuration
Information:

    Windows RE status:         Disabled
    Windows RE location:
    Boot Configuration Data (BCD) identifier: 00000000-0000-0000-0000-000000000000
    Recovery image location:
    Recovery image index:      0
    Custom image location:
    Custom image index:        0

REAGENTC.EXE: Operation Successful.
```
5. Reboot in safe mode (minimal)
```
PS C:\> bcdedit.exe /enum '{default}'

Windows Boot Loader
-------------------
identifier              {current}
device                  partition=C:
path                    \WINDOWS\system32\winload.efi
description             Windows Server
locale                  en-US
inherit                 {bootloadersettings}
displaymessageoverride  Recovery
recoveryenabled         No
isolatedcontext         Yes
allowedinmemorysettings 0x15000075
osdevice                partition=C:
systemroot              \WINDOWS
resumeobject            {8e73986a-289e-11f0-bf53-a4d06eada308}
nx                      OptOut
PS C:\> bcdedit.exe /set '{default}' safeboot minimal
The operation completed successfully.
PS C:\> Restart-Computer
```
### Move Partitions
Since the partitions are now *not in use*, we need to back up the meaningful ones -- *Recovery* and *CACHE* -- before delete them and re-create them in the proper order.
1. Confirm that we booted in safe mode
```
PS C:\> gwmi Win32_ComputerSystem | select BootupState

BootupState
-----------
Fail-safe boot

```
2. Confirm the actual page file location
```
PS C:\> gwmi Win32_PageFileSetting

MaximumSize Name            Caption
----------- ----            -------
          0 X:\pagefile.sys X:\ 'pagefile.sys'

```
3. Un-map the *PAGEFILE* partition and attach drive R:\ to the *Recovery*  partition. Get a view of the final volume mappings
```
PS C:\> 'list vol' | diskpart

Microsoft DiskPart version 10.0.26100.1150

Copyright (C) Microsoft Corporation.
On computer: SERVER-01

DISKPART>
  Volume ###  Ltr  Label        Fs     Type        Size     Status     Info
  ----------  ---  -----------  -----  ----------  -------  ---------  --------
  Volume 0     D   VBox_GAs_7.  CDFS   CD-ROM        52 MB  Healthy
  Volume 1     C                NTFS   Partition     69 GB  Healthy    Boot
  Volume 2         PAGEFILE     NTFS   Partition     16 GB  Healthy
    C:\VMEM\
  Volume 3     K   CACHE        NTFS   Partition    937 GB  Healthy
  Volume 4                      FAT32  Partition    100 MB  Healthy    System
  Volume 5                      NTFS   Partition    674 MB  Healthy    Hidden
  Volume 6     X                NTFS   Partition   1023 GB  Healthy    Pagefile

DISKPART>

PS C:\> 'sel vol 2;remove mount=c:\vmem;sel disk 0;sel par 4;assign letter r;list vol' -split ';' | diskpart

Microsoft DiskPart version 10.0.26100.1150

Copyright (C) Microsoft Corporation.
On computer: SERVER-01

DISKPART>
Volume 2 is the selected volume.

DISKPART>
DiskPart successfully removed the drive letter or mount point.

DISKPART>
Disk 0 is now the selected disk.

DISKPART>
Partition 4 is now the selected partition.

DISKPART>
DiskPart successfully assigned the drive letter or mount point.

DISKPART>
  Volume ###  Ltr  Label        Fs     Type        Size     Status     Info
  ----------  ---  -----------  -----  ----------  -------  ---------  --------
  Volume 0     D   VBox_GAs_7.  CDFS   CD-ROM        52 MB  Healthy
  Volume 1     C                NTFS   Partition     69 GB  Healthy    Boot
  Volume 2         PAGEFILE     NTFS   Partition     16 GB  Healthy
  Volume 3     K   CACHE        NTFS   Partition    937 GB  Healthy
  Volume 4                      FAT32  Partition    100 MB  Healthy    System
* Volume 5     R                NTFS   Partition    674 MB  Healthy    Hidden
  Volume 6     X                NTFS   Partition   1023 GB  Healthy    Pagefile

DISKPART>
```
4. Backup the *Recovery* and the *CACHE* partitions
```
PS C:\> dism /Capture-Image /ImageFile:X:\recovery.wim /CaptureDir:R:\ /Name:Recovery

Deployment Image Servicing and Management tool
Version: 10.0.26100.1150

Saving image
[==========================100.0%==========================]
The operation completed successfully.
PS C:\> dism /Capture-Image /ImageFile:X:\cache.wim /CaptureDir:K:\ /Name:Cache

Deployment Image Servicing and Management tool
Version: 10.0.26100.1150

Saving image
[==========================100.0%==========================]
The operation completed successfully.

PS C:\>
```
5. Remove the partitions to be re-located
```
PS C:\> 'sel disk 0;sel par 4;del par override;sel par 5;del par;sel par 6;del par; list par' -split ';' | diskpart

Microsoft DiskPart version 10.0.26100.1150

Copyright (C) Microsoft Corporation.
On computer: SERVER-01

DISKPART>
Disk 0 is now the selected disk.

DISKPART>
Partition 4 is now the selected partition.

DISKPART>
DiskPart successfully deleted the selected partition.

DISKPART>
Partition 5 is now the selected partition.

DISKPART>
DiskPart successfully deleted the selected partition.

DISKPART>
Partition 6 is now the selected partition.

DISKPART>
DiskPart successfully deleted the selected partition.

DISKPART>
  Partition ###  Type              Size     Offset
  -------------  ----------------  -------  -------
  Partition 1    System             100 MB  1024 KB
  Partition 2    Reserved            16 MB   101 MB
  Partition 3    Primary             69 GB   117 MB

DISKPART>

```
6. Create the new partitions
```
PS C:\Users\Administrator> 'sel disk 0;create par pri size=16384;format fs=ntfs quick label=PAGEFILE;assign mount=C:\VMEM;create par pri size=959488;format fs=ntfs quick label=CACHE;assign letter k;create par pri;format fs=ntfs quick label=WinRE;set id=de94bba4-06d1-4d40-a16a-bfd50179d6ac;gpt attributes=0X8000000000000001;assign letter r;list par;list vol' -split ';' | diskpart

Microsoft DiskPart version 10.0.26100.1150

Copyright (C) Microsoft Corporation.
On computer: SERVER-01

DISKPART>
Disk 0 is now the selected disk.

DISKPART>
DiskPart succeeded in creating the specified partition.

DISKPART>
  100 percent completed

DiskPart successfully formatted the volume.

DISKPART>
DiskPart successfully assigned the drive letter or mount point.

DISKPART>
DiskPart succeeded in creating the specified partition.

DISKPART>
  100 percent completed

DiskPart successfully formatted the volume.

DISKPART>
DiskPart successfully assigned the drive letter or mount point.

DISKPART>
DiskPart succeeded in creating the specified partition.

DISKPART>
  100 percent completed

DiskPart successfully formatted the volume.

DISKPART>
DiskPart successfully set the partition ID.

DISKPART>
DiskPart successfully assigned the attributes to the selected GPT partition.

DISKPART>
DiskPart successfully assigned the drive letter or mount point.

DISKPART>
  Partition ###  Type              Size     Offset
  -------------  ----------------  -------  -------
  Partition 1    System             100 MB  1024 KB
  Partition 2    Reserved            16 MB   101 MB
  Partition 3    Primary             69 GB   117 MB
  Partition 4    Primary             16 GB    69 GB
  Partition 5    Primary            937 GB    85 GB
* Partition 6    Recovery          1581 MB  1022 GB

DISKPART>
  Volume ###  Ltr  Label        Fs     Type        Size     Status     Info
  ----------  ---  -----------  -----  ----------  -------  ---------  --------
  Volume 0     D   VBox_GAs_7.  CDFS   CD-ROM        52 MB  Healthy
  Volume 1     C                NTFS   Partition     69 GB  Healthy    Boot
  Volume 2         PAGEFILE     NTFS   Partition     16 GB  Healthy
    C:\VMEM\
  Volume 3     K   CACHE        NTFS   Partition    937 GB  Healthy
  Volume 4                      FAT32  Partition    100 MB  Healthy    System
* Volume 5     R   WinRE        NTFS   Partition   1581 MB  Healthy    Hidden
  Volume 6     X                NTFS   Partition   1023 GB  Healthy    Pagefile

DISKPART>
```
7. Change the page file to the *PAGEFILE* partition
```
PS C:\> Set-WmiInstance Win32_PageFileSetting -Arguments @{ Name='C:\VMEM\pagefile.sys'; InitialSize=0;MaximumSize=0} | Out-Null
PS C:\> gwmi Win32_PageFileSetting

MaximumSize Name                 Caption
----------- ----                 -------
          0 X:\pagefile.sys      X:\ 'pagefile.sys'
          0 C:\VMEM\pagefile.sys C:\ 'VMEM\pagefile.sys'


PS C:\> (gwmi Win32_PageFileSetting -EnableAllPrivileges | ?{ $_.Name -match 'X:' }).Delete()
PS C:\> gwmi Win32_PageFileSetting

MaximumSize Name                 Caption
----------- ----                 -------
          0 C:\VMEM\pagefile.sys C:\ 'VMEM\pagefile.sys'

```
8. Restore *Recovery* and *CACHE* data
```
PS C:\> dism /Apply-Image /ImageFile:x:\cache.wim /ApplyDir:K:\

Deployment Image Servicing and Management tool
Version: 10.0.26100.1150

Applying image
[==========================100.0%==========================]
The operation completed successfully.
PS C:\> dism /Apply-Image /ImageFile:x:\recovery.wim /ApplyDir:R:\

Deployment Image Servicing and Management tool
Version: 10.0.26100.1150

Applying image
[==========================100.0%==========================]
```
9. Un-map the *Recovery* partition
```
PS C:\> 'sel vol r;remove letter r' -split ';' | diskpart

Microsoft DiskPart version 10.0.26100.1150

Copyright (C) Microsoft Corporation.
On computer: SERVER-01

DISKPART>
Volume 5 is the selected volume.

DISKPART>
DiskPart successfully removed the drive letter or mount point.

DISKPART>
```
10. Reboot in normal mode
```
PS C:\> bcdedit /deletevalue '{default}' safeboot
The operation completed successfully.

PS C:\> shutdown /r /f
```
### Cleanup the environment
1. Confirm we are on normal mode
```
PS C:\> gwmi Win32_ComputerSystem | select BootupState

BootupState
-----------
Normal boot

```
2. Check that the page file was moved out of X:\ and re-initialize the disk
```
PS C:\> 'sel disk 0;list par;list vol' -split ';' | diskpart

Microsoft DiskPart version 10.0.26100.1150

Copyright (C) Microsoft Corporation.
On computer: SERVER-01

DISKPART>
Disk 0 is now the selected disk.

DISKPART>
  Partition ###  Type              Size     Offset
  -------------  ----------------  -------  -------
  Partition 1    System             100 MB  1024 KB
  Partition 2    Reserved            16 MB   101 MB
  Partition 3    Primary             69 GB   117 MB
  Partition 4    Primary             16 GB    69 GB
  Partition 5    Primary            937 GB    85 GB
  Partition 6    Recovery          1581 MB  1022 GB

DISKPART>
  Volume ###  Ltr  Label        Fs     Type        Size     Status     Info
  ----------  ---  -----------  -----  ----------  -------  ---------  --------
  Volume 0     D   VBox_GAs_7.  CDFS   CD-ROM        52 MB  Healthy
  Volume 1     C                NTFS   Partition     69 GB  Healthy    Boot
  Volume 2         PAGEFILE     NTFS   Partition     16 GB  Healthy
    C:\VMEM\
  Volume 3     K   CACHE        NTFS   Partition    937 GB  Healthy
  Volume 4                      FAT32  Partition    100 MB  Healthy    System
  Volume 5         WinRE        NTFS   Partition   1581 MB  Healthy    Hidden
  Volume 6     X                NTFS   Partition   1023 GB  Healthy

DISKPART>

PS C:\> gwmi Win32_PageFileSetting -EnableAllPrivileges

MaximumSize Name                 Caption
----------- ----                 -------
          0 C:\VMEM\pagefile.sys C:\ 'VMEM\pagefile.sys'
```
2. Clean disk 1
```
PS C:\> 'sel disk 1;clean' -split ';'  | diskpart

Microsoft DiskPart version 10.0.26100.1150

Copyright (C) Microsoft Corporation.
On computer: SERVER-01

DISKPART>
Disk 1 is now the selected disk.

DISKPART>
DiskPart succeeded in cleaning the disk.

DISKPART>
```
2. Re-enable the recovery agent
```
PS C:\> ReAgentc.exe /info
Windows Recovery Environment (Windows RE) and system reset configuration
Information:

    Windows RE status:         Disabled
    Windows RE location:
    Boot Configuration Data (BCD) identifier: 00000000-0000-0000-0000-000000000000
    Recovery image location:
    Recovery image index:      0
    Custom image location:
    Custom image index:        0

REAGENTC.EXE: Operation Successful.
PS C:\> ReAgentc.exe /enable
REAGENTC.EXE: Operation Successful.
PS C:\> ReAgentc.exe /info
Windows Recovery Environment (Windows RE) and system reset configuration
Information:

    Windows RE status:         Enabled
    Windows RE location:       \\?\GLOBALROOT\device\harddisk0\partition6\Recovery\WindowsRE
    Boot Configuration Data (BCD) identifier: 8e73986e-289e-11f0-bf53-a4d06eada308
    Recovery image location:
    Recovery image index:      0
    Custom image location:
    Custom image index:        0

REAGENTC.EXE: Operation Successful.

PS C:\>
```
