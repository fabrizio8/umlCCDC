# SSH Demo

## Objective

Install a secure OpenSSH server on CentOS

## End Product

Run a secure OpenSSH server on CentOS and demonstrate a successful login from host to VM with a key.

## Security Analysis

- Deny all users at deployment, and then add users to be white listed as needed
- Use only ecdsa for the host key
- Change port to a non-standard port
- Disable password logins and force public key authentication
- Disable root and rhost login
- Disable X11 forwarding
- Prevent brute forcing by reducing max logins to 3

## Testing

Scripted: 
- Run `sudo sshd -t` to verify sshd_config is valid
- Use ssh-audit to test sshd_config security
- Run `sudo systemctl status sshd` to verify sshd is running

Active:
- Attempt and fail to login as root (disabled)
- Attempt and fail to login as a user (no key)
- Generate a key pair, add it to the authorized_folder, and demonstrate a login
- Attempt and fail to login as user with key (not whitelisted)
- Whitelist user with key by adding to `AllowUsers`
- Verify successful login as user with key


## Documentation
SSH should be installed by default, if not run the following command:
```sudo yum –y install openssh-server openssh-clients```

To start the SSH daemon on the OpenSSH server:

```sudo systemctl start sshd```
When active, sshd continuously listens for client connections from any of the client tools. When a connection request occurs, sshd sets up the correct connection.

Check the status of the SSH daemon:

```sudo systemctl status sshd```
As we have previously started the service, the output confirms that it is active.


# Edit the configuration using the /etc/ssh/sshd_config file:
The settings seem to be consistent with the Arch Linux set up

Run ```sudo vim /etc/ssh/sshd_config```

```#Disable root logins and uncomment the default port number:
Port 22
PermitRootLogin no

#Disable protocol 1
Protocol 2 
#Disable password authentication forcing use of keys
PasswordAuthentication no

#Default deny all users until there is at least one for AllowUsers,
#then remove DenyUsers and just add to AllowUsers

#AllowUsers
DenyUsers * 
```

Then restart the sshd service and run 
```sudo systemctl status sshd```
to verify it is running

In the future, we may want to change the SSH port to run on a non-standard port. Reason: "For a hacker to determine ssh is running on your machine, he'll most likely scan port 22 to determine this". This will require making any necessary changes to port forwarding in the router and any applicable firewall rules.

TODO: Test this script on CentOS, was having issues with Yum where it would fail to install anything.



```bash
#!/bin/bash

touch sshd_config

sudo cat <<EOF > sshd_config
#Do not allow client to choose 1, so we can use keys
Protocol 2

#Default deny all users until there is at least one for AllowUsers,
#   then remove DenyUsers and just add to AllowUsers
#AllowUsers
DenyUsers *

#Set port number
Port 22

#Put keys for allowed users here
AuthorizedKeysFile	./authorized_keys

#Use this host file, eliptic curve for best affect
HostKey /etc/ssh/ssh_host_ecdsa_key

#Disable password login and only enable public keys
PasswordAuthentication no
ChallengeResponseAuthentication no
PubkeyAuthentication yes

#Disabling other insecure features
PermitRootLogin no
X11Forwarding no
IgnoreRhosts yes

#stricter retry
MaxAuthTries 3

#Restricting us to secure algorithms (entered new lines for readability-- not available in actual sshd_config)
KexAlgorithms
  curve25519-sha256,
  diffie-hellman-group-exchange-sha256,
  diffie-hellman-group-exchange-sha256,
  diffie-hellman-group16-sha512,
  diffie-hellman-group18-sha512,
  diffie-hellman-group14-sha256       

EOF

#Replacing sshd_config
sudo rm /etc/ssh/sshd_config
sudo cp sshd_config /etc/ssh/
sudo rm sshd_config

#Verifying the config is operable
sudo sshd -t

#Downloading and running the audit tool
git clone https://github.com/arthepsy/ssh-audit
python ssh-audit/ssh-audit.py --verbose --no-colors localhost > report.txt
rm -rf ssh-audit
cat report.txt | less

sudo systemctl restart sshd
sudo systemctl status sshd
```

## ToDo

- Research adjusting firewall settings to restrict ip addresses
- [ ] Research and document curve25519 (Enabled for now) 
- [ ] Research and improve ecdsa-sha2-nistp256 appearance in hostkeys-- Do we need to reroll keys and not use the system generated ones? (Reccs from ssh_audit)
- [ ] Add more secure hostkey options for greatest secure compatability (Reccs from ssh_audit)
- [ ] Remove insecure MAC algorithms (Reccs from ssh_audit)
- [ ] Develop a script to automatically roll public keys for users, edit the `AllowUsers` config options, and place their public key in `./AuthorizedKeys`

## References
- Securing SHH on CentOS https://wiki.centos.org/HowTos/Network/SecuringSSH
- [SSH audit tool](https://github.com/arthepsy/ssh-audit)
- [Extra authentican type information](https://blog.tankywoo.com/linux/2013/09/14/ssh-passwordauthentication-vs-challengeresponseauthentication.html)
- [sshd_config man](https://linux.die.net/man/5/sshd_config)
- [sshd man](http://man.openbsd.org/cgi-bin/man.cgi/OpenBSD-current/man8/sshd.8?query=sshd%26sec=8)
- [Security recommendations](https://www.putorius.net/how-to-secure-ssh-daemon.html)
- [IRTF ID on SSH KEX Recommendations](https://tools.ietf.org/id/draft-ietf-curdle-ssh-kex-sha2-10.html) 
