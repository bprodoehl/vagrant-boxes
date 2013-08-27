This is lovingly borrowed from here: http://cbednarski.com/articles/creating-vagrant-base-box-for-centos-62/


Before We Begin
=============

You should have Vagrant 1.2 and VirtualBox 4.2 or VMware Fusion Professional 5 or VMware Workstation 5 installed. You can verify the version of vagrant via:

```
$ vagrant -v
Vagrant version 1.2.7
```

Also, all of the networking for vagrant requires your base box to use NAT ("Share with my Mac" in VMware). This is the default for VirtualBox and VMware, but if you’ve changed this somehow make sure your new boxes are created with the proper settings.

Configuring the System
=============

Minimal Install
-------------

I started out with a CentOS 6.4 minimal image. It has a few quirks (which we’ll work through below) but it also produces a very small base box size with little effort. Smaller boxes are faster to download and clone, so your iteration loop will be faster overall. Our target boxes will end up around 400-500 MB each.

Networking
-------------

The CentOS 6.4 minimal install does not enable networking by default. If you want to connect to the internet (you do want to connect to the internet), you’ll need to turn it on.

The default connection is located in:

```
/etc/sysconfig/network-scripts/ifcfg-eth0
```

If you open this with vi you’ll notice the configuration says ONBOOT="no", which means this connection won’t be started at boot time. This is obviously not what we want so we’ll need to change a few things in this configuration to get the networking to start up:

```
DEVICE="eth0"
NM_CONTROLLED="no"
ONBOOT="yes"
BOOTPROTO="dhcp"
```

Make sure you remove the HWADDR line. This specifies that the config file should only be used on a device with a matching mac address. Since the VM manager will reinitialize the NIC when you clone the VM, the mac will change and the config will stop working.

Next, we also need to modify /etc/udev/rules.d/70-persistent-net.rules since it contains a reference to the mac address, too. Replace the eth0 line with:

```
SUBSYSTEM=="net", ACTION=="add", DRIVERS="?*", ATTR{type}=="1", KERNEL=="eth*", NAME="eth0"
```

At this point, you can restart your system with shutdown -r now. When the vm comes back up your network connection will activate. You can verify your network connection is working by pinging something from inside the vm, e.g.:

```
$ ping google.com
```

If the command hangs or says there’s a timeout, you have a problem. Press Ctrl+C to interrupt the ping command and troubleshoot (i.e. go google things).

Install VMWare Tools
-------------

Note: for VirtualBox, see the next section.

From VMware’s Virtual Machine menu, Install VMware Tools. Then hop into the VM and mount the VMware tools CD.

```
$ mkdir /media/cdrom
$ mount /dev/cdrom /media/cdrom
mount: block device /dev/sr0 is write-protected, mounting read-only
```

Next, we’ll need to extract and install the vmware tools (after typing VM, below, press the tab key to autocomplete the rest of it, since your version will probably be different):

```
$ cd /tmp
$ tar -xzf /media/cdrom/VM[tab].tar.gz
$ yum install -y perl fuse-libs
$ /tmp/vmware-tools-distrib/vmware-install.pl --default
```

Finally, let’s unmount and do a bit of cleanup:

```
$ umount /dev/cdrom
$ rm -rf /tmp/*
```

Install the Virtual Box Guest Additions
-------------

If you’re creating a box for VirtualBox, use this process instead.

In order to install the VirtualBox guest additions for doing fun stuff like mounting devices inside the guest OS, you’ll need to install a kernel module. This requires the kernel source, so let’s start by updating those:

```
$ yum update kernel* -u
$ shutdown -r now
```

From the Devices menu, click Install Guest Additions, which will attach the Guest Additions iso to the VM. If you need to access the iso directly, it’s located in /Applications/VirtualBox.app/Contents/Resources/VirtualBoxVM.app/Contents/MacOS/VBoxGuestAdditions.iso after you install VirtualBox. Next:

```
$ mkdir /media/cdrom
$ mount /dev/cdrom /media/cdrom
mount: block device /dev/sr0 is write-protected, mounting read-only
```

