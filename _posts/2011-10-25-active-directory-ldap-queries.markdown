---
layout: post
title: "Useful LDAP Queries against Active Directory"
date: 2011-10-25 09:00:00 +0000
categories: [Infra]
tags: [openldap, infra]
---


# LDAP Search Examples Using `ldapsearch`

I want to share some useful LDAP queries that can be executed against directory services using the `ldapsearch` utility. The examples listed below were tested against an **Active Directory Domain Controller Global Catalog**.

---

## 1. Get Specific Attributes from a Search Filter

```bash
ldapsearch -LLL \
  -H ldap://x.x.x.x:3268/ \
  -x \
  -D "test@domain.com" \
  -w 123456 \
  -b "dc=com" \
  -s sub \
  '(&(objectClass=user)(sAMAccountName=sghaida))' \
  dn cn title sAMAccountName userPrincipalName mail
```

This query retrieves only specific attributes for a user matching the given `sAMAccountName`.

---

## 2. Search User by Email (Exclude Contacts)

This query searches for a user by email address while excluding any objects that inherit from the `contact` object class.

```bash
ldapsearch -L \
  -b "dc=com" \
  -D "test@domain.com" \
  -x \
  -w 123456 \
  -h 10.1.0.75 \
  -p 3268 \
  "(&(!(objectClass=contact))(objectClass=user)(mail=$1))"
```

---

## 3. Search Users by `sAMAccountName` (Exclude Computers)

In Active Directory, machines are also represented as users. This query excludes objects that inherit from the `computer` object class.

```bash
ldapsearch -L \
  -b "dc=com" \
  -D "test@domain.com" \
  -x \
  -w 123456 \
  -h 10.1.0.75 \
  -p 3268 \
  "(&(!(objectClass=computer))(objectClass=user)(sAMAccountName=$1))"
```

---

## 4. Get Email Addresses for a Specific UPN

The following Bash script retrieves email addresses associated with a specific `userPrincipalName`.

```bash
#!/bin/bash

email=$(ldapsearch \
  -b "dc=com" \
  -D "DOMAIN\\test" \
  -x \
  -w 123456 \
  -h x.x.x.x \
  -p 3268 \
  "(userPrincipalName=$1)" | \
  grep ^mail: | awk '{printf $2" "}')

echo -e " $1 $email "
```
