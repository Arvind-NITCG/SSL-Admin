# SSL Admin 2026 - Task Documentation
**Author:** Arvind K N
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
## Task 1: Initial Setup
### Context
To begin the induction tasks, I provisioned an Ubuntu 22.04 LTS Virtual Machine (VM) hosted on Microsoft Azure. The VM was assigned a public IP address (`20.219.226.123`), and an RSA private key (`akn4ssl_key.pem`) was generated for secure authentication. 

### Concept Learned
**SSH (Secure Shell) & Public Key Cryptography:** SSH is a client-server protocol that provides an encrypted terminal session over an unsecured network. Instead of passwords, it uses asymmetric cryptography (like RSA or ED25519). The server holds a "Public Key" (a lock), and the client holds the "Private Key" (the `.pem` file). If the private key file has loose permissions (e.g., readable by other users), the SSH client is hardcoded to block the connection to prevent a security breach.

**Linux Permissions (Octal System):**
File permissions are calculated using a point system: Read = 4, Write = 2, Execute = 1. These points are added together and assigned to three categories: [Owner] [Group] [Others]. 
- `700` means the Owner gets 7 (Read+Write+Execute), and everyone else gets 0.
- `400` means the Owner gets 4 (Read-only), and everyone else gets 0.

### Implementation & Command Breakdown

**Step 1: Securing the SSH Directory and Key**
SSH requires strict permissions to function. So I locked down the hidden .ssh folder and the private key using the following commands:
```bash 
chmod 700 ~/.ssh
chmod 400 ~/.ssh/akn4ssl_key.pem
```
- chmod: "Change mode" - the command to alter file access permissions.

- 700: Grants the owner full access (4+2+1) and strips all access from everyone else.

- 400: Grants the owner read-only access (4) and strips all access from everyone else.

- ~: A shortcut symbol that always points to the current user's home directory.

- /.ssh: The hidden directory containing SSH configuration files (the . makes it hidden).

**Step 3: Connecting to the Azure VM**
With the key secured, we established the connection to the server.
```bash
ssh -i ~/.ssh/akn4ssl_key.pem akn4ssl@20.219.226.123
```
- ssh: Invokes the Secure Shell client program.

- -i: "Identity" flag, telling SSH we are providing a specific private key file rather than a password.

- akn4ssl: The specific username we are trying to log in as on the remote server.

- @: Separates the username from the destination address.

- 20.219.226.123: The public IP address of the Azure VM.

**Step 4:System Updates and Security**
Once logged into the server, we updated the system packages.
```bash
sudo apt update && sudo apt upgrade -y
```
- apt: The package manager for Ubuntu/Debian systems (Advanced Package Tool).

- update: Downloads the latest list of available software versions from the Ubuntu servers (does not install them).

- upgrade: Actually downloads and installs the newer versions of the packages.

- -y: "Yes" - automatically answers "yes" to any installation prompts so the process doesn't pause.

**Step 5: Automating Security Patches**
To ensure the system always receives the latest security updates automatically, we configured unattended upgrades.
```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```
- install unattended-upgrades: Downloads the specific software package that manages automatic background updates.

- dpkg-reconfigure: A tool used to re-run the initial configuration setup of an installed package.

- --priority=low: Ensures that all configuration questions (even the low-priority ones) are displayed to the user via an interactive menu.

End of task 1
## Task 2: Enhanced SSH Security
# Task 2.1 - SSH Configuration
Before proceeding we login to the azure server by the command we did before : ssh -i ~/.ssh/akn4ssl_key.pem akn4ssl@20.219.226.123
**Disabling root login,Disabling password-based authentication and Enabling and configuring the public key authentication.**
Right now to the server we are having two login's one is through the user account, in this case akn4ssl, and another is via 'root'. Hackers and bots would try to login into the server through root and injecting keys through the ssh. so we have to disable the root login into the server by teaching the sshd to disable the root login, password guessing for added security becuase in other case it may affect the servers speed and network bandwidth. Hence we use the following commands:
```bash
sudo nano /etc/ssh/sshd_config
```
And this would open the sshd_config file and in that we change the following 
- PermitRootLogin no => This remove root access.
- PasswordAuthentication no => This permanently disables password guessing.
- PubkeyAuthentication yes => This enforces that only cryptographic keys are allowed.
and the  finally restard the server to apply the changes:
```bash
sudo systemctl restart ssh
```
**Restrict SSH access to specific IP addresses**
##Concept
The azure server is having a public IP address, here in this case (`20.219.226.123`) that is the destination IP. Anyone on the internet needs to know this IP to talk to the server but hackers and other users uses tools like nmap ,masscan etc to scan every single ip address hence in that case our server IP would be found and it further adds on to risk hence in this stage we are locking in the server to look into the Source IP and if it is not in the range specified, do not even give a chance to enter the password and such stuffs. So that the next time when someone knocks into the servers door ssh deamon looks through both the user and source IP to access the server.

- Step 1:Open the Config File Again**
``` bash
    sudo nano /etc/ssh/sshd_config
```
    Find the IP range using the followinf command : "curl ifconfig.me" and thsi returns an ip address in my case it has given 61.1.180.57 so we buil based on 61.1.*.*
- Step 2:Move through the configuration file**
    Move through the configuration file and make : AllowUsers akn4ssl@61.1.*.*
    and then apply "
