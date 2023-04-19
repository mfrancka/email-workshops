## Playground for Kids

Today we will Play arround with e-mails.

---
## Prepare environment:

## Setup DNS

As far as we know e-mails base on DNS so we need to have a domain.
All you vms have their domains let find them.
Let's assigne it like that

```bash
export mydomain="mydomain.com"
```
First of all we need to setup DNS MX record which will have domain name so:
To in case of error it is good idea to set TTL for record to 60 - not recomended for Production

Configuration:

Subdomain: 

mx1 A IP_of_server

MX record
MX mx1.${mydomain} - priority 1



## Install requirements:

For sure we need to have password for our account and sudo to have root access.

As root:
```bash
apt update && apt install git python3-pip -y
pip install ansible
```

Now prepare ansible to run and setup for us dovecot, postfix and mysql

```bash
git clone https://github.com/mfrancka/ansible-role-postfix-dovecot.git $HOME/ansible-role-postfix-dovecot
git clone https://github.com/mfrancka/email-workshops.git $HOME/workshop

ansible-galaxy install geerlingguy.mysql jdauphant.ssl-certs 
```


## Preparations to installation

Ansible will do all the work but it need some small configuration.

```bash
cd $HOME/workshop/ansible
vim setup_dovecot.yml
```
Modify this value using your server domain:
```
 vars:
    mail_domain: ${mydomain} -> This is not ansible syntax just put there your domain in ""
```
```
echo ${mydomain} > /etc/mailname
ansible-playbook -i 'localhost,' -c local setup_dovecot.yml --ask-become-pass
```

Generate crypted password for our user.
Keep also plain version we will need it :)

```bash
sudo doveadm pw -s SHA512-CRYPT
export password="PLAIN_PASSWORD"
```


## Add domain and users

Question?

Which way we do it? :D

* Hard way -> directly in mysql
* Easy way -> using ansible

### Easy way:

Modify setup_dovecot.yml
modify: dovecot_add_example_users to true
add all vmail_virtual_* and your user and domain
example below:
Password need to be crypted with method above

```yaml
      dovecot_add_example_users: true
      [...] - this is to remove  ;)
      vmail_virtual_domains:
        - id: 1
          domain: {{mail_domain}}

      vmail_virtual_users:
        - id: 1
          domain_id: 1
          password: '{SHA512-CRYPT}$6$reUYY3hlFsTtj/Vs$0ebwi0exN43VFOdMCf1eqgTSzROOVLBLgASLZnWZBxnl3ABXE00qdlaF557PJ.Zu0.QF5AyayF9mt0JmgInye1'
          email: 'user@{{mail_domain}}'

      vmail_virtual_aliases:
        - id: 1
          domain_id: 1
          source: 'alias@{{mail_domain}}'
          destination: 'user@{{mail_domain}}'
```

Run ansible again:

```bash
cd $HOME/workshop/ansible
ansible-playbook -i 'localhost,' -c local setup_dovecot.yml --ask-become-pass
```

### Lets test our server

Check if some programs are listening on emails ports. The most important are:
* 25
* 143
* 993
* 587

```bash
ss -ntlp
```

## IMAP

Let's try if we can login to mailbox so if user is created:

```bash
curl imaps://127.0.0.1/INBOX -u user@${mydomain}:${password}  --request 'SELECT INBOX' -v -k
```

Yeah we have account :D

## Let's play with SMTP

We can do all this steps using nc or swaks so first time we will do it 

```
nc 127.0.0.1 25
```
```
EHLO ${mydomain}
```
```
MAIL FROM: devnull@example.com
```
```
RCPT TO: our_user
```
```
DATA
```
```
Date: Thu, 20 Apr 2023 17:15:56 +0200
To: our_recipient 
From: devnull@example.com
Subject: test message
Message-Id: <20230419204256.037704@myhostname>
X-Mailer: nc by hand

```
```
This is a test message send by nc.

.
```
```
QUIT
```

Great you sent your first e-email.

## IMAP again

So now check if there is a message:

```
curl imaps://127.0.0.1/INBOX -u user@${mydomain}:${password}  --request 'SELECT INBOX' -v -k
```
Find all Unseen messages (ids)

```
curl imaps://127.0.0.1/INBOX -u user@${mydomain}:${password}  --request 'SEARCH UNSEEN' -v -k
```

Get only subjects in range of messages from 1 to 4

```
curl imaps://127.0.0.1/INBOX -u user@${mydomain}:${password}  --request 'fetch 1:4 (BODY[HEADER.FIELDS (Subject)])' -v -k
```

Fetch full messsage with ID:

```
curl imaps://127.0.0.1/INBOX -u user@${mydomain}:${password}  --request 'fetch ID RFC822' -v -k
```

Bonus: IMAP commands to delete message:
```
store MESSAGE_ID +FLAGS (\Deleted)
expunge
```

Now send message using swaks:

```
swaks --from user@${mydomain} --to user@${mydomain} -s 127.0.0.1
```

Much easier, isn't it? 

## Play time

You have some time to send email to your colegues using nc ;) 
But fist you need to open Firewall on port ?? ;)


### IDLING

Also we will check something alse using tcpdump:

