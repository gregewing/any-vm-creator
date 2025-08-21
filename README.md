# any-vm-creator

Create guest VMs so that they can be run inside docker containers.  Using this, you can install any OS you like and have it run in a docker container on your linux docker host.

## Install

Pull the container image from hub.docker.com with ````docker pull gregewing/any-vm-creator:latest````

## Issues and Support

Please raise any questions or issues here by raising issues in this project.

This image is intended as a tool to help in the creation of other docker containers with any operating system you want inside them.  

## Update 21 August 2025

- I've made some changes to thsi project recently as I need to make more use of it.  It now fully supports UEFI and Legacy BIOS operating systems, though you need to specify which you want to use in the ENV variables when creating the container, see below.
- There are ew BIOS firmware files for both Seabios Legacy Boot and also for UEFI boot, giving flexability and greater support for a wide variety of Operating Systems.
- The container now uses qemu directly instead of beig dependent on 'virsh install' for managing the guest VM images.
- The majority of the updates are in the startup.sh that is started with the container, which has been completely re-written.

## What does it do?

Specifically this container enables the creation of a Virtual Machine which is ready to be hosted on a qemu/kvm (and optionally virt-manager) hypervisor platform.  Most likely, the output of this container will then be used as the VM disk image for running whatever Operating System you installed in the first place on the destination hosting platform.  That destination platform could be my any-vm-runner docker container, allowing any OS to be hosted as a docker contaier on your linux based docker platform.

To put it another way, this provides the virtual disk image that you can use to host windows (or any other OS) inside a docker container on linux.

## To get started you will need

- a .iso file to install from 
- this container
- (optional) "virtio-win.iso" windows virtio drivers from <a href="https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/latest-virtio/">https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/latest-virtio/</a> which should be downloaded and placed in the mounted as a volume when creating the container. If you don't need this, you should delete the relelvant virtio-win.iso line from your docker command.

## What happens when you run this container

Start the container with the following, or something similar:
````
docker run -d --privileged \
  --name any-vm-creator \
  --cgroupns host \
  --security-opt seccomp=unconfined \
  --device=/dev/kvm \
  --device=/dev/net/tun \
  -e MEMORY=4096 \
  -e CPU=4 \
  -e DISK=100 \
  -e OS_VARIANT=win11 \
  -e BIOS_TYPE=UEFI \
  -v <<FULL-PATH-to-my-favourite-.ISO>>:/guestVM.iso \
  -v <<FULL-PATH-to-virtio-win.iso>>:/virtio-win.iso \
  -v /sys/fs/cgroup:/sys/fs/cgroup:rw \
  -v any-vm-creator:/guestVMdata \
  -p 5911:5910 \
  -p 3390:3389 \
  --cap-add=NET_ADMIN \
  --cap-add=SYS_ADMIN \
  gregewing/any-vm-creator:latest

````

When the container starts, it will automatically create a sparse .qcow2 file inside the any-vm-creator volume defined above.  It will then boot from the provided .ISO and install the desired operating system into that qcow2 vidtual disk file.  You can keep track of progress by connecting a vnc viewer to port 5900 (unless you changed it) and make anyinstall time configurations interactively.

Once you have completed the install, you can shut down the VM and shut down the container. You might want to reboot instead of to install additional tools or software or make customisations to the VM. The .qcow2 file that was created in the any-vm-creator volume will contain the installed operating system and any customisations that you made along the way.    

You now have a .qcow2 virtual disk image file that will operate as per any other virtual disk file you may have used in the past in full-fat hypervisors such as HyperV or VMWare.  It is portable and will be bootable on any qemu/kvm instance, or as the VM image for an instance of any-vm-runner.  

## Volumes

There are three key volumes or bind-mounts used by this container:
 - ```-v <<FULL-PATH-to-my-favourite-.ISO>>:/guestVM.iso```  replace <<FULL-PATH-to-my-favourite-.ISO>> with the path to the ISO file that you want to install from.
 - ```-v any-vm-creator:/guestVMdata```  replace -v any-vm-creator with the name of a volume that you created beforehand.  If you don't create a volume and provide it here, one will be created for you with the name any-vm-creator.
 - ```-v <<FULL-PATH-to-virtio-win.iso>>:/virtio-win.iso```  replace <<FULL-PATH-to-virtio-win.iso>> with the full path to the virtio-win.iso that you downloaded from elsewhere on the internet. This is where drivers for windows systems come from and you will have to load them in manually at the appropriate stage of the build for your chosen Windows OS.

## Environment Variables

The above docker run command makes use of the following environment variables, which you may want to alter to suit your requirements:
- MEMORY - this is the RAM that will be allocated to the guest VM that you are creating, this is used for the OS install and build only. If you don't provide a value in your docker command, it will default to 4096MB (4GB).
- CPU - this is the number of vCPU cores to allocate to the guest VM during the install and build phase. If you don't provide a value in your docker command, it will default to 4.
- DISK - this is the size of the virtual disk file that will be crete to build into, this is important to get right. Note that the container will create a sparse file, so it will look smaller on disk that the guestVM sees when it is running. If you don't provide a value in your docker command, it will default to 50GB.
- OS_VARIANT - this is where we specify as accurately as we can the operating system that we are installing.  its not strictly necessary, but i have found that leaving it as 'generic' creates problems with some image types (specifically windows).  There are a wide range of possible values here, I was not able to find a definitive list, but valid values include : 'debian9' 'ubuntu24.04' 'rhel9.3' 'rocky9.5' 'centos-stream9' 'win11'.   If you don't provide a value in your docker command, it will default to 'win11'
- BIOS_TYPE - this is where you specify 'UEFI' or 'BIOS' for Legacy boot mode.  This will cause the conainer to start up the embedded qemu hypervisor with the right bios image for your ISO image to boot properly.  Most OSs now use UEFI, but you might have some specific requirements.  The image includes the necessary BIOS and UEFI files, so you dont need to mount them in manually.

## Tested and working with

- Win11_24H2_EnglishInternational_x64.iso
- Rocky-9.6-x86_64-dvd.iso
- ubuntu-24.10-live-server-amd64.iso
- Win10_22H2_EnglishInternational_x64v1.iso
- WinSrv2025.iso
- WinSrv2022.iso
- android-x86_64-9.0-r2.iso
- debian-13.0.0-amd64-DVD-1.iso
- debian-12.11.0-amd64-DVD-1.iso
- debian-12.11.0-amd64-netinst.iso
- proxmox-ve_9.0-1.iso

Yes using Legacy BIOS, and specifically no virtio-win.iso cd mounted.
- archlinux-x86_64.iso
- kali-linux-2025.2-installer-amd64.iso
