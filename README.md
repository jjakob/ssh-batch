# ssh-batch

```
usage: /usr/local/bin/ssh-batch [-v] PROMPTGLOBFILE COMMANDFILE SSHARG [SSHARG]...
Execute ssh with SSHARGs and wait for the prompt defined in PROMPTGLOBFILE
before executing commands in COMMANDFILE.

  -v:             verbose (print all ssh interaction)
  PROMPTGLOBFILE: file with glob that matches expected prompt line
                  (only one line with Tcl "string match" pattern)
  COMMANDFILE:    file with list of commands to execute, one command per line
                  (can contain comments and empty lines)
  SSHARG:         one or more ssh arguments (connection string, batch mode ...)
```

## Installing

- download the latest release
- verify its PGP signature
- extract it
- copy the script to /usr/local/bin

## Examples

### Scripting iLO

```
root@host:~# mkdir /etc/ssh-batch

root@host:~# cat > /etc/ssh-batch/192.0.2.1_prompt <<'EOF'
hpiLO-> $
EOF

root@host:~# cat > /etc/ssh-batch/192.0.2.1 <<-'EOF'
	# get fan state (will only show on first connection after iLO reset)
	fan info
	
	# turn off temperature sensor 11 (HD Max)
	fan t 11 off
EOF

root@host:~# cat >> ~/.ssh/config <<'EOF'
	# add your id_rsa.pub to iLO for fanuser (iLO only works with RSA)
	Host 192.0.2.1
	    User fanuser
	    IdentityFile ~/.ssh/id_rsa
	    # the following are for iLO4 v2.77
	    KexAlgorithms +diffie-hellman-group14-sha1
	    HostKeyAlgorithms +ssh-rsa
	    PubkeyAcceptedAlgorithms +ssh-rsa
EOF

root@host:~# ssh-batch -v /etc/ssh-batch/192.0.2.1_prompt /etc/ssh-batch/192.0.2.1 192.0.2.1
spawn ssh 192.0.2.1
The authenticity of host '192.0.2.1 (<no hostip for proxy command>)' can't be established.
RSA key fingerprint is SHA256:redacted.
No matching host key fingerprint found in DNS.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.0.2.1' (RSA) to the list of known hosts.
User:fanuser logged-in to ILO.(192.0.2.1 / FE80::CACB:0259:725F:C312)

iLO Standard 2.77 at  Dec 07 2020
Server Name: host is unnamed
Server Power: On

</>hpiLO->
fan info



</>hpiLO->
fan t 11 off



</>hpiLO-> 

Success, sent 2 commands
```

To automatically run on boot, create a systemd service:
```
root@host:~# cat > /etc/systemd/system/ssh-batch@.service <<-'EOF'
	[Unit]
	Description=Run a batch of commands via SSH
	After=network-online.target
	Wants=network-online.target
	
	[Service]
	Type=oneshot
	RemainAfterExit=true
	# add -v if you want the entire ssh session to be logged in the journal
	ExecStart=ssh-batch "%E/%p/%i_prompt" "%E/%p/%i" -o BatchMode=yes "%i"
	
	[Install]
	WantedBy=multi-user.target
EOF

root@host:~# systemctl daemon-reload
root@host:~# systemctl enable ssh-batch@192.0.2.1.service
Created symlink /etc/systemd/system/multi-user.target.wants/ssh-batch@192.0.2.1.service → /etc/systemd/system/ssh-batch@.service.
root@host:~# systemctl start ssh-batch@192.0.2.1.service

```

## License

All rights reserved. Contact me if you want to use it for commercial purposes.
TODO: add proper license.