```
tcpdump -i any port 143 -s 0 -A
```

Still from curl command this part:

```
nc 127.0.0.1 143
A002 AUTHENTICATE PLAIN ......
A003 SELECT INBOX
A004 IDLE
```

Now we are waiting and send message to our account

And in mean time let's check some logs:

when we send message at the end there is information like that:

```
<-  250 2.0.0 Ok: queued as 687A8C2B99
 -> QUIT
<-  221 2.0.0 Bye
```

So we have a queue ID let's check it in logs:

```
grep 687A8C2B99 /var/log/mail.log
```

Now lets check also dovecot logs:
```
grep dovecot /var/log/mail.log
```

This is finishing idiling
``
DONE
A004 FETCH ID_OF_NEW_MESSAGE RFC822
A005 LOGOUT
```

## DKIM

We need to have our private and public key:

```
cd $HOME/workshop
openssl genrsa -out dkim_private.pem 2048
openssl rsa -in dkim_private.pem -pubout -outform der 2>/dev/null | openssl base64 -A

openssl rsa -in dkim_private.pem -pubout -outform der 2>/dev/null | openssl base64 -A >dkim_public.pem
```

Prepare our domains adding DKIM record.
We need to add TXT field in subdomain
selector._domainkey.${mydomain}

I propose TTL 60s

I choose as selector "test" so record need to be:
```
test._domainkey TXT  "v=DKIM1;k=rsa;s=email;p=OUR_PUBLIC_KEY;t=s;"
```

Create message to send
```
swaks --from user@${mydomain} --to user@${mydomain} --dump-mail > mail.eml
```

We need to remove last lines using vim - last two lines need to be removed
```
X-Mailer: swaks v20190914.0 jetmore.org/john/code/swaks/

This is a test mailing

^M
```

So we have:
```
X-Mailer: swaks v20190914.0 jetmore.org/john/code/swaks/

This is a test mailing
```

Now sign message:
```
dkimsign test mydomain.pl dkim_private.pem < mail.eml >signed_email.eml
```

Now send it using swaks

```
swaks --from user@${mydomain} --to user@${mydomain} -d signed_email.eml -s 127.0.0.1
```

Now get message using openssl:
```
openssl s_client -connect 127.0.0.1:993
A002 AUTHENTICATE PLAIN ......
A01 SELECT INBOX
A02 SEARCH UNSEEN
A03 FETCH ID_OF_MESSAGE RFC822
```
Response should looks like:

```
A01 FETCH 315 RFC822
* 315 FETCH (RFC822 {1353}
Return-Path: <user@${mydomain}>
Delivered-To: user@${mydomain}
Received: from mail.dedyk.mydomain.eu
        by mariusz-Virtual-Machine with LMTP
        id MCmvCiU/QGSilAAA2+a9PA
        (envelope-from <user@${mydomain}>)
        for <user@${mydomain}>; Wed, 19 Apr 2023 21:21:09 +0200
Received: from mariusz-Virtual-Machine (localhost [127.0.0.1])
        by mail.dedyk.mydomain.eu (Postfix) with ESMTP id 250329E2
        for <user@${mydomain}>; Wed, 19 Apr 2023 21:21:09 +0200 (CEST)
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/simple; d=mydomain.pl;
 i=@mydomain.pl; q=dns/txt; s=test; t=1681932053; h=date : to : from :
 subject : message-id : from;
 bh=ecGWgWCJeWxJFeM0urOVWP+KOlqqvsQYKOpYUP8nk7I=;
 b=kfUUFVKBEP6Hk2ilZc/xkSzxRNHAob9gLYJrGkystSwPxco+xiYve7ogPE4HM0D4CvbTF
 kbLWdgAgc9SN3Wo8saD2NClaifKKTs6n0wTD6sKub9R558bF8647l2D2mH1CJ4oQ4gbj70G
 2/RWBQFyibYCQXoYvvNme1Xd1u+rcg8kNf648/dXWlwEWecr7a2EKOi1s+TBbSOTUkDsRH3
 5eY9fxrLSujFoZ0ChBFqHOCmo+CkHwgAYMYDOmy59Q02Zfviy2ZHXLuRbqkLI1eXK1xegT3
 qR3YDA7shH9Bq1wbe0YxZ2w+XQOsozfAyVDZr0xFyh2sO7Ab9U7rUW4Q427A==
Date: Wed, 19 Apr 2023 21:18:40 +0200
To: user@${mydomain}
From: user@${mydomain}
Subject: test Wed, 19 Apr 2023 21:18:40 +0200
Message-Id: <20230419211840.038020@mariusz-Virtual-Machine>
X-Mailer: swaks v20190914.0 jetmore.org/john/code/swaks/

This is a test mailing

)
A01 OK Fetch completed (0.003 + 0.000 + 0.002 secs).
```

Copy Message to file without ")" starting after "* 315 FETCH (RFC822 {1353}"
Store it as test.eml

Now let's do dkim signature verification.
To do that our domain need to respond with proper TXT record for DKIM

```
dkimverify < test.eml
signature ok
```

No let's check if it do not lie. Modify test.eml for example "From" header.

```
dkimverify < test.eml
signature verification failed
```
