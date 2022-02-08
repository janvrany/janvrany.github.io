---
title: "msmtpd: how to setup super-light relay-only MTA"
created_at: 2022-02-08 20:41:01 +0000
kind: article
published: true
---

This post describes how to setup `msmtpd` as local relay-only
SMTP server on Debian.

<!-- more -->

1. Install [msmtpd][1]

       sudo apt install msmtp-mta bsd-mailx


2. Since we'd like to use socket-activated service (to save resources),
   completely disable `msmtpd.service`:

       sudo systemctl mask msmtpd.service

3. Create socket unit for `msmtpd`. Run...

       sudo systemctl edit --force --full msmtpd.socket

   ...and when editor opens, type in:

       [Socket]
       ListenStream=25
       Accept=yes

       [Install]
       WantedBy=sockets.target

4. Create service unit for `msmtpd`. Run...

       sudo systemctl edit --force --full msmtpd@.service


   ...and when editor opens, type in:


       [Unit]
       Description=msmtp daemon
       Documentation=man:msmtpd(1)

       [Service]
       User=msmtp
       ProtectHome=true
       PrivateTmp=true
       NoNewPrivileges=true
       SupplementaryGroups=msmtp

       StandardInput=socket
       ExecStart=/usr/sbin/msmtpd --inet '--command=/usr/bin/msmtp -C /etc/msmtprc.relay -f %%F'


5. Now it's time to configure relay SMTP server. Run...

       sudo touch /etc/msmtprc.relay
       sudo chmod go-rwx /etc/msmtprc.relay
       sudo chown msmtp /etc/msmtprc.relay
       sudo nano /etc/msmtprc.relay


   ...and when editor opens, type in:


       tls on
       tls_starttls on
       host smtp.gmail.com
       port 587
       tls_starttls on
       auth on
       user joe@gmail.com
       password joessecret
       from joe@gmail.com
       aliases /etc/aliases

   <p style="color: red; font-size: 200%">BIG FAT WARNING:</p>

   This leaves passeword in clear text in file `/etc/msmtprc.relay` - this is *insecure*. The file is readable only by user `msmtp` but still. Unfortunately, I could not find another, more secure way to pass down password.

6. Disable apparmor for `msmtp`:

       sudo ln -s /etc/apparmor.d/usr.bin.msmtp /etc/apparmor.d/disable
       sudo apparmor_parser -R /etc/apparmor.d/usr.bin.msmtp


   Sadly, I could not make it working otherwise.

7. Enable and start `msmtpd` socket unit:

       sudo systemctl enable msmtpd.socket
       sudo systemctl start msmtpd.socket


8. Now configure local mail:

       sudo touch /etc/msmtprc
       sudo chmod ugo+r /etc/msmtprc
       sudo nano /etc/msmtprc


   ...and when editor opens, type in:


       account default
       host localhost
       port 25

   This causes local mail (sent by cron and other tools) to use just
   configured local SMTP relay. Other machines in local network
   (virtual or not) may be configured the same way.

[1]: https://marlam.de/msmtp/