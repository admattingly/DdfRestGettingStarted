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

<pre><code><b>curl -s -u ADCDA:****** -H "Accept: application/json" http://192.168.0.61:5035/services</b></code></pre>
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

Use the `-u user:password` option of the cURL command to pass the credentials of a valid z/OS user to DDF. The `-s` option asks cURL not to produce a progress bar.

**In this example, SSL is not being used, but if you have SSL configured for DDF, then it is highly desirable to use SSL for access to DDF RESTful services.**

For guidance on setting up SSL (strictly, AT-TLS) for DDF, consult the following Redpaper:
http://www.redbooks.ibm.com/redpieces/abstracts/redp4799.html?Open

### 4. Create a RESTful service
In this example, we will use the sample table, `DSN8B10.EMP`, but feel free to use any existing table in your DB2 subsystem.

Define a `getEmployee` service to return the results of the following SQL query:

`SELECT * FROM DSN8B10.EMP WHERE EMPNO = :EMPLOYEE_CODE`

To do this we invoke the `DB2ServiceManager` service (shown in the previous step), as follows:

<pre><code><b>curl -s -X POST -u ADCDA:******** -H "Accept: application/json" -H "Content-Type: application/json" --data "{\"requestType\": \"createService\", \"sqlStmt\": \"SELECT * FROM DSN8B10.EMP WHERE EMPNO = :EMPLOYEE_CODE\", \"collectionID\": \"MYCOLL\", \"serviceName\": \"getEmployee\", \"description\": \"Get employee by EMPNO\"}" http://192.168.0.61:5035/services/DB2ServiceManager</b></code></pre>
<pre>{
  "StatusCode":201,
  "StatusDescription":"DB2 Rest Service getEmployee was created successfully.",
  "URL":"http:\/\/192.168.0.61:5035\/services\/MYCOLL\/getEmployee"
}</pre>

Note that the user creating this service requires `BINDADD` privilege and `CREATE IN` privilege for the collection specified in the above command (`MYCOLL` in the above example).  This is because DDF is creating a new package for our service, called `location.MYCOLL.getEmployee`, in this case.

### 5. Execute the RESTful service

Now we can test our service, by invoking it using cURL, as follows:

<pre><code><b>curl -s -X POST -u ADCDA:******** -H "Accept: application/json" -H "Content-Type: application/json" --data "{\"EMPLOYEE_CODE\": \"000010\"}" http://192.168.0.61:5035/services/MYCOLL/getEmployee</b></code></pre>
<pre>{
  "ResultSet Output":[
    {
      "EMPNO":"000010",
      "FIRSTNME":"CHRISTINE",
      "MIDINIT":"I",
      "LASTNAME":"HAAS",
      "WORKDEPT":"A00",
      "PHONENO":"3978",
      "HIREDATE":"1965-01-01",
      "JOB":"PRES ",
      "EDLEVEL":18,
      "SEX":"F",
      "BIRTHDATE":"1933-08-14",
      "SALARY":52750.00,
      "BONUS":1000.00,
      "COMM":4220.00
    }
  ],
  "StatusCode":200,
  "StatusDescription":"Execution Successful"
}</pre>

Again, the above output has been formatted by hand for clarity.

To allow other users to access this service, you need to grant `EXECUTE` authority to the underlying package that DDF created.  For example, execute the DB2 statement below to give user `ADCDB` access to the package:

<pre><code><b>GRANT EXECUTE ON PACKAGE MYCOLL.getEmployee TO ADCDB</b></code></pre>

### 6. (Optional) Discover your service
You, or others, can discover how to invoke your service, and what output to expect from it, by invoking the service using a HTTP `GET` request with no parameters.

Issue the following cURL command to discover how to invoke your service:

