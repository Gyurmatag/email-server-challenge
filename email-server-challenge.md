# Email Server Setup Challenge

## Overview

This 30-minute challenge will test your ability to set up a functional email server using Postfix and Dovecot on a virtual Linux machine. You'll configure the core components of an email system and verify its functionality through various testing methods.

## Prerequisites

- Virtual machine with a fresh Ubuntu Server 22.04 LTS installation
- Basic understanding of Linux command line
- sudo/root access on the virtual machine
- Basic networking knowledge
- Familiarity with email server concepts from the presentation

## Challenge Objectives

1. Install and configure Postfix as an MTA
2. Set up Dovecot for IMAP access
3. Configure secure authentication
4. Create test email accounts
5. Verify email delivery
6. Test IMAP connectivity
7. Troubleshoot any issues that arise

## Detailed Instructions

### Phase 1: Installation (5 minutes)

1. Update your package repositories:
   ```bash
   sudo apt update
   sudo apt upgrade -y
   ```

2. Install the required packages:
   ```bash
   sudo apt install -y postfix dovecot-imapd dovecot-lmtpd mailutils
   ```
   
3. During Postfix installation, select "Internet Site" and enter your FQDN (e.g., `mail.example.com`) when prompted.

### Phase 2: Postfix Configuration (10 minutes)

1. Edit the main Postfix configuration file:
   ```bash
   sudo nano /etc/postfix/main.cf
   ```

2. Update or add the following parameters:
   ```
   # Basic Settings
   myhostname = mail.example.com
   mydomain = example.com
   myorigin = $mydomain
   
   # Network Settings
   inet_interfaces = all
   inet_protocols = ipv4
   
   # Recipient Settings
   mydestination = $myhostname, localhost.$mydomain, $mydomain
   
   # Mail Delivery Settings
   mailbox_transport = lmtp:unix:private/dovecot-lmtp
   
   # TLS Settings
   smtpd_tls_cert_file = /etc/ssl/certs/ssl-cert-snakeoil.pem
   smtpd_tls_key_file = /etc/ssl/private/ssl-cert-snakeoil.key
   smtpd_tls_security_level = may
   smtpd_sasl_auth_enable = yes
   smtpd_sasl_type = dovecot
   smtpd_sasl_path = private/auth
   ```

3. Edit the master.cf file to configure submission service:
   ```bash
   sudo nano /etc/postfix/master.cf
   ```

4. Uncomment the submission service section and add parameters:
   ```
   submission inet n       -       y       -       -       smtpd
     -o syslog_name=postfix/submission
     -o smtpd_tls_security_level=encrypt
     -o smtpd_sasl_auth_enable=yes
     -o smtpd_client_restrictions=permit_sasl_authenticated,reject
   ```

5. Restart Postfix:
   ```bash
   sudo systemctl restart postfix
   ```

### Phase 3: Dovecot Configuration (10 minutes)

1. Edit the main Dovecot configuration file:
   ```bash
   sudo nano /etc/dovecot/dovecot.conf
   ```

2. Ensure the following line is there or add it:
   ```
   !include conf.d/*.conf
   ```

3. Configure authentication:
   ```bash
   sudo nano /etc/dovecot/conf.d/10-auth.conf
   ```

4. Update the authentication settings:
   ```
   disable_plaintext_auth = no
   auth_mechanisms = plain login
   ```

5. Configure mail location:
   ```bash
   sudo nano /etc/dovecot/conf.d/10-mail.conf
   ```

6. Set the mail location:
   ```
   mail_location = maildir:~/Maildir
   ```

7. Configure service settings for LMTP and authentication:
   ```bash
   sudo nano /etc/dovecot/conf.d/10-master.conf
   ```

8. Update the service settings:
   ```
   service lmtp {
     unix_listener /var/spool/postfix/private/dovecot-lmtp {
       mode = 0600
       user = postfix
       group = postfix
     }
   }

   service auth {
     unix_listener /var/spool/postfix/private/auth {
       mode = 0666
       user = postfix
       group = postfix
     }
   }
   ```

9. Restart Dovecot:
   ```bash
   sudo systemctl restart dovecot
   ```

### Phase 4: Testing (5 minutes)

1. Create a test user:
   ```bash
   sudo adduser testuser
   ```

2. Check if Postfix and Dovecot services are running:
   ```bash
   sudo systemctl status postfix
   sudo systemctl status dovecot
   ```

3. Test mail delivery using the mail command:
   ```bash
   echo "This is a test email" | mail -s "Test Email" testuser@localhost
   ```

4. Check mail logs for delivery status:
   ```bash
   sudo tail -f /var/log/mail.log
   ```

5. Verify that the mail was delivered:
   ```bash
   sudo ls -la /home/testuser/Maildir/new/
   ```

6. Test IMAP connectivity using telnet:
   ```bash
   telnet localhost 143
   ```

7. At the telnet prompt, test authentication:
   ```
   a LOGIN testuser yourpassword
   a SELECT INBOX
   a LOGOUT
   ```

## Extra Challenges (if time permits)

1. Configure a web-based email client like Roundcube
2. Set up PostfixAdmin for virtual domain management
3. Configure SPF record for your domain
4. Implement DKIM signing for outgoing emails
5. Set up a simple anti-spam solution

## Troubleshooting Guide

### Common Issues:

#### Postfix Not Starting
- Check configuration syntax: `sudo postfix check`
- Verify log files: `sudo tail -f /var/log/mail.log`
- Check file permissions for main.cf and master.cf

#### Dovecot Not Starting
- Verify configuration syntax: `sudo dovecot -n`
- Check log files: `sudo tail -f /var/log/mail.log`
- Examine process list: `ps aux | grep dovecot`
- Try starting manually with debug: `sudo dovecot -F`

#### Mail Delivery Failures
- Check mail logs: `sudo tail -f /var/log/mail.log`
- Verify mail queue: `sudo mailq`
- Test mail flow with telnet:
  ```
  telnet localhost 25
  EHLO localhost
  MAIL FROM:<testuser@localhost>
  RCPT TO:<testuser@localhost>
  DATA
  Subject: Test Mail
  
  This is a test.
  .
  QUIT
  ```

#### Authentication Issues
- Verify user exists: `getent passwd testuser`
- Check Dovecot auth settings: `grep -r auth_mechanisms /etc/dovecot/`
- Test authentication mechanisms: `doveadm auth test testuser yourpassword`

## Evaluation Criteria

The challenge is successfully completed when:

1. Postfix is properly configured and accepting mail
2. Dovecot is running and providing IMAP access
3. Mail can be sent between local users
4. You can authenticate to the IMAP server
5. The mail delivery chain (Postfix -> LMTP -> Dovecot) is working

## Learning Outcomes

After completing this challenge, you will:

1. Understand the core components of an email server
2. Be able to configure Postfix as an MTA
3. Know how to set up Dovecot for IMAP access
4. Understand the mail flow from sending to delivery
5. Have basic troubleshooting skills for email servers
6. Be familiar with testing methods for email server components
