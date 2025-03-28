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

4. Check mail logs for successful delivery:
   ```bash
   sudo tail -f /var/log/mail.log
   ```
   
   Look for messages indicating successful delivery such as:
   ```
   postfix/lmtp[1234]: 5678ABCDEF: to=<testuser@localhost>, relay=none, delay=0.08, delays=0.05/0.01/0.00/0.02, dsn=2.0.0, status=sent (250 2.0.0 <testuser@localhost> deliveredToMailbox)
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

## Detailed Troubleshooting for Bounced Emails

If your test emails are still bouncing back, follow this systematic approach to diagnose and fix the issue:

### 1. Understand the Exact Bounce Reason

First, identify the specific reason for the bounce:

```bash
grep "status=bounced" /var/log/mail.log
```

Look for the DSN (Delivery Status Notification) code and message, such as:
- `dsn=5.1.1, status=bounced (user unknown)`
- `dsn=5.2.0, status=bounced (mailbox unavailable)`
- `dsn=5.3.0, status=bounced (other mail system problem)`

### 2. Check Mail Delivery Agent Settings

Verify that Postfix is properly configured to use Dovecot's LMTP:

```bash
sudo postconf -n | grep transport
```

Ensure that the following settings are correct:
```
mailbox_transport = lmtp:unix:private/dovecot-lmtp
```

If missing or incorrect, set it:
```bash
sudo postconf -e "mailbox_transport = lmtp:unix:private/dovecot-lmtp"
sudo systemctl restart postfix
```

### 3. Verify LMTP Socket Existence and Permissions

Check if the LMTP socket exists and has correct permissions:

```bash
ls -la /var/spool/postfix/private/dovecot-lmtp
```

You should see something like:
```
srw-rw---- 1 postfix postfix 0 Mar 28 12:34 /var/spool/postfix/private/dovecot-lmtp
```

If missing, verify Dovecot LMTP configuration:
```bash
sudo cat /etc/dovecot/conf.d/10-master.conf | grep -A 10 "service lmtp"
```

### 4. Check Local User and Mailbox Configuration

Verify the user exists and test mail delivery directly:

```bash
# Check if user exists
id testuser

# Test delivery with Dovecot's deliver command
sudo -u testuser doveadm deliver -d testuser@localhost
```

### 5. Debug Dovecot LMTP

Enable verbose logging for Dovecot LMTP:

```bash
sudo nano /etc/dovecot/conf.d/10-logging.conf
```

Add or modify these lines:
```
mail_debug = yes
verbose_ssl = yes
auth_verbose = yes
auth_debug = yes
```

Restart Dovecot and check logs:
```bash
sudo systemctl restart dovecot
sudo tail -f /var/log/mail.log
```

### 6. Test Dovecot LMTP Connection Directly

Test the LMTP service directly from command line:

```bash
telnet localhost 24
```

If not using TCP, you can test with netcat:
```bash
echo -e "LHLO localhost\nQUIT\n" | nc -U /var/spool/postfix/private/dovecot-lmtp
```

### 7. Check Local Mail Aliases

Verify that there are no conflicting aliases:

```bash
sudo postconf -n | grep alias
sudo cat /etc/aliases
```

### 8. Verify Mail Directory Structure

Create Maildir structure if missing:

```bash
sudo -u testuser mkdir -p /home/testuser/Maildir/{new,cur,tmp}
sudo -u testuser chmod -R 700 /home/testuser/Maildir
sudo -u testuser touch /home/testuser/Maildir/subscriptions
```

### 9. Test Local Mail Command Directly

Try sending mail directly with mailx:

```bash
echo "Test message body" | mailx -s "Test subject" testuser@localhost
```

### 10. Reset and Create a Clean User for Testing

Sometimes, creating a fresh user can help isolate issues:

```bash
sudo adduser testuser2
sudo -u testuser2 mkdir -p /home/testuser2/Maildir/{new,cur,tmp}
sudo -u testuser2 chmod -R 700 /home/testuser2/Maildir
echo "Test message for new user" | mailx -s "Test for clean user" testuser2@localhost
```

After making any changes, always restart the relevant services:

```bash
sudo systemctl restart postfix dovecot
```

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
