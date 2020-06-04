# Using Edition-Based Redefinition

## Introduction

### Objectives

-    Learn how to use Oracle Database Edition-Based Redefinition to enable online application upgrade with uninterrupted availability of the application.

### Scenario

The standard sample schema HR has an EMPLOYEES table. We’ll pretend that this table has historically been part of an U.S. based company’s application, so the phone numbers are stored in a format that a U.S. based company would recognize easily. U.S. phone numbers are stored with just an area code and then the seven-digit number, such as 650.507.9876. International numbers are stored in a format that has the U.S. “escape” code (for dialing an international number), followed by the country code and then the phone number, for example:  011.44.1644.429262.

Now the company grown to be a global company. It's need change how it stores phone numbers in order to adhere to the standards . It will store and display all phone numbers in two fields: a country code field and a phone number field. For example, using two sample phone numbers, the before and after storage will be as follows:

| **Before**         | **After**    |              |
| ------------------ | ------------ | ------------ |
| PHONE_NUMBER       | COUNTRY_CODE | PHONE#       |
| 650.507.9876       | +1           | 650.507.9876 |
| 011.44.1644.429262 | +44          | 1644.429262  |

Now the change involves a couple of steps. We have to

- Modify the schema so the EMPLOYEE table has two new columns: `COUNTRY_CODE` and `PHONE#`
- Mass-move the existing data from `PHONE_NUMBER` to `COUNTRY_CODE` and `PHONE#`
- Modify the schema so the EMPLOYEE table doesn’t have a `PHONE_NUMBER` column anymore
- Replace the application code in the database that is reliant on the `PHONE_NUMBER` column

Any one of these steps could take a significant amount of time. Our goal is to minimize any downtime incurred by this application upgrade, so we’d like to do all these steps while version 1.0 of the application is up and running and stage all the changes in the database for version 2.0 of the application. Then we’d like to online switch over to the new application.

## Step 1: Prepare the Version 1.0 application

1. Connect to the database host using the public ip. Sudo to the oracle user.

   ```
   ssh -i labkey opc@xxx.xxx.xxx
   sudo su - oracle
   ```
   

   
2. Connect to the PDB **orclpdb** with **sysdba** user.

   ```
   <copy>
   sqlplus sys/Ora_DB4U@orclpdb as sysdba
   </copy>
   ```

3. Install HR sample schema. Use the values as following:

   ```
   SQL> @?/demo/schema/human_resources/hr_main.sql
   
   specify password for HR as parameter 1:
   Enter value for 1: hr
   
   specify default tablespeace for HR as parameter 2:
   Enter value for 2: USERS
   
   specify temporary tablespace for HR as parameter 3:
   Enter value for 3: TEMP
   
   specify log path as parameter 4:
   Enter value for 4: $ORACLE_HOME/demo/schema/log/
   ```

   

4. Create a lab user and grant to sufficient privileges.

   ```
   <copy>
   create user demo identified by demo;
   grant connect, resource to demo;
   grant select on hr.employees to demo;
   alter user demo quota unlimited on users;
   </copy>
   ```

   

5. Connect with the lab user: 

   ```
   <copy>
   connect demo/demo@orclpdb
   </copy>
   ```

  

5. Let’s start with the version 1.0 application setup, first we will create the sample table and sequence.

   ```
   <copy>
   create table employees as select * from hr.employees;
   create sequence emp_seq start with 500;
   </copy>
   ```

The sequence was created to start with a value higher than any existing value in the EMPLOYEES table for this demonstration.

