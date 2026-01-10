---
layout: post
title: "SNMP MIBS Extensions and how to use"
date: 2011-10-3 09:00:00 +0000
categories: [network, infra]
tags: [network, infra]
---

# Using `extend` in SNMP to Expose Custom Script Output

I don’t know how many people are familiar with SNMP configuration directives, but one directive I find especially interesting is **`extend`**.

What makes `extend` powerful is that you can define your own script, execute it through SNMP, and expose its output under variables such as **`nsExtendOutput`**. Later, you can use these values to build convenient graphs using tools like **MRTG** or **RRD**, tailored to your needs 🙂.

Let’s start.

---

## 1) Create a Script to Get Network Traffic Stats

The following script retrieves **packets received**, **packets sent**, and **both per second** from an interface (e.g., `eth0`) over the last 5 minutes.

```bash
#!/bin/bash

stats=$(sar -n DEV -s $(date --date='5 mins ago' +%T) | grep eth0 | tail -n2 | head -n1)

case $1 in
    received ) echo "$stats" | awk '{ print $6 }' ;;
    sent     ) echo "$stats" | awk '{ print $7 }' ;;
    both     ) echo "$stats" | awk '{ print $6 + $7 }' ;;
esac
```

---

## 2) Save the Script and Update `snmp.conf`

* Save the script anywhere you want
* Make it executable
* Then update `snmp.conf` with the following `extend` entries:

```conf
extend networktraffic      /etc/snmp/myscript.sh both
extend networktrafficrcvd  /etc/snmp/myscript.sh received
extend networktrafficsent  /etc/snmp/myscript.sh sent
```

---

## 3) Test Using `snmpwalk`

### If you are using SNMP v3

```bash
snmpwalk -v3 -u testuser2 -l authNoPriv -a MD5 -A "pass" localhost .1 | grep networktraffic
```

### If you are using SNMP v2c

```bash
snmpwalk -v2c -c public localhost .1 | grep networktraffic
```

---

## Sample Output

You should see output similar to the following (pay attention to the `nsExtendOutput` entries):

```text
HOST-RESOURCES-MIB::hrSWRunParameters.12938 = STRING: "networktraffic"
NET-SNMP-EXTEND-MIB::nsExtendCommand."networktraffic" = STRING: /etc/snmp/myscript.sh
NET-SNMP-EXTEND-MIB::nsExtendCommand."networktrafficrcvd" = STRING: /etc/snmp//myscript.sh
NET-SNMP-EXTEND-MIB::nsExtendCommand."networktrafficsent" = STRING: /etc/snmp//myscript.sh
NET-SNMP-EXTEND-MIB::nsExtendArgs."networktraffic" = STRING: both
NET-SNMP-EXTEND-MIB::nsExtendArgs."networktrafficrcvd" = STRING: received
NET-SNMP-EXTEND-MIB::nsExtendArgs."networktrafficsent" = STRING: sent
NET-SNMP-EXTEND-MIB::nsExtendInput."networktraffic" = STRING:
NET-SNMP-EXTEND-MIB::nsExtendInput."networktrafficrcvd" = STRING:
NET-SNMP-EXTEND-MIB::nsExtendInput."networktrafficsent" = STRING:
NET-SNMP-EXTEND-MIB::nsExtendCacheTime."networktraffic" = INTEGER: 5
NET-SNMP-EXTEND-MIB::nsExtendCacheTime."networktrafficrcvd" = INTEGER: 5
NET-SNMP-EXTEND-MIB::nsExtendCacheTime."networktrafficsent" = INTEGER: 5
NET-SNMP-EXTEND-MIB::nsExtendExecType."networktraffic" = INTEGER: exec(1)
NET-SNMP-EXTEND-MIB::nsExtendExecType."networktrafficrcvd" = INTEGER: exec(1)
NET-SNMP-EXTEND-MIB::nsExtendExecType."networktrafficsent" = INTEGER: exec(1)
NET-SNMP-EXTEND-MIB::nsExtendRunType."networktraffic" = INTEGER: run-on-read(1)
NET-SNMP-EXTEND-MIB::nsExtendRunType."networktrafficrcvd" = INTEGER: run-on-read(1)
NET-SNMP-EXTEND-MIB::nsExtendRunType."networktrafficsent" = INTEGER: run-on-read(1)
NET-SNMP-EXTEND-MIB::nsExtendStorage."networktraffic" = INTEGER: permanent(4)
NET-SNMP-EXTEND-MIB::nsExtendStorage."networktrafficrcvd" = INTEGER: permanent(4)
NET-SNMP-EXTEND-MIB::nsExtendStorage."networktrafficsent" = INTEGER: permanent(4)
NET-SNMP-EXTEND-MIB::nsExtendStatus."networktraffic" = INTEGER: active(1)
NET-SNMP-EXTEND-MIB::nsExtendStatus."networktrafficrcvd" = INTEGER: active(1)
NET-SNMP-EXTEND-MIB::nsExtendStatus."networktrafficsent" = INTEGER: active(1)
NET-SNMP-EXTEND-MIB::nsExtendOutput1Line."networktraffic" = STRING: 12145.6
NET-SNMP-EXTEND-MIB::nsExtendOutput1Line."networktrafficrcvd" = STRING: 6309.75
NET-SNMP-EXTEND-MIB::nsExtendOutput1Line."networktrafficsent" = STRING: 5835.86
NET-SNMP-EXTEND-MIB::nsExtendOutputFull."networktraffic" = STRING: 12145.6
NET-SNMP-EXTEND-MIB::nsExtendOutputFull."networktrafficrcvd" = STRING: 6309.75
NET-SNMP-EXTEND-MIB::nsExtendOutputFull."networktrafficsent" = STRING: 5835.86
NET-SNMP-EXTEND-MIB::nsExtendOutNumLines."networktraffic" = INTEGER: 1
NET-SNMP-EXTEND-MIB::nsExtendOutNumLines."networktrafficrcvd" = INTEGER: 1
NET-SNMP-EXTEND-MIB::nsExtendOutNumLines."networktrafficsent" = INTEGER: 1
NET-SNMP-EXTEND-MIB::nsExtendResult."networktraffic" = INTEGER: 0
NET-SNMP-EXTEND-MIB::nsExtendResult."networktrafficrcvd" = INTEGER: 0
NET-SNMP-EXTEND-MIB::nsExtendResult."networktrafficsent" = INTEGER: 0
NET-SNMP-EXTEND-MIB::nsExtendOutLine."networktraffic".1 = STRING: 12145.6
NET-SNMP-EXTEND-MIB::nsExtendOutLine."networktrafficrcvd".1 = STRING: 6309.75
NET-SNMP-EXTEND-MIB::nsExtendOutLine."networktrafficsent".1 = STRING: 5835.86
```

The values you’ll typically graph are:

* `NET-SNMP-EXTEND-MIB::nsExtendOutput1Line."networktraffic"`
* `NET-SNMP-EXTEND-MIB::nsExtendOutput1Line."networktrafficrcvd"`
* `NET-SNMP-EXTEND-MIB::nsExtendOutput1Line."networktrafficsent"`
