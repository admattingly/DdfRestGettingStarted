# Getting Started with RESTful Web Services Support in the Distributed Data Facility of DB2 for z/OS
_The easiest way on the planet to create RESTful Web Services!_
## Overview
This recipe walks you through a minimal configuration of the Distributed Data Facility (DDF) of DB2 for z/OS V11 or V12 to support the provision of RESTful Web Services. A simple Web Service is created and tested using the UNIX cURL command.
## Ingredients
- DB2 for z/OS version 12, or DB2 for z/OS version 11 with APAR PI66828 (Note: update APAR PI70477 now available)
- Access to any UNIX environment with the "cURL" command installed (doesn't have to be z/OS UNIX)

A complete description of the process for configuring DDF and managing RESTful services can be found in the DB2 Knowledge Center:
http://www.ibm.com/support/knowledgecenter/SSEPEK_11.0.0/restserv/src/tpc/db2z_restservices.html
## Step-by-step
### 1. Prepare DDF to receive RESTful services
After you install DB2 V12 or install APAR `PI66828` on DB2 V11, you must customise and submit DB2 installation job, `DSNTIJRS`.

This job creates the table, `SYSIBM.DSNSERVICE`, which holds the RESTful services that DDF will provide.

Once this job executes successfully, stop and restart DDF.  **Note that this action will disconnect any applications which access your DB2 subsystem via DDF.**

For example, for a DB2 subsystem with command prefix, -DBBG, we issue MVS operator commands:

<pre><b>-DBBG STOP DDF</b> 
DSNL021I -DBBG STOP DDF COMMAND ACCEPTED
DSNL005I -DBBG DDF IS STOPPING 
DSNL006I -DBBG DDF STOP COMPLETE 
<b>-DBBG START DDF</b> 
DSNL021I -DBBG START DDF COMMAND ACCEPTED 
DSNL003I -DBBG DDF IS STARTING 
DSNL519I -DBBG DSNLILNR TCP/IP SERVICES AVAILABLE 474 
 FOR DOMAIN S0W1.DAL-EBIS.IHOST.COM AND PORT 5035
DSNL004I -DBBG DDF START COMPLETE 475 
 LOCATION DALLASB 
 LU NETD.DBBGLU1 
 GENERICLU -NONE 
 DOMAIN S0W1.DAL-EBIS.IHOST.COM 
 TCPPORT 5035 
 SECPORT 5037 
 RESPORT 5036 
 IPNAME -NONE 
 OPTIONS: 
 PKGREL = COMMIT 
DSNL519I -DBBG DSNLIRSY TCP/IP SERVICES AVAILABLE 476 
 FOR DOMAIN S0W1.DAL-EBIS.IHOST.COM AND PORT 5036</pre>

### 2. Authorise users to access DDF RESTful services
DDF will test access to profile <subsys>.REST in the DSNR class to determine whether a user can invoke RESTful services.  So, to permit users to access these services, permit READ access to this profile through your SAF security provider.