```bash
    sudo systemctl restart sshd
```
**Set up fail2ban to protect against brute-force attacks.**
We have added security for the server right now any user could try the doing brute force attack on the server so now we implement fail2ban so that if the same IP tries for multiple attacks automaticaly this tool will act to restrict that IP from accessing the server.Hence acting as a Firewall protection for the vm.
- Step 1:Install and enable Fail2ban
```bash 
sudo apt install fail2ban -y
```
- Step 2: Enable the service to start automatically on system boot
``` bash
sudo systemctl enable fail2ban
```
- Step 3: Start the monitoring service
```bash
sudo systemctl start fail2ban
```
# Task 2.2 - Setting up Multi factor authentication
**Objective:Implement Time-Based One-Time Password (TOTP) Multi-Factor Authentication for SSH logins**
Concepts learned: 
 1. Pluggable Authentication Modules (PAM): Linux uses PAM as a modular architecture to handle authentication. Instead of building MFA directly into the SSH program, SSH delegates the secondary authentication check  to a PAM module.
 2. Defense-in-Depth: By requiring both "something you have" (the RSA .pem key) and "something you generate" (the Google Authenticator TOTP code), the server remains secure even if the primary private key is stolen from the local client machine.
**Implementation**
- Step 1 : Install Google Authenticator PAM module
``` bash
sudo apt install libpam-google-authenticator -y
```
- Step 2 : Executed the generator to create the secret QR code and emergency scratch codes
``` bash
google-authenticator
```
- Step 3 : Configuring PAM to Enforce the Module and Resolve Password Conflicts
``` bash
sudo nano /etc/pam.d/sshd
```
 Traverse to the end of file and add the following line : auth required pam_google_authenticator.so This would enforce the google authenticator and as we are allowing the disabled keyboard interactive prompts, we   gotta remove the @include common-auth becuase in other case it would ask the password for the vm which we havent set and would automativally lock us down.

- Step 4 :Modifying the SSH Daemon to Prompt for MFA
Instructed the SSH server to allow interactive keyboard prompts and explicitly require both authentication methods.
``` bash
sudo nano /etc/ssh/sshd_config
```
Modify the file : Allowed interactive prompts: KbdInteractiveAuthentication yes and Enforced both key and interactive auth: AuthenticationMethods publickey,keyboard-interactive

- Step 5 : Apply the changes using sudo sysatemctl restart sshd

Hence we have set up Multi-Factor Authentication for our VM, hence making it more secure.

# Task 3 : Firewall and Network Security
**Setting up Firewall**
- The question why we need a firewall still awaits, the exact reason being that an Ubuntu server exposes all 65,535 virtual network ports to the public internet. If a malicious botnet scans the server's IP address and knocks on a random closed port (lets say Port 8080), the Linux operating system must actively process that request and send back a "TCP RST" (Reset) packet to say the port is closed. Under a heavy automated scan, this active rejection process consumes valuable CPU cycles, memory, and network bandwidth, slowing down the server.

- Hence we setup UFW aka Uncomplicated FireWall: By enabling UFW, we implemented a "Default Deny" network architecture. Instead of the operating system actively rejecting unauthorized traffic, the firewall acts as a silent lobby guard. If a packet arrives for an unapproved port, UFW simply "Drops" it into a black hole. The server sends no response back to the attacker, completely hiding its internal state and preserving computational resources.

- Another major problem being that by default all the automated bots know that port 22 is the standard for ssh connection making it bypass the UFW so we migrate the ssh port from 22 to port 2222. While it does not stop a targeted, sophisticated attack, it immediately filters out the massive volume of automated "background noise" attacks, keeping our logs clean and our CPU usage low.

- Shifting the port required a coordinated change across both the Cloud Hardware layer and the OS Software layer:
Cloud Level : First, an Inbound Port Rule was created in the Azure Portal's Network Security Group to allow TCP traffic on Port 2222 to pass through Microsoft's outer hardware firewall.
OS level : 
- Step 1 : Log in Normally in port 22
``` bash
ssh -i ~/.ssh/akn4ssl_key.pem akn4ssl@bazzlunk.sslnitc.site
```
- Step 2: Update the UFW Lobby Guard
``` bash
sudo ufw allow 2222/tcp
```
- Step 3: update port 22 to port 2222 in the sshd_config file. Open it using the command:
``` bash
sudo nano /etc/ssh/sshd_config
```
- Step 4: Apply changes by using the command:
``` bash
sudo systemctl restart sshd
```
- Step 5 : Test the change by logging in thorugh the new terminal
``` bash
ssh -i ~/.ssh/akn4ssl_key.pem -p 2222 akn4ssl@bazzlunk.sslnitc.site
```
- Step 6: And before ending this we disallow the entry acdcess through port 22; we tell ufw to close any connections made to port 22
```bash
sudo ufw delete allow 22/tcp
```

Done so this has effectively made the port 22 to port 2222 for ssh connection.

Right now we have all th ports closed to the server and ufw is prevemting any access to the server other than port 2222 for ssh connection and that port is heavily protected by MFA, fail2ban and other security services. This has made the system secure but right now suppose we have to host a web service or deploy any AI models inside this server the other users or the external users shoudl be able to connect to it and hence for this we need to open two ports namely port 80 and port 443 as they are universally standard for unencrypted and encrypted web traffic. Hence type in these commands:
``` bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```
Along with this currently we havent allowed logging what happened when someone tries to access the server as the ufw would blindly put into a loop by giving no response and hence for the maintainer and for the user it becomes difficult to debug, morover as we have implemented fail2ban the tool needs to know which ip has accessed the ufw or the ports multiple times. And this is required by the tool to ban that address from hitting the server again. Hence we implement Logging for our server.
```bash
sudo ufw logging on
```
Finally we enable the new settings which we implemented :
```bash
sudo ufw enable
```
# Task 3.2 - Setting up Intrusion Detection System (IDS)
Right now UFW is a lobby guard and looks only on the envelope and suppose if it says port 80 then UFW allows access and suppose that packet is having a malicious SQL Injection then we are still exposed to attacks hence we set up IDS. It performs Deep Packet Inspection (DPI). As packets flow into the server, the IDS actually rips open the envelopes and reads the data payload inside and it does:

- Signature Matching: It compares the contents of every packet against a massive database of known hacker signatures, malware patterns, and suspicious behaviors.

- The Security it Adds: If it sees a packet containing the text string for a known vulnerability (like the infamous Log4j exploit), it instantly triggers a massive alarm, writes it to a security log, and alerts the sysadmin, even if the firewall thought the packet was safe.

Hence we are going to deploy suricata the IDS engine. For that we have to install suricata
``` bash
sudo apt update
sudo apt install suricata -y
```
Lets write some custom rule such that if any client tries to knock the forbidden port 22 then we tell IDS to produce a warning message
``` bash
sudo nano /etc/suricata/rules/local.rules
```
Here we write the custom rule:
alert tcp any any -> any 22 (msg:"[AKN] Hacker Probing Legacy SSH Port 22!"; sid:1000001; rev:1;)

Now we tell the IDS to read the custom rule
``` bash
sudo nano /etc/suricata/suricata.yaml
```
and in the opened yaml file add local.rules if it is missing there under the rule-files.

Restart the engine:
``` bash
sudo systemctl restart suricata
```
This was a test script done to see whether suricata or the IDS engine could filter and protect the server or not but for real applications we cannot fix rules suricata itself is having a massive rule book called Emerging Threat Open rule book we download it and use it. In this way the IDS engine sits there waiting for any package asking the server it reads the data and decides whether to alert the user or not and the ufw sits there parellely blocking disallowed ports. In this way we have reached a great security by even protecting the ports 80 and 443.

# Task 4 - User and Permission Management
**Task 4.1 - Setting up users and their permission.**
We setup Users and give them permissions as to what they have to approach to. Here we create 5 users exam_1 , exam_2, exam_3 , examadmin , examaudit. exam_1,exam_2 ,exam_3 are normal users without any super user previliages meaning that the work done by exam_1 cannot be accessed by the other users and those users cannot chabge the configurations of the vm they are just the end users.They are individual users having their own private folders and have only read only access. Examauditor is the next user and this user has previlages not to change the configuartions of the vm but they have read access meaning that they could read and see what exam_1,exam_2,exam_3 have done but cant update any folders. examadmin is the super user. He have both read and write access to the vm, could update the configurations and supervise everything hence for seetting this up the following steps were used:

- Step 1: Add exam_1,exam_2,exam_3:
``` bash
sudo useradd -m -s /bin/bash exam_1
sudo useradd -m -s /bin/bash exam_2
sudo useradd -m -s /bin/bash exam_2
```
 This would add all three users and give them a home directory and -s /bin/bash would give them a secure terminal and -m makes a home directory.
 
- Step 2: Add examadmin:
``` bash
sudo useradd -m -s /bin/bash examadmin
sudo usermod -aG sudo examadmin
```
 This would append examadmin to the super users group -aG means append Group.
 
 - Step 3: Add examaudit:
 Linux permissions make it very hard to give one specific user read-only access to multiple private folders. So, we use a tool called ACL(Access Control Lists).
``` bash
sudo useradd -m -s /bin/bash examaudit
sudo apt update
sudo apt install acl -y
sudo setfacl -R -m u:examaudit:rx /home/exam_1
sudo setfacl -R -m u:examaudit:rx /home/exam_2
sudo setfacl -R -m u:examaudit:rx /home/exam_3
```
 The sudo setfacl -R -m u:examaudit:rx /home/exam_1 command breaksdown to the following:
 setfacl: Set File Access Control List. Standard Linux permissions (like chmod) are blunt instruments. ACL allows us to use surgical precision to give one specific person a highly customized permission without affecting anyone else.
 -R: Recursive.
 -m : modify the ACL ruleset.
 u:examaudit:rx means the user examaudit is given a power to look into i.e. r - read and x - execute the files or folders in the permitted directories.
Hence this command would give examaudit the power to read and execute files and folders in all exam_1,exam_2 and exam_3 as well.
**Task 4.2 - Directory Access Control and Resource Limitation**
Currently we created Users and now we face two problems One being that exam_1 could read files of exam_2 which means even though exam_1 and exam_2 are not given write access but linux automatically gives read access hence we need to secure the system from this. Hence we apply chmode 700 which means only the user and the admin could now access the files inside that respective user.

``` bash
sudo chmod 700 /home/exam_1
sudo chmod 700 /home/exam_2
sudo chmod 700 /home/exam_3
```

The second problem we face is that , the VM right now faces an issue suppose a hacker tries to enter or lets say the user itself is a hacker then he could destroy the VM in no time by just writing a 3 line of python code to print HACKER into a text file in an infinite loop. This would make the text file of size 30GB eventually and consume the entire Azure hard drive, and the server will violently crash.Hence we implemnt Disk quota, Disk Quotas put a hard limit on the user. If we set a 1GB quota, the moment exam_1's script hits 1GB, the Linux kernel physically cuts off their write access and saves the server.

