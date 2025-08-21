---
title: "GoADv3 Installation"
toc: true
toc_label: "Table of Contents"
toc_icon: "folder-open"
toc_sticky: true
category: GOAD
tags: GOAD AD Guide
excerpt: "This is a guide on how I installed GOADv3 in VMware using WSL in Windows 11"
header:
  teaser: "/assets/images/GOADv3-Installation-Images/logo_GOAD3.png"
---

{: .notice--info}
>**Requirements:**
>
>We are going to install the main `GOAD` lab, so be sure to have at least:  
>`24GB RAM`  
>`120GB Disk Space`

[GOAD Official Documentation](https://orange-cyberdefense.github.io/GOAD/installation/)

For this installation I will be installing GOAD using a Windows 11 host, with VMware Workstation 17 Pro and WSL

Some sections of this guide will be taken directly from the above listed official documentation but with any additional steps taken to resolve issues faced.

# 1 - Preparing Windows

## VMware Workstation

{: .notice--info}
>**Important:**  
>VMware Workstation Pro Is now free for public use. Though the broadcom pages are rubbish and require an account to use.

Install VMware Workstation : [Vmware Workstation Pro Download](https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware+Workstation+Pro)

{: .notice--danger}
>**VMware Installation Bug:**  
>if you got an error about groups and permission during vmware workstation install consider running this in an administrator cmd prompt: 
>
>```powershell
>net localgroup /add "Users"
>net localgroup /add "Authenticated Users"
>```


## Vagrant

If you want to create the lab on your windows computer you will need vagrant. Vagrant will be responsible to automate the process of VM download and creation.

Download and install visual c++ 2019 : [vc_redist.x64.exe](https://aka.ms/vs/17/release/vc_redist.x64.exe)  
Install vagrant : [Vagrant Main Download Page](https://developer.hashicorp.com/vagrant/install)

{: .notice--info}
>**Vagrant Binaries Page:**  
>This Link is important as the main WSL install of Vagrant is a different version from the Main Windows Version, This page will become useful later on when installing Vagrant on WSL to match versions.
>[Vagrant Binaries Pages](https://releases.hashicorp.com/vagrant/)

Install vagrant VMware utility : [Vagrant VMware Utility](https://developer.hashicorp.com/vagrant/install/vmware)

Once vagrant is installed on windows, run the following in a command prompt to install the relevant dependencies:

```powershell
vagrant.exe plugin install vagrant-reload vagrant-vmware-desktop winrm winrm-fs winrm-elevated
```

## WSL Installation

We are going to be installing WSL on the windows host to manage the provisioning of the Lab.
For this we will need a distribution we are comfortable using, to list all current WSL distros run the following in a terminal on windows.

```powershell
wsl --list --online
```

For this installation I will be using Debian

```powershell
wsl --install debian
```

{: .notice--warning}
>**WSL Setup:**  
>Run through the prompts such as setting a root password and restart your host when required, it just makes it easier.

{: .notice--info}
>**Install WSL 1**  
>New Linux installations, installed using the wsl --install command, will be set to WSL 2 by default. The wsl --set-version command can be used to downgrade from WSL 2 to WSL 1 or to update previously installed Linux distributions from WSL 1 to WSL 2. To see whether your Linux distribution is set to WSL 1 or WSL 2, use the command: `wsl -l -v`. To change versions, use the command: `wsl --set-version <distro name> <wsl_version>` replacing with the name of the Linux distribution that you want to update. As an example: `wsl --set-version Debian 1` will set your Debian distribution to use WSL 1.

### Prepare WSL distribution

To open your new Debian type the following in PowerShell / Terminal

```powershell
wsl
```

Verify you are using python version >= 3.8:

```bash
python3 --version
```

{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/python-version.png" alt="python-version"%}{: .align-center}

Install python packages:

```bash
sudo apt update
sudo apt install python3 python3-pip python3-venv libpython3-dev git
```

### Install Vagrant on WSL 

This is where the Vagrant [Binaries page](https://releases.hashicorp.com/vagrant/) works wonders, if you just run a `apt install vagrant` type of command, you will get some obscure WSL dev version, however for the setup to work correctly, you need to ensure that both the Windows version of Vagrant and the WSL version of Vagrant match!

Download the version of Vagrant that you have installed on windows, in my case 2.4.8

Copy the `.deb` file to a suitable place on your drive, to make things easier I added it to the `vagrant` directory in `c:\GOAD`

**Install:**
```bash
dpkg -i vagrant_2.4.8-1_amd64.deb
```

Check the vagrant versions across the windows host and the WSL session to ensure they now match; you should be able to use the same command for both PowerShell and WSL.

```bash
vagrant --version
```

## Clone GOAD

Make a directory off of one of your main drives, I am making a directory in `C:\` called `GOAD`: `C:\GOAD`

{: .notice--info}
>This path HAS to be on one of your drives (C:/D:/E:/...)

We will be using `git` to download all of the files used to create the lab, if you have it installed on windows i.e. `git desktop` or `git` then clone using Terminal/PowerShell as Admin

If not, you can also do it from within WSL, but you may need to install `git`

### Windows
```powershell
cd c:\
git clone https://github.com/Orange-Cyberdefense/GOAD.git
cd c:\GOAD
```

### WSL
```
cd /mnt/c/whatever_folder_you_want
git clone https://github.com/Orange-Cyberdefense/GOAD.git
cd GOAD
```  


# 2 - Preparing the VM's

{: .notice--danger}
>**Hell**  
>This is the section where all hell breaks loose and the [Official Documentation](https://orange-cyberdefense.github.io/GOAD/providers/vmware/) does not cover any of it in detail.

First thing is to check that you have the designated Prerequisites installed and ready to go.
## Prerequisites

- Providing
    
    - [Vmware workstation](https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware+Workstation+Pro)
    - [Vagrant](https://developer.hashicorp.com/vagrant/docs)
    - [Vmware utility driver](https://developer.hashicorp.com/vagrant/install/vmware)
    - Vagrant plugins:
        - vagrant-reload
        - vagrant-vmware-desktop
        - winrm
        - winrm-fs
        - winrm-elevated
- Provisioning
    
    - Python3 >=3.8
    - goad requirements
    - ansible-galaxy goad requirements


## Configuration Files

First of all, we are going to address some of the issues I faced from the get go, to hopefully prevent them later.

Taken from the [Troubleshooting Page](https://orange-cyberdefense.github.io/GOAD/troobleshoot/), the below error has caused me no end of issues, and there are several steps I have taken to try and address this.

## ansible persistent "unreachable error"

- Unreachable means ansible can't contact the VM's.
  - Maybe the VM's didn't get the right IP? (try to connect with `vagrant:vagrant` on the VM and check it's IP)
  - Or you have a firewall on the VM which does the provisioning which could be blocking the WinRM connection?
  - Or maybe it is a vagrant issue: https://github.com/Orange-Cyberdefense/GOAD/issues/12
  - You could try to switch on port 5985 to connect without SSL as suggest here : https://github.com/Orange-Cyberdefense/GOAD/issues/98 by uncommenting the lines in the inventory file you use:

```
# ansible_winrm_transport=basic
# ansible_port=5985
```

## Inventory File

As its not explained above, the default inventory file for the main `GOAD` Lab is located (assuming you're following along) at: 

```
C:\GOAD\ad\GOAD\data\inventory
```

This is essentially the main file that does the initial setup of the Virtual Machines, it points to which `.yml` file is to be used for which machine, and also configures the default local admin account for each machine `vagrant:vagrant`

In this file we are going to uncomment the configuration lines listed above (Remove the `#`):

```
ansible_winrm_transport=basic
ansible_port=5985
```

{: .notice--info}
> **Speed up Failure**  
> During troubleshooting it was a painful experiance to have to wait upwards oft 30 minutes for the installation to fail each time, if you are comfortable changing the timeouts its worth changing the values to:
> ```
> ansible_winrm_operation_timeout_sec=50
> ansible_winrm_read_timeout_sec=60
> ```
> The Operation Timeout has to be lower than the Read Timeout.

{: .notice--warning}
>Further changes to our `Vagrantfile` will need to be made, however that cannot be done yet as it is not generated for our specific lab environment yet

## WSL Session Configuration

## WSL as Root

Scary stuff, but to make things easier, its time to run the installation as Admin/Root! A few things need to be addressed before we can do this properly.

First things first, Run Terminal/PowerShell as admin
{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/terminal-admin.png" alt="terminal-admin"%}{: .align-center}

And navigate to our GOAD folder:

```powershell
cd C:\GOAD
```

So, we are going to be running WSL as root, so the install goes smoothly, but we need the `$PATH` from our standard user; if its working as expected, so first thing is first, we need to check if our `$PATH` is working.

The best way to check, is to run the GOAD shell script, and perform a check, Fire up WSL:

```powershell
WSL
```

Run the Script

```bash
./GOAD.sh -p vmware
```

When the script is running, we want to use the `check` command:

```bash
check
```

{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/check-command.png" alt="check-command"%}{: .align-center}

Hopefully you will see that everything is green, this means its all ready and prepared. however, we want to run this as Root, However running as `sudo` will give the following results when using `check`:

{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/broken-check.png" alt="broken-check"%}{: .align-center}

This is because the root user's `$PATH` is different, to solve this issue, were going to copy the `$PATH` from the standard user, and use that for this current session.

{: .notice--warning}
>This is a change only for this current terminal session, if the session is closed at any point the `$PATH` will need to be imported again.

Exit out of the `GOAD.sh` script and `echo` the `$PATH` of the standard user

```
echo $PATH
```

{: .notice--danger}
>**Very Important!**  
>The commands `echo $PATH` and `$PATH` are two different things; It took me far too long to realize this so I'm letting you know now.

You'll get a lovely long `$PATH` but just check that the following entries are there:
```
/mnt/c/Program Files/Vagrant/bin/vagrant.exe
/mnt/c/Program Files/Vagrant/bin
```

It's of note that WSL requires `.exe` extensions when addressing specific executables.

Next we need to add this whole path to the super user's path (temporarily), upgrade your WSL session to the super user:

```bash
sudo su
```

And import your copied `$PATH` from the standard user:

```bash
export PATH="<Paste Path Here>"
```

Open up the Script again and run the `check` command to see if everything is now green:

```bash
./GOAD.sh -p vmware
check
```

{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/check-command.png" alt="check-command"%}{: .align-center}

# 3 - Installing the Base VM's to Work With

{: .notice--warning}
>**Failure**  
>This is going to fail; at least from my experience.
>The reason we are running it now, is to begin the construction of our lab and install the operating system on each device, setting up the `vagrantfile` and setting the initial network settings.


Setting up the lab's configuration is pretty simple through the `GOAD.sh` script.
As you may have noticed we have already set the provider with the `-p` switch when running the script. 

To see the current settings that will be applied use the `config` command.

```bash
config
```

Were looking at the top section for the current configurations:  
{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/goad-config.png" alt="goad-config"%}{: .align-center}

Set the lab you want to install, for this guide we are going with the full installation `GOAD`, the other options are:  
`GOAD/GOAD-Light/NHA/SCCM`

```bash
GOAD/vmware/local/192.168.56.X > set_lab <lab>
GOAD/vmware/local/192.168.56.X > set_lab GOAD
```  

Setting the IP range, by default the IP range is `192.168.56.0/24` you can specify the first 3 octets if you want to change it, or if you're installing a second lab and want a different range.

```bash
GOAD/vmware/local/192.168.56.X > set_ip_range <ip_range>
GOAD/vmware/local/192.168.56.X > set_ip_range 10.0.1
GOAD/vmware/local/10.0.1.X >
```  

Time to install, type:

```
install
```

After some time; at least in my experience it fails, this isn't great but its a start and now the lab should be at least all up and running.

{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/first-install-fail.png" alt="first-install-fail"%}{: .align-center}

You will most likely get errors like above and as previously mentioned in [this section](#ansible-persistent-unreachable-error).  

If you have followed along with this guide, you have enabled set the `inventory` file to use HTTP for WinRM connections, and you will still be getting errors like above. 
In short, i have tried with HTTPS and HTTP and they both didn't work, so i left it as HTTP and carried on with the remaining steps to get it to work. I'm not sure if i could have ran it with the default HTTPS but I'm just going through what got it to work for me and to hopefully also work for you.

TL;DR: We will get it to work just keep going.

## Getting it to Work

Open up VMware Workstation on your windows host, and open the library (side panel), then right click in the panel and make a new folder for all of your GOAD VM's to be organized in.

{: .notice--info}
>The VM's will not be listed here yet, we will add them in a second.

{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/vmware-folder.png" alt="vmware-folder"%}{: .align-center}

Next, you'll want to add the VM's into that folder, right click an open space in the panel again and select `Scan for Virtual Machines`

{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/vmware-scan.png" alt="vmware-scan"%}{: .align-center}

Select your `C:\GOAD` folder and scan for VM's, the list should populate and you can drag the VM's into the GOAD folder.

# 4 - Configuring the VM's

In this section we will be configuring the VM's to not be a jumbled mess of default base boxes, but to be the fully functioning AD LAB known as GOAD!

So where are we?

At this stage we should have Windows configured, A bunch of boxes imported into VMware, and WSL running the GOAD script as Root in an Administrator Terminal with lots of networking errors.

{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/first-install-fail.png" alt="first-install-fail"%}{: .align-center}

First things first, we want to make another change to the configuration files, this couldn't be done previously as the config file we are looking for did not exist until we created the initial VM's

## Vagrantfile

We need to find the `vagrantfile` this will be located in the `C:\GOAD\workplace` directory.
There will be a new folder with the ID of your new lab, in my instance `fdafab-goad-vmware` from there within the `provider` directory you will find `vagrantfile`

We will be adding the following lines to to the file:

```
  config.winrm.transport = "plaintext"
  config.winrm.basic_auth_only = true
```

The best place to add the above lines is with the rest of the `config.*` section:

```
  config.vm.boot_timeout = 600
  config.vm.graceful_halt_timeout = 600
  config.winrm.retry_limit = 30
  config.winrm.retry_delay = 10
```

Like this:

```
  config.vm.boot_timeout = 600
  config.vm.graceful_halt_timeout = 600
  config.winrm.retry_limit = 30
  config.winrm.retry_delay = 10
  config.winrm.transport = "plaintext"
  config.winrm.basic_auth_only = true
```

## The Networking Section

{: .notice--info}
>**The Default IP's of each box:**  
>
>| Hostname   | Expected IP   |
>| ---------- | ------------- |
>| GOAD-DC01  | 192.168.56.10 |
>| GOAD-DC02  | 192.168.56.11 |
>| GOAD-DC03  | 192.168.56.12 |
>| GOAD-SRV02 | 192.168.56.22 |
>| GOAD-SRV03 | 192.168.56.23 |
>{: .display}

Ok, so Host Adaptors in VMware; good fun.

Firstly, you need to be aware that due to the failure of the installation, most of the configuration has not been set up. the machines are basically barebones at the moment potentially with the correct networking, potentially not.

So we need to first of all, check the networking, this is going to involve checking the VMware Adaptor settings, checking the VM's assigned networking hardware configuration then logging on to each machine, checking the IP's (possibly altering them), checking the Host Adaptor settings (possibly altering it), and then disabling the firewall on each VM.

### Checking the VMware Adaptor Settings

In VMware, Press Edit in the top options bar, and select `Virtual Network Editor...`

{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/vmware-vna.png" alt="vmware-vna"%}{: .align-center}

From here we are looking for the Adaptor with the IP address range that we set earlier in the GOAD script.  

{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/vmware-vmnet2.png" alt="vmware-vmnet2"%}{: .align-center}  

Other Adaptors may be present, some are default, some will be part of GOAD such as `VMnet8` in this instance.
We are only interested in the adaptor with the IP range that we specified, as mentioned before the default is `192.168.56.0/24` in my case `VMnet2`

Now go to your control panel and go to the network and sharing center, we need to check our Host adaptor.

Find the Adaptor with the same name as the one we found in VMware in my case `VMnet2`

{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/network-adaptor.png" alt="network-adaptor"%}{: .align-center} 

Right click and select, Properties

From here were going to save some pain, and set a static IP address for our host, on that network.
I chose an IP address out of the default IP's `192.168.56.100`

{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/setting-static-ip.png" alt="setting-static-ip"%}{: .align-center} 

Accept those changes and we can move on to checking each VM

### Fixing each VM

Starting with the first machine `GOAD-DC01` as mentioned above the default IP address for this machine is `192.168.56.10`, we need to first check the hardware assigned adaptors, as expected we have 2 for each machine.

Right Click the Device and go to `settings`,  from there, select `Network Adaptor 2` it is important to note that whilst the listed adaptor in the left panel says `Custom (VMnet2)` half of the time, that is just not the case and another adaptor is selected.

From the options on the right hand side, make sure that `VMnet2` is selected.
You will probably have to come back in here and check each machine again at some point, because for me it changed back to `VMnet0` for some reason.

{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/vmware-host-only-vmnet2.png" alt="vmware-host-only-vmnet2"%}{: .align-center} 

Now do this for every machine...

Next, you will need to log in to each machine, to check the currently assigned IP's and to disable the firewall.

From making the above changed you most likely have every VM open in the VMware Workstation window.

Starting again with `GOAD-DC01` were going to send the `CTRL+ALT+Delete` signal and log into the Vagrant account, credentials `vagrant:vagrant`

From there, we need to open up a PowerShell / CMD window, whichever you prefer and do and `ipconfig`  

{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/ipconfig.png" alt="ipconfig"%}{: .align-center} 

Looking at Ethernet adapter Ethernet1 we can see that the expected IP address of `192.168.56.10` is present.

If it is not correct, you'll need to go into the adaptor settings on this VM, as we did above with the Host, and just set a static IP address, matching the following table:  

| Hostname   | Expected IP   |
| ---------- | ------------- |
| GOAD-DC01  | 192.168.56.10 |
| GOAD-DC02  | 192.168.56.11 |
| GOAD-DC03  | 192.168.56.12 |
| GOAD-SRV02 | 192.168.56.22 |
| GOAD-SRV03 | 192.168.56.23 |
{: .display}  

Next is to disable the firewall, the easiest way to do this is to open the start menu, and type `firewall`
{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/firewall-status.png" alt="firewall-status"%}{: .align-center} 

Select `Check Firewall Status`

Select `Turn Windows Defender Firewall on or off`  
{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/firewall-off.png" alt="firewall-off"%}{: .align-center} 

And select `Turn off Windows Defender Firewall (not recommended)`, on both the `Private network settings` and the `Public network settings`

{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/firewall-networks.png" alt="firewall-networks"%}{: .align-center} 

## Install Again

With all of that out of the way, its worth testing a ping to each device from your host, you should now get a response:

{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/goad-ping.png" alt="goad-ping"%}{: .align-center} 

{: .notice--warning}
>Give each machine a restart, to make sure that the settings have stuck, and once each machine is back up, try pinging again.


### Back to WSL

From the Host WSL session you should still be in the Administrator Terminal running GOAD.sh as Root.

Time to try the install again.

```bash
install
```

This time you should see a lot better results:
{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/goad-install-ok.png" alt="goad-install-ok"%}{: .align-center} 

However, after some time, you may get the same error message as me:
{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/goad-domain-error.png" alt="goad-domain-error"%}{: .align-center} 

This one was a simple fix, after the installation script failed 3 times, and the installation was canceled. 
I restarted `GOAD-DC02` (because its the DC for `north.sevenkingdoms.local`) and the machine that did not want to connect to the domain `north.sevenkingdoms.local` in this case `GOAD-SRV02`

Once the machines were restarted, and give 2-3 minutes to fully boot up, I ran the installation again.

```bash
install
```  

# Finally

And after and hour and a half i was greeted with the following message in my terminal

{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/goad-install-done.png" alt="goad-install-done"%}{: .align-center} 

```
[*] Lab successfully provisioned in 01:39:04
```

After all of that, it's now time to start using the lab!