# 1. VMWARE Installation


# 2. WINDOWS Installation
- Install windows without network configuration
- Setup the network with NAT
- ipconfig and Update hosts file


# 3. JAVA Installation 
- Install jdk inside c:/u01/java27

Set User Variable
```
JAVA_HOME: C:\U01\app\JAVA27 
```

Set System Variable
```
C:\u01\app\JAVA27
```

# 4. MAVEN INSTALLATION

Set User Variable
```
MAVEN_HOME: c:\u01\app\apache-maven-3.10.1-bin\apache-maven-3.10.1\bin
```

Set System Variable
```
%MAVEN_HOME%\bin
```



#5. WINRAR Installation
- Install winrar


# 6. MYSQL INSTALLATION

- Install mysql 5.7.17
- Set path for bin in system variables

Connect to mysql
```
> mysql -u root -p
```

Change root password
```
> alter user 'root'@'localhost' identified by 'root';
```

Run this command
```
> install plugin validate_password soname 'validate_password.dll';
```

Run this command and check if installations are properly completed.
```
> select plugin_name, plugin_status from information_schema.plugins where plugin_name like 'validate%'
```

```
> show variables like 'validate_password%';
```

```
> set global validate_password_policy=LOW;
```

```
> set global validate_password_length=6;
```

```
> set global validate_password_number_count=0;
```

Shutdown
```
> mysqladmin -u root -p shutdown
```


# 7. OPENIDM INSTALLATION
1. Download the .zip file from github.com/OpenIdentityPlatform/OpenIDM/releases
2. Download file: openidm:7.1.2.zip. (DON'T DOWNLOAD Source code option).
3. Update file boot.properties

Set User Variable
```
OPENIDM_HOME: C:\U01\app\openidm 
```

Set System Variable
```
C:\u01\app\openidm
```

Start OpenIDM
```
> startup.bat
```
USERNAME: openidm-admin
PASSWORD: openidm-admin

Stop OpenIDM (done from felix console)
```
> shutdown
```


# 8. Task: Load the OpenIDM schema into mysql database

Step 1. Loading a schema creates a user in openidm.

Step 2. Before loading a schema, we can change the default username/password in  openidm/db/mysql/conf/datasource.jdbc-default.json file.

Step 3. SQL Scripts are available in openidm/db/mysql/scripts/ folder

Step 4. Execute below 6 SQL scripts.

Step 5. Open the command prompt in folder openidm/db/mysql/scripts

```
> mysql -u root -p
mysql> source openidm.sql
mysql> source activiti.mysql.create.engine.sql
mysql> source activiti.mysql.create.history.sql
mysql> source activiti.mysql.create.identity.sql
mysql> show tables
```



# 9. Task: Switch your repository to MYSQL

Step 1. Locate openidm/conf/repo.orientdb.json and delete.
   
Step 2. Copy the MySQL configuration files
- Locate datasource.jdbc-default.json and repo.jdbc.json inside folder openidm/db/mysql/conf/.
- Copy both files and paste it inside openidm/conf

Step 3. Add the MySQL JDBC driver
- Download mysql-connector-java-8.0.16.jar and place it inside openidm/bundle/

Step 4. Locate file openidm/conf/boot/boot.properties and open it. Validate below lines.
```   
   openidm.repo.host = localhost
   openidm.repo.port = 3306
```

Step 5. Delete felix-cache and Start OpenIDM
```   
   openidm > startup.bat
```

Step 6. Download sqldeveloper tool and unzip
https://www.oracle.com/in/database/sqldeveloper/technologies/download
```
sqldeveloper-26.2.0.186.2220-x64.zip
```

Step 7. Configure MySQL into sqldeveloper tool
- Double click sqldeveloper
- Open tools > preperences > Database > Third Party JDBC Driver > Add Entry
- Select mysql-connector-j-26.10.0.jar folder from C:\u01\openidm\bundle
- click ok
- Click + sign in connections
- Select Database Type as MySQL
- Choose Database openidm
- Click Connect