6. The version 1.0 application code will perform two functions: show a report of EMPLOYEES with their phone numbers and e-mails when given a search string and hire a new employee, adding the information of the new employee to the existing table. 

   ```
   <copy>
   create or replace package emp_pkg
   as
     procedure show
     ( last_name_like in employees.last_name%type );
   
     function add
     ( FIRST_NAME in employees.FIRST_NAME%type := null,
       LAST_NAME in employees.LAST_NAME%type,
       EMAIL in employees.EMAIL%type,
       PHONE_NUMBER in employees.PHONE_NUMBER%type := null,
       HIRE_DATE in employees.HIRE_DATE%type,
       JOB_ID in employees.JOB_ID%type,
       SALARY in employees.SALARY%type := null,
       COMMISSION_PCT in employees.COMMISSION_PCT%type := null,
       MANAGER_ID in employees.MANAGER_ID%type := null,
       DEPARTMENT_ID in employees.DEPARTMENT_ID%type := null  )
     return employees.employee_id%type;
   end;
   /
   </copy>
   ```

   

   The `emp_pkg` body for application version 1.0:

   ```
   <copy>create or replace package body emp_pkg
   as
   procedure show
   ( last_name_like in employees.last_name%type )
   as
   begin
       for x in
       ( select first_name, last_name,
                phone_number, email
           from employees
          where last_name like
                show.last_name_like
          order by last_name )
       loop
           dbms_output.put_line
           ( rpad( x.first_name || ' ' ||
                     x.last_name, 40 ) ||
             rpad( nvl(x.phone_number, ' '), 20 ) ||
             x.email );
       end loop;
   end show;
   function add
   ( FIRST_NAME in employees.FIRST_NAME%type := null,
     LAST_NAME in employees.LAST_NAME%type,
     EMAIL in employees.EMAIL%type,
     PHONE_NUMBER in employees.PHONE_NUMBER%type := null,
     HIRE_DATE in employees.HIRE_DATE%type,
     JOB_ID in employees.JOB_ID%type,
     SALARY in employees.SALARY%type := null,
     COMMISSION_PCT in employees.COMMISSION_PCT%type := null,
     MANAGER_ID in employees.MANAGER_ID%type := null,
     DEPARTMENT_ID in employees.DEPARTMENT_ID%type := null
   )
   return employees.employee_id%type
   is
       employee_id  employees.employee_id%type;
   begin
       insert into employees
       ( EMPLOYEE_ID, FIRST_NAME, LAST_NAME,
         EMAIL, PHONE_NUMBER, HIRE_DATE,
         JOB_ID, SALARY, COMMISSION_PCT,
         MANAGER_ID, DEPARTMENT_ID )
       values
       ( emp_seq.nextval, add.FIRST_NAME, add.LAST_NAME,
         add.EMAIL, add.PHONE_NUMBER, add.HIRE_DATE,
         add.JOB_ID, add.SALARY, add.COMMISSION_PCT,
         add.MANAGER_ID, add.DEPARTMENT_ID )
       returning employee_id into add.employee_id;
       return add.employee_id;
   end add;
   end;
   /</copy>
   ```

7. Now we can review how this application works in sqlplus.

   ```
   SQL> set serveroutput on 
   SQL> exec emp_pkg.show( '%C%' );
   Anthony Cabrio				650.509.4876	    ACABRIO
   Nanette Cambrault			011.44.1344.987668  NCAMBRAU
   Gerald Cambrault			011.44.1344.619268  GCAMBRAU
   John Chen				515.124.4269	    JCHEN
   Kelly Chung				650.505.1876	    KCHUNG
   Karen Colmenares			515.127.4566	    KCOLMENA
   Samuel McCain				650.501.3876	    SMCCAIN
   Donald OConnell 			650.507.9833	    DOCONNEL
   
   PL/SQL procedure successfully completed.
   
   SQL> begin
     dbms_output.put_line
     ( emp_pkg.add
       ( first_name      => 'Tom',
         last_name       => 'Cruise',
         email           => 'TCRUISE',
         phone_number    => '703.123.9999',
         hire_date       => sysdate,
         job_id          => 'MK_REP' ) );
   end;
   /  
   500
   
   PL/SQL procedure successfully completed.
   
   SQL> exec emp_pkg.show( '%C%' );
   Anthony Cabrio				650.509.4876	    ACABRIO
   Gerald Cambrault			011.44.1344.619268  GCAMBRAU
   Nanette Cambrault			011.44.1344.987668  NCAMBRAU
   John Chen				515.124.4269	    JCHEN
   Kelly Chung				650.505.1876	    KCHUNG
   Karen Colmenares			515.127.4566	    KCOLMENA
   Tom Cruise				703.123.9999	    TCRUISE
   Samuel McCain				650.501.3876	    SMCCAIN
   Donald OConnell 			650.507.9833	    DOCONNEL
   
   PL/SQL procedure successfully completed.
   
   SQL> commit;
   
   Commit complete.
   
   SQL>
   ```

   Now, the version 1.0 application is ready.



