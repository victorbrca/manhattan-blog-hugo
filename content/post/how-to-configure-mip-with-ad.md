---
title: "How to Configure MIP With Active Directory"
date: 2020-11-09T16:00:00-04:00
draft: false
author: "Victor Mendonça"
description: "Instructions on how to configure MIP with AD"
tags: ["MIP", "AD"]
categories: ["MIP"]
---

One of the most common configurations changes for WMOS is to have users authenticate against Active Directory instead of the user store in the WMOS database (which is usually the MDA schema in the WMOS database). This allows for an easier and more granular control of the user authentication, and can sometimes help the project meet coporate requirements.

ℹ️ _**Note:** The AD configuration can also be done during the install, however it's advisable to do it after the application has been installed and confirmed to be working._

Understanding MIP Authentication Source Types
---

MIP has 3 different Authentication Source types (`msf.authentication.option`). It's important that we understand the differences in order to pick the one that will work the best for our use case.

#### Database

Users are authenticated against the WMOS database. All user authentication changes are managed via MDA.

#### LDAP

Users exist in the WMOS database, however authentication is done via LDAP to the Active Directory server.

#### LDAP-DB

Users exist in the WMOS database and authentication is done via LDAP to the Active Directory server. However, some users can alternatively be configured to authenticate against the WMOS database.

This can be very handy in situations where:

+ You are doing a proof of concept for using AD in WMOS
+ You are doing the initial testing for AD authentication in WMOS
+ If RF/Vocollect users cannot meet the corporate password policy (like having special characters in the password)

The authentication is controlled via the entry `ISPASSWORDMANAGEDINTERNALLY` in the WMOS database under the `UCL_USER` table. The first time a user tries to authenticate MIP will try the AD, and if it fails MIP will try authenticating the user against the WMOS DB. If at any point the authentication against the AD is successful, MIP will change `ISPASSWORDMANAGEDINTERNALLY` to '0' and the user will no longer be able to authenticate against the WMOS database.

_Example cases:_
![](/img/how-to-configure-mip-with-ad/authentication-sources.png)

Before we Get Started
---

We will need a few things before we can get started.

#### SSL Certificate for the AD

I strongly advise you to connect to your active directory via SSL. Without SSL all authentication traffic between MIP and the AD server can be snooped by anyone in the network.

Your infrastructure team should be able to provide you with a certificate file that can be used to connect to the AD. Make sure to ask for a non-binary file in the `.cer` format.

#### The FQDN or IP address for the AD(s) server

This will be either the Fully Qualified Name (like '_myadserver.mycompany.com_') or an IP address. It's a good idea to have a secondary AD server as it can help in the event that the primary AD server goes down (so your users can continue to login).

**Related config:**

+ `ldap.primary.host` - FQDN or IP for the primary AD server
+ `ldap.primary.port` - Port for the primary AD server (TCP/UDP: 389 or LDAPS: 636)
+ `ldap.secondary.host` - FQDN or IP for the secondary AD server
+ `ldap.secondary.port` - Port for the secondary AD server(TCP/UDP: 389 or LDAPS: 636)

#### AD structure information

We need to know how to search for users in your company's Active Directory. Your system administrator should be able to help with this info.

|Field|Description|Examples|
|:----:|:-----:|:---:|
| `ldap.base.ctx.dn` |  Base context DN to establish connection to the LDAP | `DC=mycompany,DC=com` or `OU=Warehouse1,DC=mycompany,DC=com` |
| `ldap.user.id.attribute` | Attribute ID to use to get the username | `sAMAccountName` |
| `ldap.user.ctx.dn` | DN to look for users | `DC=mycompany,DC=com` or `CN=Users,OU=Warehouse1,DC=mycompany,DC=com` |
| `ldap.user.filter` | Filter to get the user | `(&amp;(sAMAccountName=%v)(objectcategory=user))` |
| `ldap.group.id.attribute` | Attribute ID to get Group Name | `cn` |
| `ldap.group.ctx.dn` | DN to look for groups in LDAP | |
| `ldap.group.filter` | Filter to query groups | `(&amp;(cn=%v)(objectcategory=group))` |
| `ldap.user.group.attribute.id` | Attribute ID in user object that give information about user groups | `memberOf` |
| `ldap.group.attribute.isDN` | Is the attibute id given above is itself a DN | `true` |
| `ldap.group.nesting.level` | How deep to traverse in the tree | `2` |
| `ldap.user.displayname.attribute` | Attribute id in user object, which gives the full name of the user   | `(&amp;(sAMAccountName=%v)(objectcategory=user))` |
| `ldap.user.dn.attribute` | Attribute id in user object  | `distinguishedName` |
| `ldap.group.dn.attribute` | Attribute id in group object  | `distinguishedName` |
| `ldap.search.scope` | Search type in LDAP  | `SUBTREE_SCOPE` |

