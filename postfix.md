### User story

There exists legacy application that requires local mail sender to send different notifications for accounts. 
Application itself is able to run without root, but since postfix comes in then e-mails outbound stopped working.
I wasn't able to find information how to run postfix in rootless mode.

I wanted to use podman run with --userns=keep-id, but I started having errors like:
>postdrop: warning: unable to look up public/pickup: No such file or directory
>postdrop: warning: mail_queue_enter: create file maildrop/468819.31: Permission denied

Luckily I figured out that it's possible to run with --userns=keep-id if
1. I change postfix mail_owner to the user I start it under my host
2. I override permissions for the host user
3. restart postfix service at the end.

### Notes
1. each my application has its own user in Linux host
2. I am aware that each RUN line means new layer. Idea is to share main information instead.

#### Dockerfile
```
FROM ubuntu:latest

ARG DEBIAN_FRONTEND=noninteractive

# Set environment variables
ENV CURRDIR=/var/spool/postfix \
    USER=example \
    POSTFIX_SMTP_PORT=1515 \
    FQDN=example.com

RUN apt update && apt upgrade -y && \
echo "postfix postfix/mailname string $FQDN" | debconf-set-selections && \
echo "postfix postfix/main_mailer_type string 'Internet with smarthost'" | debconf-set-selections && \
apt install postfix 

RUN useradd -m -s /bin/bash $USER && postconf -e "mail_owner = $USER" && \
    chown -R $USER: $CURRDIR && \
    mkdir -p $CURRDIR/etc/ssl $CURRDIR/lib/x86_64-linux-gnu && \
    chown root: $CURRDIR/ $CURRDIR/pid $CURRDIR/etc $CURRDIR/lib $CURRDIR/usr $CURRDIR/usr/lib $CURRDIR/usr/lib/sasl2 $CURRDIR/usr/lib/zoneinfo && \
    chown $USER: /var/lib/postfix/ && \
    chown $USER:postdrop $CURRDIR/public $CURRDIR/maildrop && \
    chown root: $CURRDIR/etc/ssl $CURRDIR/lib/x86_64-linux-gnu

RUN postconf -Me "smtp/inet = ${POSTFIX_SMTP_PORT} inet n - n - - smtpd" && \
postconf -e "maillog_file=/var/log/mail.log" && \
...
postconf -e "header_checks = regexp:/etc/postfix/header_checks" && \
echo '/^Subject:/     WARN' > /etc/postfix/header_checks
```

~/.config/systemd/user/example.service
```
...
ExecStartPost=/usr/bin/podman exec --user 0 -it example-container service postfix restart \
        --cidfile=%t/%n.ctr-id
...
```
