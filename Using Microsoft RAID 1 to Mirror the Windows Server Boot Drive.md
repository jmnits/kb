## Environment
The current example occurred on a customer's server installed with Windows Server 2025 Core.
The server has the following characteristics:
* [UEFI](https://en.wikipedia.org/wiki/UEFI) Firmware
- 4 CPU's
- 8 GB RAM
- 1 NIC
- 1 hard drive of 1 TB (SATA)

The hard drive is formatted as a [Basic Disk](https://learn.microsoft.com/en-us/windows/win32/fileio/basic-and-dynamic-disks#basic-disks), using the [GPT Partition Format](https://en.wikipedia.org/wiki/GUID_Partition_Table) and contains the following partitions:

| Partition                                                         | File System | Size   | Remarks                                                     |
| :---------------------------------------------------------------- | :---------- | :----- | :---------------------------------------------------------- |
| [EFI](https://en.wikipedia.org/wiki/EFI_system_partition)         | FAT32       | 100 MB | EFI Boot Disk                                               |
| [MSR](https://en.wikipedia.org/wiki/Microsoft_Reserved_Partition) | --          | 16 MB  | Microsoft Reserved Partition                                |
| *BOOT*                                                            | NTFS        | 69 GB  | C:                                                          |
| Recovery                                                          | NTFS        | 674 MB | Windows Recovery Partition                                  |
| *PAGEFILE*                                                        | NTFS        | 16 GB  | Partition to host the Windows *pagefile*                    |
| *CACHE*                                                           | NTFS        | 937 GB | Partition used by the server main application to cache data |
The customer wanted to protect the main disk with RAID-1, using the Microsoft RAID-1 Software Solution, which is an integral part of the Windows Server OS.

## Updated Environment and initial considerations
In order to enable the RAID-1 setup, we required a second hard drive, with at least 1 TB (same capacity as the primary disk).

Adding the second SATA disk is a requirement but the system requires additional tweaks, as Microsoft RAID-1 solution requires both disks to be configured as [Dynamic disks](https://learn.microsoft.com/en-us/windows/win32/fileio/basic-and-dynamic-disks#basic-disks).

However, the conversion of disk 0 to dynamic can't be achieved as the *Recovery partition* is in between the *BOOT* and *PAGEFILE* partitions.

> Dynamic disks require all data partitions to be contiguous

If tried, the system reports:

```
The selected GPT formatted disk contains a partition which is not of type 
'PARTITION_BASIC_DATA_GUID', and is both preceeded and followed by a partition 
of type 'PARTITION_BASIC_DATA_GUID'.
```

To solve this situation, we need to re-organize the partitions in the disk, placing the recovery partition at the end of the disk.

Only after that operation are we able to initiate the RAID-1 setup of the boot disk.

## Log Book

### Re-order Windows Partitions

Our objective is to end with the following partition layout:

| Partition                                                         | File System |   Size | Remarks                                                     |
| ----------------------------------------------------------------- | ----------- | -----: | ----------------------------------------------------------- |
| [EFI](https://en.wikipedia.org/wiki/EFI_system_partition)         | FAT32       | 100 MB | EFI Boot Disk                                               |
| [MSR](https://en.wikipedia.org/wiki/Microsoft_Reserved_Partition) | --          |  16 MB | Microsoft Reserved Partition                                |
| *BOOT*                                                            | NTFS        |  69 GB | C:                                                          |
| *PAGEFILE*                                                        | NTFS        |  16 GB | Partition to host the Windows *pagefile*                    |
| *CACHE*                                                           | NTFS        | 937 GB | Partition used by the server main application to cache data |
| Recovery                                                          | NTFS        | 674 MB | Windows Recovery Partition                                  |

In order to do this, we need to delete the *Recovery*, *PAGEFILE* and *CACHE* partitions and re-build them again.

The following approach and remarks apply to these partitions:

| Partition  | Considerations                                                                                                                                                  | Actions                                                                                                       |
| ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| *Recovery* | Can contain recovery data that should be preserved upon re-creation.<br>This is a special partition, meaning that its GUID and attributes should be replicated. | Backup the partition using the `dism` utility.<br>The image can be stored on C: (< 1 GB)                      |
| *PAGEFILE* | The pagefile must be removed from this drive before deleting the the partition.                                                                                 | Remove the pagefile from the partition. No other special actions are required.                                |
| *CACHE*    | No applications should be using this partition. <br>The partition should be backup to be later restored.                                                        | Boot the server in `safemode` so all applications are stopped.<br>Use the `dism` utility to backup the drive. |


### Prepare Secondary Boot Disk

### Setup RAID-1 on Boot Disk

### Replicate EFI and Configure Boot Sequence