#### AD username and credential

We will also need an AD user (bind DN) that has access to active directory. I would advised having separate users for your test environment and production. If this account gets locked users will not be able to login to WMOS.

**Related config:**

+ `ldap.bind.dn` - Username for bind DN
+ `ldap.bind.password` - Password for bind DN

#### User in the WMOS Database

Remember I mentioned before that the users need to exist in the WMOS database? So make sure that we have at least one user in the database that we can test with. The user ID should be exactly the same as the username in AD.

#### Identify the Application Paths

There are two paths that we should know before we continue: the application root folder (`MIP_ROOT` or `mip_root`) and the application profile folder (`mip_profile` or `MIP_PROFILE`). They will be the base paths of our configuration.

My examples will use the following paths:

+ **MIP_ROOT** - `/opt/manhattan/apps/mip`
+ **MIP_PROFILE** - `/opt/manhattan/apps/mip/profile-root/mip_dev`

#### Backing up Files

I advise making a backup of all the files that we are going to modify. This way you can revert the changes should something go wrong.

_Make a backup of the following files:_

|File|Description|
|:--|:--|
| `${MIP_ROOT}/distribution/mip.properties` | Main LDAP configuration |
| `${MIP_ROOT}/distribution/tomcat/bin/catalina.sh` | JAVA Options |
| `${MIP_PROFILE}/conf/msf-jaas-config.xml` | Runtime configuration |
| `${MIP_ROOT}/distribution/resources/templates/mip/mip-trustcerts.jks` | * Certificate truststore |

_\* The truststore is where we need to save the certificate that will be used to connect to the AD. Manhattan support will sometimes advised you to create a new one, however, I would advise you to use the existing one to keep uniformity and predictability between the environments._

#### Files to Delete

The `msf-jaas-config.xml` is a runtime file that would usually be recreated with an install command (more about it later). However, it so happens that it does not. So we need to delete the file to force it to be recreated (make sure you backed it up in the previous step).

|File|Description|
|:--|:--|
| `${MIP_PROFILE}/conf/msf-jaas-config.xml` | Runtime configuration |


Getting Started
---

We can now start to make all the required changes for AD to work. Most of these changes can be made with the application running. I will let you know when to stop the application.

#### Importing the Certificate

Upload the certificate to the server. I advise setting up a folder that will be the same throughout all the environments (for example `/opt/manhattan/certificates`). This will keep environments consistent and allow you to easily automate tasks in the future.

Once the certificate is uploaded we can import it to the truststore. The command format is:

```bash
$ keytool -import -trustcacerts -noprompt -file \
 "[certificate_path]/[certificate_name].cer" -alias "[alias for my certificate]" \
 -keystore "${MIP_ROOT}/distribution/resources/templates/mip/mip-trustcerts.jks" \
 -storepass "[truststore_password]"
```

_Example:_
```bash
$ keytool -import -trustcacerts -noprompt -file \
 "/opt/manhattan/certificates/my_certificate.cer" -alias "AD-CERT" \
 -keystore "/opt/manhattan/apps/mip/distribution/resources/templates/mip/mip-trustcerts.jks" \
 -storepass "idpkeys"
```

**⚠️ WARNING:** _I'm using the default password above. I strongly advise that you change it to something else (you will need another set of commands for that)._

#### Configuring MIP to use the truststore

Once the truststore has the certificate we can configure MIP to use it at run time. This is usually done with the Java option `Djavax.net.ssl.trustStore`.