## Step 2: Preparing the Application to Use Editioning Views

1. Connect as sysdba user, create a new edition named **version2**. Permit the DEMO account to use editions, and grant the DEMO account to use the version2 edition.

   ```
    connect sys/Ora_DB4U@orclpdb as sysdba;
    create edition version2 as child of ora$base;
    alter user demo enable editions;
    grant use on edition version2 to demo;
    grant create view to demo;
    grant create job to demo;
    connect demo/demo@orclpdb;
   ```
   
    
   
2. Let’s prepare the schema to allow for an online application upgrade that includes physical schema updates. Remember, this involves one outage to put the editioning views in place. In this case, we will rename the base table and creating the editioning view with the original name of its base table's. The editioning view maps physical column names (used by the base table) to logical column names (used by the application). When you application has based on the editioning view, you can skip this step during the next upgrade.

   ```
    <copy>
    alter table employees rename to employees_rt;
    create editioning view employees
      as
      select 
        EMPLOYEE_ID, FIRST_NAME,
        LAST_NAME, EMAIL, PHONE_NUMBER,
        HIRE_DATE, JOB_ID, SALARY,
        COMMISSION_PCT, MANAGER_ID,
        DEPARTMENT_ID
      from employees_rt;
    </copy>
   ```


​    

3. Once that is done, we are online again. Furthermore, the existing application will be 100 percent unaffected by this; the editioning view we put in place looks and behaves just like a table. The existing application runs as before.

    ```
    SQL> set serveroutput on;
    SQL> exec emp_pkg.show( '%C%' );
    Anthony Cabrio				650.509.4876	    ACABRIO
    Gerald Cambrault			011.44.1344.619268  GCAMBRAU
    Nanette Cambrault			011.44.1344.987668  NCAMBRAU
    John Chen				515.124.4269	    JCHEN
    Kelly Chung				650.505.1876	    KCHUNG
    Karen Colmenares			515.127.4566	    KCOLMENA
    Tom Cruise				703.123.9999	    TCRUISE
    Samuel McCain				650.501.3876	    SMCCAIN
    Donald OConnell 			650.507.9833	    DOCONNEL
    
    PL/SQL procedure successfully completed.
    
    SQL> 
    ```

    

4. Now we are ready to add the new columns in preparation for the new application release:

    ```
    <copy>
    alter table employees_rt
    add
    ( country_code varchar2(3),
      phone# varchar2(20)
    );
    </copy>
    ```

    The addition of the columns is an online operation. So it's not interrupt the current application.

## Step 3: Transforming Data from Pre- to Post-Upgrade Representation
Now we need migrating the data from the pre-upgrade edition to the new edition. To accomplish this, we will rely on a forward cross-edition trigger. The trigger we’ll use to make sure that when data is inserted or updated by the pre-upgrade edition, the changes are accurately reflected in the new editions. We’ll use this trigger not only to capture the changes made by the legacy application but also to do the mass move as well.

1. First let's switch to the new edition version2.

    ````
    SQL> SELECT SYS_CONTEXT('userenv','current_edition_name') sc FROM DUAL;
    
    SC
    --------------------------------------------------------------------------------
    ORA$BASE
    
    SQL> alter session set edition = version2;
    
    Session altered.
    
    SQL> SELECT SYS_CONTEXT('userenv','current_edition_name') sc FROM DUAL;
    
    SC
    --------------------------------------------------------------------------------
    VERSION2
    
    SQL> 
    ````
    
    In the edition version2, all the editioned objects are inherited from the existing version in the ORA$BASE edition.
    
