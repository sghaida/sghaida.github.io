---
layout: post
title: "Parsing Cisco Switche config using TCL and shell script"
date: 2012-3-25 09:00:00 +0000
categories: [infra, networking]
tags: [networking, infra]
---

# Extracting Cisco Configurations Using TCL and Bash

Today I’d like to talk about extracting Cisco configurations—such as **switches, routers, ASA, etc.**—from network devices to your Linux machine for further manipulation.

To achieve this, I wrote **two scripts**:

1. **A TCL script** for exporting or configuring Cisco devices  
2. **A Bash script** for manipulating the extracted data in a fairly complex way  

---

## Overview

We’ll start with the TCL script. I downloaded part of it from a website and modified it, since TCL can be quite cryptic for me 🙂.

What the TCL script does:

- Retrieves the VLAN brief
- Saves the output to a log file
- Retrieves interface status
- Executes a Bash script to parse the generated files

This approach can be used for large-scale deployments. You can even push configurations through a TCL script, although I avoided that here since it is very similar to manually entering passwords or executing shell commands.

I hope this will be useful for some people and educational for others 🙂

---

## What the TCL Script Does (Step by Step)

To simplify things, the TCL script performs the following actions:

- Connects to a Cisco device (in my case, a switch stack)
- Sends the login password, then enters enable mode with the enable password
- Starts logging and disables the Cisco pager to get full output without interaction
- Executes `show vlan brief` to retrieve VLAN ID, description, name, and ports
- Stops logging after writing results to a file
- Starts logging to a different file and disables the pager again
- Executes `show interfaces status` to retrieve interface name, VLAN ID, and status
- Stops logging and executes a Bash script to parse the generated files
- Requires the IP address to be passed as an argument when executing the script

---

## TCL / Expect Script

```tcl
#!/usr/bin/expect -f
#!/bin/bash

set force_conservative 1 ;
# set to 1 to force conservative mode even if
# script wasn't run conservatively originally

if {$force_conservative} {
    set send_slow {1 .1}
    proc send {ignore arg} {
        sleep .1
        exp_send -s -- $arg
    }
}

set timeout 3000
log_user 1

set var1 [lindex $argv 0]
set var2 [lindex $argv 1]

puts $var1
puts $var2

spawn ssh -1 -l username $var1
match_max 100000

expect "*assword: " {send -- "password1\r"}
sleep .5

expect ">"
send -- "en\r"

expect "*assword:" {
    send -- "password2\r"
}

log_file /root/scripts/vlans_per_id
expect "#" {send -- "term len 0\r"}
expect "#" {send -- "sh vlan brief\r"}
expect "#" {send -- "\r"}
sleep .5
log_file

log_file /root/scripts/vlans_per_interface
expect "#" {send -- "term len 0\r"}
expect "#" {send -- "sh interfaces status\r"}
expect "#" {send -- "\r"}
expect "#" {send -- "exit\r"}
expect "#" {send -- "exit\r"}
log_file

exec /bin/bash parser.sh $var1
exit
```

---

## What the Bash Script Does

The Bash script performs the following steps:

- Reads the `vlans_per_id` file and removes leftovers (headers and footers) from the TCL log
- Reads the switch hostname and generates a file named after the hostname
- Writes the hostname and IP address of the switch into the file header
- Reads `vlans_per_interface`, removes speed and duplex information, and replaces VLAN IDs with VLAN names using data from `vlans_per_id`
- Generates a third file named `switch_hostname.map`

---

## Notes

- `awk` is used to replace `vlan_id` with `vlan_name`
- All intermediate generated files are deleted, keeping only the necessary output
- The parser was written in Bash because I’m comfortable with Bash scripting
- `awk` and `sed` are used in different ways for educational purposes
- The script could be written in many different ways, but this approach was chosen for clarity and learning—so please excuse me, geeks of the world 🙂

---

## Bash Parser Script

```bash
#!/bin/bash -x
# Prepared by sghaida
# Date: Wed Mar 21 21:11:48 EET 2012

FILE=vlans_per_id
VLANS=vlans_per_interface

sed -i -e '1,2d' $FILE
sed -i -e '2,4d' $FILE
sed -i -e '$d' $FILE

DICTIONARY=$(cat $FILE | head -n 1 | awk -F\# '{print $1}')
sed '1d' $FILE > $DICTIONARY
rm -rf $FILE

echo "Hostname   : $DICTIONARY" > $DICTIONARY.MAP
echo "IP Address : $1" >> $DICTIONARY.MAP
echo "" >> $DICTIONARY.MAP
echo -e "port \t vlan \t status" >> $DICTIONARY.MAP
echo "" >> $DICTIONARY.MAP
echo -e "---- \t ---- \t ------" >> $DICTIONARY.MAP
echo "" >> $DICTIONARY.MAP

sed -i -e '1,5d' $VLANS
sed -i -e '$d' $VLANS
sed -i -e '$d' $VLANS
sed -i -e '$d' $VLANS

cat $VLANS | awk '{print $1 "\t\t" $3 "\t\t" $2}' > $VLANS.TMP
mv $VLANS.TMP $VLANS

while read line; do
    vlan_id=$(echo $line | awk '{print $2}')

    if [ "$vlan_id" == "trunk" ]; then
        vlan_name="trunk"
        continue
    fi

    while read another_line; do
        vlan_id2=$(echo $another_line | awk '{print $1}')
        vlan_name2=$(echo $another_line | awk '{print $2}')

        if [ "$vlan_id" -eq "$vlan_id2" ]; then
            vlan_name=$vlan_name2
            break
        fi
    done < $DICTIONARY

    echo -e "$line" | awk -v v1="$vlan_name" -v v2="$vlan_id" \
        '{ sub(v2,v1,$2); print $1 "\t" $2 "\t" $3 }' >> $DICTIONARY.MAP

done < $VLANS

rm -rf $VLANS
```
