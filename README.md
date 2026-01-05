# Running Metasploitable2 on WSL2

To record a CMD.exe session, start your Ubuntu distribution. Then use asciinema with CMD.exe as a shell
and record the commands in the DOS session. There are couple of gotchas, tl;dr; this is the command:

```
(cd /mnt/c && asciinema rec --overwrite --command "$(which cmd.exe) /D /K cd /d c:\\" /tmp/kali-linux-on-wsl2.cast)
```

## Install Windows Terminal

Install the [free Windows Terminal](https://apps.microsoft.com/detail/9N0DX20HK701) from the Microsoft
Store.


## Enabling WSl2 on Windows

From an elevated command prompt (*Run As Administrator*), run the following commands:

```
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all
```

If you have to reboot, please do so - just in case. When you are back, open another elevated command
prompt and run the following command to make sure you are using WLS2.:

```
wsl --set-default-version 2
```

## Install Kali Linux

Kali Linux can now be installed from the Windows store. From a command prompt in Windows, type the
following commands:

```
wsl --install --distribution kali-linux
```

Your Kali is already protected by your Windows account, so you can just stick with the default user
`kali` and password `kali` when prompted.

```
sudo apt update
sudo apt install kali-linux-default kali-win-kex burpsuite --assume-yes
```

## Enabling nested virtualization

This requires Windows 11, it is not supported on Windows 10. The steps should work, but performance
will suffer.

Follow the [steps documented on ServerFault](https://serverfault.com/a/1115773/81170) to make nested
virtualization permanent. But you can just do the following in Kali everytime:

```
sudo usermod -a -G kvm ${USER}
# Use kvm_amd if you have an AMD CPU
modprobe kvm_intel
sudo chown root:kvm /dev/kvm && sudo chmod 660 /dev/kvm
```

## Install the virtual machine manager tools

```
sudo apt update
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients virtinst libguestfs-tools virt-manager
```

## Preparing the Metasploitable2 image

You must firt download the virtual machine archive from source.

```
curl --location --continue-at - --remote-name https://downloads.metasploit.com/data/metasploitable/metasploitable-linux-2.0.0.zip
```

There are a few files, but we only need the VMDK. Extract it with this command:

```
unzip -j metasploitable-linux-2.0.0.zip Metasploitable2-Linux/Metasploitable.vmdk
rm metasploitable-linux-2.0.0.zip
```

## Create the lab network

We create a lab named `attack` on `192.168.3.0/24`

```
sudo virsh net-define attack-network.xml
sudo virsh net-start attack-network
sudo virsh net-autostart attack-network
```

Might as well add the IP address of the VM right away

```
sudo sed -zi '/192.168.3.20 msf2.test/!s/$/\n192.168.3.20 msf2.test\n/' /etc/hosts
```

## Create the virtual machine

We convert the disk to a format that is native to QEMU:

```
sudo qemu-img convert -p -f vmdk -O qcow2 \
    Metasploitable.vmdk \
    /var/lib/libvirt/images/metasploitable2.qcow2
```

And we import the disk in a new virtual machine.

```
sudo virt-install \
   --name Metasploitable2 \
   --os-variant ubuntu8.04 \
   --disk path=/var/lib/libvirt/images/metasploitable2.qcow2,bus=ide \
   --network network=attack-network,model=e1000 \
   --graphics vnc,listen=0.0.0.0 \
   --import \
   --noautoconsole \
   --ram 512 \
   ;
```

## Connect to the virtual machine

Your Metasploitable2 virtual machine should be ready to go! It uses a graphical console by
default, so you can use virt-manager's VNC console to connect to the VM:

```
sudo virt-manager --connect qemu:///system --show-domain-console Metasploitable2
```