2. Then we create a forward cross-edition trigger. We created this in our new edition verion2. Our goal is to not disturb the existing application, so we’ll do our editioning work in the new edition only.

    ````
    <copy>
    create or replace trigger employees_fwdxedition
    before insert or update of phone_number on employees_rt
    for each row
    forward crossedition
    DISABLE
    declare
        first_dot  number;
        second_dot number;
    begin
        if :new.phone_number like '011.%'
       then
            first_dot:= instr( :new.phone_number, '.' );
            second_dot:= instr( :new.phone_number, '.', 1, 2 );
            :new.country_code:= '+'||substr( :new.phone_number,first_dot+1,second_dot-first_dot-1 );
            :new.phone#:= substr( :new.phone_number,second_dot+1 );
        else
            :new.country_code := '+1';
            :new.phone# := :new.phone_number;
        end if;
    end;
    /
    </copy>
    ````

3. Enable the forward cross-edition trigger if there is no problem.

    ````
    <copy>
    ALTER TRIGGER employees_fwdxedition ENABLE;
    </copy>
    ````

     

4. Before data transform, we need wait until pending changes are either committed or rolled back to prevent lost updates:

    ```
    <copy>
    DECLARE
      scn              NUMBER  := NULL;
      timeout CONSTANT INTEGER := NULL;
    BEGIN
      IF NOT DBMS_UTILITY.WAIT_ON_PENDING_DML(Tables  => 'EMPLOYEES_RT',
                                              timeout => timeout,
                                              scn     => scn)
      THEN
        RAISE_APPLICATION_ERROR(-20000,
         'Wait_On_Pending_DML() timed out. CETs were enabled before SCN: '||SCN);
      END IF;
    END;
    /
    </copy>
    ```

    

5. Now we are ready to transforming data from pre- to post-upgrade representation:

    ````
    <copy>
    DECLARE
      c NUMBER := DBMS_SQL.OPEN_CURSOR();
      x NUMBER;
    BEGIN
      DBMS_SQL.PARSE(
        c                          => c,
        Language_Flag              => DBMS_SQL.NATIVE,
        Statement                  => 'update employees set phone_number = phone_number',
        Apply_Crossedition_Trigger => 'employees_fwdxedition'
      );
      x := DBMS_SQL.EXECUTE(c);
      DBMS_SQL.CLOSE_CURSOR(c);
    END;
    /
    </copy>
    ````

6. Check the result:

    ```
    SQL> select phone_number,country_code,phone# from employees_rt;
    
    PHONE_NUMBER	     COU PHONE#
    -------------------- --- --------------------
    515.123.4567	     +1  515.123.4567
    515.123.4568	     +1  515.123.4568
    515.123.4569	     +1  515.123.4569
    590.423.4567	     +1  590.423.4567
    ......
    ......
    011.44.1345.429268   +44 1345.429268
    011.44.1345.829268   +44 1345.829268
    011.44.1346.229268   +44 1346.229268
    011.44.1343.929268   +44 1343.929268
    011.44.1343.329268   +44 1343.329268
    011.44.1644.429263   +44 1644.429263
    650.509.1876	     +1  650.509.1876
    650.505.2876	     +1  650.505.2876
    650.501.2876	     +1  650.501.2876
    
    108 rows selected.
    
    SQL> 
    ```

    When you applying the transform, you can invoke either the `DBMS_SQL`.`PARSE` procedure or the subprograms in the `DBMS_PARALLEL_EXECUTE` package. The latter is recommended if you have a lot of data. The subprograms enable you to incrementally update the data in a large table in parallel, in two high-level steps:

    - Group sets of rows in the table into smaller chunks.

    - Apply the desired UPDATE statement to the chunks in parallel, committing each time you have finished processing a chunk.

    The advantages are:

    - You lock only one set of rows at a time, for a relatively short time, instead of locking the entire table.
    - You do not lose work that has been done if something fails before the entire operation finishes.

     

    In the next step, we will perform the above mass task using the `DBMS_PARALLEL_EXECUTE` package. So, We rollback the `employees_rt` table to the pre-upgrade state.

    ```
    SQL> rollback;
    
    Rollback complete.
    SQL>
    ```

    

