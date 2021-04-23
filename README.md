# bigip-vs-simple-stats

## Overview

This is a simple Perl script that will compute a few useful statistics on a BIG-IP (v12+) for LTM Virtual Servers.  It collects the data using snmp v2c against the BIG-IP.  It uses the `snmpbulkwalk` binary on the local system.  It is possible to run this script on the BIG-IP control plane itself.

## Synopsis

```bash
collect-performance-stats [-i <sleep_interval>] [-c <community>] [-a <snmp_host_ip>] [-d <delimiter>] [--help]
```

## Description

The script will bulkwalk the following table: `F5-BIGIP-LOCAL-MIB::ltmVirtualServStat`.  This table is indexed by partition-qualified Virtual Server (e.g., `/Common/vs01`) names.  For each Virtual Server, the following table rows are retained: `ClientBytesIn`, `ClientBytesOut`, `ClientTotConns` and `ClientMaxConns`.  The script performs a walk to get a baseline for data.  It then sleeps for *sleep_interval* seconds (the default in 5).  It continues this cycle (poll then sleep) indefinitely.  For each Virtual Server, between one collection and the immediately previous collection, it determines the maximum concurrent client connections, the client-side bytes in/out delta, and the throughput (in bytes/sec) for the period.  It prints these values, joining them using the specified *delimiter* (the default is a tab).  For the *delimiter* and string can be used, but the literal '\t' is taken to mean "a tab", while the literal '\n' is taken to mean "a newline".  The data collection uses the binary `/bin/snmpbulkwalk`, which must be able to resolve the logical label `F5-BIGIP-LOCAL-MIB::ltmVirtualServStat`.  The script queries the IP address specified as *snmp_host_ip* (the default is `127.0.0.1`).  It uses the SNMP community specified as *community* (the default is `default`).

This script runs indefinitely and prints to STDOUT.  It does not buffer output.

## Running on a BIG-IP

In order to run this on the BIG-IP which you are probing, checkout this repository, copy the script to the BIG-IP, make it executable, then start running.  For example:

```bash
# on your local machine
git clone https://github.com/blorticus/bigip-vs-simple-stats.git
scp bigip-vs-simple-stats/collect-performance-stats root@192.168.10.1:/var/tmp

# on the BIG-IP cli
chmod 775 /var/tmp/collect-performance-stats
/var/tmp/collect-performance-stats | tee /var/tmp/vs-data-collection-$(date +%F).log
```

Hit ^c to terminate data collection.

## Interpretation of Statistics

The script performs an snmpbulkwalks periodically.  After each walk, the data are processed, then the script sleeps for the provided <sleep_interval>.  The actual <interval> between polls, therefore, will be longer than <sleep_interval> by the amount of time it takes to actually perform the walk and subsequently process the data.

The output fields are as follows:
* Partition: the BIG-IP partition that contains the Virtual Server;
* VS Name: the name of the Virtual Server;
* New Connections this Interval: the total number of new client connections added toward the Virtual Server during the last interval;
* Max CC: the maximum concurrent connections that have been processed by this Virtual Server since the counters were last reset;
* CPS this Interval: the rate of client connections/sec toward this Virtual Server during this interval;
* Client Bytes In this Interval: the total number of bytes received on all client connections by this Virtual Server during this interval;
* Client Bytes Out this Interval: the total number of bytes sent to all client connections by this Virtual Server during this interval;
* Throughput this Interval: the sum of Client Bytes In this Interval and Client Bytes Out this Interval;
* Interval Length (in ms): the length of this interval in milliseconds.