Edit `${MIP_ROOT}/distribution/tomcat/bin/catalina.sh` and look for the line that has `Dorg.apache.catalina.security.SecurityListener`. Add the lines below right after (note the path and the password. Set it to reflect your environment).

```bash
# Added for LDAPS  
JAVA_OPTS="$JAVA_OPTS -Djavax.net.ssl.trustStore=/opt/manhattan/apps/mip/profile-root/mip_dev/conf/mip-trustcerts.jks -Djavax.net.ssl.trustStorePassword=idpkeys"
```

#### Configuring MIP

Now let's edit `${MIP_ROOT}/distribution/mip.properties` with the values we gathered before. I have a configuration below that you can use as example for your configuration.


```bash
# ######################################################## #
# What is the authentication option for MSF?               #
# Possible values are ldap/db/ldap-db/external             #
# ######################################################## #
msf.authentication.option=ldap-db
 
# ############### #
# LDAP properties #
# ############### #
ldap.primary.host=ad_server1.mydomain.com
ldap.primary.port=636

# if we need a active/passive ldap support, fill in values
ldap.secondary.host=ad_server2.mydomain.com
ldap.secondary.port=636

# Username and password to connect to the LDAP server
ldap.bind.dn=domain\\bind_user
ldap.bind.password=[bind password]

# Base context DN to establish connection to the LDAP
ldap.base.ctx.dn=DC=mycompany,DC=com

# Attribute ID to use to get the username
ldap.user.id.attribute=sAMAccountName

# DN to look for users
ldap.user.ctx.dn=DC=mycompany,DC=com

# Filter to get the user
ldap.user.filter=(&amp;(sAMAccountName=%v)(objectcategory=user))

# Attribute ID to get Group Name
ldap.group.id.attribute=cn

# DN to look for groups in LDAP
ldap.group.ctx.dn=

# Filter to query groups
ldap.group.filter=(&amp;(cn=%v)(objectcategory=group))

# Attribute ID in user object that give information about user groups
ldap.user.group.attribute.id=memberOf

# Is the attibute id given above is itself a DN. If yes, the code will query that DN to get the group informaiton
# If not, the attribute value is itself used as a grop name
ldap.group.attribute.isDN=true

# We support multiple level of groups .Specifies how deep to traverse in the tree.
ldap.group.nesting.level=2

# Attribute id in user object, which gives the full name of the user
ldap.user.displayname.attribute=(&amp;(sAMAccountName=%v)(objectcategory=user))

# Attribute id in user object , which give the DN of the user.
ldap.user.dn.attribute=distinguishedName

# Attribute id in group object , which give the DN of the user.
ldap.group.dn.attribute=distinguishedName
 
# Search type in LDAP
ldap.search.scope=SUBTREE_SCOPE

# Search timeout
ldap.search.timeout=1000

ldap.context.factory=com.sun.jndi.ldap.LdapCtxFactory

# Enable connection pool for LDAP  
ldap.connect.pool=true

# If connection pool is enabled  
ldap.connect.pool.maxsize=20
ldap.connect.pool.prefsize=10
ldap.connect.pool.timeout=300000

# LDAP protocol set to ldap or ldaps, if using ldaps you must configure or add a java truststore with ldaps server ssl public certificates
ldap.protocol=ldaps
```

#### Installing the Changes

We now need to stop MIP and install the new files. You don't have to stop other apps (users currently logged in will not be affected, only new users trying to login).

a. Source the MIP runtime variables

```bash
$ cd ${MIP_ROOT}/distribution
$ . ./setenv.sh
```

b. Stop MIP

```bash
$ ./scpp-ant.sh stop
```

c. Run the install command

```bash
$ ./scpp-ant.sh install
```

d. Start MIP

```bash
$ ./scpp-ant.sh start
```

Try to login to WMOS (or any other app) with the AD user that you had configured in both the WMOS database as well as in AD. You can use the MIP logs (`${MIP_PROFILE}/logs`) to troubleshoot any possible issues.

- - -

Closing Notes
---

While this seems like a long process, once you have a working configuration it's very easy to enable AD authentication, and almost seamless for the users.

If your project still has a high influx of SDNs, having all these steps automated (with Bash, Ansible or anything else) would be the next step. SDNs with full builds of MIP are known to overwrite AD configuration.
