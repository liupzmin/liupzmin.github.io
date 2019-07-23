---
title: ORA-31617 unable to open dump file
tags: oracle
categories: oracle
date: 2016-03-10 16:21
---


**Question:**  I am getting the ORA-31693 error during a data pump export in RAC:

ORA-31693: Table data object "xxx" failed to load/unload and is being skipped due to error:
ORA-31617: unable to open dump file "/u01/expdp/dump_test.dmp" for write
ORA-19505: failed to identify file "/u01/expdp/dump_test.dmp"
ORA-27037: unable to obtain file status

What is the cause of the ORA-31693 error?

**Answer:**  You can use the oerr utility to look-up the ORA-31693 error:

**ORA-31617:** unable to open dump file "xxx" for write

**Cause:** Export was unable to open the export file for writing. This message is usually followed by device messages from the operating system.

**Action:**  Take appropriate action to restore the device.

The root cause of the ORA-31617 is likely in RAC is that this error occurs when parallel parameter is specified in using expdp in RAC databases  and same physical directories does not exists in all nodes.

This is a bug in 11g and it is fixed in 11by setting the "cluster" parameter cluster=N :