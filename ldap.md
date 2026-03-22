# ldap command ref

## concepts

| Term | Description |
|------|-------------|
| **DN** | Distinguished Name — unique path to an entry (e.g. `cn=ak,ou=users,dc=kdn,dc=cloud`) |
| **RDN** | Relative Distinguished Name — leftmost component of a DN (e.g. `cn=ak`) |
| **DC** | Domain Component — part of a domain name (e.g. `dc=kdn,dc=cloud` → `kdn.cloud`) |
| **CN** | Common Name — name of the object |
| **OU** | Organizational Unit — grouping container |
| **O** | Organization |
| **UID** | User ID attribute |
| **objectClass** | Defines what attributes an entry can/must have |
| **Base DN** | Root DN for a search scope |
| **Bind DN** | DN used to authenticate to the directory |
| **LDIF** | LDAP Data Interchange Format — plaintext format for entries and changes |

## tools

```bash
# ldap-utils (Debian/Ubuntu)
apt install ldap-utils

# macOS (via Homebrew)
brew install openldap

# common binaries
ldapsearch     # query the directory
ldapadd        # add entries from LDIF
ldapmodify     # modify entries from LDIF
ldapdelete     # delete entries
ldappasswd     # change passwords
ldapwhoami     # check bind identity
ldapmodrdn     # rename/move entries
```

## connection flags (common to all tools)

```bash
-H ldap://host:389          # server URI (ldap or ldaps)
-H ldaps://host:636         # TLS
-H ldapi:///                # Unix socket (local)
-x                          # simple auth (not SASL)
-D "cn=admin,dc=kdn,dc=cloud"  # bind DN
-w password                 # bind password (inline)
-W                          # prompt for bind password
-Z                          # use STARTTLS
-ZZ                         # require STARTTLS
-v                          # verbose
-d 1                        # debug level 1 (increase for more)
```

## ldapsearch

```bash
# basic syntax
ldapsearch -x -H ldap://localhost \
  -D "cn=admin,dc=kdn,dc=cloud" -w password \
  -b "dc=kdn,dc=cloud" \
  "(filter)" attr1 attr2

# search everything under base DN
ldapsearch -x -H ldap://localhost \
  -D "cn=admin,dc=kdn,dc=cloud" -w password \
  -b "dc=kdn,dc=cloud" "(objectClass=*)"

# find a specific user by uid
ldapsearch -x -H ldap://localhost \
  -D "cn=admin,dc=kdn,dc=cloud" -w password \
  -b "dc=kdn,dc=cloud" \
  "(uid=ak)"

# find all users in an OU
ldapsearch -x -H ldap://localhost \
  -D "cn=admin,dc=kdn,dc=cloud" -w password \
  -b "ou=users,dc=kdn,dc=cloud" \
  "(objectClass=inetOrgPerson)"

# return specific attributes only
ldapsearch -x -H ldap://localhost \
  -D "cn=admin,dc=kdn,dc=cloud" -w password \
  -b "dc=kdn,dc=cloud" \
  "(uid=ak)" cn mail memberOf

# return no attributes (just DNs)
ldapsearch ... "(filter)" 1.1

# search scope
-s base     # base DN only (single entry)
-s one      # one level below base
-s sub      # full subtree (default)

# limit results
-z 100      # size limit (max 100 results)
-l 30       # time limit (30 seconds)

# paged results (for large directories)
-E pr=100/noprompt

# output as LDIF
ldapsearch ... -LLL          # clean LDIF (suppress comments/version)
ldapsearch ... -LL           # suppress comments only
ldapsearch ... -L            # standard LDIF with comments
```

## search filters