7. We’ll have to (for purposes of demonstration) scale up our `EMPLOYEES_RT` table, because it is very small right now. First we’ll make it 1000 times larger.

    ```
    SQL> insert into employees
    select * from
    (
    with data(r)
    as
    (select 1 r from dual
     union all
     select r+1 from data where r <= 1000
    )
    select rownum+(select max(employee_id)
                          from employees_rt),
           FIRST_NAME, LAST_NAME, EMAIL,
           PHONE_NUMBER, HIRE_DATE, JOB_ID,
           SALARY, COMMISSION_PCT, MANAGER_ID,
           DEPARTMENT_ID
      from employees_rt, data
    );
    
    108108 rows created.
    
    SQL> commit;
    
    Commit complete.
    ```

    

8. Make sure your current edition is version2:

    ```
    SQL> SELECT SYS_CONTEXT('userenv','current_edition_name') sc FROM DUAL;
    
    SC
    --------------------------------------------------------------------------------
    VERSION2
    
    SQL> 
    ```

    

9. Check the blocks of the table:

    ```
    SQL> select count(*), count(distinct dbms_rowid.rowid_block_number(rowid)) cnt_blk from employees_rt;
    
      COUNT(*)    CNT_BLK
    ---------- ----------
       108216	  1111
    
    SQL> 
    ```

    

10. Our existing table is about 1111 blocks, and we’d like to update about 10 percent of it at a time. (On a larger table, you’d likely use a much smaller percentage to avoid locking too much of the table at a time.) So we’ll break it up that about 100 blocks in one chunk.

   ```
   <copy>
   begin
       dbms_parallel_execute.create_task('update employees_rt');
       dbms_parallel_execute.create_chunks_by_rowid
       ( task_name   => 'update employees_rt',
         table_owner => user,
         table_name  => 'EMPLOYEES_RT',
         by_row      => false,
         chunk_size  => 100);
   end;
   /
   </copy>
   ```

   

11. Looking at chunks in `USER_PARALLEL_EXECUTE_CHUNKS`:

    ```
    SQL> select chunk_id, status, start_rowid, end_rowid
      from user_parallel_execute_chunks
     where task_name = 'update employees_rt'; 
    
      CHUNK_ID STATUS		START_ROWID	   END_ROWID
    ---------- -------------------- ------------------ ------------------
    	34 UNASSIGNED		AAAR6DAAMAAAAFIAAA AAAR6DAAMAAAAGzH//
    	35 UNASSIGNED		AAAR6DAAMAAAAG0AAA AAAR6DAAMAAAAJHH//
    	36 UNASSIGNED		AAAR6DAAMAAAAJIAAA AAAR6DAAMAAAAKrH//
    	37 UNASSIGNED		AAAR6DAAMAAAAKsAAA AAAR6DAAMAAAAMPH//
    	38 UNASSIGNED		AAAR6DAAMAAAAMQAAA AAAR6DAAMAAAANzH//
    	39 UNASSIGNED		AAAR6DAAMAAAAN0AAA AAAR6DAAMAAAAPXH//
    	40 UNASSIGNED		AAAR6DAAMAAAAPYAAA AAAR6DAAMAAAAQ7H//
    	41 UNASSIGNED		AAAR6DAAMAAAAQ8AAA AAAR6DAAMAAAASfH//
    	42 UNASSIGNED		AAAR6DAAMAAAASgAAA AAAR6DAAMAAAAUDH//
    	43 UNASSIGNED		AAAR6DAAMAAAAUEAAA AAAR6DAAMAAAAVnH//
    	44 UNASSIGNED		AAAR6DAAMAAAAVoAAA AAAR6DAAMAAAAXLH//
    
      CHUNK_ID STATUS		START_ROWID	   END_ROWID
    ---------- -------------------- ------------------ ------------------
    	45 UNASSIGNED		AAAR6DAAMAAAAXMAAA AAAR6DAAMAAAAX/H//
    
    12 rows selected.
    
    SQL> 
    ```

    

