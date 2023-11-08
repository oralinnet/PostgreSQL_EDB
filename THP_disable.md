### Disable Transparent Huge Pages CentOS/RHEL 8

- Check the current status of Transparent Huge Pages.
```sh
cat /sys/kernel/mm/transparent_hugepage/enabled
[always] madvise never
### THP are currently enabled in your Linux operating system.
```
```doc
To disable Transparent Huge Pages, you have to edit GRUB boot menu. For this purpose, 
Edit the GRUB configuration file in vim text editor.
```
```sh
vim /etc/default/grub

# Locate GRUB_CMDLINE_LINUX and append "transparent_hugepage=never" at the end of that line.
GRUB_CMDLINE_LINUX="resume=/dev/mapper/cl-swap rd.lvm.lv=cl/root rd.lvm.lv=cl/swap rhgb quiet transparent_hugepage=never"
```
- Generate new GRUB boot menu based on customized configuration file.
```sh
grub2-mkconfig -o /boot/grub2/grub.cfg
## Generating grub configuration file ...
## done
```
- Restart Linux operating system to apply new settings.
```sh
reboot
```
- After restart, check the status of THP again.
```sh
cat /sys/kernel/mm/transparent_hugepage/enabled
always madvise [never]
```
```t
You have disabled Transparent Huge Pages on your Linux operating system. 
But you are also required to configure tuned service as well.
Create a Custom Profile for tuned Service:
tuned service is used for monitoring and tuning in a Linux system. It can be used to disable or enable THP.
Therefore, you need to create a profile for tuned service and disable the Transparent Huge Pages in that profile.
```
- Create a directory for new tuned profile.
```sh
mkdir /etc/tuned/no-thp
```
- Create a tuned configuration file.
```sh
vim /etc/tuned/no-thp/tuned.conf
# Add following lines in that file.

[main]
include=virtual-guest

[vm]
transparent_hugepages=never

```
- Enable the no-thp profile by using following command.
```sh
tuned-adm profile no-thp
```
