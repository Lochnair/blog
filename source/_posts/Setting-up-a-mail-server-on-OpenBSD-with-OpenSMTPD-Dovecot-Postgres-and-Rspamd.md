---
title: >-
  Setting up a mail server on OpenBSD with OpenSMTPD, Dovecot, Postgres and
  Rspamd
date: 2019-11-08 23:01:22
tags:
---

## Introduction

Most people use one of the big free e-mail providers (Gmail, Hotmail, etc.) for their personal e-mail and find them perfectly adequate. 

However there's no such thing as a free lunch, and so most of them will use the data they have on you for targeted advertising. Alternatively one can use a paid e-mail service. There's quite a lot out there to choose from.

Personally I've opted for FastMail the past few years. They've served me well, and I've no real complaints about them.

That said, I've had a wish for quite some time to self-host my e-mail, for educational purposes if nothing else.

My MTA (mail transfer agent) of choice is OpenSMTPD, Dovecot for MDA (mail delivery agent), PostgreSQL for the database containing virtual users. We'll also set up Rspamd to filter out as much spam as possible. I'll assume Debian here. Please note that at the time of writing there wasn't any OpenSMTPd 6.6.x packages for Debian buster, so I've built my own based on the unstable source.

To be sure we won't lose any e-mails if our server go down, we'll set up a backup MX in another location.

I'm spinning up a cloud server for each service, but if you wanna run it all on one host that's fine too.

## Setting up primary MX

### Installing OpenSMTPD and OpenSMTPD-extras

```
wget https://build.lochnair.net/job/debian/job/opensmtpd-buster-backports/lastSuccessfulBuild/artifact/opensmtpd_6.6.1p1-1_amd64.deb
wget https://build.lochnair.net/job/debian/job/opensmtpd-extras-buster-backports/lastSuccessfulBuild/artifact/opensmtpd-extras_6.6.0-1_amd64.deb
wget https://build.lochnair.net/job/debian/job/filter-rspamd/lastSuccessfulBuild/artifact/filter-rspamd_0.1.4+git20191116.4c393bb-1_amd64.deb
wget https://build.lochnair.net/job/debian/job/filter-senderscore/lastSuccessfulBuild/artifact/filter-senderscore_0.1.0+git20191111.9531272-1_amd64.deb
sudo dpkg -i opensmtpd_6.6.1p1-1_amd64.deb opensmtpd-extras_6.6.0-1_amd64.deb filter-rspamd_0.1.4+git20191116.4c393bb-1_amd64.deb filter-senderscore_0.1.0+git20191111.9531272-1_amd64.deb
sudo apt install -f
```

### Setting up PostgreSQL 12