12. Now we are ready to perform our update.

     ```
     <copy>
     begin
         dbms_parallel_execute.run_task
         ( task_name      => 'update employees_rt',
           sql_stmt       => 'update employees_rt
                                 set phone_number = phone_number
                               where rowid between :start_id
                                               and :end_id',
           language_flag  => DBMS_SQL.NATIVE,
           apply_crossedition_trigger => 'employees_fwdxedition',
           parallel_level => 2 );
     end;
     /
     </copy>
     ```

     When running our task, using two threads of execution `(parallel_level=>2)` just to demonstrate that you can chunk something up into many more chunks than you ultimately run concurrently. It's locking only a small subset of the table at a time. This enabled the existing application to function normally while we did our mass move of data at the same time.

13. Check the result:

     ```
     SQL> select count(*) from employees_rt where country_code is not null;
     
       COUNT(*)
     ----------
         108216
     
     SQL>
     ```
     
     
     
14. After that operation is done and we are satisfied with the results, we can drop the task we created:

     ```
     <copy>
     begin
       dbms_parallel_execute.drop_task('update employees_rt');
     end;
     /
     </copy>
     ```



## Step 4: Upgrade the new application code

1. Make sure you are in the new edition:

   ```
   SQL> SELECT SYS_CONTEXT('userenv','current_edition_name') sc FROM DUAL;
   
   SC
   --------------------------------------------------------------------------------
   VERSION2
   
   SQL> 
   ```

   

2. Right now all the code is inherited from the existing version (version 1) of our application in the ORA$BASE edition.

   ```
   SQL> select object_name, object_type, status, edition_name
     from user_objects_ae
     where object_name in ( 'EMPLOYEES', 'EMP_PKG' ); 
   
   OBJECT_NAME	         OBJECT_TYPE	           STATUS  EDITION_NAME
   -------------------- ----------------------- ------- --------------------
   EMPLOYEES	           VIEW		                 VALID   ORA$BASE
   EMP_PKG 	           PACKAGE		             VALID   ORA$BASE
   EMP_PKG 	           PACKAGE BODY	           VALID   ORA$BASE
   
   SQL> 
   ```

   