- Step 1 : Install qouta tracking tool:
``` bash
sudo apt update
sudo apt install quota quotatool -y
```
- Step 2: We need to tell the hardrive to start counting the bytes so we update the /etc/fstab 
 we open the file using sudo nano /etc/fstab and once open we add usrquota into the main hardrive file. So this would instructs the hardive to count the bytes used by an user.
 
- Step 3: Remount the hard drive 
``` bash
sudo mount -o remount /
```
 This would apply the new quota rule without even rebooting the server.

- Step 4: We tell the server to scan the hard drive and build the initial tracker file:
``` bash
sudo quotacheck -cum /
sudo quotaon /
```
so here quotacheck is a utility that runs through every bytes of the hard drive and counts every single byte owned by every single user and build the tracking database (aquota.user).

- Step 5: Now we setup the limits.
``` bash
sudo setquota -u exam_1 500000 550000 0 0 /
sudo setquota -u exam_2 500000 550000 0 0 /
sudo setquota -u exam_3 500000 550000 0 0 /
```
this command sets the limit for hard drive for all users we have fixed the soft limit of 500 MB and a hard limit of 550 MB the momnet a user tries to save afile of more than 550 MB the server stops and prevents the user from further storing the files. The last two 0's implies that we do not keep any limit for the number of individual files meaning we do not count on the bases of number of files rather based on the size the files takes.

Hence this makes an end of task 4.2 
**Task 4.3 - Backup script**
There are several reasons why we write a backup script. So here we implement an automated script using cron that will make our work in a zip file and add it to our server at midnight.
- Step 1 : Setup the folder
As we made chmod 700 for our exam_1 exam_2 and exam_3 we need sudo previlage to copy the files so we use ACL for allowing the execute function.
``` bash
sudo mkdir -p /home/examadmin/backups
sudo chown examadmin:examadmin /home/examadmin/backups
```
- Step 2: Give the ability to read the locked student folders so the script doesn't get blocked for sudo password
``` bash
sudo setfacl -R -m u:examadmin:rx /home/exam_1
sudo setfacl -R -m u:examadmin:rx /home/exam_2
sudo setfacl -R -m u:examadmin:rx /home/exam_3
```
- Step 3: Now we will write the backup script to run at every midnight and we create it in the folder which we just created.
``` bash
sudo nano /home/examadmin/backup_script.sh
```
and we write the bash script:
``` bash
DESTINATION="/home/examadmin/backups/exam_vault_$(date +'%Y-%m-%d').tar.gz"
TARGETS="/home/exam_1 /home/exam_2 /home/exam_3"
tar -czf $DESTINATION $TARGETS
chmod 600 $DESTINATION
```
The following script will convert the updates in the users exam_1 and exam_2 and exam_3 into a compressed file and store it in the /home/examadmin/backups and we have made given only read access to admin.
- Step 4:Giving examadmin the power to read, write and execute the files..
``` bash
sudo chown examadmin:examadmin /home/examadmin/backup_script.sh
sudo chmod 700 /home/examadmin/backup_script.sh
```
- Step 5: Automating with cron
Cron is the timkeeper of linux so we are going to edit examadmin's personal crontab schedule to trigger the script daily.
```bash
sudo crontab -u examadmin -e
```
Now we scroll into the bottom of the crontab and add the following co
```bash
0 0 * * * /home/examadmin/backup_script.sh
```
here 0 0 * * * means Minute 0, Hour 0 [Midnight], Every Day, Every Month, Every Day of the Week.
and the source file which is our compresesed file
hence now cron would run the script where we add the comprosed files into our vm situated in the cloud.


# Task 5 - Reverse Proxy configuration via Nginx
Nginx is one of the most powerful web server software package. So in this stage we use NginX as a reverse proxy. 
There are two types of proxy:

- Forward Proxy : Protects the client.The best case when we are using VPN in that case we are the client accessing the internet or software so in that case if we use forward proxy then the server cannot identify the client tehreby protecting the client.

- Reverse Proxy : Protects the server. Here NginX acts as the reverse proxy so that the underlying software in the server is not known by the client as the client feels like it is talking to NginX and NginX talks to the underlying software and replies. Hence protecting the software and server. 
Also NginX is powerful becuase it is having a big dataset of known attacks we have already enabled suricata or the IDS system but here we include added security. Especially in attacks like Slowloris where the hacker sends 1 byte by byte to freeze the server, couldnt be easily found by IDS but NginX is a pro in finding those. Hence they both work together.

In this stage we include two powerfull apps named app1 in port 8008 and app2 in port 3000 where both ports are closed and locked by the UFW. And we also add a URI path. On setting the URI path we set it in such a way that when a user tries to make an https request via port 443 which is open, we include the URI path along with so lets say here we have /server1, Then on doing https://x.ssl.airno.de/server1/ NginX will do the traffic routing it will route the /server1 via port 8008 to app1 and make the app1 work. Hence the client talks via 443 to the server and NginX communicated or does the trafficing making the connection to port 8008 which is closed to access for a normal client.

