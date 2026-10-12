#SSH Hardening Guide
# /etc/ssh/sshd_config — Hardened Configuration

#11-10-2026 | K M Nurazzaman
#======================================================================
#Before editing the SSH configuration, create a backup:
#cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup

#After each change, test the configuration before restarting SSH:
#sudo sshd -t

# RHEL/CentOS/Rocky [Restart the SSH service]
#sudo systemctl restart sshd

#sudo systemctl status sshd #Verify the service is running

#Set public key  authentication (ssh-keygen -t ed25519 -C "your_email@example.com")
#=======================================================================


# ----- Network -----
Port 22    #Not good security to change port number
AddressFamily inet
Protocol 2    

# ----- Authentication ----- 

PermitRootLogin no    #Disable root SSH login
PasswordAuthentication no    #Force key-based authentication
PubkeyAuthentication yes
ChallengeResponseAuthentication no
UsePAM yes
MaxAuthTries 3    #Limit failed login attempts
MaxStartups 10:30:60
LoginGraceTime 60    #Shorten the authentication window

# ----- Access Control -----
#AllowUsers alice bob carol    #Restrict SSH access to listed users
# AllowGroups admins

# ----- Session -----
ClientAliveInterval 300
ClientAliveCountMax 2    #Disconnect after 2 missed keep-alives

# ----- Logging -----
#LogLevel VERBOSE

# ----- Misc -----
X11Forwarding no    #Disable X11 forwarding
AllowAgentForwarding no    #Disable agent forwarding
AllowTcpForwarding no    #Disable SSH tunnels for users who do not need them
PermitEmptyPasswords no    #Block empty passwords

#KbdInteractiveAuthentication	no	#Disable keyboard-interactive password prompts