3. Now we replace the view and package to the new version. It's not effect the editioned objects in version1.

   ```
   <copy>
   create OR REPLACE editioning view employees
   as
     select
       employee_id, first_name,
       last_name, email, COUNTRY_CODE, PHONE#,
       hire_date, job_id, salary,
       commission_pct, manager_id,
       department_id
     from employees_rt
   /
   
   create or replace package emp_pkg
   as
     procedure show
     ( last_name_like in employees.last_name%type );
   
     function add
       ( FIRST_NAME in employees.FIRST_NAME%type := null,
       LAST_NAME in employees.LAST_NAME%type,
       EMAIL in employees.EMAIL%type,
       -- replaced
       COUNTRY_CODE in employees.COUNTRY_CODE%type := null,
   	  PHONE# in employees.PHONE#%type := null,
   	  -- replaced
       HIRE_DATE in employees.HIRE_DATE%type,
       JOB_ID in employees.JOB_ID%type,
       SALARY in employees.SALARY%type := null,
       COMMISSION_PCT in employees.COMMISSION_PCT%type := null,
       MANAGER_ID in employees.MANAGER_ID%type := null,
       DEPARTMENT_ID in employees.DEPARTMENT_ID%type := null  )
     return employees.employee_id%type;
   end;
   /
   
   create or replace package body emp_pkg
   as
   procedure show
   ( last_name_like in employees.last_name%type )
   as
   begin
       for x in
       ( select first_name, last_name,
                country_code, phone#, email
           from employees
          where last_name like
                show.last_name_like
          order by last_name )
       loop
           dbms_output.put_line
           ( rpad( x.first_name || ' ' ||
                     x.last_name, 40 ) ||
             rpad( nvl(x.country_code, ' '), 5 ) ||
             rpad( nvl(x.phone#, ' '), 20 ) ||
             x.email );
       end loop;
   end show;
   function add
   ( FIRST_NAME in employees.FIRST_NAME%type := null,
     LAST_NAME in employees.LAST_NAME%type,
     EMAIL in employees.EMAIL%type,
     COUNTRY_CODE in employees.COUNTRY_CODE%type := null,
     PHONE# in employees.PHONE#%type := null,
     HIRE_DATE in employees.HIRE_DATE%type,
     JOB_ID in employees.JOB_ID%type,
     SALARY in employees.SALARY%type := null,
     COMMISSION_PCT in employees.COMMISSION_PCT%type := null,
     MANAGER_ID in employees.MANAGER_ID%type := null,
     DEPARTMENT_ID in employees.DEPARTMENT_ID%type := null
   )
   return employees.employee_id%type
   is
       employee_id  employees.employee_id%type;
   begin
       insert into employees
       ( EMPLOYEE_ID, FIRST_NAME, LAST_NAME,
         EMAIL, COUNTRY_CODE, PHONE#, HIRE_DATE,
         JOB_ID, SALARY, COMMISSION_PCT,
         MANAGER_ID, DEPARTMENT_ID )
       values
       ( emp_seq.nextval, add.FIRST_NAME, add.LAST_NAME,
         add.EMAIL, add.COUNTRY_CODE, add.PHONE#, add.HIRE_DATE,
         add.JOB_ID, add.SALARY, add.COMMISSION_PCT,
         add.MANAGER_ID, add.DEPARTMENT_ID )
       returning employee_id into add.employee_id;
       return add.employee_id;
   end add;
   end;
   /
   </copy>
   ```

   

4. And now we can see that we have both versions installed.

   ```
   SQL> select object_name, object_type, status, edition_name
     from user_objects_ae
     where object_name in ( 'EMPLOYEES', 'EMP_PKG' ); 
   
   OBJECT_NAME	         OBJECT_TYPE	           STATUS  EDITION_NAME
   -------------------- ----------------------- ------- --------------------
   EMPLOYEES	           VIEW		                 VALID   ORA$BASE
   EMP_PKG 	           PACKAGE		             VALID   ORA$BASE
   EMP_PKG 	           PACKAGE BODY	           VALID   ORA$BASE
   EMPLOYEES	           VIEW		                 VALID   VERSION2
   EMP_PKG 	           PACKAGE		             VALID   VERSION2
   EMP_PKG 	           PACKAGE BODY	           VALID   VERSION2
   
   6 rows selected.
   
   SQL> 
   ```

   

5. The editions are installed but not quite ready to go yet. The code in pre-upgrade edition `ORA$BASE` is all set, but the code in VERSION2 is not quite ready. What if we call the `EMP_PKG.ADD` routine in VERSION2? It will populate the `COUNTRY_CODE` and `PHONE#` column but not the `PHONE_NUMBER` legacy column! So if you want the pre-upgrade and post-upgrade application coexist for sometimes, you need to create a reverse crossedition trigger:

   ```
   <copy>
   create or replace trigger employees_revxedition
   before insert or update of country_code,phone# on employees_rt
   for each row
   reverse crossedition
   DISABLE
   declare
       first_dot  number;
       second_dot number;
   begin
           if :new.country_code = '+1'
           then
              :new.phone_number :=
                 :new.phone#;
           else
              :new.phone_number :=
                 '011.' ||
                 substr( :new.country_code, 2 ) ||
                 '.' || :new.phone#;
           end if;
   end;
   /
   </copy>
   ```

   

6. If there is no problem, enable reverse crossedition trigger:

   ```
   <copy>ALTER TRIGGER employees_revxedition ENABLE;</copy>
   ```

   