**Implementation**
- Creating a non-previlaged user : We create a non-previlaged user so that in case someone hacks the app1 then they are still in powerless account and cannot do anything else in it. So we create a user to run apps lets call it apprunner
``` bash
sudo useradd -m -s /bin/bash apprunner
```
Right now we have made many accounts as per stage 4 so suppose if we do like this and we download files in this terminal then the user apprunner being an unprevilaged one cannot access the file of examadmin, which is our present admin hence we need to move to the apprunner terminal 
```bash
sudo su - apprunner
```
- Download App 1 & Its Security Signature similarly the app2
Now we pull the binary file and its SHA256 signature from the provided URLs.
``` bash
wget https://do.edvinbasil.com/ssl/app -O app1
wget https://do.edvinbasil.com/ssl/app.sha256.sig -O app1.sha256.sig
git clone https://gitlab.com/tellmeY/issslopen.git
```
- Verify SHA256 by comapring the fingerprints : What if when we downloaded the app1 some hacker interupted or suppose we downloaded a hacked version in that case we should not use the faulty app1. So inorder to check that, we use SHA256 hashing, this is a fingerprint; suppose even if a line of code in the app1 is changed then the fingerprint completely changes.
``` bash 
sha256sum app1
cat app1.sha256.sig
```
so right now we get two long text both being the signatures..one is that of app1 and the other is the original signature file we downloaded, if they are same then app1 is good to use.
Now we know that app1 is safe to run.
- Give execution power for app1: Right now, app1 is just a text file. We need to flip the executable bit to tell Linux, "This is a program, you are allowed to run it."
```bash
chmod +x app1
```
- Start the app in background : If we just type ./app1, the app will take over your terminal. The moment we close the terminal, the app will die. We want this to run permanently in the background.
We will use nohup (No Hangup) and the & symbol to push it to the background.
```bash
nohup ./app1 > app1.log 2>&1 &
```
 also we could test app1 is working by using the curl command : curl http://127.0.0.1:8008/ and if the app1 is alive it returns : SSL Onboarding. path: /

- Setting up app2 : Because we pulled this app from GitLab, it is sitting inside its own directory. So we execute the commands to kniw the instruction sfor us to execute:
``` bash
cd ~/issslopen
ls -la
cat README.md
```
The README suggest us to use bun and get going with it. Bun is a super-fast engine for running JavaScript apps. The beauty of Bun is that we can install it locally inside apprunner's home directory without needing any sudo privileges.
- Creating the secret Token : Copy the template file to create your actual .env file:
```bash
cp .env.example .env
```
now we open the .env to edit the token:
```bash
nano .env
```
when the .env opens add the token in the ADMINS= section i have added the token as ADMINS=AKN:Token_ssl
As per the README file app3 conetents are wrritten in javascript or typescript but our azure enegines doesnt know how to execute it so we use a compiler and Node.js were used but the developer has instructed to use bun so we use a very fast compiler named bun which will do the required. So we install bun here.
```bash
curl -fsSL https://bun.sh/install | bash
source ~/.bashrc
bun install
nohup bun run index.ts > app2.log 2>&1 &
```
so in the first commands we install bun in the apprunner user and we add it to the linux path so we could directly type bun install and for that we need to refresh the terminal so we use : source ~/.bashrc and we install further dependencies of bun and we add it to the no hang up (nohup)  program, making the index.ts run even without the clients intervention.
Hence both app1 and app2 is perfectly running now we add the Nginx reverse proxy here.Presently we are in apprunner terminal so we need to go back to the orginal terminal so we use the : exit command.

- Install Nginx and write the configuration file :  Nginx uses the configuration file to naviagte or do traffic routing 
```bash
sudo apt update
sudo apt install nginx -y
```

- Configuation of the configuration file:

sudo nano /etc/nginx/sites-available/default
and remove everything in it and add these: 
```bash
server {
    listen 80;
    server_name bazzlunk.sslnitc.site;
    
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https://images.ctfassets.net;";
    
    location /server1/ {
        proxy_pass http://127.0.0.1:8008;
    }
    
    location /server2/ {
        proxy_pass http://127.0.0.1:8008/;
    }

    location /sslopen {
        proxy_pass http://127.0.0.1:3000;
    }
}
```
Right now we are listeing at HTTP or port 80 but that is not safe as hackers could encrypt the datas easily. so We need to implement it in port 443 or HTTPS means HTTP Secure; so we need SSL certificates for that as well. Also in the configuration we have added the command for Nginx to act as a shield against XSS attacks and this is called Content Security policy.

- Test whether the configuartion is correct : 
```bash
sudo nginx -t
```
if it says synatx is ok and test is successful then do : sudo systemctl restart nginx

- Install certbot :  Presently we need to talk to app1 and app2 and in our nginx which we set up we have added URI paths to them and they could eb accessed via https://bazzlunk.sslnitc.site/server1 and all but that requires an https request and as https is a secure request we need SSL certification for the server to connect this via https. Why we add HTTPS is so that hackers cant change the information being passed.
```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d bazzlunk.sslnitc.site
```
If the network configurations are correct we will be given an ssl certificate for our domain and on trying the following we get the following outputs:
 - https://bazzlunk.sslnitc.site/server1/ return SSL Onboarding path /server1
 - https://bazzlunk.sslnitc.site/server2/ return SSL Onboarding. path: /
 - https://bazzlunk.sslnitc.site/sslopen returns a RED UI saying SSL is Closed
 - https://bazzlunk.sslnitc.site/sslopen/edit/Token_ssl returns a response to change time and once changed a GREEN UI saying SSL is OPEN
 
 Which ensures that our implemetaion is correct and right now we have to apps ready in our server. Now we add security to the databases stored in our server.
 
