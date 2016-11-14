# Database configuration 

## Database usage

#### Main concepts

Bonita BPM uses an RDBMS (Relational DataBase Management System) for the following three purposes:
 - One database schema is required by Bonita BPM Engine to store information about deployed process definitions, process configurations, history of process execution, users, and such. 
- Starting with Bonita BPM 7.3.0, the same database is also used to store Bonita BPM Platform configuration information, in dedicated tables.
- Business data are stored in a database connected to Bonita BPM. We recommend that you configure a different database scheme for this purpose.

Connections to the databases are done through the Hibernate library (version 4.2).  
This provides a level of abstraction between the engine and the RDBMS.  

Bonita BPM comes with a set of optimized initialization scripts for the [supported RDBMSs](https://customer.bonitasoft.com/support-policies).

#### Default H2 database

Bonita BPM Studio, the WildFly bundle, the Tomcat bundle, and the deploy bundle come by default with an embedded H2 RDBMS. The database is automatically created at first startup, an can be used for all three purposes described earlier.

However, H2 is only suitable for testing purposes. When building a production environment, you must switch to another RDBMS.  

#### Steps to switch to another RDBMS

As suggested earlier, if you use the BDM (Business Data Model) feature to store business data related to the scope of your applications and processes, you will need to configure two databases: one for Bonita BPM Platform configuration data and engine data, and one for business data.

Here are the steps to follow:
    1. Create the database(s)
    2. Customize RDBMS to make it work with Bonita BPM    
    3. Add the JDBC driver to the bundle
    4. Configure the bundle with database information 
    
::: warning
All RDBMSs require specific customization, which must be done before you complete your installation.  
If you do not complete the specific configuration for your RDBMS, your installation may fail.
:::

::: warning
There are known issues with the management of XA transactions by MySQL engine and driver: see MySQL bugs [17343](http://bugs.mysql.com/bug.php?id=17343) and [12161](http://bugs.mysql.com/bug.php?id=12161) for more details.  
Therefore, using MySQL database in a production environment is not recommended.
:::

::: warning
There is another known issue between Bitronix (the Transaction Manager shipped by Bonitasoft for the Tomcat bundle & inside Deploy bundle for Tomcat) and Microsoft SQL Server driver (refer to: [MSDN note](https://msdn.microsoft.com/en-us/library/aa342335.aspx), [Bitronix note](http://bitronix-transaction-manager.10986.n7.nabble.com/Failed-to-recover-SQL-Server-Restart-td148.html)).  
Therefore, using Bitronix as a Transaction Manager with SQL Server is not recommended.  
We recommend that you use the WildFly bundle provided by Bonitasoft.
:::


<a id="database_creation" />

## Create the database(s)

The first step in configuring Bonita BPM with your RDBMS is to create one or two new database(s) (i.e. new schema(s)).

To do so, you need a RDBMS user account that has sufficient privileges (i.e. privileges to create new schema).

Also, note that the owner of the new schemas must own the following privileges:

* CREATE TABLE
* CREATE INDEX
* SELECT, INSERT, UPDATE, DELETE on created TABLE

:::info
CREATE TABLE and CREATE INDEX are not required after first start in normal use.  
If the same SQL user is used with the [migration tool](migrate-from-an-earlier-version-of-bonita-bpm.md), then this user needs such grants.
:::

To create the database(s), we recommend that you refer to your RDBMS documentation:

* [PostgreSQL](http://www.postgresql.org/docs/9.3/static/app-createdb.html)
* [Oracle database](https://docs.oracle.com/cd/E11882_01/server.112/e25494/create.htm#ADMIN002)
* [SQL Server](https://technet.microsoft.com/en-us/library/dd207005(v=sql.110).aspx)
* [MySQL](http://dev.mysql.com/doc/refman/5.5/en/database-use.html)

Your database(s) must be configured to use the UTF-8 character set. 
Also, you are recommended to configure the database(s) to be case-insensitive so that searches in Bonita BPM Portal are case-insensitive.

<a id="specific_database_configuration" />

## Customize RDBMS to make it work with Bonita BPM    


### PostgreSQL

Configure the database to use UTF-8\.

Edit `postgresql.conf` and set a non-zero value for `max_prepared_transactions`. The default value, 0, disables prepared transactions, which is not recommended for Bonita BPM Engine.  
The value should be at least as large as the value set for `max_connections` (default is typically 100).  
See the [PostgreSQL documentation](https://www.postgresql.org/docs/9.3/static/runtime-config-resource.html#GUC-MAX-PREPARED-TRANSACTIONS) for details.

### Oracle Database

Make sure your database is configured to use the AL32UTF8 character set.  
If your database already exists, see the Oracle documentation for details of how to [migrate the character set](http://docs.oracle.com/cd/E11882_01/server.112/e10729/ch11charsetmig.htm#NLSPG011).

Bonita BPM Engine uses datasources that handle global transactions that span resources (XADataSource), so the Oracle user used by Bonita BPM Engine, requires some specific privileges, and there are also specific settings for XA activation.

##### **Important information for a successful connection**

The procedure below is used to create the settings to enable Bonita BPM Engine to connect to the Oracle database.

It is assumed in the procedure that:

* Oracle product is already installed and running
* An 'Oracle' OS user is already created
* A database already exists
* The environment is already set:
  ```
  ORACLE_HOME=/u01/app/oracle/product/11.2.0/dbhome_1
  ORACLE_SID=...
  ```

1. Connect to the database as the System Administrator.
   ```bash
   oracle@ubuntu:~$ sqlplus / as sysdba
   ```

2. Check that the following components exist and are valid:  
   SQL query \>  
   ```sql
   select comp_name, version, status from dba_registry;
   ```  

   | Comp\_name | Version | Status |
   |:-|:-|:-|
   | Oracle Database Catalog Views | 11.2.0.1.0 | VALID |
   | Oracle Database Packages and Types | 11.2.0.1.0 | VALID |
   | JServer JAVA Virtual Machine | 11.2.0.1.0 | VALID |
   | Oracle XDK | 11.2.0.1.0 | VALID |
   | Oracle Database Java Packages | 11.2.0.1.0 | VALID |

3. Add XA elements:

   SQL query \>
   ```sql
   @/u01/app/oracle/product/11.2.0/dbhome_1/javavm/install/initxa.sql
   ```
   This only needs to be done once, after the installation of Oracle.

4. Create the database user to be used by the Bonita BPM Engine and grant the required rights:

   SQL query \>
   ```sql
   @/u01/app/oracle/product/11.2.0/dbhome_1/rdbms/admin/xaview.sql
   ```
   The following queries must be done for each new user: i.e. one user = one database schema.

   SQL query \>
   ```sql
   CREATE USER bonita IDENTIFIED BY bonita;
   GRANT connect, resource TO bonita IDENTIFIED BY bonita;
   GRANT select ON sys.dba_pending_transactions TO bonita;
   GRANT select ON sys.pending_trans$ TO bonita;
   GRANT select ON sys.dba_2pc_pending TO bonita;
   GRANT execute ON sys.dbms_system TO bonita;
   GRANT select ON sys.v$xatrans$ TO bonita;
   GRANT execute ON sys.dbms_xa TO bonita;
   GRANT FORCE ANY TRANSACTION TO bonita;
   ```

#### SQL Server

::: warning
There is a known issue between Bitronix (the Transaction Manager shipped by Bonitasoft in the Tomcat bundle and in the Tomcat directories of the Deploy bundle) and the Microsoft SQL Server driver
(refer to: [MSDN note](https://msdn.microsoft.com/en-us/library/aa342335.aspx), [Bitronix note](http://bitronix-transaction-manager.10986.n7.nabble.com/Failed-to-recover-SQL-Server-Restart-td148.html)).
Therefore, using Bitronix as a Transaction Manager with SQL Server is not recommended. Our recommendation is to use the WildFly bundle provided by Bonitasoft.
:::

##### XA Transactions

To support XA transactions, SQL Server requires a specific configuration.  
You can refer to [MSDN](https://msdn.microsoft.com/en-us/library/aa342335(v=sql.110).aspx) for more information.  
Here is the list of steps to perform (as an example, the database name BONITA\_BPM is used):

1. Make sure you have already downloaded and installed the [Microsoft SQL Server JDBC Driver 4.0](https://www.microsoft.com/en-us/download/details.aspx?displaylang=en&id=11774).
2. Copy the `sqljdbc_xa.dll` from `%JDBC_DRIVER_INSTALL_ROOT%\sqljdbc_4.0\enu\xa\x64\` (x64 for 64 bit version of Windows, x86 for 32 bit version of Windows) to `%SQLSERVER_INSTALL_ROO%\Instance_root\MSSQL10.MSSQLSERVER\MSSQL\Binn\.`
3. Copy/paste the content of xa\_install.sql file (located in %JDBC\_DRIVER\_INSTALL\_ROOT%\\sqljdbc\_4.0\\enu\\xa) to SQL Server Management Studio's Query Editor.
4. Execute the query in the Query Editor.
5. To confirm successful execution of the script, open the "Object Explorer" and go to: **Master** \> **Programmability** \> **Extended Stored Procedures**.  
   You should have 12 new procedures, each with a name starting with `dbo.xp.sqljdbc_xa_`.
6. Assign the new role 'SqlJDBCXAUser' to the user who owns the Bonita BPM Engine database (`bonitadev` in our example). To do this, execute the following commands in SQL editor:
   ```sql
   USE master;
   GO
   CREATE LOGIN bonitadev WITH PASSWORD = 'secret_password';
   GO
   CREATE USER bonitadev FOR LOGIN bonitadev;
   GO
   EXEC sp_addrolemember [SqlJDBCXAUser], 'bonitadev';
   GO
   ```

7. In the Windows "Start" menu, select **Administrative Tools**-\> **Services**.
8. In the "Services" window, make sure that the **Distributed Transaction Coordinator** service is set to start automatically. If it's not yet started, start it.
9. Make sure that the other services it depends on, namely "Remote Procedure Call" and "Security Accounts Manager", are also set to start automatically.
10. Run the `dcomcnfg` command, or go to the "Start" menu, then Administrative Tools \> Component Services.
11. In the left navigation pane, navigate to **Component Services** \> **Computers** \> **My Computer** \> **Distributed Transaction Coordinator**.
12. Select and right-click on _**Local DTC**_ and then _**Properties**_.
13. Click on _**Security**_ tab. Ensure that the checkbox for **Enable XA Transactions** is checked.
14. Click _**Apply**_, then click _**OK**_
15. Then stop and restart SQLServer.
16. Create the BONITA\_BPM database: `CREATE DATABASE BONITA_BPM GO`.
17. Set `bonitadev` as owner of BONITA\_BPM database (use, for example, 'Microsoft SQL Management Studio')

##### Recommended configuration for lock management

Run the script below to avoid deadlocks:

```sql
ALTER DATABASE BONITA_BPM SET SINGLE_USER WITH ROLLBACK IMMEDIATE
ALTER DATABASE BONITA_BPM SET ALLOW_SNAPSHOT_ISOLATION ON
ALTER DATABASE BONITA_BPM SET READ_COMMITTED_SNAPSHOT ON
ALTER DATABASE BONITA_BPM SET MULTI_USER
```

See [MSDN](https://msdn.microsoft.com/en-us/library/ms175095(v=sql.110).aspx).

#### MySQL

##### Maximum packet size

MySQL defines a maximum packet size on the server side. The default value for this settings are appropriate for most standard use cases.
However, you need to increase the packet size if you see the following error:
`Error: 1153 SQLSTATE: 08S01 (ER_NET_PACKET_TOO_LARGE) Message: Got a packet bigger than 'max_allowed_packet' bytes`

You need to update the file `my.ini` (for Windows) or `my.cnf` (for Linux) to avoid the `ER_NET_PACKET_TOO_LARGE` problem.
Look for `max_allowed_packet` settings and reduce the value.

For more information, see the [MySQL website](http://dev.mysql.com/doc/refman/5.5/en/packet-too-large.html).

##### Surrogate characters not supported

MySQL does not support [surrogate characters](https://en.wikipedia.org/wiki/Universal_Character_Set_characters#Surrogates).
If you want to use surrogate characters in your processes, you need to use another type of database.

## Add the JDBC driver to the bundle

#### Download JDBC driver

First, you need to download the JDBC driver for your database system. Use the links below to download the driver.

| Database vendor | Download link |
| :- | :- |
| PostgreSQL (use "Current Version") | [download](https://jdbc.postgresql.org/download.html#current) |
| Oracle Database | [download](http://www.oracle.com/technetwork/database/features/jdbc/index-091264.html) |
| Microsoft SQL Server | [download](http://go.microsoft.com/fwlink/?LinkId=245496) |
| MySQL | [download](http://dev.mysql.com/downloads/connector/j/) |

**Note:** If you run on Linux, the JDBC driver might also be available in the distribution packages repository. On Ubuntu and Debian, you can, for example, install the `libpostgresql-jdbc-java` package to get the PostgreSQL JDBC Driver (install in `/usr/share/java`).

<a id="jdbc_driver"/>

#### Add JDBC driver to application server

The way to install the JDBC driver depends on the application server:

##### WildFly 10

WildFly 10 manages JDBC drivers as modules, so to add a new JDBC driver, complete these steps:
(see [WildFly documentation](https://docs.jboss.org/author/display/WFLY10/DataSource+configuration) for full reference):

* Create a folder structure under `<WILDFLY_HOME>/modules` folder.  
  Refer to the table below to identify the folders to create.  
  The last folder is named `main` for all JDBC drivers.
* Add the JDBC driver jar file to the `main` folder.
* Create a module description file `module.xml` in `main` folder.

| Database vendor | Module folders | Module description file |
| :- | :- | :- |
| PostgreSQL | modules/org/postgresql/main | [module.xml](images/special_code/postgresql/module.xml) |
| Oracle | modules/com/oracle/main | [module.xml](images/special_code/oracle/module.xml) |
| SQL Server | modules/com/sqlserver/main | [module.xml](images/special_code/sqlserver/module.xml) |
| MySQL | modules/com/mysql/main | [module.xml](images/special_code/mysql/module.xml) |

Put the driver jar file in the relevant `main` folder.

In the same folder as the driver, add the module description file, `module.xml`.
This file describes the dependencies the module has and the content it exports.
It must describe the driver jar and the JVM packages that WildFly 10 does not provide automatically.
The exact details of what must be included depend of the driver jar.
**Warning:** You might need to edit the `module.xml` in order to match exactly the JDBC driver jar file name.

::: info  
**Note:** By default, when WildFly, it removes any comments from `standalone/configuration/standalone.xml` and formats the file.
If you need to retrieve the previous version of this file, go to `standalone/configuration/standalone_xml_history`.
:::

#### Tomcat 7

For Tomcat, simply add the JDBC driver jar file in the appropriate folder:

* Bonita BPM Tomcat bundle: in the bundle folder add the driver to the `lib/bonita` folder.
* Bonita BPM deploy bundle: in the Tomcat folder add the driver to the `lib` folder.
* Ubuntu/Debian package: add the driver to `/usr/share/tomcat7/lib`.
* Windows as a service: add the driver to `C:\Program Files\Apache Software Foundation\Tomcat 7.0\lib`

## Configure the bundle with database information 

Go to the bundle setup/database.properties file and edit the values, that handle both engine/platform configuration and business data.

The table below shows the match between the database vendor you want to use and the property to add in the file: 

| Database vendor | Property value |  
| :- | :- |  
| PostgreSQL | postgres |
| Oracle database | oracle |
| SQL Server | sqlserver |
| MySQL | mysql |
| H2 (default for testing, not for production) | H2 |

</div></div>

As an example, if you want to use PostgreSQL, the line will be:
`db.vendor=postgres`

<!-- [Nath must add all locations to change] 

#########################################
# Bonita BPM internal database properties
#########################################

# valid values are (h2, postgres, sqlserver, oracle, mysql)
db.vendor=h2
# when using h2, no server or port setting is needed since connexion is made using file protocol mode using relative directory.
# db.server.name=localhost
# db.server.port=5432
db.database.name=bonita_journal.db
db.user=sa
db.password=

###################################
# Business Data database properties
###################################
# valid values are (h2, postgres, sqlserver, oracle, mysql)
bdm.db.vendor=h2
# bdm.db.server.name=localhost
# bdm.db.server.port=5432
bdm.db.database.name=business_data.db
bdm.db.user=sa
bdm.db.password=


# IMPORTANT NOTE regarding H2 database:
# in case you move whole setup folder to another directory, you must change property below
# to point to original folder containing h2 database folder
# new value can be relative or absolute since it still points to the right folder
h2.database.dir=../database
* Address of the RDBMS server
* Port number of the RDBMS server
* Database (schema) name
* User name to connect to the database
* Password to connect to the database
* JDBC Driver fully qualified class name (see table below)
* XADataSource fully qualified class name (see table below)
An alternative to setting the JVM system property (`sysprop.bonita.db.vendor`) is to set `db.vendor` property value in the
[`bonita-platform-community-custom.properties`](BonitaBPM_platform_setup.md) file.
The default value of `db.vendor` indicates that the value of the JVM system property value must be used.
If the property is not defined, the fallback value is H2: `db.vendor=${sysprop.bonita.db.vendor:h2}`
-->

Bonita BPM Engine requires the configuration of two data sources. The data source declaration defines how to connect to the RDBMS. The following information is required to configure the data sources:


<!-- still needed? -->
| Database vendor | Driver class name | XADataSource class name |
|:-|:-|:-|
| PostgreSQL | org.postgresql.Driver | org.postgresql.xa.PGXADataSource |
| Oracle Database | oracle.jdbc.driver.OracleDriver | oracle.jdbc.xa.client.OracleXADataSource |
| Microsoft SQL Server | com.microsoft.sqlserver.jdbc.SQLServerDriver | com.microsoft.sqlserver.jdbc.SQLServerXADataSource |
| MySQL | com.mysql.jdbc.Driver | com.mysql.jdbc.jdbc2.optional.MysqlXADataSource |
| h2 (not for production) | org.h2.Driver | org.h2.jdbcx.JdbcDataSource |

The following sections show how to configure the datasources for WildFly and Tomcat.
There is also an [example of how to configure datasources for Weblogic](red-hat-oracle-jvm-weblogic-oracle.md).

<!-- say earlier that this is the same between Tomcat and WildFly, introduce the templates system, and explain the files that are overwirtten by the templates -->

#### WildFly

This section explains how to configure the data sources if you are using WildFly:

1. Open the file `<WILDFLY_HOME>/standalone/configuration/standalone.xml`.
2. Comment out the default definition for h2\.
3. Uncomment the settings matching your RDBMS vendor.
4. Modify the values for following settings to your configuration: server address, server port, database name, user name and password.

**Note:** For a first test, you might want to keep the H2 section related to Business Data Management (BDM) feature (driver and data sources configuration). You can update the configuration related to BDM later.

#### Tomcat

Configuration of data source for Tomcat is in two parts: because Tomcat doesn't support JTA natively, one data source will be configured in the Bitronix configuration file and the other data source will be configured in the standard Tomcat context configuration file.

##### JTA data source (managed by Bitronix)

1. Open `<TOMCAT_HOME>/conf/bitronix-resources.properties` file.
2. Remove or comment out the lines regarding the h2 database.
3. Uncomment the line matching your RDBMS.
4. Update the value for each of the following settings:
   * For `resource.ds1.driverProperties.user`, put your RDBMS user name.
   * For `resource.ds1.driverProperties.password`, put your RDBMS password.
   * For `resource.ds1.driverProperties.serverName`, put the address (IP or hostname) of your RDBMS server.
   * For `resource.ds1.driverProperties.portNumber`, put the port of your RDBMS server.
   * For `resource.ds1.driverProperties.databaseName`, put the database name.
5. Save and close the file.

##### Non-transactional data source

The second data source run SQL queries outside any transaction. To configure it:

1. Open `<TOMCAT_HOME>/conf/Catalina/localhost/bonita.xml` file.
2. Remove or comment out the lines regarding h2 database.
3. Uncomment the line matching your RDBMS.
4. Update following attributes value:
   * `username`: your RDBMS user name.
   * `password`: your RDBMS password.
   * `url`: the URL, including the RDBMS server address, RDBMS server port and database (schema) name.
