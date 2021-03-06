<!--
{
    "title": "postfix&dovecot邮件服务相关",
    "create": "2018-05-16 14:30:06",
    "modify": "2018-12-02 15:46:43",
    "tag": [
        "postfix",
        "dovecot",
        "email"
    ],
    "info": [
        "测试不完全成功//todo"
    ]
}
-->

## 修改主机名

```bash
hostnamectl set-hostname <mail.example.com>
#或者
hostnamectl set-hostname <example.com>
```

## 添加hosts记录

```txt
xxx.xx.xxx.xx <mail.example.com>
#或者
xxx.xx.xxx.xx <example.com>
```

## DNS设置

添加MX记录指向mail.example.com，添加A，A-mail记录指向服务器IP，添加TXT记录做SPF反垃圾邮件设置

```txt
类型            主机记录            记录值                优先级             TTL
MX               @          <mail.example.com>            10               600
A              mail            xxx.xx.xxx.xx              --               600
A                @             xxx.xx.xxx.xx              --               600
TXT              @        v=spf1 ipv4:xxx.xx.xxx.xx       --               600
#或者
类型            主机记录            记录值                优先级             TTL
MX               @               <example.com>            10               600
A                @               xxx.xx.xxx.xx            --               600
TXT              @        v=spf1 ipv4:xxx.xx.xxx.xx       --               600
```

## Postfix设置

配置文件`/etc/postfix/main.cf`

```conf
myhostname = <example.com> #邮局系统的主机名，或是<mail.example.com>据主机名而定
mydomain = <example.com> #邮局系统的域名
myorigin = $mydomain #从本机发出邮件的域名名称
inet_interfaces = all #设置网络接口以便Postfix能接收到邮件
inet_protocols = all #网络协议
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain #指定哪些邮件地址允许在本地发送邮件
mynetworks = 0.0.0.0/0 #指定受信任SMTP的列表，受信的SMTP客户端允许通过Postfix传递邮件
home_mailbox = mail/ #
#设置邮箱路径与用户目录有关，也可以指定要使用的邮箱风格。带/为maildir风格，不带为mailbox风格
#maildir为邮件分文件存放，mailbox为邮件存一个文件由特定标识分割
smtpd_banner = $myhostname ESMTP unknow #不显示SMTP服务器的相关信息
smtpd_sasl_auth_enable = yes #开启sasl用户认证模式
smtpd_sasl_type = dovecot #设置smtpd使用什么样的sasl方式进行验证，默认是cyrus
smtpd_sasl_path = private/auth #unix域协议所用文件位置
smtpd_sasl_local_domain = #如果采用SASL进行认证，那么这里不做设置，默认为空
smtpd_sasl_security_options = noanonymous #不允许匿名登录
broken_sasl_auth_clients = yes #表示是否兼容非标准的SMTP认证，开启后会出现250-AUTH=PLAIN LOGIN
smtpd_relay_restrictions=permit_mynetworks,permit_sasl_authenticated,reject_unauth_destination
#允许通过验证的用户使用转发服务，以前叫做smtpd_recipient_restrictions
#smtpd_recipient_restrictions = permit_sasl_authenticated,permit_mynetworks,reject_unauth_destination #
#表示通过收件人地址对客户端发来的邮件进行过滤
#permit_mynetworks：当客户端建立连接时，若它来自mynetworks或mynetworks_style定义的网络，返回OK
#permit_sasl_authenticated：表示允许转发通过SASL认证，如果用户已经通过sasl验证登录的用户，同样会返回OK
#reject_unauth_destination：如果邮件的收件人不在postfix所管辖的网域（由mydestination定义，包括虚拟网域），返回REJECT
smtp_tls_security_level = may #设置服务器发送邮件的TLS安全等级，三个参数none、may和encrypt
smtpd_tls_security_level = may #设置服务器接收邮件的TLS安全等级，三个参数none、may和encrypt
#may在SMTPD接收端表示向客户端通告服务器支持加密但允许客户端使用不加密方式传输邮件，在SMTP发送端表示如果服务器通告支持加密即使用加密方式发送邮件
smtp_tls_note_starttls_offer = yes
smtpd_tls_loglevel = 1
smtpd_tls_key_file = /etc/postfix/ssl/server.key
smtpd_tls_cert_file = /etc/postfix/ssl/server.crt
smtpd_tls_received_header = yes
smtpd_tls_session_cache_timeout = 3600s
tls_random_source = dev:/dev/urandom
```

配置文件`/etc/poostfix/master.cf`

```conf
# service type  private unpriv  chroot  wakeup  maxproc command + args
#               (yes)   (yes)   (yes)   (never) (100)
# ==========================================================================
smtp      inet  n       -       n       -       -       smtpd

submission     inet  n       -       n       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
smtps     inet  n       -       n       -       -       smtpd
  -o syslog_name=postfix/smtps
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
```

## Dovecot设置

配置文件`/etc/dovecot/conf.d/10-master.conf`

```conf
# Postfix smtp-auth
unix_listener /var/spool/postfix/private/auth {
mode = 0660
user = postfix
group = postfix
}
```

配置文件`/etc/dovecot/conf.d/10-auth.conf`

```conf
# Space separated list of wanted authentication mechanisms:
#   plain login digest-md5 cram-md5 ntlm rpa apop anonymous gssapi otp skey
#   gss-spnego
# NOTE: See also disable_plaintext_auth setting.
auth_mechanisms = plain login
```

配置文件`/etc/dovecot/conf.d/10-mail.conf`

```conf
#mail_location =
mail_location = maildir:~/mail
```