```bash
# basic equality
(uid=ak)
(cn=John Doe)
(objectClass=posixAccount)

# presence (attribute exists)
(mail=*)
(memberOf=*)

# wildcard
(cn=a*)                      # starts with a
(cn=*doe*)                   # contains doe
(mail=*@kdn.cloud)           # ends with domain

# negation
(!(uid=ak))

# AND (all must match)
(&(objectClass=inetOrgPerson)(uid=ak))
(&(objectClass=posixAccount)(uidNumber>=1000))

# OR (any must match)
(|(uid=ak)(mail=ak@kdn.cloud))

# combined
(&(objectClass=inetOrgPerson)(|(uid=ak)(cn=AK*)))

# comparison operators
(uidNumber>=1000)
(uidNumber<=9999)

# approximate match (~=)
(cn~=john)

# extensible match (check attribute in DN)
(ou:dn:=admins)
```

## ldapadd — adding entries

```bash
ldapadd -x -H ldap://localhost \
  -D "cn=admin,dc=kdn,dc=cloud" -w password \
  -f new_entry.ldif

# or pipe LDIF directly
ldapadd -x -H ldap://localhost \
  -D "cn=admin,dc=kdn,dc=cloud" -w password << 'EOF'
dn: uid=newuser,ou=users,dc=kdn,dc=cloud
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: newuser
cn: New User
sn: User
mail: newuser@kdn.cloud
uidNumber: 1001
gidNumber: 1001
homeDirectory: /home/newuser
loginShell: /bin/bash
EOF
```

## ldif examples

### new user entry

```ldif
dn: uid=ak,ou=users,dc=kdn,dc=cloud
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: ak
cn: AK
sn: Klein
givenName: A
mail: ak@kdn.cloud
uidNumber: 1000
gidNumber: 1000
homeDirectory: /home/ak
loginShell: /bin/bash
userPassword: {SSHA}hashedpasswordhere
```

### new group entry

```ldif
dn: cn=admins,ou=groups,dc=kdn,dc=cloud
objectClass: groupOfNames
objectClass: posixGroup
cn: admins
gidNumber: 1000
member: uid=ak,ou=users,dc=kdn,dc=cloud
```

### organizational unit

```ldif
dn: ou=services,dc=kdn,dc=cloud
objectClass: organizationalUnit
ou: services
description: Service accounts
```

## ldapmodify — modifying entries

```bash
ldapmodify -x -H ldap://localhost \
  -D "cn=admin,dc=kdn,dc=cloud" -w password \
  -f changes.ldif
```

### modify operations in LDIF

```ldif
# replace an attribute value
dn: uid=ak,ou=users,dc=kdn,dc=cloud
changetype: modify
replace: mail
mail: newemail@kdn.cloud

# add an attribute
dn: uid=ak,ou=users,dc=kdn,dc=cloud
changetype: modify
add: telephoneNumber
telephoneNumber: +1-555-0100

# delete a specific attribute value
dn: uid=ak,ou=users,dc=kdn,dc=cloud
changetype: modify
delete: telephoneNumber
telephoneNumber: +1-555-0100

# delete all values of an attribute
dn: uid=ak,ou=users,dc=kdn,dc=cloud
changetype: modify
delete: telephoneNumber

# multiple changes in one operation (separated by -)
dn: uid=ak,ou=users,dc=kdn,dc=cloud
changetype: modify
replace: mail
mail: ak@kdn.cloud
-
add: description
description: KDN Lab admin
-
delete: telephoneNumber

# add member to a group
dn: cn=admins,ou=groups,dc=kdn,dc=cloud
changetype: modify
add: member
member: uid=newuser,ou=users,dc=kdn,dc=cloud

# remove member from a group
dn: cn=admins,ou=groups,dc=kdn,dc=cloud
changetype: modify
delete: member
member: uid=olduser,ou=users,dc=kdn,dc=cloud
```

## ldapdelete