# Task 6 - Database Security
**Task 6.1 Implementation**
- First we install mariadb as said:
```bash
sudo apt update
sudo apt install mariadb-server -y
```
If an app gets hacked, and that app's database user has "root" powers, the hacker owns the whole server. We are going to create a restricted user that can only read and write data, but cannot delete the actual tables or mess with other databases also we do not want this db connected to the internet. We will force MariaDB to only listen to localhost (127.0.0.1). This means the only way to talk to the database is to already be inside the server.

- Locking down Root : MariaDB comes with a built-in script that automatically fixes the most common security vulnerabilities , including the assignment requirement to Disable remote root login. So we run this scipt:
```bash
sudo mysql_secure_installation
```
so during this step we will , Change the root password , Remove anonymous users, Disallow root login remotely, Remove test database and access to it, Reload privilege tables.

- Create Database & The restricted User
we enter the SQL command line as the ultimate administrator (root)
``` bash
sudo mysql -u root -p
```
and then we type the following sql commands
```bash
CREATE DATABASE secure_onboarding;
CREATE USER 'db_worker'@'localhost' IDENTIFIED BY 'NITC_secure_pass_123';
GRANT SELECT, INSERT, UPDATE, DELETE ON secure_onboarding.* TO 'db_worker'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
So the first command creates the database secure_onboarding and we create a user named db_worker and gave a password credental for him: NITC_Secure_pass_123 and then we assign him as localhost meaning that he could access the database only insisde the server.
And we allow SELECT,INSERT,UPDATE,DELETE commands inside the database and do not provide permission for DROP command. Hence we have set up the database such that the user is having lower previlages to edit in the database.

- Database Security : In order to ensure that MariaDB is only accessible from localhost and not even from a LAN machine, we follow these steps:
```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```
and in this we update the bind-address to 127.0.0.1 which is the address of the local host. and once updated we type in sudo systemctl restart mariadb
- Set up regular automated backups of the database.
So we open a bash file named db_backup.sh
```bash
nano ~/db_bacup.sh
```
and we add the following lines into it :
```bash
#!/bin/bash
mysqldump -u db_worker -pNITC_secure_pass_123 secure_onboarding > /home/akn4ssl/secure_onboarding_backup.sql
```
so here we have used mysqldump it is a tool availabel in mariadb. The use of it is that mysqldump will take the sanshot of the database and dump all the sql queries required to build the database from scratch upto this level.
Now we have to mae the script an executable so that we could run it and backup our database
```bash
chmod +x ~/db_backup.sh
```
And along with it we automate it with cron
```bash
crontab -e
```
and in the file add the line : 0 2 * * * /home/akn4ssl/db_backup.sh

Hence this would at 2:00 AM daily run the script we built which will in turn take a snapshot of the database and we dump those queries required to build the database from scratch. 

# Task 7 - VPN Configuration
As the public IP is known any automated bot can attack or try an ssh enternce in port 22 and all so inorder to secure the system we built a channel for the admin to connect directly into the server. We still need ssh to communicate to the server. But we can lock the ssh to allow only the admins like provide check for those who come through that channel this is what VPN does. VPN essenatially connects the admins system to the servers Network centre basically like azure server would be saved in the virtual network of the admin using it and VPN makes the data send through this cahnnel as really complicated mathematical strings hence making it practically impossible for a hacker to guess and attack on it.
- Step 1 :Add Port 51820 hence allowing UDP Protocol.
```bash
sudo ufw allow 51820/tcp
```
- Step 2: Download and Execute the WireGuard Deployment Script
Configuring kernel routing rules manually is risky and can permanently break the server's internet connection if a typo occurs. We used the industry-standard Angristan deployment script to safely compile WireGuard and calculate the IP Forwarding (NAT) rules automatically.
```bash
curl -O https://raw.githubusercontent.com/angristan/wireguard-install/master/wireguard-install.sh
chmod +x wireguard-install.sh
sudo ./wireguard-install.sh
```
During the installation wizard, we accepted the default settings to ensure the AllowedIPs was set to 0.0.0.0/0, which routes all traffic through the VPN, satisfying the requirement to access both the local network and the public internet.

- Step 3:Creating Credentials for Two Users
The script automatically prompted the creation of the first user (ssms_me). To fulfill the requirement for at least two users, the script was executed a second time.
```bash
sudo ./wireguard-install.sh
```
We selected "Option 1: Add a new client" and created client2. This generated two configuration files containing the cryptographic public/private keypairs: wg0-client-ssms_me.conf and wg0-client-client2.conf.
- Step 4: Patching the Internal IP Trap
Because Azure hides the server's real public IP behind a hardware firewall, the script mistakenly used the server's internal private IP (10.0.0.4) as the VPN Endpoint. If left unchanged, the VPN connection would fail because the client laptop cannot route to a private Azure IP over the open internet.
```bash
sudo nano wg0-client-ssms_me.conf
sudo nano wg0-client-client2.conf
```
In both files, we located the Endpoint variable at the bottom and updated it to our secure public domain:
Endpoint = bazzlunk.sslnitc.site:51820
- Step 5: Extracting the Keys for Local Use
Finally, to connect the local Windows laptop to the Azure server, we output the contents of the configuration files to the terminal:
```bash
cat wg0-client-ssms_me.conf
```
The output text (from [Interface] down to AllowedIPs) was copied, saved locally as a .conf file using Notepad, and imported into the WireGuard desktop client. Upon activation, the local machine was successfully assigned a Virtual IP inside the Azure server's internal network, completing the secure tunnel setup!

# Task 8 -Docker Fundamentals and Personal Website Deployment
**Context & Concepts Learned**
- Containerization vs. Virtual Machines: In previous tasks, we used an entire VM running a full Ubuntu Operating System. Containerization (via Docker) is a lightweight, modern alternative. Instead of running a whole OS, Docker packages an application and its exact dependencies into an isolated "container" that shares the host VM's kernel. This solves "Dependency Hell"—ensuring that code (like a complex AI model or a web app) runs exactly the same way on a local laptop at NIT Calicut as it does in a production cloud environment.
- Docker Volumes: Containers are temporary by design. If a container crashes or is deleted, any data inside it is destroyed. A "Volume" is a secure bridge that mounts a physical folder on the host VM directly into the container, ensuring data persistence and allowing live code updates without needing to rebuild the container itself.
- Detached Daemon: A method of running a process or container persistently in the background so it does not block or depend on the active terminal session.

**Implementation & Command Breakdown**
- Step 1: Basic Docker Setup
First, we installed the Docker engine and configured it to wake up automatically if the Azure VM undergoes a system reboot.
``` bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker
```
- Step 2: Non-Root Access Configuration
By default, Docker is tied directly to the Linux kernel and requires sudo (root) privileges to execute. To safely allow the standard akn4ssl user to run containers without superuser privileges, we added the user to the docker group and refreshed the session.
```bash
sudo usermod -aG docker $USER
newgrp docker
```
We then verified the installation by pulling and running the official test image: docker run hello-world.

- Step 3: Creating the Portfolio Website and Securing the Volume (Bonus)
We created a local directory to act as the persistent volume for our website and drafted the index.html file.
```bash
mkdir -p ~/portfolio
nano ~/portfolio/index.html
```
To achieve the bonus objective for secure access, we explicitly changed the ownership of this folder to UID 101 (the default user ID for the Nginx worker process inside the container) and restricted the write permissions. This ensures the container has the exact access it needs, but hackers cannot easily modify the host files if the container is compromised.
```bash
sudo chown -R 101:101 ~/portfolio
sudo chmod -R 755 ~/portfolio
```
- Step 4: Deploying the Container
We launched the portfolio website using a single Docker command that fulfilled multiple architectural requirements at once:
```bash
docker run -d \
  --name akn_portfolio \
  --restart unless-stopped \
  -p 8080:80 \
  -v /home/akn4ssl/portfolio:/usr/share/nginx/html:ro \
  --cap-add=NET_ADMIN \
  nginx:alpine