<pre><code><b>curl -s -u ADCDA:******** -H "Accept: application/json" http://192.168.0.61:5035/services/MYCOLL/getEmployee</b></code></pre>
<pre>{
  "getEmployee":{
    "serviceName":"getEmployee",
    "serviceCollectionID":"MYCOLL",
    "serviceProvider":"db2service-1.0",
    "serviceDescription":"Get employee by EMPNO",
    "serviceURL":"http:\/\/192.168.0.61:5035\/services\/MYCOLL\/getEmployee",
    "serviceStatus":"started",
    "RequestSchema":{
      "$schema":"http:\/\/json-schema.org\/draft-04\/schema#",
      "type":"object",
      "properties":{
        "EMPLOYEE_CODE":{
          "type":[
            "null",
            "string"
          ],
          "maxLength":6,
          "description":"Nullable CHAR(6)"
        }
      },
      "required":[
        "EMPLOYEE_CODE"
      ],
      "description":"Service getEmployee invocation HTTP request body"
    },
    "ResponseSchema":{
      "$schema":"http:\/\/json-schema.org\/draft-04\/schema#",
      "type":"object",
      "properties":{
        "ResultSet Output":{
          "type":"array",
          "items":{
            "type":"object",
            "properties":{
              "EMPNO":{
                "type":"string",
                "maxLength":6,
                "description":"CHAR(6)"
              },
              "FIRSTNME":{
                "type":"string",
                "maxLength":12,
                "description":"VARCHAR(12)"
              },
              "MIDINIT":{
                "type":"string",
                "maxLength":1,
                "description":"CHAR(1)"
              },
              "LASTNAME":{
                "type":"string",
                "maxLength":15,
                "description":"VARCHAR(15)"
              },
              "WORKDEPT":{
                "type":[
                  "null",
                  "string"
                ],
                "maxLength":3,
                "description":"Nullable CHAR(3)"
              },
              "PHONENO":{
                "type":[
                  "null",
                  "string"
                ],
                "maxLength":4,
                "description":"Nullable CHAR(4)"
              },
              "HIREDATE":{
                "type":[
                  "null",
                  "string"
                ],
                "minLength":8,
                "maxLength":10,
                "pattern":"^(?![0]{4})([0-9]{4})-(0?[1-9]|1[0-2])-(0?[1-9]|[1-2][0-9]|3[0-1])$",
                "description":"Nullable DATE yyyy-[m]m-[d]d"
              },
              "JOB":{
                "type":[
                  "null",
                  "string"
                ],
                "maxLength":8,
                "description":"Nullable CHAR(8)"
              },
              "EDLEVEL":{
                "type":[
                  "null",
                  "integer"
                ],
                "multipleOf":1,
                "minimum":-32768,
                "maximum":32767,
                "description":"Nullable SMALLINT"
              },
              "SEX":{
                "type":[
                  "null",
                  "string"
                ],
                "maxLength":1,
                "description":"Nullable CHAR(1)"
              },
              "BIRTHDATE":{
                "type":[
                  "null",
                  "string"
                ],
                "minLength":8,
                "maxLength":10,
                "pattern":"^(?![0]{4})([0-9]{4})-(0?[1-9]|1[0-2])-(0?[1-9]|[1-2][0-9]|3[0-1])$",
                "description":"Nullable DATE yyyy-[m]m-[d]d"
              },
              "SALARY":{
                "type":[
                  "null",
                  "number"
                ],
                "multipleOf":0.01,
                "minimum":-999999.99,
                "maximum":999999.99,
                "description":"Nullable DECIMAL(9,2)"
              },
              "BONUS":{
                "type":[
                  "null",
                  "number"
                ],
                "multipleOf":0.01,
                "minimum":-999999.99,
                "maximum":999999.99,
                "description":"Nullable DECIMAL(9,2)"
              },
              "COMM":{
                "type":[
                  "null",
                  "number"
                ],
                "multipleOf":0.01,
                "minimum":-999999.99,
                "maximum":999999.99,
                "description":"Nullable DECIMAL(9,2)"
              }
            },
            "required":[
              "EMPNO",
              "FIRSTNME",
              "MIDINIT",
              "LASTNAME",
              "WORKDEPT",
              "PHONENO",
              "HIREDATE",
              "JOB",
              "EDLEVEL",
              "SEX",
              "BIRTHDATE",
              "SALARY",
              "BONUS",
              "COMM"
            ],
            "description":"ResultSet Row"
          }
        },
        "StatusDescription":{
          "type":"string",
          "description":"Service invocation status description"
        },
        "StatusCode":{
          "type":"integer",
          "multipleOf":1,
          "minimum":100,
          "maximum":600,
          "description":"Service invocation HTTP status code"
        }
      },
      "required":[
        "StatusDescription",
        "StatusCode",
        "ResultSet Output"
      ],
      "description":"Service getEmployee invocation HTTP response body"
    }
  }
}</pre>

### 7. (Optional) Delete your service

To remove your service from DDF, issue the following cURL command:

<pre><code><b>curl -s -X POST -u ADCDA:******** -H "Accept: application/json" -H "Content-Type: application/json" --data "{\"requestType\": \"dropService\", \"collectionID\": \"MYCOLL\", \"serviceName\": \"getEmployee\"}" http://192.168.0.61:5035/services/DB2ServiceManager</b></code></pre>
<pre>{
  "StatusCode":200,
  "StatusDescription":"DB2 Rest Service getEmployee was dropped successfully."
}</pre>