```bash
# delete a single entry
ldapdelete -x -H ldap://localhost \
  -D "cn=admin,dc=kdn,dc=cloud" -w password \
  "uid=olduser,ou=users,dc=kdn,dc=cloud"

# delete multiple entries (one DN per line)
ldapdelete -x -H ldap://localhost \
  -D "cn=admin,dc=kdn,dc=cloud" -w password \
  -f dns_to_delete.txt

# delete a subtree (requires server support or manual ordering leaf→parent)
ldapdelete -x -H ldap://localhost \
  -D "cn=admin,dc=kdn,dc=cloud" -w password \
  -r "ou=temp,dc=kdn,dc=cloud"
```

## ldappasswd — changing passwords

```bash
# set password for a user (prompted)
ldappasswd -x -H ldap://localhost \
  -D "cn=admin,dc=kdn,dc=cloud" -w adminpass \
  -S "uid=ak,ou=users,dc=kdn,dc=cloud"

# set password inline (not recommended — appears in shell history)
ldappasswd -x -H ldap://localhost \
  -D "cn=admin,dc=kdn,dc=cloud" -w adminpass \
  -s newpassword \
  "uid=ak,ou=users,dc=kdn,dc=cloud"

# user changes own password
ldappasswd -x -H ldap://localhost \
  -D "uid=ak,ou=users,dc=kdn,dc=cloud" -W \
  -S
```

## ldapwhoami — test bind

```bash
# verify who you're bound as
ldapwhoami -x -H ldap://localhost \
  -D "cn=admin,dc=kdn,dc=cloud" -w password

# anonymous bind check
ldapwhoami -x -H ldap://localhost
```

## common objectClasses & attributes

### inetOrgPerson (general users)

```
MUST: cn, sn
MAY:  uid, mail, givenName, displayName, telephoneNumber,
      mobile, title, description, jpegPhoto, userPassword
```

### posixAccount (Linux/Unix users)

```
MUST: cn, uid, uidNumber, gidNumber, homeDirectory
MAY:  loginShell, gecos, userPassword, description
```

### posixGroup (Linux/Unix groups)

```
MUST: cn, gidNumber
MAY:  memberUid, description
```

### groupOfNames (general groups)

```
MUST: cn, member
MAY:  description, owner
```

### organizationalUnit

```
MUST: ou
MAY:  description
```

## password hashing

```bash
# generate SSHA hash (salted SHA1 — widely supported)
slappasswd -s mypassword

# generate SHA256 (if supported)
slappasswd -h '{SHA256}' -s mypassword

# generate in LDIF-ready format
echo "userPassword: $(slappasswd -s mypassword)"
```

## slapd config (OpenLDAP)

```bash
# config locations
/etc/ldap/slapd.conf          # legacy flat file config
/etc/ldap/slapd.d/            # OLC (cn=config) directory config — modern

# test config syntax
slaptest -f /etc/ldap/slapd.conf
slaptest -F /etc/ldap/slapd.d/

# reload / restart
systemctl restart slapd
systemctl status slapd

# slapd logs (adjust log level in config)
journalctl -u slapd -f
```

### useful slapd ACLs

```
# allow users to change their own password
access to attrs=userPassword
  by self write
  by anonymous auth
  by * none

# allow authenticated users to read all
access to *
  by self write
  by users read
  by anonymous none
```

## ldap with tls/ldaps

```bash
# test LDAPS connection
ldapsearch -x -H ldaps://ldap.kdn.cloud:636 \
  -D "cn=admin,dc=kdn,dc=cloud" -w password \
  -b "dc=kdn,dc=cloud" "(objectClass=*)"

# STARTTLS
ldapsearch -x -H ldap://ldap.kdn.cloud:389 \
  -Z \
  -D "cn=admin,dc=kdn,dc=cloud" -w password \
  -b "dc=kdn,dc=cloud" "(objectClass=*)"

# skip cert verification (dev/testing only)
LDAPTLS_REQCERT=never ldapsearch -x -H ldaps://ldap.kdn.cloud ...

# or in /etc/ldap/ldap.conf
TLS_REQCERT never
```