```
- Step 5: Nginx Reverse Proxy on the Host VM
Although the container is running securely on internal port 8080, we needed the public to access it via standard HTTP (Port 80) using the VM's public IP address. We configured the host's existing Nginx reverse proxy to route this specific traffic.
```bash
sudo nano /etc/nginx/sites-available/default
```
We appended a new server block to capture traffic hitting the raw IP and route it to the Docker container:
```bash
server {
    listen 80;
    server_name 20.219.226.123;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```
Finally, we tested the configuration syntax and restarted the host Nginx service:
```bash
sudo nginx -t
sudo systemctl restart nginx
```
Result: The portfolio website is successfully containerized, isolated from the host operating system, and globally accessible via the VM's public IP address.

# Task 9 : Ansible Automation in Dockerized lab environment
**1. Architectural Overview**
The goal of this task was to move from manual server management to Infrastructure as Code (IaC). We built a hierarchical environment within our Azure VM:
- The Control Node: A Docker container acting as the "Brain," where Ansible is installed and playbooks are executed.
- The Target Nodes: Two isolated Ubuntu containers (target1, target2) acting as the "Fleet" to be configured.
- The Network: A dedicated Docker bridge network (ansible-net) allowing the containers to communicate via hostnames.
**2. Phase 1: Environment Provisioning & Connectivity**
In this phase, we built the physical lab and established a "Trust Relationship" between the nodes using RSA Keypairs.
IMPLEMENTATION:
- Network and Container Creation:
```bash
docker network create ansible-net
docker run -itd --name control --network ansible-net ubuntu:22.04
docker run -itd --name target1 --network ansible-net ubuntu:22.04
docker run -itd --name target2 --network ansible-net ubuntu:22.04
```
- Target Preparation (Installing SSH & Python):
```bash
docker exec -it target1 bash -c "apt update && apt install openssh-server python3 -y && service ssh start"
docker exec -it target2 bash -c "apt update && apt install openssh-server python3 -y && service ssh start"
```
- SSH Key Exchange:
Generated a key in control and injected the public key into targets for passwordless login.
```bash
docker exec -it control bash -c "ssh-keygen -q -t rsa -N '' -f /root/.ssh/id_rsa"
PUB_KEY=$(docker exec control cat /root/.ssh/id_rsa.pub)
docker exec target1 bash -c "mkdir -p /root/.ssh && echo '$PUB_KEY' > /root/.ssh/authorized_keys"
docker exec target2 bash -c "mkdir -p /root/.ssh && echo '$PUB_KEY' > /root/.ssh/authorized_keys"
```
Now inorder to finish with phase 1 we need to orchastrate all the targets from the control node so we make these files:
- inventory.ini
```bash
[targets]
target1
target2
```
- basic_playbook.yml
```bash
---
- name: Phase 1 Basic Lab Checks
  hosts: targets
  tasks:
    - name: 1. Ping the targets to test connectivity
      ansible.builtin.ping:

    - name: 2. Check disk space (df -h)
      ansible.builtin.command: df -h
      register: disk_space

    - name: Show disk space output
      ansible.builtin.debug:
        msg: "{{ disk_space.stdout_lines }}"

    - name: 3. Show system uptime
      ansible.builtin.command: uptime
      register: sys_uptime

    - name: Display system uptime output
      ansible.builtin.debug:
        msg: "{{ sys_uptime.stdout }}"
```
- Bypass the SSH Prompt and EXECUTE:
Normally, the very first time you SSH into a new server, Linux asks, "Are you sure you want to connect? Type yes/no." Because Ansible is automated, this prompt will freeze the script.
Run this command to tell Ansible to disable host key checking and ignore that prompt:
```bash
export ANSIBLE_HOST_KEY_CHECKING=False
```
- running the orchastrator
```bash
ansible-playbook -i inventory.ini basic_playbook.yml
```
Hence phase 1 is complete we just orchestrated multiple servers simultaneously from a single control node.

**Phase 2: Configuration Management & Security Hardening**
We developed a comprehensive playbook (lab_config.yml) to automate software and security. During this phase, we encountered an OOM (Out of Memory) Crash because the Azure VM hit its RAM limit.
When the system crashed, we used Sequential Execution to protect the RAM:
- lab_config.yml
```bash
---
- name: Phase 2 - Comprehensive Lab Configuration Management
  hosts: targets
  
  # DEFINING VARIABLES: Making the playbook adaptable
  vars:
    lab_packages:
      - python3
      - git
      - vim
      - htop
      - curl
      - wget
      - tmux
      - ufw
    ssh_port: 22

  tasks:
    # 1. SOFTWARE INSTALLATION (Using Variables and Loops)
    - name: Install required lab software packages
      ansible.builtin.apt:
        name: "{{ item }}"
        state: present
        update_cache: yes
      loop: "{{ lab_packages }}"

    # 2. BASH ALIAS CONFIGURATION
    - name: Configure global bash aliases (ll for ls -la)
      ansible.builtin.lineinfile:
        path: /etc/bash.bashrc
        line: "alias ll='ls -la'"
        state: present

    # 3. VIM SETTINGS
    - name: Enable syntax highlighting in vim for all users
      ansible.builtin.lineinfile:
        path: /etc/vim/vimrc
        line: "syntax on"
        state: present

    # 4. SECURITY POLICY: SSH HARDENING
    - name: Disable Root Login via SSH
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PermitRootLogin'
        line: 'PermitRootLogin no'
        state: present
      notify: Restart SSH Service

    # 5. SECURITY POLICY: UFW RULES
    - name: Configure UFW to allow SSH traffic
      community.general.ufw:
        rule: allow
        port: "{{ ssh_port }}"
        proto: tcp

    # 6. SECURITY POLICY: FILE PERMISSIONS
    - name: Ensure strict file permissions on SSH config
      ansible.builtin.file:
        path: /etc/ssh/sshd_config
        mode: '0644'
        owner: root
        group: root

  # HANDLERS: Only runs if the SSH config was actually changed
  handlers:
    - name: Restart SSH Service
      ansible.builtin.service:
        name: ssh
        state: restarted
```
We tried running this file but we ended up in errors like Out Of Memory and especially in python modules.
```bash
ansible-playbook -i inventory.ini lab_config.yml
```
Hence we retry with raw module of ansible
```bash
ansible targets -i inventory.ini -m raw -a "apt-get update && apt-get install python3-apt -y" -f 1
```
The -f 1 prevents Out Of Memory errors.
Now retry running the playbook:
```bash
ansible-playbook -i inventory.ini lab_config.yml -f 1
```
- Remove ufw if errors exist because kernel-level firewalls cannot run inside unprivileged Docker containers

Hence on successfull completion this would result in automating the software installation and security hardening in the targets.

**Phase 3: Modular Infrastructure with Ansible Roles**
To meet industry standards, we refactored our code into Roles. This allows for reusable, modular "building blocks" of infrastructure.
Role Directory Structure:
We used ansible-galaxy init to create the standard file hierarchy:
- roles/lab-base/: System updates, Timezone, and Monitoring.
- roles/student-workstation/: User creation and Dev environments.
The "Container-Safe" Timezone Fix:
Because timedatectl fails in containers, we used a file-linking task in roles/lab-base/tasks/main.yml:
```bash
---
# tasks file for lab-base
- name: Ensure system packages are updated
  ansible.builtin.apt:
    update_cache: yes
    cache_valid_time: 3600

- name: Install tzdata and monitoring tools
  ansible.builtin.apt:
    name:
      - tzdata
      - htop
      - sysstat
    state: present

- name: Set timezone to Asia/Kolkata (Container-friendly way)
  ansible.builtin.file:
    src: /usr/share/zoneinfo/Asia/Kolkata
    dest: /etc/localtime
    state: link
    force: yes
```
- Create the site.yml
```bash
---
- name: Phase 3 - Apply Infrastructure Roles
  hosts: targets
  roles:
    - lab-base
    - student-workstation
```
- Finally we run the site.yml sequentially :
```bash
ansible-playbook -i inventory.ini site.yml -f 1
```
This brings the phase 3 to a conclusion.

Hence we have achieved:
- Infrastructure Scaling: Configured multiple servers in seconds using a single command.
- Resource Management: Mastered the -f (forks) flag to manage low-memory environments.
- Container Security: Implemented SSH hardening and non-root user creation (student_dev).
- DevOps Proficiency: Successfully debugged rc: 137 (OOM Killer) and rc: 1 (Systemd missing) errors.

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Submitted by : Arvind K N
For : SSL Admin Task