7. We can use the new application code in VERSION2:

   ```
   <copy>
   begin
    dbms_output.put_line
    ( emp_pkg.add
      ( first_name   => 'Tom',
        last_name    => 'Hanks',
        email        => 'THANKS',
        country_code => '+44',
        phone#       => '703.123.4567',
        hire_date    => sysdate,
        job_id       => 'MK_REP' ) );
   end;
   /
   </copy>
   ```

   

8. Check the result:

   ```
   SQL> set serveroutput on;
   SQL> exec emp_pkg.show('Hanks')
   Tom Hanks				+44  703.123.4567	 THANKS
   
   PL/SQL procedure successfully completed.
   
   SQL> 
   ```

   

9. We can see that the data is input correctly. Further, in the pre-upgrade edition ORA$BASE we can verify that the legacy data format is still valid:

   ```
   SQL> connect demo/demo@orclpdb
   Connected.
   SQL> SELECT SYS_CONTEXT('userenv','current_edition_name') sc FROM DUAL;
   
   SC
   --------------------------------------------------------------------------------
   ORA$BASE
   
   SQL> set serveroutput on
   SQL> exec emp_pkg.show('Hanks')
   Tom Hanks				011.44.703.123.4567 THANKS
   
   PL/SQL procedure successfully completed.
   
   SQL> 
   ```

   

10. We can still run the old application in Version 1:

    ```
    SQL> begin
      dbms_output.put_line
      ( emp_pkg.add
        ( first_name      => 'Steve',
          last_name       => 'Jobs',
          email           => 'SJOBS',
          phone_number    => '555.123.8888',
          hire_date       => sysdate,
          job_id          => 'MK_REP' ) );
    end;
    / 
    502
    
    PL/SQL procedure successfully completed.
    
    SQL> exec emp_pkg.show('Jobs')
    Steve Jobs				555.123.8888	    SJOBS
    
    PL/SQL procedure successfully completed.
    
    SQL> commit;
    
    Commit complete.
    
    SQL>
    ```

    

11. Switch to the new edition Version 2, we can verify that the new data format is still valid:

    ```
    SQL> alter session set edition = version2;
    
    Session altered.
    
    SQL> exec emp_pkg.show('Jobs')
    Steve Jobs				+1   555.123.8888	 SJOBS
    
    PL/SQL procedure successfully completed.
    
    SQL> 
    ```



## Step 5: Set the new edition to default 

1. Once the application is upgraded, We can make using the new edition to default.

   ```
   SQL> connect sys/Ora_DB4U@orclpdb as sysdba
   Connected.
   
   SQL> ALTER DATABASE DEFAULT EDITION = VERSION2;
   
   Database altered.
   
   SQL>
   ```

   

2. Now, when we log in to the database, we’ll observe:

   ```
   SQL> connect demo/demo@orclpdb
   Connected.
   SQL> SELECT SYS_CONTEXT('userenv','current_edition_name') sc FROM DUAL;
   
   SC
   --------------------------------------------------------------------------------
   VERSION2
   
   SQL> set serveroutput on
   SQL> exec emp_pkg.show( 'Hanks' )
   Tom Hanks				+44  703.123.4567	 THANKS
   
   PL/SQL procedure successfully completed.
   
   SQL> 
   ```

   Now all that remains is cleaning up. The cleanup takes place after everyone is finished using the `ORA$BASE` edition, after no existing sessions are using the old code, we can perform our cleanup. In this case, the cleanup consists of

   - Dropping the forward and reverse crossedition triggers
   - Optionally, dropping or setting as unused the `PHONE_NUMBER` column

   And that is it. We are done with our online application upgrade!

   


## Conclusion

- Edition-based redefinition allows multiple versions of PL/SQL objects, views and synonyms in a single schema, which makes it possible to perform upgrades of database applications with zero down time.

For more information, please see the EBR Guide at [https://www.oracle.com/database/technologies/high-availability/ebr.html](https://www.oracle.com/database/technologies/high-availability/ebr.html)

