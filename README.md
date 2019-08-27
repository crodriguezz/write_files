# write_files
write files pl 
link 1

https://docs.oracle.com/cd/E18283_01/appdev.112/e16760/d_ftran.htm#i999064


https://community.oracle.com/thread/3999444?start=15&tstart=0


Microsoft Windows [Version 6.1.7601]
c:\>dir c:\temp\another_new_dir

Volume in drive C is OSDisk
Volume Serial Number is 38A0-49EF 

Directory of c:\temp
File Not Found 

c:\>


2. Client is Linux:
$ uname

Linux
$ sqlplus scott@sol11

 

SQL*Plus: Release 12.1.0.2.0 Production on Thu Dec 15 08:13:14 2016
Copyright (c) 1982, 2014, Oracle.  All rights reserved.
Enter password:
Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options

SQL> set serveroutput on
SQL> -- output zero indicates success
SQL> exec dbms_output.put_line(OSCommand_Run('mkdir c:\temp\another_new_dir'));
0

 

PL/SQL procedure successfully completed.
SQL>

 

3. Now check MS Windows for directory:
c:\>ver

 Microsoft Windows [Version 6.1.7601]

 

c:\>dir c:\temp\another_new_dir
Volume in drive C is OSDisk
Volume Serial Number is 38A0-49EF

 
Directory of c:\temp\another_new_dir

12/15/2016  08:14 AM    <DIR>          .
12/15/2016  08:14 AM    <DIR>          ..

              0 File(s)              0 bytes
              2 Dir(s)  101,635,698,688 bytes free

c:\>

 

SY.



