# LVM Resize
## To expand an LVM Partition on Linux Carry out the following steps. 

``` Bash
NOTE: (x is the partition you want to resize)
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
