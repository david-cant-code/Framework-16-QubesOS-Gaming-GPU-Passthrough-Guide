# Framework-16 QubesOS Gaming GPU Passthrough Guide - Verified to work on Qubes 4.3
A step by step guide to pass the 7700s GPU module on a Framework 16 laptop through to create a gaming HVM qube in QubesOS. This should also work with the 5070 nvidia GPU they just released if you adjust the commands for the output of lspci. This will get updated more as time goes on and I feel like it, kind of rough but wanted to share since I have wanted to do this for a long time and finally figured it out, which was a pain because there are no clear guides on how to do this. Now there is, and you can play Helldivers on it :100:

As a side note, I have noticed slightly better battery life after doing this, I assume from excluding the GPU from dom0 resulting in only idle background usage unless running the gaming qube. For those of you into self hosting AI, you can also use this to run an ollama server (or other local LLM) on your laptop with some tinkering, since this gives you an OS running on bare metal with your GPU.

![Democracy!!!](https://github.com/david-cant-code/Framework-16-QubesOS-Gaming-GPU-Passthrough-Guide/blob/main/pictures/smallerDemocracy.gif)

## Step 1: Grub Modifications
Before going too far, verify that your GPU has the same pcie ID as mine does, in dom0 run:
```
lspci | grep -i vga
lspci | grep -i audio
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
Apply those changes, still in dom0 run:
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
Before Qubes 4.3: Create a qube, name it gaming, select standalone VM, and set the template to none and check to launch settings after creation.
Qubes 4.3: this version of qubes has a new gui for creating qubes, ensure you select "standalone" in the column on the left and under "Clone from existing" select do not clone. On the bottom check you can leave "Install system from device" unchecked since it is available from the settings. The rest of the HVM creation is the same as prior to 4.3.

Next, in the settings, change the RAM to whatever you can give it. Go to the devices tab and add the GPU and its audio device and hit apply. Note, if it crashes here, you did not properly hide the GPU and/or audio device. You may note that in the device passthrough part of the qube setting it shows a different pcie address than what we used, you can ignore this and pass it through.

Later we will need to pass through a USB controller, but for now you can install the VM. Now plug your GPU into a monitor, it should do nothing because the GPU is not held by dom0. 

On the basic tab of the settings for your gaming HVM qube, increase the system storage size to at least 25GB (assuming a Fedora install), this is where your / partition will go. Then increase the private storage, its your call here on how much to give it, but remember the amount you give it will be the total size available for anything you download. I recommend installing assigning /home to the private storage. In a Fedora installer, the private storage shows up as xvdb and the system storage shows up as xvda.

Download an iso, I used Fedora 42, verify the ISO, and then in the settings for your gaming HVM, select boot from device/cd in the Advanced tab, select the qube the iso you downloaded is in, then hit the three dots and a file picker will pop up. Then select the iso and boot the hvm.

If you haven't already, plug in an external monitor to your GPU, the gaming qube will be very slow if you use the windown on your laptop screen (I think its because its using the vCPU cores passed through to render). Once you boot the gaming qube, it will output video to a window on your laptop, as well as your external monitor. The external monitor will not have a qubes color barrier around it, it will look like a normal fedora installer. I recommend from within your installer, or once installed and booted into your gaming qube, disable the window that is on the laptop screen (it shows up as a monitor). I have noticed a rare instance where I need to boot it without the external monitor plugged in during initial install, and then plug it in once the iso is booted. Try doing that if you get stuck at the end of the boot sequence without the desktop starting up. This is not a problem once installed.

Note on mouse capture: Until you pass through a usb controller, in order to get the mouse on the external monitor, you must hover over the window for the gaming qube on your laptop, whether you disabled it in your install or not. 

Install your OS and reboot.

Once you have the HVM installed and running, the window will still pop up every time you run the qube, and inside the HVM that window is seen as a display, which can be shutoff in settigns like any other display. But the window stays on your laptop screen. In order to have your mouse work inside the gaming qube, you need to keep it within the window on your laptop's screen, then it shows on the external monitor also. This is why later we have to pass through a USB controller.

## Step 3 - Networking
For whatever reason you will have to manually configure your network, but it is easy to do. On your qubes setting, you will see netowrk info, including an IP address, gateway, netmask, dns, etc. You will want to leave the net qube as sys-firewall default.

In gnome settings, open up network settings, then click the gear icon on the wired connection. Disable ipv6, then go to the ipv4 section and change it to manual.

Now you need to enter in the IP, netmask, and gateway. Then enter the two DNS addresses. It will now connect (assuming your qubesOS host is connected to the internet. Mind that in the screenshot, those are my specific HVMs IP, it may be different on your system, but will be similar since that is how qubes assigns internal IPs.

![Gnome Netowrk Settings Example](https://github.com/david-cant-code/Framework-16-QubesOS-Gaming-GPU-Passthrough-Guide/blob/main/pictures/Screenshot%20From%202025-08-17%2016-57-29.png)


Now, from here you have a working HVM that has your GPU and can game. I am able to play Helldivers 2 with 6 vCPU passthrough, 12 GB RAM, and of course the GPU.

![NVTOP showing GPU info](https://github.com/david-cant-code/Framework-16-QubesOS-Gaming-GPU-Passthrough-Guide/blob/main/pictures/Screenshot%20From%202025-08-17%2016-27-11.png)


BUT you will have to keep your mouse within the window of the HVM, it doesn't capture it, and you can't pass through a keyboard and mouse to the HVM like you normally would.

## Step 4 - USB Passthrough

I am sure someone smarter than me will be able to do this in a better way, but for me, this works.
Open up your qubes settings again, go to devices, and add the main USB controller to the settings. This must be done with the HVM shutoff. And you can't boot it up while sys-usb is active. Sounds lame, and it is because I stopped looking for a workaround when this worked. It works because your touchpad stays active, even with sys-usb shutoff, so you can use the qubes manager GUI to shutoff sys-usb, and then boot your gaming HVM. Once the HVM is running, it will own all USB ports and your monitor will look and feel like a bare metal install of Fedora, hotplugging USB and all. 
![Devices Screenshot](https://github.com/david-cant-code/Framework-16-QubesOS-Gaming-GPU-Passthrough-Guide/blob/main/pictures/IMG_20250817_171405_531.jpg)

Some downsides to doing it my lazy way:
1: You will have no ability to type into dom0 at all, only the track pad, while the gaming HVM qube is booted up
2: Sys-usb abd the gaming HVM can't be on at the same time
3: If you let dom0 go to a lock screen, you'll have to reboot in order to get your keyboard back

## Step 5 - Install games, have fun

Just do what you'd normally do to install steam or whatever you use. 
