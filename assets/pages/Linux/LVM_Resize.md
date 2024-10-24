# LVM Resize
## To expand an LVM Partition on Linux Carry out the following steps. 

``` Bash
NOTE: (x is the partition you want to resize)
for i in /sys/class/scsi_host/host*/scan; do echo "- - -" > $i; done
for i in /sys/class/scsi_device/*/device/rescan; do echo "1" > $i; done
growpart /dev/sda x 
lsblk
pvresize /dev/sda(x)
lsblk
pvs
vgextend vg_main /dev/sda(x)
vgs
df -h
lvextend -l+100%FREE /dev/mapper/vg_main-lv_root /dev/sda(x)      
lvs
resize2fs /dev/mapper/vg_main-lv_root
```

> Note that for XFS volumes you will need to expand using xfs_growfs instead of resize2fs like the below example

``` Bash
xfs_growfs /dev/mapper/vg_main-lv_root
```