######lelel
https://www.oracle.com/technical-resources/articles/it-infrastructure/admin-linux-from-database.html

    Create a directory in which you'll place a shell script wrapping the command above:

    $ mkdir /home/oracle/ext_tbl_dir

    2. In the directory, create the ex_ls.sh shell script with the following content:

    #!/bin/bash
    /bin/ls /mnt/dbfs/staging_area -a -l -X

    3. Allow execution of the ex_ls.sh script:

    # chmod +x /home/oracle/ext_tbl_dir/ex_ls.sh

    4. Try to execute the script to make sure it works as expected at this stage:

    $ /home/oracle/ext_tbl_dir/ex_ls.sh

    5. Launch a SQLPlus session:

    CONNECT  /AS SYSDBA;

    6. Create the external table as follows:

        a. Specify an alias in the database for the directory where the external table data file (the ex_ls.sh script, in this particular example) is located.

        CREATE OR REPLACE DIRECTORY my_dir AS '/home/oracle/ext_tbl_dir';

        b. Grant the necessary privileges on the files in the my_dir directory to the dbfs_user1 schema (here, you're using the same user to interact with the DBFS file system and the external table):

        GRANT READ,WRITE,EXECUTE ON DIRECTORY my_dir TO dbfs_user1;

        c. Now, connect as the dbfs_user1 user:

        CONNECT dbfs_user1/pswd

        d. Create the external table, as shown in Listing 1:

        CREATE TABLE ls_tbl(
                   file_priv     VARCHAR2(11),
                   file_links    NUMBER,
                   file_owner    VARCHAR2(25),
                   file_owner_gr VARCHAR2(25),
                   file_sz       NUMBER,
                   file_month    VARCHAR2(3),
                   file_day      NUMBER,
                   file_tm       VARCHAR2(6), 
                   file_nm       VARCHAR2(25))
           ORGANIZATION EXTERNAL (
               TYPE ORACLE_LOADER
               DEFAULT DIRECTORY my_dir
               ACCESS PARAMETERS (
                   records delimited by newline
                   preprocessor my_dir:'ex_ls.sh'
                   skip 1
                   fields terminated by whitespace ldrtrim 
                 ) 
                 LOCATION(my_dir:'ex_ls.sh')
               )
        /

        Listing 1

After completing the above steps, you can issue a query against the ls_tbl external table to make sure that everything is OK so far:

SELECT file_nm "file", file_priv "privileges", file_owner "owner", file_sz "size" FROM ls_tbl WHERE file_sz>0;

The output generated might look like this:

file                 privileges  owner                 size
-------------------- ----------- --------------- ----------
test01.bmp           -rw-r--r--  oracle                1110
test02.bmp           -rwxrwxrwx  oracle                1398
test01.java          -r--r-----  oracle                1765
test01.jpg           -rw-r--r--  oracle               71962
test02.jpg           -rwxrwxrwx  oracle               10197
test04.jpg           -rw-r--r--  oracle                9546
test01.xml           -rw-r--r--  oracle                 477

As you can see, the row order specified by the -X parameter of the ls command remains the same in the table query output above. To recap, using -X makes ls sort the outputted files alphabetically by extension. Doing this by means of SQL would be still possible, of course, but a bit tricky.
Joining External Table Data with Database Data

Wherever the data in an external table has come from, you can query it as if it were regular relational data, joining it, if necessary, with some other data that you can obtain from within the database. Turning back to the example in the preceding section, you might want to consolidate the external table data and the data available via the DBFS_CONTENT view.

Schematically, this might look like Figure 2:

Figure 2: Joining the data obtained from an external table and the DBFS_CONTENT view.

The following query provides an example of such a join:

SELECT e.file_nm, e.file_owner, dbms_lob.getlength(d.filedata)  FROM ls_tbl e, dbfs_content
 d WHERE e.file_nm = substr(d.pathname,instr(d.pathname,'/',-1,1)+1,length(d.pathname));

The output generated by the query above should look like this:

FILE_NM              FILE_OWNER      DBMS_LOB.GETLENGTH(D.FILEDATA)
-------------------- --------------- ------------------------------
test01.bmp           oracle                                    1110
test02.bmp           oracle                                    1398
test01.java          oracle                                    1765
test01.jpg           oracle                                   71962
test02.jpg           oracle                                   10197
test04.jpg           oracle                                    9546
.sfs                 root
test01.xml           oracle                                     477

8 rows selected.

The example above illustrates how you might join the output generated by a Linux utility with regular SQL data. Actually, you might use this same approach to join the operating system data to any data available from within the database, including the data dictionary.
Launching Linux Utilities from Within PL/SQL

As mentioned, using the preprocessor feature of an external table is not the only way to launch an operating system utility from within the database. Thus, you might use the CREATE_PROGRAM procedure of DBMS_SCHEDULER to wrap an external executable in a program that might be then called from within the database. Moreover, you could specify some schedule on which the program will be executed automatically.

The following example illustrates how you might define a scheduler program that wraps a shell script that invokes the ls command and saves the output into an external text file. Then, you'll be able to define an external table that derives its data from that text file, thus making the ls output available in the form of relational data. Figure 3 gives a high-level view of how it works.

Figure 3: Launching a Linux utility with the help of DBMS_SCHEDULER.

The following steps walk you through how to create such a scheduler program wrapping a shell script, as well as how to create an external table upon the text file to which the shell script saves its output:

    1. First, create a shell script with the following content (save it as ls_to_file.sh in the /home/oracle/ext_tbl_dir directory).

    #!/bin/bash
    /bin/ls /mnt/dbfs/staging_area -a -l -X  > /home/oracle/ext_tbl_dir/flist.txt

    2. In a SQLPlus session, connect as sysdba:

    CONNECT  /AS SYSDBA;

    3. Issue the PL/SQL block shown in Listing 2 to create a scheduler program and the scheduler job to execute that program:

    BEGIN
      DBMS_SCHEDULER.CREATE_PROGRAM(
        program_name   => 'sys.ls_to_file',
        program_type   => 'executable',
        program_action => '/home/oracle/ext_tbl_dir/ls_to_file.sh',
        enabled        =>  TRUE);

      DBMS_SCHEDULER.CREATE_JOB(
        job_name        => 'sys.ls_job',
        program_name    => 'sys.ls_to_file',
        start_date      => '15-JUL-13 1.00.00AM US/Pacific',
        repeat_interval => 'FREQ=DAILY;BYHOUR=1;BYMINUTE=0',
        enabled         =>  TRUE);
    END;
    /

    Listing 2
    4. Now, manually run the job:

    BEGIN
      DBMS_SCHEDULER.RUN_JOB('sys.ls_job');
    END;
    /


    After this step, you should have the flist.txt file containing the ls_to_file.sh script output.
    5. Define an external table upon the flist.txt file:
        a. Connect as the dbfs_user1 user:

        CONNECT dbfs_user1/pswd

        b. Create the external table, as shown in Listing 3:

        CREATE TABLE ls_ext_tbl(
        file_priv     VARCHAR2(11),
        file_links    NUMBER,
        file_owner    VARCHAR2(25),
        file_owner_gr VARCHAR2(25),
        file_sz       NUMBER,
        file_month    VARCHAR2(3),
        file_day      NUMBER,
        file_tm       VARCHAR2(6), 
        file_nm       VARCHAR2(25))
        ORGANIZATION EXTERNAL (
               TYPE ORACLE_LOADER
               DEFAULT DIRECTORY my_dir
               ACCESS PARAMETERS (
                   records delimited by newline
                   skip 1
                   fields terminated by whitespace ldrtrim 
                 ) 
                 LOCATION('flist.txt')
               )
        /

        Listing 3
    6. To make sure that everything works as expected, issue a query against the newly created external table:

    SELECT count(*) FROM ls_ext_tbl; 

      COUNT(*)
    ----------
            10


    It's important to note that the ls_job job created in Step 3 will be executed automatically on the schedule specified. So, launching the job manually is possible but not required.

Conclusion

This article illustrated Oracle Database features that can be useful for Linux administrators, including how to use external tables to query operating system data and then join that data with the database data. It also covered how to launch Linux utilities from within PL/SQL code, using the DBMS_SCHEDULER package.
About the Author

Yuli Vasiliev is a software developer, freelance author, and consultant currently specializing in open source development, Java technologies, business intelligence (BI), databases, service-oriented architecture (SOA) and, more recently, virtualization. He is the author of a series of books on Oracle technology, the most recent one being Oracle Business Intelligence: An Introduction to Business Analysis and Reporting (Packt, 2010).


#####iif exist
    Create a directory in which you'll place a shell script wrapping the command above:

    $ mkdir /home/oracle/ext_tbl_dir

    2. In the directory, create the ex_ls.sh shell script with the following content:

    #!/bin/bash
    /bin/ls /mnt/dbfs/staging_area -a -l -X

    3. Allow execution of the ex_ls.sh script:

    # chmod +x /home/oracle/ext_tbl_dir/ex_ls.sh

    4. Try to execute the script to make sure it works as expected at this stage:

    $ /home/oracle/ext_tbl_dir/ex_ls.sh

    5. Launch a SQLPlus session:

    CONNECT  /AS SYSDBA;

    6. Create the external table as follows:

        a. Specify an alias in the database for the directory where the external table data file (the ex_ls.sh script, in this particular example) is located.

        CREATE OR REPLACE DIRECTORY my_dir AS '/home/oracle/ext_tbl_dir';

        b. Grant the necessary privileges on the files in the my_dir directory to the dbfs_user1 schema (here, you're using the same user to interact with the DBFS file system and the external table):

        GRANT READ,WRITE,EXECUTE ON DIRECTORY my_dir TO dbfs_user1;

        c. Now, connect as the dbfs_user1 user:

        CONNECT dbfs_user1/pswd

        d. Create the external table, as shown in Listing 1:

        CREATE TABLE ls_tbl(
                   file_priv     VARCHAR2(11),
                   file_links    NUMBER,
                   file_owner    VARCHAR2(25),
                   file_owner_gr VARCHAR2(25),
                   file_sz       NUMBER,
                   file_month    VARCHAR2(3),
                   file_day      NUMBER,
                   file_tm       VARCHAR2(6), 
                   file_nm       VARCHAR2(25))
           ORGANIZATION EXTERNAL (
               TYPE ORACLE_LOADER
               DEFAULT DIRECTORY my_dir
               ACCESS PARAMETERS (
                   records delimited by newline
                   preprocessor my_dir:'ex_ls.sh'
                   skip 1
                   fields terminated by whitespace ldrtrim 
                 ) 
                 LOCATION(my_dir:'ex_ls.sh')
               )
        /

        Listing 1

After completing the above steps, you can issue a query against the ls_tbl external table to make sure that everything is OK so far:

SELECT file_nm "file", file_priv "privileges", file_owner "owner", file_sz "size" FROM ls_tbl WHERE file_sz>0;

The output generated might look like this:

file                 privileges  owner                 size
-------------------- ----------- --------------- ----------
test01.bmp           -rw-r--r--  oracle                1110
test02.bmp           -rwxrwxrwx  oracle                1398
test01.java          -r--r-----  oracle                1765
test01.jpg           -rw-r--r--  oracle               71962
test02.jpg           -rwxrwxrwx  oracle               10197
test04.jpg           -rw-r--r--  oracle                9546
test01.xml           -rw-r--r--  oracle                 477

As you can see, the row order specified by the -X parameter of the ls command remains the same in the table query output above. To recap, using -X makes ls sort the outputted files alphabetically by extension. Doing this by means of SQL would be still possible, of course, but a bit tricky.
Joining External Table Data with Database Data

Wherever the data in an external table has come from, you can query it as if it were regular relational data, joining it, if necessary, with some other data that you can obtain from within the database. Turning back to the example in the preceding section, you might want to consolidate the external table data and the data available via the DBFS_CONTENT view.

Schematically, this might look like Figure 2:

Figure 2: Joining the data obtained from an external table and the DBFS_CONTENT view.

The following query provides an example of such a join:

SELECT e.file_nm, e.file_owner, dbms_lob.getlength(d.filedata)  FROM ls_tbl e, dbfs_content
 d WHERE e.file_nm = substr(d.pathname,instr(d.pathname,'/',-1,1)+1,length(d.pathname));

The output generated by the query above should look like this:

FILE_NM              FILE_OWNER      DBMS_LOB.GETLENGTH(D.FILEDATA)
-------------------- --------------- ------------------------------
test01.bmp           oracle                                    1110
test02.bmp           oracle                                    1398
test01.java          oracle                                    1765
test01.jpg           oracle                                   71962
test02.jpg           oracle                                   10197
test04.jpg           oracle                                    9546
.sfs                 root
test01.xml           oracle                                     477

8 rows selected.

The example above illustrates how you might join the output generated by a Linux utility with regular SQL data. Actually, you might use this same approach to join the operating system data to any data available from within the database, including the data dictionary.
Launching Linux Utilities from Within PL/SQL

As mentioned, using the preprocessor feature of an external table is not the only way to launch an operating system utility from within the database. Thus, you might use the CREATE_PROGRAM procedure of DBMS_SCHEDULER to wrap an external executable in a program that might be then called from within the database. Moreover, you could specify some schedule on which the program will be executed automatically.

The following example illustrates how you might define a scheduler program that wraps a shell script that invokes the ls command and saves the output into an external text file. Then, you'll be able to define an external table that derives its data from that text file, thus making the ls output available in the form of relational data. Figure 3 gives a high-level view of how it works.

Figure 3: Launching a Linux utility with the help of DBMS_SCHEDULER.

The following steps walk you through how to create such a scheduler program wrapping a shell script, as well as how to create an external table upon the text file to which the shell script saves its output:

    1. First, create a shell script with the following content (save it as ls_to_file.sh in the /home/oracle/ext_tbl_dir directory).

    #!/bin/bash
    /bin/ls /mnt/dbfs/staging_area -a -l -X  > /home/oracle/ext_tbl_dir/flist.txt

    2. In a SQLPlus session, connect as sysdba:

    CONNECT  /AS SYSDBA;

    3. Issue the PL/SQL block shown in Listing 2 to create a scheduler program and the scheduler job to execute that program:

    BEGIN
      DBMS_SCHEDULER.CREATE_PROGRAM(
        program_name   => 'sys.ls_to_file',
        program_type   => 'executable',
        program_action => '/home/oracle/ext_tbl_dir/ls_to_file.sh',
        enabled        =>  TRUE);

      DBMS_SCHEDULER.CREATE_JOB(
        job_name        => 'sys.ls_job',
        program_name    => 'sys.ls_to_file',
        start_date      => '15-JUL-13 1.00.00AM US/Pacific',
        repeat_interval => 'FREQ=DAILY;BYHOUR=1;BYMINUTE=0',
        enabled         =>  TRUE);
    END;
    /

    Listing 2
    4. Now, manually run the job:

    BEGIN
      DBMS_SCHEDULER.RUN_JOB('sys.ls_job');
    END;
    /


    After this step, you should have the flist.txt file containing the ls_to_file.sh script output.
    5. Define an external table upon the flist.txt file:
        a. Connect as the dbfs_user1 user:

        CONNECT dbfs_user1/pswd

        b. Create the external table, as shown in Listing 3:

        CREATE TABLE ls_ext_tbl(
        file_priv     VARCHAR2(11),
        file_links    NUMBER,
        file_owner    VARCHAR2(25),
        file_owner_gr VARCHAR2(25),
        file_sz       NUMBER,
        file_month    VARCHAR2(3),
        file_day      NUMBER,
        file_tm       VARCHAR2(6), 
        file_nm       VARCHAR2(25))
        ORGANIZATION EXTERNAL (
               TYPE ORACLE_LOADER
               DEFAULT DIRECTORY my_dir
               ACCESS PARAMETERS (
                   records delimited by newline
                   skip 1
                   fields terminated by whitespace ldrtrim 
                 ) 
                 LOCATION('flist.txt')
               )
        /

        Listing 3
    6. To make sure that everything works as expected, issue a query against the newly created external table:

    SELECT count(*) FROM ls_ext_tbl; 

      COUNT(*)
    ----------
            10


    It's important to note that the ls_job job created in Step 3 will be executed automatically on the schedule specified. So, launching the job manually is possible but not required.

Conclusion

This article illustrated Oracle Database features that can be useful for Linux administrators, including how to use external tables to query operating system data and then join that data with the database data. It also covered how to launch Linux utilities from within PL/SQL code, using the DBMS_SCHEDULER package.
About the Author

Yuli Vasiliev is a software developer, freelance author, and consultant currently specializing in open source development, Java technologies, business intelligence (BI), databases, service-oriented architecture (SOA) and, more recently, virtualization. He is the author of a series of books on Oracle technology, the most recent one being Oracle Business Intelligence: An Introduction to Business Analysis and Reporting (Packt, 2010).

#####


exec dbms_scheduler.create_job('mkdir','executable','/bin/mkdir',1,auto_drop=>false);
exec dbms_scheduler.set_job_argument_value('mkdir',1,'/tmp/newdir');
exec dbms_scheduler.run_job('mkdir');