Follow the instructions on their [download page](https://www.postgresql.org/download/linux/debian/) to install the server.

Open /etc/postgresql/12/main/postgresql.conf and set it to listen on all addresses.

```
listen_addresses = '*'
```

If you'd like you can also change the password hashing from MD5 to SHA256. Please keep in mind that older clients doesn't support SCRAM, so if that matters to you keep the default.

```
password_encryption = scram-sha-256
```

Start it with

```bash
sudo systemctl start postgresql@12-main
```

Change to the postgres user and create the users for mail and replication to the backup MX

```bash
sudo -u postgres -i
psql
```

```sql
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'secret';
CREATE USER mailuser WITH PASSWORD 'secret';
```

Create the database to hold domain/user information and grant the `mailuser` user access to it.

```sql
CREATE DATABASE maildb;
GRANT ALL PRIVILEGES ON DATABASE "maildb" to mailuser;
```

Create the tables and insert some data into them

```sql
create table aliases
(
    destination varchar not null,
    alias varchar not null,
    active boolean default true not null,
    constraint aliases_pk
        primary key (destination, alias)
);



create table domains
(
    domain varchar(255) not null
        constraint domains_pk
            primary key,
    active boolean default true not null
);

create table users
(
    username varchar(128) not null,
    domain varchar(255) not null
        constraint users_domains_domain_fk
            references domains
                on update restrict on delete restrict,
    password varchar(64) not null,
    active boolean default true not null,
    constraint users_pk
        primary key (username, domain)
);

INSERT INTO domains VALUES ('example.com', true);
INSERT INTO users (username, domain, password) VALUES ('user', 'example.com', '<encrypted password from smtpctl>');
INSERT INTO aliases (destination, alias) VALUES ('user@example.com', 'post@example.com');
```

For the slaves to be able to connect to the PostgreSQL master on the primary MX we have to add a couple of lines to the /etc/postgresql/12/main/pg_hba.conf file.

```
host    replication     replicator      <backup mx IP>          scram-sha-256
host    replication     replicator      <imap host IP>          scram-sha-256
```

### Configuring OpenSMTPD

#### Setting up the lookup tables

Enter the connection details for the PostgreSQL database.

```
conninfo host='localhost' user='mailuser' password='<password for the mailuser>' dbname='maildb'
```

Set up the alias lookup, this lets OpenSMTPD check if an address is an alias, and if so where to forward it  to. To make OpenSMTPD happy we'll also need to make sure the primary addresses point towards the virtual users. Because we use the same user for everyone, we can add a `UNION SELECT` to the query adding a `vmail` row, *if* the address exists in the `users` table.

```
query_alias             SELECT destination FROM aliases WHERE alias=$1 UNION SELECT 'vmail' FROM users WHERE (username||'@'||domain)=$1;
```

Set up a lookup for domains, so OpenSMTPD can see what domains it's supposed to accept e-mails for.

```
query_domain            SELECT domain FROM domains WHERE domain=$1 AND active=true;
```

The credentials query is used to look up the password for connecting clients. Since the name and domain is split in the database we concatenate them together in the query, since it's the whole e-mail address that's used to sign in.

```
query_credentials       SELECT (username||'@'||domain), password FROM users WHERE (username||'@'||domain)=$1 and active=true;
```

This is a an optional part. What this does is check against the alias table what addresses a user is allowed to send e-mails with. A users primary e-mail address won't be in that table, so we add it to the result with the `UNION SELECT $1` addition.

```
query_mailaddrmap       SELECT alias FROM aliases WHERE destination=$1 AND active=true UNION SELECT $1;
```

Show OpenSMTPD where to find your TLS certificates. Note that I'm using two different domains here and in the rest of the configuration. mx1.example.com is used for dealing with other SMTP servers, while smtp.example.com is used for e-mail clients.

```
pki mx1.example.com key "/etc/letsencrypt/live/mx1.example.com/privkey.pem"
pki mx1.example.com cert "/etc/letsencrypt/live/mx1.example.com/fullchain.pem"
pki smtp.example.com key "/etc/letsencrypt/live/smtp.example.com/privkey.pem"
pki smtp.example.com cert "/etc/letsencrypt/live/smtp.example.com/fullchain.pem"
```

We'll also add some filters I shamelessly stole from Gilles Chehade's [post on OpenSMTPD](https://poolp.org/posts/2019-09-14/setting-up-a-mail-server-with-opensmtpd-dovecot-and-rspamd/). Go read his post if you'd like more details on what these filters does.

```
filter check_dyndns phase connect match rdns regex { '.*\.dyn\..*', '.*\.dsl\..*' } \
 disconnect "550 no residential connections"

filter check_rdns phase connect match !rdns \
 disconnect "550 no rDNS is so 80s"

filter check_fcrdns phase connect match !fcrdns \
 disconnect "550 no FCrDNS is so 80s"

filter senderscore \
 proc-exec "/usr/bin/filter-senderscore -blockBelow 10 -junkBelow 70 -slowFactor 5000"

filter rspamd proc-exec "/usr/bin/filter-rspamd"
```

Point OpenSMTPD towards the postgres.conf file we set up earlier. We also need a users table to tell it what user to use for virtual users. It's not actually used when delivering e-mail with LMTP, but it'll complain if it's not there.

```
table postgres postgres:/etc/smtpd/postgres.conf
table users { vmail = "5000:5000:/var/mail" }
```

Listen for e-mail locally.

```
listen on lo
```

Listen for incoming e-mail from other servers. Use STARTTLS so the sender have the option to start a encrypted connection. Authentication should be optional, and I'm not aware of a situation where that's actually used. Enable all the filters we defined earlier.

```
listen on ens3 port 25 \
 tls pki mx1.example.com \
 hostname mx1.example.com \
 auth-optional <postgres> \
 filter { check_dyndns, check_rdns, check_fcrdns, senderscore, rspamd }
```

Listen for clients connections. On port 465 we enforce TLS for clients, avoiding the possibility of a man-in-the-middle attack. Authenticate clients against the PostgreSQL database. Check if the client is allowed to use a certain address with the `senders` directive. Use `mask-src` to remove the clients address from the headers. Enable only the `rspamd` filter for DKIM signing.

```
listen on ens3 port 465 \
 smtps pki smtp.example.com \
 hostname smtp.example.com \
 auth <postgres> \
 senders <postgres> \
 mask-src \
 filter rspamd
```

Same as above, but on port 587 with STARTTLS.

```
listen on ens3 port 587 \
 tls pki smtp.example.com \
 hostname smtp.example.com \
 auth <postgres> \
 senders <postgres> \
 mask-src \
 filter rspamd
```

E-mails destined for local addresses should be sent with LMTP to Dovecot. `rcpt-to` needs to be enabled so Dovecot sees the e-mail address, and not `vmail`.

```
action "deliver" lmtp "<IMAP IP address>:24" rcpt-to userbase <users> virtual <postgres>
```

Relay all other e-mail to their MTA's.

```
action "relay" relay helo "mx1.example.com"
```

Any e-mail for domains registered in the table gets delivered to Dovecot.

```
match from any for domain <postgres> action "deliver"
```

Any e-mails generated locally is delivered to Dovecot.

```
match from local for any action "deliver"
```

If the client is authenticated, relay the e-mail to the destinations MTA. 

**Warning:** Getting this right is important. If you configure this wrong, you'll risk running an open relay, meaning spammers can abuse it.

```
match auth from any for any action "relay"
```

## Setting up the IMAP server



### Installing packages

```bash
sudo apt install dovecot-imapd dovecot-lmtpd dovecot-sieve dovecot-pgsql dovecot-managesieved
```

For PostgreSQL follow the instructions on their [download page](https://www.postgresql.org/download/linux/debian/) to install the server.

### Setting up replication against the PostgreSQL master

Change to the postgres account on the backup MX

```
sudo -u postgres -i
```

Make sure PostgreSQL is stopped

```bash
systemctl stop postgresql@12-main
```

Delete or move the data folder

```bash
rm -rf /var/lib/postgresql/12/main/*
```

Start the initial sync from the master

```bash
pg_basebackup -h  -U replicator -p 5432 -D /var/lib/postgresql/12/main -Fp -Xs -P -R
```

Start PostgreSQL back up

```bash
systemctl start postgresql@12-main
```



## Setting up the backup MX

##### Sources

- https://www.percona.com/blog/2019/10/11/how-to-set-up-streaming-replication-in-postgresql-12/

- https://poolp.org/posts/2019-09-14/setting-up-a-mail-server-with-opensmtpd-dovecot-and-rspamd/

- https://xec.net/dovecot-migration/
