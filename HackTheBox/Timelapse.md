# Timelapse
Timelapse is an easy machine from HackTheBox that starts with publicly accessible SMB shares and ends with abusing account permissions to pull the Administrator password from LAPS.

As with any CTF from HackTheBox, I always start with a Nmap scan of the target. Before I do that though, I set the target IP to a variable first:

```bash
target=10.129.227.113
```
I don't lways do this, sometimes it just helps me work through the box quicker so I don't always have to go and copy the IP.

From there, I run the following Nmap command:

```bash
sudo nmap -Pn -p- --min-rate 5000 -oN nmap-ports.txt $target && ports=$(grep ^[0-9] nmap-ports.txt | cut -d'/' -f1 | tr '\n' ',' | sed 's/,$//') && sudo nmap -Pn -n -p$ports -A -oN nmap-full.txt $target
```

This one-liner does a full scan of all 65,535 TCP ports, then whatever ports come back as open, those are saved to the ports variable. From there, those are fed into another Nmap scan with the -A flag. This performs OS detection, service version detection, and script scanning. I also send the output to a .txt file for future reference if needed.

Once this is complete, we see the following:

<img width="1953" height="389" alt="image" src="https://github.com/user-attachments/assets/f100168f-eb1b-4958-9997-31fb229142b7" />

<img width="1291" height="751" alt="image" src="https://github.com/user-attachments/assets/408f3b4f-53b8-4031-bbc7-9326d28e855f" />

The main ports I am interested in here are 88, 389, 445, and 5986:
- tcp/88 is going to be Kerberoas, indicating that we are dealing with a domain controller
- tcp/389 is going to be LDAP, further strenghtining the domain controller theory
- tcp/445 is SMB
- tcp/5986 is the SSL implementation of WinRM
  
If SMB is open on a host, I'll typically start there, looking for quick wins. I'll try null authentication, and if that fails I'll try Guest authentication:

```bash
nxc smb $target -u '' -p '' --shares
nxc smb $target -u 'Guest' -p '' --shares
```

<img width="1438" height="326" alt="image" src="https://github.com/user-attachments/assets/40396ea9-ad37-4fad-8449-0425ac76fb23" />

In this case, we have Guest authentication and read access over a non-standard share named "Shares". We also can see that the target's hostname is DC01 and the domain is timelapse.htb. 

I'll utilize smbclient to access this share and see what we can find:

<img width="1379" height="618" alt="image" src="https://github.com/user-attachments/assets/f85cc200-8523-4bda-8e7f-024d2e35c216" />

We see two directories, "Dev" and "HelpDesk". Inside of "Dev", we see "winrm_backup.zip", so we grab that. Inside of HelpDesk, we see documentation for LAPS, hinting that LAPS is likely configured on this domain controller.

Upon attempting to unzip the .zip file we retrieved, we see it is password protected. Luckily, JohnTheRipper has a module available to create a hash for this, zip2john. Putting that hash through John with the rockyou wordlist, we see the password to unzip the file is `supremelegacy`

<img width="1202" height="250" alt="image" src="https://github.com/user-attachments/assets/1d00d96b-f5c6-440a-8776-730a71a2e187" />

Inside of the .zip, we see "legacyy_dev_auth.pfx". A .pfx file a file that contains both a users public key and private key. In this case, it appears they were used for WinRM authentication. This is also password protected though. We must use pfx2john in order to get a password hash.

<img width="1318" height="492" alt="image" src="https://github.com/user-attachments/assets/c54e19d3-63df-461d-bb05-01f08389f92f" />

Once again, we can crack that with John and the rockyou wordlist.

<img width="900" height="227" alt="image" src="https://github.com/user-attachments/assets/80912610-1a5c-4a4b-876f-314df6263fda" />

The password for our .pfx file is `thuglegacy`.

Next, we can utilize openssl in order to separate the private key and SSL certificate for authentication

```bash
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out key.pem -nodes
openssl pkcs12 -in legacyy_dev_auth.pfx -clcerts -nokeys -out cert.crt -nodes
evil-winrm -S -k ./key.pem -c ./cert.crt -i 10.129.227.113
```

<img width="1148" height="341" alt="image" src="https://github.com/user-attachments/assets/c01e0c55-b3ff-4d6a-8f45-5ee28557201a" />

Awesome! We have initial access as the user legacyy.

Running a quick check (`whoami /all`) to see our current user's groups and privileges, I see nothing of interest:

<img width="1668" height="731" alt="image" src="https://github.com/user-attachments/assets/e3d9ae4a-8a66-4706-8249-9b675feaa8d7" />

Next, I transferred winPEAS to the target via PowerShell's Invoke-WebRequest and saved that as wp.exe, then executed it:

<img width="1866" height="759" alt="image" src="https://github.com/user-attachments/assets/328a06d3-24fa-4561-832e-5e0c1d618a35" />

While this was running, I had a lightbulb moment and remembered LAPS was likely enabled on this host. However, I knew that in order to abuse it, I was going to need an account with credentials.

winPEAS confirmed my LAPS suspicions, and it also noted that I had a PowerShell console history text file with some data inside.

<img width="775" height="125" alt="image" src="https://github.com/user-attachments/assets/cda1c09e-1d71-41fa-97f7-274a0f72ab15" />

Upon listing out the contents of the PowerShell history file, I saw credentials for the user "svc_deploy":

<img width="1183" height="202" alt="image" src="https://github.com/user-attachments/assets/3236e5d9-bef7-44bc-97bf-da70aa0b6c3d" />

Using those credentials, I could now attempt to retrieve the Administrator password using the Netexec laps module:

```bash
nxc ldap 10.129.227.113 -u 'svc_deploy' -p 'E3R$Q62^12p7PLlC%KWaxuaV' -M laps
```

<img width="1398" height="127" alt="image" src="https://github.com/user-attachments/assets/801f058d-fd82-4e54-873a-be19b3bde8e3" />

And that is a success! We can confirm the credentials are valid by using a tool I like to use called nxcspray - https://github.com/NTHSec/nxcspray

```bash
nxcspray all 10.129.227.113 -u 'Administrator' -p 'ySma$LzFUKpX1[I/++0+K+58'
```

<img width="1492" height="349" alt="image" src="https://github.com/user-attachments/assets/4714f212-3aea-41c2-9200-addcdfe3a568" />

Our credentials are valid, so we can utilize impacket-psexec to gain access

<img width="1085" height="283" alt="image" src="https://github.com/user-attachments/assets/46f4e7ec-b9b5-41f4-af96-2e6a536c95f9" />

And with that, we have a SYSTEM shell, and that is the box!