Now that the image is mounted, we need to install a few tools and run the installer:

```
$ export KERN_DIR=/usr/src/kernels/`uname -r`
$ yum install -y gcc make perl kernel-devel kernel-headers
$ /media/cdrom/VBoxLinuxAdditions.run --nox11
```

Note: Some of the steps like OpenGL will fail, but this is OK. you can verify that everything is working correctly afterward by running:

```
$ VBoxControl --version
$ VBoxService --version
```

Setting up the vagrant User
=============

Add the vagrant User
-------------

Next, we’ll need to install a vagrant user so vagrant can login to the box. This enables several critical vagrant features like vagrant ssh and vagrant provision.

```
$ yum install -y sudo
$ useradd -m vagrant
$ usermod -aG wheel vagrant
$ echo vagrant | passwd vagrant --stdin
$ echo "vagrant ALL=(ALL) ALL" >> /etc/sudoers
$ echo "%wheel ALL=NOPASSWD: ALL" >> /etc/sudoers
$ echo 'Defaults env_keep="SSH_AUTH_SOCK"' >> /etc/sudoers
```

Finally, edit the /etc/sudoers file and add a bang ! to the requiretty line. This enables vagrant to sudo remotely.

```
Default !requiretty
```

Install opensshd
-------------

Since we want to be able to login to the box remotely we’ll need to install opensshd:

```
$ yum install -y openssh-server
$ echo "UseDNS no" >> /etc/ssh/sshd_config
```

You can verify that it’s running via:

```
$ service sshd status
$ /etc/init.d/sshd status
```

Add the vagrant public key
-------------

In order for the vagrant user to connect to the vm we’ll need to add the vagrant public key to the vagrant user’s authorized_keys file.

```
$ mkdir -m 0700 /home/vagrant/.ssh
$ curl -s https://raw.github.com/mitchellh/vagrant/master/keys/vagrant.pub > \
> /home/vagrant/.ssh/authorized_keys
$ chown -R vagrant:vagrant /home/vagrant/.ssh
$ chmod -R 0600 /home/vagrant/.ssh/*
```

Install Chef
================

Wait…what’s Chef? Vagrant uses something called a “provisioner” to deploy code and configuration to your box after you run vagrant up. Chef is one of these provisioners.

If you want to install the latest version of chef, it’s pretty easy:

```
$ curl -L https://www.opscode.com/chef/install.sh | bash
```

If you want to install a specific version of chef, it’s two lines:

```
$ curl -L https://www.opscode.com/chef/install.sh > /root/chef-install.sh
$ bash /root/chef-install.sh -v 10.18.2
```

This process installs chef using the chef omnibus, which includes a sandboxed version of ruby (so you don’t need system ruby to cooperate, or even be installed). Afterwards, you can verify the installation via:

```
$ chef-client -v
Chef: 10.18.2
```

Opscode has a page that documents this process, which you can reference if you run into problems.

Reboot
-------------

When you’re done making these changes, restart the machine.

```
$ shutdown -r now
```

Verify you can login with the vagrant user and sudo ls /root.

Packaging the Box
================

Once the OS is setup you’ll need to package the box for shipment. This process differs depending on whether you’re targeting VirtualBox or VMware.

First step in both cases, shut down your vm:

```
$ shutdown -h now
```

VMware
---------------

VMware puts its vms into ~/Documents/Virtual Machines.localized/, so I found mine via:

```
$ cd ~/Documents/Virtual Machines.localized/vagrant-centos62-x64.vmwarevm
```

In this directory you’ll find a handful of files:

```
vagrant-cent62-x64.vmdk
vagrant-centos62-x64.nvram
vagrant-centos62-x64.plist
vagrant-centos62-x64.vmsd
vagrant-centos62-x64.vmx
vagrant-centos62-x64.vmxf
```

This is almost everything we need, but first we’ll want to defrag (-d) and shrink (-k) the .vmdk file. Note that your .vmdk may be split into multiple files, but this doesn’t matter.