## authentik / ldap proxy

```bash
# Authentik exposes an LDAP interface at port 389 (or 636 for LDAPS)
# Bind DN format: cn=<username>,ou=users,dc=ldap,dc=goauthentik,dc=io
# Search base: dc=ldap,dc=goauthentik,dc=io

# test Authentik LDAP bind
ldapwhoami -x -H ldap://authentik.lab.kdn.cloud:389 \
  -D "cn=ak,ou=users,dc=ldap,dc=goauthentik,dc=io" -W

# search users via Authentik LDAP
ldapsearch -x -H ldap://authentik.lab.kdn.cloud:389 \
  -D "cn=ak,ou=users,dc=ldap,dc=goauthentik,dc=io" -W \
  -b "dc=ldap,dc=goauthentik,dc=io" \
  "(objectClass=user)" cn mail

# service account bind (ldapservice user in Authentik)
ldapsearch -x -H ldap://authentik.lab.kdn.cloud:389 \
  -D "cn=ldapservice,ou=users,dc=ldap,dc=goauthentik,dc=io" -w password \
  -b "ou=users,dc=ldap,dc=goauthentik,dc=io" \
  "(objectClass=user)"
```

## troubleshooting

```bash
# connection refused
# → check slapd is running: systemctl status slapd
# → check port: ss -tlnp | grep 389

# invalid credentials (49)
# → wrong bind DN or password
# → test with ldapwhoami first

# insufficient access (50)
# → ACLs blocking the operation
# → check slapd ACL config

# no such object (32)
# → base DN doesn't exist yet
# → check DN spelling exactly

# object class violation (65)
# → missing required attribute for objectClass
# → check MUST attributes for all objectClasses used

# can't contact LDAP server
# → check -H URI and port
# → check TLS cert issues: try LDAPTLS_REQCERT=never

# enable verbose client debug
ldapsearch -d 1 ...           # basic
ldapsearch -d 255 ...         # maximum verbosity

# check server-side logs
journalctl -u slapd -f
# increase log level in slapd config:
# loglevel 256   (stats)
# loglevel 512   (stats2)
# loglevel -1    (all — very verbose)
```

## useful one-liners

```bash
# list all users
ldapsearch -x -LLL -H ldap://localhost \
  -D "cn=admin,dc=kdn,dc=cloud" -w password \
  -b "ou=users,dc=kdn,dc=cloud" "(objectClass=posixAccount)" uid cn mail

# list all groups and their members
ldapsearch -x -LLL -H ldap://localhost \
  -D "cn=admin,dc=kdn,dc=cloud" -w password \
  -b "ou=groups,dc=kdn,dc=cloud" "(objectClass=groupOfNames)" cn member

# check which groups a user belongs to
ldapsearch -x -LLL -H ldap://localhost \
  -D "cn=admin,dc=kdn,dc=cloud" -w password \
  -b "dc=kdn,dc=cloud" \
  "(&(objectClass=groupOfNames)(member=uid=ak,ou=users,dc=kdn,dc=cloud))" cn

# export entire directory to LDIF
ldapsearch -x -LLL -H ldap://localhost \
  -D "cn=admin,dc=kdn,dc=cloud" -w password \
  -b "dc=kdn,dc=cloud" "(objectClass=*)" > backup.ldif

# bulk import LDIF
ldapadd -x -H ldap://localhost \
  -D "cn=admin,dc=kdn,dc=cloud" -w password \
  -f backup.ldif

# count entries under base DN
ldapsearch -x -LLL -H ldap://localhost \
  -D "cn=admin,dc=kdn,dc=cloud" -w password \
  -b "dc=kdn,dc=cloud" "(objectClass=*)" dn | grep "^dn:" | wc -l

# test anonymous bind (check if allowed)
ldapsearch -x -H ldap://localhost \
  -b "dc=kdn,dc=cloud" "(objectClass=*)" dn
```
