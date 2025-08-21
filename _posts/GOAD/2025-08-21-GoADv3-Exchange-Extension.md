---
title: "GoADv3 Exchange Extension"
toc: true
toc_label: "Table of Contents"
toc_icon: "folder-open"
toc_sticky: true
category: GOAD
tags: GOAD AD Guide Exchange
excerpt: "This is a guide on how I installed the Exchange Extension for GOADv3"
header:
  teaser: "/assets/images/Exchange-Installation-Images/Exchange-Server-2019.jpg"
---

In this guide we are going to be extending the capability of our GoADv3 installation to include an exchange server.

As I am following along with the book "Pentesting Active Directory and windows-based infrastructure" by Denis Isakov the first chapter is about pentesting exchange. It initially mentions the installation of GoADv2 and then immediately changes to use DetectionLab for this first chapter due to the lack of Exchange in GoADv2. 

Good News, GoADv3 has an Exchange module, so I will be using that instead of DetectionLab.

Here is how I installed it.

# 1 - Running The Script

## WSL as Root and set PATH

As with the previous installation, we are going to want to be running PowerShell/Terminal as administrator, and be running WSL as Root.

First things first, Run Terminal/PowerShell as admin
{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/terminal-admin.png" alt="terminal-admin"%}{: .align-center}

And navigate to our GOAD folder:

```powershell
cd C:\GOAD
```

As with the first installation we are going to need the `$PATH` from our standard user; if its working as expected, so first thing is first, we need to check if our `$PATH` is working.

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

Hopefully you will see that everything is green, this means its all ready and prepared. however, we want to run this as Root, 

However running as `sudo` will give the following results when using `check`

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

## First Installation

So we are now all caught up with the previous installation and we now want to install an exchange server, lovely.

From the `goad.sh` script, we can view all currently installed labs by using `ls`:

{% include figure popup=true image_path="/assets/images/Exchange-Installation-Images/goad-ls.png" alt="goad-ls"%}{: .align-center}

If you are following along with the installation series, we will only have one lab so far, and by default, it will be selected.

if you have multiple labs, you will need to select the lab you are going to install the extension on to.

```
load <instance id>
```

Next, we are going to need the rest of the lab to be started, this is so that the new virtual server can connect to `sevenkingdoms.local` domain, and the installation can configure the new virtual server correctly.

```
start
```

Give the `start` command some time to bring up the lab

{: .notice--warning}
>It is possible that the lab has not fully started, check your VMware Workstation to see if the green play icon is present next to each machine, if not, fire those servers up manually.

Now to start the installation.

```
install_extention exchange
```

As with the main installation, this did not go smoothly for me.

# 2 - Connection Errors Again

{% include figure popup=true image_path="/assets/images/Exchange-Installation-Images/exchange-connection-error.png" alt="exchange-connection-error"%}{: .align-center}

Thankfully, the vagrant section of this installation went well and the new virtual server is up and running, albeit without any of the Ansible configuration, much like the initial install we have the box installed but nothing on it.

As with the main installation, we will need to ensure that the host-only adaptor is set, make sure that the IP address is correct and, Disable the firewall.


## Adding the VM to VMware

{: .notice--info}
>This section has been stolen from the main setup guide, and shortened into a more TL;DR version, as we have already done this before.

You'll want to add the new VM into the `GOAD` folder in VMware, right click an open space in the panel and select `Scan for Virtual Machines`

{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/vmware-scan.png" alt="vmware-scan"%}{: .align-center}

Select your `C:\GOAD` folder and scan for VM's, the new VM should be found and it can be added to the GOAD folder along with the rest of the lab.

## Setting Host-Adaptor

Select the new `GOAD-SRV01` machine this should have the default IP address of `192.168.56.21`, we need to first check the hardware assigned adaptors, as expected we have 2 for each machine.

Right Click the Device and go to `settings`,  from there, select `Network Adaptor 2` it is important to note that whilst the listed adaptor in the left panel says `Custom (VMnet2)` half of the time, that is just not the case and another adaptor is selected.

From the options on the right hand side, make sure that `VMnet2` is selected.
You will probably have to come back in here and check again after the machine has been restarted, because for me it changed back to `VMnet0` for some reason.

{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/vmware-host-only-vmnet2.png" alt="vmware-host-only-vmnet2"%}{: .align-center}

Next, log in to check the currently assigned IP and to disable the firewall.

Send the `CTRL+ALT+Delete` signal and log into the Vagrant account, credentials `vagrant:vagrant`

Run an `ipconfig` in PowerShell/CMD

Looking at Ethernet adapter `Ethernet1` we are looking for the expected IP address of `192.168.56.21`

If it is not correct, you'll need to go into the adaptor settings and just set the IP address statically.

As a reminder, here is a table of the IP addresses for each machine on the GoADv3 Lab including the new `GOAD-SRV01`

| Hostname   | Expected IP   |
| ---------- | ------------- |
| GOAD-DC01  | 192.168.56.10 |
| GOAD-DC02  | 192.168.56.11 |
| GOAD-DC03  | 192.168.56.12 |
| GOAD-SRV01 | 192.168.56.21 |
| GOAD-SRV02 | 192.168.56.22 |
| GOAD-SRV03 | 192.168.56.23 |
{: .display}  

## Dropping the Firewall

Next is to disable the firewall, the easiest way to do this is to open the start menu, and type `firewall`
{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/firewall-status.png" alt="firewall-status"%}{: .align-center} 

Select `Check Firewall Status`

Select `Turn Windows Defender Firewall on or off`
{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/firewall-off.png" alt="firewall-off"%}{: .align-center} 

And select `Turn off Windows Defender Firewall (not recommended)`, on both the `Private network settings` and the `Public network settings`

{% include figure popup=true image_path="/assets/images/GOADv3-Installation-Images/firewall-networks.png" alt="firewall-networks"%}{: .align-center} 


Now back to the `goad.sh` script on the windows host to try the installation again.


```
install_extention exchange
```

# 3 - AD Role Error

This time, we got further, but there are still weird configuration issues. 

After some playing around, and investigating the Ansible files, and investigating this error, it looked like it was not finding some of the other configuration files, specifically a role called `ad`

{% include figure popup=true image_path="/assets/images/Exchange-Installation-Images/exchange-ad-role-error.png" alt="exchange-ad-role-error"%}{: .align-center}

To resolve this, i simply copied all of the roles from the root Ansible folder to the exchange extension folder.

```
copy C:\GOAD\ansible\roles

paste it in C:\GOAD\ansible\extentions\exchange\ansible
```

This will most likely repeat some of the initial installation steps, adding users and so on, but it allows us to get further into the installation and that's what is important here.

I also found that in the `install.yml` file for the exchange extension the `data_path` did not seem to go to the correct data file, it was not going back enough directories to be able to find the global configuration files.

The file is located in `C:\GOAD\extensions\exchange\ansible\install.yml`

It reads:

```yaml
# read global configuration file and set up adapters
- import_playbook: "../../../ansible/data.yml"
  vars:
    data_path: "../ad/{{domain_name}}/data/"
  tags: 'data'
```

So i changed it to:

```yaml
# read global configuration file and set up adapters
- import_playbook: "../../../ansible/data.yml"
  vars:
    data_path: "../../../ad/{{domain_name}}/data/"
  tags: 'data'
```


Back to the main `goad.sh` script:

```
install_extention exchange
```

# 4 - Exchange Installation Error

Another Error, this time it is to do with the installation of Exchange 2019 on the new server.

{% include figure popup=true image_path="/assets/images/Exchange-Installation-Images/exchange-install-error.png" alt="exchange-install-error"%}{: .align-center}

For some reason after much pondering, it seems that exchange does not like being unattended when installed, so were going to need to essentially reset the installation by removing some entries from the registry and start the installation again from the GUI, this way it correctly installs the dependencies and resolves the role errors.

As mentioned the failed role's  `watermark` and `action` registry keys need to be deleted, then the setup.exe needs to be ran as the enterprise admin of the main domain `sevenkingdoms.local`

The registry keys are located in:

```
HKLM\SOFTWARE\Microsoft\ExchangeServer\v15
```

In my case the role `ClientAccess` is the offending role, and the registry key for it is located in:

```
HKLM\SOFTWARE\Microsoft\ExchangeServer\v15\ClientAccessRole
```

The affected role may be different for someone else, but the information is here on how to reset any failed role installation.


{: .notice--info}
>I also turned the **Domain** firewall and Windows Defender Real time protection off for the install, based on some online suggestions for other errors during installation, then turned it back on when the installation was done.

Delete the `watermark` and `action` keys on the failed role to be able to re-run the install

{: .notice--info}
>You will need to delete these keys for each time the installation fails, its essentially a rearm.

{% include figure popup=true image_path="/assets/images/Exchange-Installation-Images/exchange-reg-keys.png" alt="exchange-reg-keys"%}{: .align-center}

As mentioned in the beginning of this section, we will need to now run this installation from the GUI because for some reason this installs the dependencies correctly.

On the machine, run PowerShell as Administrator, then run the following commands.

```powershell
cd E:\
runas /u:sevenkingdoms.local\Administrator setup.exe

Password: 8dCT-DJjgScp

```

{: .notice--info}
>Credentials for the Lab are found in the config file `config.json` located at `C:\GOAD\ad\GOAD\data\config.json`
  
{: .notice--warning}
>**Quote from Microsoft:**  
>"Select the check box in the Exchange Setup Wizard to install Windows prerequisites."  
>I did not get a check box to select, the installation just worked, but if you do get a check box, check it.

Once the installation is complete, restart the server.

Then once the machine is back up and running, from the main `goad.sh` script, run the installation again.

```
install_extention exchange
```

# 5 - Success!

{% include figure popup=true image_path="/assets/images/Exchange-Installation-Images/exchange-install-done.png" alt="exchange-install-done"%}{: .align-center}

Great, this looks like its finished this time, a great way to check is to visit the the following URL in a browser to see if OWA is enabled on the server.

```
https://192.168.56.21
```

{% include figure popup=true image_path="/assets/images/Exchange-Installation-Images/exchange-owa.png" alt="exchange-owa"%}{: .align-center}

Fantastic, time to start exploring.