```
$ vmware-vdiskmanager -d vagrant-cent62-x64.vmdk
$ vmware-vdiskmanager -k vagrant-cent62-x64.vmdk
```

Once the disk is smallified a bit, add a metadata.json file:

```
metadata.json
{
    "provider":"vmware_fusion"
}
```

And finally, tar it all up into a box.

```
$ tar -czvf vagrant-centos62-x64.vmware.box ./*
```

Aaand we’re done! Skip to Adding and Upping the Box, below, to continue.

The vagrant docs have some notes on this process if you want to read more.

VirtualBox
------------------

Navigate into the directory where your box is located. On my system, this is:

```
$ cd ~/VirtualBox VMs/vagrant-cent62-x64.virtualbox
```

From here we’ll need to do a few things. First, compact the disk:

```
$ VBoxManage modifyhd vagrant-cent62-x64.virtualbox.vdi --compact
```

This will squash the disk a bit so it will take up less space when we package it. Next, we’ll need to export the VM into an odf package so vagrant can use it. I did this through the GUI.

Select your vm in the VirtualBox interface, click File menu, and then Export. In the dialog that appears, change the file extension to .odf. Click Continue and then Export.

When the process is complete you should have a handful of files that look like this:

```
vagrant-cent62-x64.virtualbox.ovf
vagrant-cent62-x64.virtualbox-disk1.vmdk
vagrant-cent62-x64.virtualbox.mf
```

For later steps, it will be easiest if these files are along in a directory so if they aren’t, create a directory and move them. For example:

```
$ mkdir ~/Desktop/vbox
$ mv vagrant-cent62* ~/Desktop/vbox/
```

Now on to the configuration. First, we need to rename the ovf file to box.ovf so vagrant will recognize it.

```
$ mv vagrant-cent62-x64.virtualbox.ovf box.ovf
```

Next, we need to add a metadata.json file to our vbox folder to indicate the provider to use with this vm. This should go in the same folder as your vagrant-cent62* files from above.

```
metadata.json
{
    "provider":"virtualbox"
}
```

And finally, we need to indicate the base mac address of the vm. I also did this through the VirtualBox GUI. Right-click the VM, Settings, Network tab, click the arrow next to Advanced, and copy the mac address. Mine is 08002710BCC4. We’re going to put this in a Vagrantfile in the same vbox folder as above.

```
Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.base_mac = "08002710BCC4"
end
```

Now the folder should look like this:

```
Vagrantfile
metadata.json
vagrant-cent62-x64.virtualbox.ovf
vagrant-cent62-x64.virtualbox-disk1.vmdk
vagrant-cent62-x64.virtualbox.mf
```

We’re ready to box! The special .box format vagrant uses is actually just a .tar with the above files and a .box extension, which we will make now, and then move to the desktop so we can find it easily.

```
$ tar -czvf vagrant-cent62-x64.virtualbox.box ./*
$ mv vagrant-cent62-x64.virtualbox.box ~/Desktop
```

Using the Box
=======================

You’re ready to add the box and test!

```
$ vagrant box add vagrant-cent62-x64 \
> ~/Desktop/vagrant-cent62-x64.virtualbox.box --provider virtualbox
```

You can do the same for the other box (if you made one) using --provider vmware_fusion. The name of the box can be the same since --provider is used to differentiate them.

Note: if you have to repeat this process a few times (like I did when I was testing), you can remove the box from vagrant using vagrant box remove vagrant-cent62-x64 virtualbox. This will delete the copy that vagrant keeps in its cache, but will not touch the copy on your desktop. This is a good way to test downloading the box if you upload it somewhere.

Finally, we’ll want to try using our box with vagrant up.

```
$ mkdir ~/Desktop/vagrant-test
$ cd ~/Desktop/vagrant-test
$ vagrant init
```

This will create a Vagrantfile in the current directory. Open it in your favorite editor and change config.vm.box to match the one we just added:

```
Vagrant.configure("2") do |config|
  config.vm.box = "vagrant-cent62-x64"
end
```

Then:

```
$ vagrant up --provider virtualbox
```

And you’re off to the races!
