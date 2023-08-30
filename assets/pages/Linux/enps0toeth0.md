# Use eth(x) instead of enps(x)
## To switch back to the old NIC Names

1. Edit /etc/default/grub
2. At the end of GRUB_CMDLINE_LINUX line append
"net.ifnames=0 biosdevname=0"
3. Save the file
4. Type "grub2-mkconfig -o /boot/grub2/grub.cfg" -- This will change depending on where the grub.cfg is. So you need to go to /boot and find where your grub config is
5. Type "reboot"
