Notes on Koji initial install
=============================

!!! note
    This document was written for the original install of koji/koji-hub version
    1.6 to `koji-hub.batlab.org`.  The document is kept for historical
    purposes, as this setup is no longer accurate with newer versions of Koji,
    machine moves and renames, cert authority changes, etc.

Machine setup
-------------

koji is run on `koji-hub.batlab.org`; right now a single machine was used for the hub (koji-hub), the web frontend (koji-web), repo generation (kojira), and el5 building (kojid). A second machine, `kojibuilder1.batlab.org` was used for el6 building.

As instructions for setting up the machine, I'm just going to take the Fedora guide at <http://fedoraproject.org/wiki/Koji/ServerHowTo>, and add modifications / comments to suit our setup.

Sections taken from the Fedora guide will be between dividers like this:

---

> -   For an overview of yum, mock, Koji (and all its subcomponents), mash, and how they all work together, see the excellent slides put together by Steve Traylen at CERN <http://indico.cern.ch/event/55091>

---

### Packages to install

---

> **On the server (koji-hub/koji-web)**
>
> -   httpd
> -   mod\_ssl
> -   postgresql-server
> -   mod\_python (>= 3.3.1 for Kerberos authentication)
>
> **On the builder (koji-builder)**
>
> -   mock
> -   setarch (for some archs you'll require a patched version)
> -   rpm-build
> -   createrepo

---

We used one machine for both these roles, so all of the above had to be installed.

The koji packages to install are:

-   koji-hub
-   koji-web
-   koji-builder
-   koji-utils

### Filesystems

---

> #### A note on filesystem space
>
> Koji will consume copious amounts of disk space under the primary KojiDir directory (as set in the kojihub.conf file).

---

This is `/mnt/koji` by default. Koji keeps *all RPMs built* in here so a complete history is kept. In addition, repositories are kept here. A repository is just a set of text files, but since a new one is generated for every build, the space will add up. Koji does clean up the repositories once they get old enough. For `koji-hub.batlab.org`, we allocated a 120G partition for `/mnt/koji`.

---

> However, as koji makes use of mock on the backend to actually create build roots and perform the builds in those build roots, it might come to a surprise to users that a running koji server will consume large amounts of disk space under /var/lib/mock and /var/cache/mock as well.

---

`kojid` (the builder daemon) will refuse to start a build if it does not have sufficient space under `/var/lib/mock`. By default, this is an 8G surplus, but we lowered it to 4G for `koji-hub.batlab.org`. The build roots will be wiped after successful builds, but the build roots for failed builds will be kept around for 2 weeks (?); as a result, disk usage of `/var/lib/mock` can swing wildly. For `koji-hub.batlab.org`, we have allocated 35G for `/var/lib/mock`. `/var/cache/mock` doesn't use up that much space, so we didn't make a separate partition for it, it just uses what's under `/`.

### Authentication

Most of the "Koji Authentication Selection" section can be skipped. We got our certs from DigiCert. **Note** that certs are sensitive to line ending issues -- you may have to run `dos2unix` on them before they would work.

We have 3 certs, with the following subjects:

-   `/DC=org/DC=opensciencegrid/O=Open Science Grid/OU=Services/CN=koji-hub.batlab.org` (for koji-hub, koji-web, and kojid)
-   `/DC=org/DC=opensciencegrid/O=Open Science Grid/OU=Services/CN=kojira/koji-hub.batlab.org` (for kojira)
-   `/DC=com/DC=DigiCert-Grid/O=Open Science Grid/OU=Services/CN=kojibuilder1.batlab.org` (for kojid on kojibuilder1)

kojid needed a different cert than kojira because the two would connect to koji-hub at the same time and log each other off.

Copies of the certs exist in AFS at `/p/condor/home/certificates/koji-hub.batlab.org/` and `/p/condor/home/certificates/kojibuilder1.batlab.org/`

Clients connecting to koji-hub need a file containing the CA certs; this is just a concatenation of the DigiCert Grid Root CA and the DigiCert Grid CA-1 certs, which can be obtained from DigiCert's website (get PEM format). We have a `digicert-chain.crt` in `/etc/pki/tls/certs/digicert-chain.crt` on `koji-hub`

We made a `kojiadmin` user, but we didn't make a cert for it so we just used user/pass authentication until we set up our own accounts and then disabled `kojiadmin`.

#### Using proxy certs

`/etc/sysconfig/httpd` needed to be changed to include the following lines:

```bash
OPENSSL_ALLOW_PROXY=1
OPENSSL_ALLOW_PROXY_CERTS=1

export OPENSSL_ALLOW_PROXY
export OPENSSL_ALLOW_PROXY_CERTS
```

The user must use RFC proxies and must have a version of the koji client of 1.6.0-6.osg or newer.

### Postgres Database

-   Install postgres:  
    `[root@koji-hub]# yum install postgresql-server`
-   Init the db:  
    `[root@koji-hub]# su - postgres -c "PGDATA=/var/lib/pgsql/data" initdb`
-   Start the db:  
    `[root@koji-hub]# service postgresql start`
-   Make a koji account:  
    `[root@koji-hub]# useradd koji; passwd -d koji`

---

<!-- I'd love to highlight the quoted stuff but the ``` is broken within blockquotes -->

> #### Setup PostgreSQL and populate schema:
>
> The following commands will create the `koji` user within PostgreSQL and will then create the koji database using the schema within the `/usr/share/doc/koji*/docs/schema.sql` directory
>
>     root@localhost$ su - postgres
>     postgres@localhost$ createuser koji
>     Shall the new role be a superuser? (y/n) n
>     Shall the new role be allowed to create databases? (y/n) n
>     Shall the new role be allowed to create more new roles? (y/n) n
>     postgres@localhost$ createdb -O koji koji
>     postgres@localhost$ logout
>     root@localhost$ su - koji
>     koji@localhost$ psql koji koji < /usr/share/doc/koji*/docs/schema.sql
>     koji@localhost$ exit
>
> **NOTE:** When issuing the command to import the psql schema into the new database it is important to ensure that the directory path `/usr/share/doc/koji*/docs/schema.sql` remains intact and is not resolved to a specific version of koji. In test it was discovered that when the path is resolved to a specific version of koji then not all of the tables were created correctly.

---

To authorize the koji-web and koji-hub resources, make the following additions to `/var/lib/pgsql/data/pg_hba.conf` (IP addresses will vary):

    # access for koji
    host    koji        postgres    127.0.0.1/32          trust
    host    koji        koji        127.0.0.1/32          trust
    host    koji        koji        128.104.100.41/32     trust
    host    koji        koji        ::1/128               trust

Next, we'll need a koji admin user to run some commands. We'll need to add it to the database manually. Once again, we used user/pass authentication for the kojiadmin user until we disabled it. All database commands should be done as the `koji` user:

```console
[root@koji-hub]# sudo -u koji psql
```
```psql
koji=>  insert into users (name, password, status, usertype) values
          ('kojiadmin', 'some-throwaway-admin-password-in-plain-text', 0, 0);
koji=>  insert into user_perms (user_id, perm_id, creator_id) values
          ((select id from users where name='kojiadmin'), 1,
           (select id from users where name='kojiadmin'));
```


### Koji-Hub

---

> Koji-hub is the center of all Koji operations. It is an XML-RPC server running under mod\_python in Apache. koji-hub is passive in that it only receives XML-RPC calls and relies upon the build daemons and other components to initiate communication. Koji-hub is the only component that has direct access to the database and is one of the two components that have write access to the file system.

---

Config files we care about:

-   `/etc/httpd/conf/httpd.conf`
-   `/etc/httpd/conf.d/kojihub.conf`
-   `/etc/httpd/conf.d/ssl.conf`
-   `/etc/koji-hub/hub.conf`

Install the necessary packages.
``` console
[root@koji-hub]# yum install koji-hub httpd mod_ssl mod_python
```

---

> #### /etc/httpd/conf/httpd.conf:
>
> The apache web server has two places that it sets maximum requests a server will handle before the server restarts. The xmlrpc interface in kojihub is a python application, and mod\_python can sometimes grow outrageously large when it doesn't reap memory often enough. As a result, it is strongly recommended that you set both instances of MaxRequestsPerChild in `httpd.conf` to something reasonable in order to prevent the server from becoming overloaded and crashing (at 100 the `httpd` processes will grow to about 75MB resident set size before respawning).
>
>     <IfModule prefork.c>
>     ...
>     MaxRequestsPerChild  100
>     </IfModule>
>     <IfModule worker.c>
>     ...
>     MaxRequestsPerChild  100
>     </IfModule>
>
> #### /etc/koji-hub/hub.conf:
>
> This file contains the configuration information for the hub. You will need to edit this configuration to point Koji Hub to the database you are using and to setup Koji Hub to utilize the authentication scheme you selected in the beginning.

---

We made multiple changes to this config file.

Notable changes:

-   We turned off `LoginCreatesUser`
-   `KojiWebURL` is <http://koji-hub.batlab.org/koji>
-   `PluginPath` is `/usr/lib/koji-hub-plugins`
-   `Plugins = sign` since there is an RPM signing plugin (package name: `koji-plugin-sign`) that we use

Other notes:

-   Should not be world-readable
-   Must be readable by the `apache` user though

Finally, there is a `[policy]` section for controlling ACLs. The contents of that are described later in this document.

---

> #### /etc/httpd/conf.d/kojihub.conf:
>
> If using SSL auth, uncomment these lines for kojiweb to allow logins.
>
>     <Location /kojihub>
>     SSLOptions +StdEnvVars
>     </Location>

---

This is actually outdated information, useful for Koji < 1.4.0. Instead, we add the following:

```apache
<Location /kojihub/ssllogin>
        SSLVerifyClient require
        SSLVerifyDepth  10
        SSLOptions +StdEnvVars
</Location>
```

---

> #### /etc/httpd/conf.d/ssl.conf:
>
> If using SSL you will also need to add the needed SSL options for apache. These options should point to where the certificates are located on the hub.

---

These are the config lines we use:

```apache
## The host cert for koji-hub
SSLCertificateFile /etc/pki/tls/certs/digicert_hostcert.crt

## The private key for koji-hub
SSLCertificateKeyFile /etc/pki/tls/private/digicert_hostkey.key

## The concatenation of the DigiCert CA certs we made above.
SSLCertificateChainFile /etc/pki/tls/certs/digicert-chain.crt

## All the CA certs on the system.
SSLCACertificateFile /etc/pki/tls/certs/ca-bundle.crt

SSLVerifyClient require
SSLVerifyDepth  10
```

Restart `httpd` after doing this.


#### Adding a real admin user

At this point, SSL should be set up on Koji Hub. Create an account for yourself (your CN) and give yourself the admin bit:

``` console
[kojiadmin@koji-hub]$ koji add-user 'Your Name 123456'
[kojiadmin@koji-hub]$ koji grant-permission admin 'Your Name 123456'
```

**Once you're done with all the setup**, you should remove the admin bit from `kojiadmin`, block the user, and remove its password from the database.

``` console
[you@koji-hub]$ koji revoke-permission admin kojiadmin
[you@koji-hub]$ koji disable-user kojiadmin
```

In the database:

``` psql
koji=>  update users set password = null where name='kojiadmin';
```

#### Koji filesystem

`KojiDir` is `/mnt/koji`, so the following tree should be created:

``` console
[root@koji-hub]# mkdir -p /mnt/koji/{packages,repos,work,scratch}
[root@koji-hub]# chown apache:apache *
```

#### Accounts for kojira and kojid

---

> It is important to note that the kojira component needs repo privileges, but if you just let the account get auto created the first time you run kojira, it won't have that privilege, so you should pre-create the account and grant it the repo privilege now.
>
>     kojiadmin@koji-hub$ koji add-user kojira
>     kojiadmin@koji-hub$ koji grant-permission repo kojira
>
> For similar technical reasons, you need to add-host each build host prior to starting kojid on that host the first time and could also do that now.

---

We turned off auto account creation, so there won't be a problem with accounts with bad perms getting created. This means they have to be created manually. We have kojid running on koji-hub, so this is the command we used to add it:

``` console
[you@koji-hub]$ koji add-host koji-hub.batlab.org i386 x86_64
```

### Koji-Web, Kojid

Installing:

``` console
[root@koji-hub]# yum install koji-web mod_ssl
[root@koji-hub]# yum install koji-builder
```

Note that we use a patched koji-builder, so be sure to install what's in the OSG repository. (The patch is not likely to be accepted upstream).

#### SSL certs

The following files should be in the `/etc/pki/tls` tree:

-   `/etc/pki/tls/private/digicert_hostkey.key`  
    the host key from DigiCert

-   `/etc/pki/tls/private/kojiweb.pem`  
    digicert\_hostcert.crt catted together with digicert\_hostkey.key (public key first)

-   `/etc/pki/tls/private/kojira.key`  
    kojira's private key from DigiCert

-   `/etc/pki/tls/private/kojira.pem`  
    kojira.crt catted together with kojira.key (public key first)

-   `/etc/pki/tls/cert.pem`  
    symlink to certs/ca-bundle.crt

-   `/etc/pki/tls/certs/ca-bundle.crt`  
    all certs available on the system; the DigiCert Grid certs should be in here

-   `/etc/pki/tls/certs/kojira.crt`  
    kojira's cert

-   `/etc/pki/tls/certs/digicert_hostcert.crt`  
    the host cert from DigiCert

-   `/etc/pki/tls/certs/digicert-chain.crt`  
    the DigiCert CAs catted together

#### Adding koji-builder host to database

```console
[you@koji-hub]$ koji add-host-to-channel koji-hub.batlab.org createrepo
```
In the database:
```psql
koji=>#  update host set capacity = 4 where name = 'koji-hub.batlab.org';
```

#### Start kojid

``` console
[root@koji-hub]# /sbin/service kojid start
```

Check `/var/log/kojid.log` to make sure everything started up.

### Kojira

Installing:

``` console
[root@koji-hub]# yum install koji-utils
```

Kojira also needs a user

``` console
[you@koji-hub]$ koji add-user kojira
[you@koji-hub]$ koji grant-permission repo kojira
```

Start it up

``` console
[root@koji-hub]# /sbin/service kojira start
```

