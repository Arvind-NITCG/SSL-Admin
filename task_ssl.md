# SSL Admin 2026 - Task Documentation
**Author:** Arvind K N

##[cite_start]Task 1: Initial Setup [cite: 14]

### Context
[cite_start]To begin the induction tasks, I provisioned an Ubuntu 22.04 LTS Virtual Machine (VM) hosted on Microsoft Azure[cite: 15, 18]. The VM was assigned a public IP address (`20.219.226.123`), and an RSA private key (`akn4ssl_key.pem`) was generated for secure authentication. 

### Concept Learned
**SSH (Secure Shell) & Public Key Cryptography:** SSH is a client-server protocol that provides an encrypted terminal session over an unsecured network. Instead of passwords, it uses asymmetric cryptography (like RSA or ED25519). The server holds a "Public Key" (a lock), and the client holds the "Private Key" (the `.pem` file). If the private key file has loose permissions (e.g., readable by other users), the SSH client is hardcoded to block the connection to prevent a security breach.

**Linux Permissions (Octal System):**
File permissions are calculated using a point system: Read = 4, Write = 2, Execute = 1. These points are added together and assigned to three categories: [Owner] [Group] [Others]. 
- `700` means the Owner gets 7 (Read+Write+Execute), and everyone else gets 0.
- `400` means the Owner gets 4 (Read-only), and everyone else gets 0.

---

### Implementation & Command Breakdown

**Step 1: Securing the SSH Directory and Key**
SSH requires strict permissions to function. So I locked down the hidden .ssh folder and the private key using the following commands:
```bash 
chmod 700 ~/.ssh
chmod 400 ~/.ssh/akn4ssl_key.pem
```
--> chmod: "Change mode" - the command to alter file access permissions.

--> 700: Grants the owner full access (4+2+1) and strips all access from everyone else.

--> 400: Grants the owner read-only access (4) and strips all access from everyone else.

--> ~: A shortcut symbol that always points to the current user's home directory.

--> /.ssh: The hidden directory containing SSH configuration files (the . makes it hidden).

**Step 3: Connecting to the Azure VM**
With the key secured, we established the connection to the server.
```bash
ssh -i ~/.ssh/akn4ssl_key.pem akn4ssl@20.219.226.123
```
--> ssh: Invokes the Secure Shell client program.

--> -i: "Identity" flag, telling SSH we are providing a specific private key file rather than a password.

--> akn4ssl: The specific username we are trying to log in as on the remote server.

--> @: Separates the username from the destination address.

--> 20.219.226.123: The public IP address of the Azure VM.

**Step 4:System Updates and Security**
Once logged into the server, we updated the system packages.
```bash
sudo apt update && sudo apt upgrade -y
```
--> apt: The package manager for Ubuntu/Debian systems (Advanced Package Tool).

--> update: Downloads the latest list of available software versions from the Ubuntu servers (does not install them).

--> upgrade: Actually downloads and installs the newer versions of the packages.

--> -y: "Yes" - automatically answers "yes" to any installation prompts so the process doesn't pause.

**Step 5: Automating Security Patches**
To ensure the system always receives the latest security updates automatically, we configured unattended upgrades.
```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```
--> install unattended-upgrades: Downloads the specific software package that manages automatic background updates.

--> dpkg-reconfigure: A tool used to re-run the initial configuration setup of an installed package.

--> --priority=low: Ensures that all configuration questions (even the low-priority ones) are displayed to the user via an interactive menu.

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
--> PermitRootLogin no => This remove root access.
--> PasswordAuthentication no => This permanently disables password guessing.
--> PubkeyAuthentication yes => This enforces that only cryptographic keys are allowed.
and the  finally restard the server to apply the changes:
```bash
sudo systemctl restart ssh
```
**Restrict SSH access to specific IP addresses**
##Concept
The azure server is having a public IP address, here in this case (`20.219.226.123`) that is the destination IP. Anyone on the internet needs to know this IP to talk to the server but hackers and other users uses tools like nmap ,masscan etc to scan every single ip address hence in that case our server IP would be found and it further adds on to risk hence in this stage we are locking in the server to look into the Source IP and if it is not in the range specified, do not even give a chance to enter the password and such stuffs. So that the next time when someone knocks into the servers door ssh deamon looks through both the user and source IP to access the server.

-->step 1:Open the Config File Again**
``` bash
    sudo nano /etc/ssh/sshd_config
```
    Find the IP range using the followinf command : "curl ifconfig.me" and thsi returns an ip address in my case it has given 61.1.180.57 so we buil based on 61.1.*.*
-->step 2:Move through the configuration file**
    Move through the configuration file and make : AllowUsers akn4ssl@61.1.*.*
    and then apply "
```bash
    sudo systemctl restart sshd
```
**Set up fail2ban to protect against brute-force attacks.**
##Concept
We have added security for the server right now any user could try the doing brute force attack on the server so now we implement fail2ban so that if the same IP tries for multiple attacks automaticaly this tool will act to restrict that IP from accessing the server.Hence acting as a Firewall protection for the vm.
--> Step 1:Install and enable Fail2ban
```bash 
sudo apt install fail2ban -y
```
--> Step 2: Enable the service to start automatically on system boot
``` bash
sudo systemctl enable fail2ban
```
--> Step 3: Start the monitoring service
```bash
sudo systemctl start fail2ban
```
# Task 2.2 - Setting up Multi factor authentication
**Objective:Implement Time-Based One-Time Password (TOTP) Multi-Factor Authentication for SSH logins**
Concepts learned: 
 1. Pluggable Authentication Modules (PAM): Linux uses PAM as a modular architecture to handle authentication. Instead of building MFA directly into the SSH program, SSH delegates the secondary authentication check  to a PAM module.
 2. Defense-in-Depth: By requiring both "something you have" (the RSA .pem key) and "something you generate" (the Google Authenticator TOTP code), the server remains secure even if the primary private key is stolen from the local client machine.
**Implementation**
step 1 : Install Google Authenticator PAM module
``` bash
sudo apt install libpam-google-authenticator -y
```
Step 2 : Executed the generator to create the secret QR code and emergency scratch codes
``` bash
google-authenticator
```
Step 3 : Configuring PAM to Enforce the Module and Resolve Password Conflicts
``` bash
sudo nano /etc/pam.d/sshd
```
Traverse to the end of file and add the following line : auth required pam_google_authenticator.so This would enforce the google authenticator and as we are allowing the disabled keyboard interactive prompts, we gotta remove the @include common-auth becuase in other case it would ask the password for the vm which we havent set and would automativally lock us down.

Step 4 :Modifying the SSH Daemon to Prompt for MFA
Instructed the SSH server to allow interactive keyboard prompts and explicitly require both authentication methods.
``` bash
sudo nano /etc/ssh/sshd_config
```
Modify the file : Allowed interactive prompts: KbdInteractiveAuthentication yes and Enforced both key and interactive auth: AuthenticationMethods publickey,keyboard-interactive

Step 5 : Apply the changes using sudo sysatemctl restart sshd

Hence we have set up Multi-Factor Authentication for our VM, hence making it more secure.




