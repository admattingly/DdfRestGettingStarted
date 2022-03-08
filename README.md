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
DDF will test access to profile _subsys_.REST in the DSNR class to determine whether a user can invoke RESTful services.  So, to permit users to access these services, permit READ access to this profile through your SAF security provider.

As a RACF example, to assign IBMUSER as the profile owner and permit access to users ADCDA and ADCDB to RESTful services provided by DB2 subsystem DBBG, you would issue the following TSO commands:

<pre><b>RDEFINE DSNR DBBG.REST OWNER(IBMUSER) UACC(NONE)
PERMIT DBBG.REST CLASS(DSNR) ID(ADCDA) ACCESS(READ)
PERMIT DBBG.REST CLASS(DSNR) ID(ADCDB) ACCESS(READ)
SETROPTS RACLIST(DSNR) REFRESH</b></pre>

### 3. Test access to DDF restful services
Use the cURL command in UNIX to attempt to list the available services from DDF.

In the example below, DDF is listening on port `5035` at IP address `192.168.0.61`. 

<pre>$ </pre><pre><code><b>curl -s -u ADCDA:****** -H "Accept: application/json" http://192.168.0.61:5035/services</b></code></pre>
<pre>{
  "DB2Services":[
    {
      "ServiceName":"DB2ServiceDiscover",
      "ServiceCollectionID":null,
      "ServiceDescription":"DB2 service to list all available services.",
      "ServiceProvider":"db2service-1.0",
      "ServiceURL":"http:&#92;/&#92;/192.168.0.61:5035&#92;/services&#92;/DB2ServiceDiscover"
    },
    {
      "ServiceName":"DB2ServiceManager",
      "ServiceCollectionID":null,
      "ServiceDescription":"DB2 service to create, drop, or alter a user defined service.",
      "ServiceProvider":"db2service-1.0",
      "ServiceURL":"http:&#92;/&#92;/192.168.0.61:5035&#92;/services&#92;/DB2ServiceManager"
    }
  ]}</pre>

  Note that the example above has been _formatted_ by hand so the JSON response is more legible – cURL displays the response as a continuous character stream without any formatting.