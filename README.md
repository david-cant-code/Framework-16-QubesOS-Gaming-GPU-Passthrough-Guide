# Framework-16 QubesOS Gaming GPU Passthrough Guide
A step by step guide to pass the 7700s GPU module on a Framework 16 laptop through to create a gaming HVM qube in QubesOS. This will get updated more as time goes on and I feel like it, kind of rough but wanted to share since I have wanted to do this for a long time and finally figured it out, which was a pain because there are no clear guides on how to do this. Now there is.

## Step 1: Grub Modifications
Before going too far, verify that your GPU has the same pcie ID as mine does, in dom0 run:
```
qvm-pci | grep -i vga
```
As long as it shows as 03:00.0 for the GPU, and 03:00.1 for the audio controller, then you can follow exactly my commands, if not you'll need to substitute whatever it is in place

In dom0:
```
sudo nano /etc/default/grub
```
Find the line named "GRUB_CMDLINE_LINUX" and append the following to it:
```
rd.drive.pre=xen-pciback xen-pciback.hide=(0000:03:00.0)(0000:03:00.1)
```
Then to the "GRUB_CMDLINE_XEN_DEFAULT" line append:
```
iommu=on
```
Aplly those changes, still in dom0 run:
```
sudo grub2-mkconfig -o /boot/efi/EFI/qubes/grub.cfg
```
Then you need to:
```
echo 'options xen-pciback hide=(0000:0a:00.0)(0000:0a:00.1)' | sudo tee /etc/modprobe.d/xen-pciback.conf
sudo dracut -f
```

Reboot, then in dom0 run these commands to verify this worked:
```
cat /proc/cmdline
```
You are looking to verify your grub command line arguments were added and show up. Next you need to check to make sure the GPU is held back from dom0:
```
lspci -k -s 03:00.0
```
What you want to see is that the kernel driver in use is pciback. That means you are good to go

## Step 2 Create HVM
Create a qube, name it gaming, select standalone VM, and set the template to none and check to launch settings after creation.

Next, in the settings, change the RAM to whatever you can give it. Go to the devices tab and add the GPU and its audio device and hit apply.

Later we will need to pass through a USB controller, but for now you can install the VM. Now plug your GPU into a monitor, it should do nothing because the GPU is not held by dom0. Download an iso, I used Fedora 42, verify the ISO, and then in the settings for your gaming HVM, select boot from device/cd, select the qube the iso you downloaded is in, then hit the three dots and a file picker will pop up. Then select the iso and boot the hvm.

This will then output video to a window on your laptop, as well as your external monitor. The external monitor will not have a qubes color barrier around it, it will look like a normal fedora installer.

Install your OS and reboot.

## Step 3 - Networking
For whatever reason you will have to manually configure your network, but it is easy to do. On your qubes setting, you will see netowrk info, including an IP address, gateway, netmask, dns, etc. You will want to leave the net qube as sys-firewall default.

In gnome settings, open up network settings, then click the gear icon on the wired connection. Disable ipv6, then go to the ipv4 section and change it to manual.

Now you need to enter in the IP, netmask, and gateway. Then enter the two DNS addresses. It will now connect (assuming your qubesOS host is connected to the internet. Mind that in the screenshot, those are my specific HVMs IP, it may be different on your system, but will be similar since that is how qubes assigns internal IPs.

![Gnome Netowrk Settings Example](https://github.com/david-cant-code/Framework-16-QubesOS-Gaming-GPU-Passthrough-Guide/blob/main/Screenshot%20From%202025-08-17%2016-57-29.png)


Now, from here you have a working HVM that has your GPU and can game. I am able to play Helldivers 2 with 5 vCPU passthrough, 24 GB RAM, and of course the GPU.

![NVTOP showing GPU info](https://github.com/david-cant-code/Framework-16-QubesOS-Gaming-GPU-Passthrough-Guide/blob/main/Screenshot%20From%202025-08-17%2016-27-11.png)

BUT you will have to keep your mouse within the window of the HVM, it doesn't capture it, and you can't pass through a keyboard and mouse to the HVM like you normally would.

## Step 4 - USB Passthrough

I am sure someone smarter than me will be able to do this in a better way, but for me, this works.
Open up your qubes settings again, go to devices, and add the main USB controller to the settings. This must be done with the HVM shutoff. And you can't boot it up while sys-usb is active. Sounds lame, and it is because I stopped looking for a workaround when this worked. It works because your touchpad stays active, even with sys-usb shutoff, so you can use the qubes manager GUI to shutoff sys-usb, and then boot your gaming HVM. Once the HVM is running, it will own all USB ports and your monitor will look and feel like a bare metal install of Fedora, hotplugging USB and all. 

Some downsides to doing it my lazy way:
1: You will have no ability to type into dom0 at all, only the track pad
2: If you let dom0 go to a lock screen, you'll have to reboot in order to get your keyboard back

## Step 5 - Install games, have fun

Just do what you'd normally do to install steam or whatever you use. 
