# Install MySQL Shell Table Encryption custom plugin
Create directory
```
mkdir -p $HOME/.mysqlsh/plugins/table_encryption
```
Create a python file
```
vi $HOME/.mysqlsh/plugins/table_encryption/init.py
```
Copy and paste the following code
```
from mysqlsh.plugin_manager import plugin, plugin_function
import mysqlsh
import os

shell = mysqlsh.globals.shell
dba = mysqlsh.globals.dba

def _user_identity():
    x = shell.parse_uri(shell.get_session().get_uri())
    vUser = x['user']
    host = x['host']
    port = x['port']
    return vUser, host, port

def _strict_root():
    vUser, host, port = _user_identity()
    if vUser != "root":
       return False
    else:
       if host != "localhost":
           return False
    return True

def _run_sql(query, getColumnNames):
    session = shell.get_session()
    result = session.run_sql(query)
    list_output = []
    if (result.has_data()):
        if getColumnNames:
            list_output = [result.get_column_names()]
        for row in result.fetch_all():
             list_output.append(list(row))
    else:
        list_output.append("0")
    return list_output

@plugin_function("table_encryption.add_user")
def add_user():
    if _strict_root():
       vName = shell.prompt("Please provide user_name: ")
       try:
         _run_sql("create database if not exists keyDB", False)
         _run_sql("create table if not exists keyDB.key_allowlist (id int auto_increment primary key, key_id varchar(50), key_name varchar(100), user varchar(50), role varchar(10))", False)
         _run_sql("create table if not exists keyDB.table_column_encryption (id int auto_increment primary key, schema_name varchar(50), table_name varchar(50), column_name varchar(50), user_name varchar(50), key_id varchar(20), encryption_date date)", False)
         _run_sql("grant select on keyDB.key_allowlist to " + vName + "@'%';", False)
         _run_sql("grant execute on *.* to " + vName + "@'%';", False)
       except:
         print("\nCheck if user exists \n")
    else:
       print("\nMust use root@'localhost' \n")

@plugin_function("table_encryption.remove_user")
def remove_user():
    if _strict_root():
       vName = shell.prompt("Please provide user_name: ")
       try:
         _run_sql("create database if not exists keyDB", False)
         _run_sql("create table if not exists keyDB.key_allowlist (id int auto_increment primary key, key_id varchar(50), key_name varchar(100), user varchar(50), role varchar(10))", False)
         _run_sql("create table if not exists keyDB.table_column_encryption (id int auto_increment primary key, schema_name varchar(50), table_name varchar(50), column_name varchar(50), user_name varchar(50), key_id varchar(20), encryption_date date)", False)         
         _run_sql("revoke select on keyDB.key_allowlist from " + vName + "@'%';", False)
         _run_sql("revoke execute on *.* from " + vName + "@'%';", False)
       except:
         print("\nCheck if user exists \n")
    else:
       print("\nMust use root@'localhost' \n")


def _check_key(vKey):
    vCheck = _run_sql("select count(1) from keyDB.key_allowlist where key_id='" + vKey + "'", False)
    return vCheck[0][0] 

def _check_key_owner(vKey, vOwner):
    vCheck=0
    if _check_key(vKey)==1:
       test = _run_sql("select count(1) from keyDB.key_allowlist where key_id='" + vKey + "' and role='owner' and user='" + vOwner + "'", False)
       vCheck=test[0][0]
    return vCheck

def _check_key_grantee(vKey, vGrantee):
    vCheck=0
    if _check_key(vKey)==1:
       test = _run_sql("select count(1) from keyDB.key_allowlist where key_id='" + vKey + "' and role='grantee' and user='" + vGrantee + "'", False)
       vCheck=test[0][0]
    return vCheck

@plugin_function("table_encryption.show_keys")
def show_keys():
    if _strict_root():
       return shell.get_session().run_sql("select * from keyDB.key_allowlist")
    else:
       print("\nMust use root@'localhost' \n")

@plugin_function("table_encryption.show_encrypted_columns")
def show_encrypted_columns():
    if _strict_root():
       return shell.get_session().run_sql("select * from keyDB.table_column_encryption")
    else:
       print("\nMust use root@'localhost' \n")


@plugin_function("table_encryption.add_key")
def add_key():
    if _strict_root():
       vUser, host, port = _user_identity()
       vUser = shell.prompt("Please provide user_name: ")
       vPassword = shell.prompt('Please provide user password: ',{"type":"password"})
       x=shell.get_session()
       try:
          y=shell.open_session(vUser + "@localhost:" + str(port), vPassword)
          shell.set_session(y)
       except:
          shell.set_session(x)
          print("\nUnable to connect using " + vUser + " \n")
          return
       vName = shell.prompt("Please provide key_id: ")
       vKey = shell.prompt('Please provide key string: ',{"type":"password"})
       try:
          if _check_key(vName)==0:
             _run_sql("select keyring_key_store('" + vName + "','DSA','" + vKey + "');", False)
             shell.set_session(x)
             _run_sql("drop function if exists keyDB.get_shared_key_" + vUser + "_" + vName, False)
             _run_sql("CREATE DEFINER = '" + vUser + "' FUNCTION keyDB.get_shared_key_" + vUser + "_" + vName + "() RETURNS BLOB READS SQL DATA RETURN if(substring_index(replace(user(),'@',' '),' ',1)=(select user from keyDB.key_allowlist where user=substring_index(replace(user(),'@',' '),' ',1) and key_id='" + vName + "'),keyring_key_fetch('" + vName + "'),null)", False)
             _run_sql("grant execute on function keyDB.get_shared_key_" + vUser + "_" + vName + " to " + vUser + "@'%'", False) 
             _run_sql("insert into keyDB.key_allowlist (key_id, key_name, user, role) values ('" + vName + "','keyDB.get_shared_key_" + vUser + "_" + vName + "','" + vUser + "', 'owner')", False)
          else:
             print("\nThis key_id is already exist \n")
             shell.set_session(x)
       except:
             print("\n" + vUser + " may not be a user or key_id exist but not on key_allowlist. Try to add user or manually run keyring_key_remove before adding this key. If you are on Read-Only node of InnoDB Cluster or ClusterSet, this is expected and please ignode this error message as the key is created successfully\n")
             shell.set_session(x)
    else:
       print("\nMust use root@'localhost' \n")

def _check_key_usage(vName):
    vCheck = _run_sql("select count(1) from keyDB.table_column_encryption where key_id='" + vName + "'", False)
    if vCheck[0][0]!=0:
       print("\nBlocked. This key_id is in use \n")
    return vCheck[0][0] 


@plugin_function("table_encryption.remove_key")
def remove_key():
    if _strict_root():
       vUser, host, port = _user_identity()
       vUser = shell.prompt("Please provide user_name: ")
       vPassword = shell.prompt('Please provide user password: ',{"type":"password"})
       x=shell.get_session()
       try:
          y=shell.open_session(vUser + "@localhost:" + str(port), vPassword)
          shell.set_session(y)
       except:
          shell.set_session(x)
          print("\nUnable to connect using " + vUser + " \n")
          return
       vName = shell.prompt("Please provide key_id: ")
       try:
          shell.set_session(x)
          if _check_key_owner(vName, vUser)==1 and _check_key_usage(vName)==0:
             shell.set_session(y)
             _run_sql("select keyring_key_remove('" + vName + "')", False)
             shell.set_session(x)
             _run_sql("drop function if exists keyDB.get_shared_key_" + vUser + "_" + vName, False)
             _run_sql("delete from keyDB.key_allowlist where key_id='" + vName + "'", False)
          else:
             print("\nThis key_id is not exist or not an owner\n")
             shell.set_session(x) 
       except:
          print("\n" + vUser + " may not be a user or key_id is not exist. Try to add user. If you are on Read-Only node of InnoDB Cluster or ClusterSet, this is expected and please ignode this error message as the key is removed successfully\n")
          shell.set_session(x)
    else:
       print("\nMust use root@'localhost' \n") 

def _check_object(vUser, vDB, vTable, vColumn, vName):
    table_schema = _run_sql("select count(1) from information_schema.COLUMNS where table_name='" + vTable + "' and table_schema='" + vDB + "' and column_name='" + vColumn + "'", False)
    if table_schema[0][0] == 1:
         routine_ = _run_sql("select count(1) from information_schema.routines where routine_schema='keyDB' and routine_type='FUNCTION' and routine_name='get_shared_key_" + vUser + "_" + vName + "';", False)
         if routine_[0][0] == 1:
               return True
         print("\nNo such key id \n")
         return False
    else:
         print("\nNo such tables \n")
         return False

@plugin_function("table_encryption.grant_access_to_key")
def grant_access_to_key():
    if _strict_root():
       vUser = shell.prompt("Please provide user_name: ")
       vName = shell.prompt("Please provide key_id: ")
       vOwner = shell.prompt("Please provide key owner: ")
       try:
         _run_sql("grant execute on function keyDB.get_shared_key_" + vOwner + "_" + vName + " to " + vUser + "@'%'", False)
         _run_sql("insert into keyDB.key_allowlist (key_id, key_name, user, role) values ('" + vName + "','keyDB.get_shared_key_" + vOwner + "_" + vName + "','" + vUser + "', 'grantee')", False)       
       except:
         print("\nuser_name " + vUser + "@'%' may be invalid or key id and key owner mismatch\n")
    else:
       print("\nMust use root@'localhost' \n")


@plugin_function("table_encryption.revoke_access_from_key")
def revoke_access_from_key():
    if _strict_root():
       vUser = shell.prompt("Please provide user_name: ")
       vName = shell.prompt("Please provide key_id: ")
       vOwner = shell.prompt("Please provide key owner: ")
       try:
         _run_sql("revoke execute on function keyDB.get_shared_key_" + vOwner + "_" + vName + " from " + vUser + "@'%'", False) 
         _run_sql("delete from keyDB.key_allowlist where key_id='" + vName + "' and user='" + vUser + "'", False)
       except:
         print("\nuser_name " + vUser + "@'%' may be invalid or key id and key owner mismatch\n")
    else:
       print("\nMust use root@'localhost' \n")


def _create_trigger_before_insert(vUser, vDB, vTable, vColumn, vName):
    os.system("echo 'DELIMITER $$' > trigger_before_insert.sql")
    os.system("echo 'create trigger " + vDB + ".before_" + vTable + "_" + vColumn + "_insert before insert on " + vDB + "." + vTable + " for each row' >> trigger_before_insert.sql")
    os.system("echo 'begin' >> trigger_before_insert.sql")
    os.system("echo '  DECLARE v_msg VARCHAR(255);' >> trigger_before_insert.sql") 
    os.system("echo '  if (substring_index(replace(user(),#@#,# #),# #,1) <> #" + vUser + "#) then' >> trigger_before_insert.sql")
    os.system("echo '     set v_msg = #Blocked. Table is encrypted and you are not the key_id owner#; ' >> trigger_before_insert.sql")    
    os.system("echo '     SIGNAL SQLSTATE #45000# SET MESSAGE_TEXT = v_msg;' >> trigger_before_insert.sql")
    os.system("echo '  end if; ' >> trigger_before_insert.sql")
    os.system("echo '  set NEW." + vColumn + "=hex(aes_encrypt(NEW." + vColumn + ", hex(keyDB.get_shared_key_" + vUser + "_" + vName + "())));' >> trigger_before_insert.sql")
    os.system("echo 'end$$' >> trigger_before_insert.sql")
    os.system("echo 'DELIMITER ;' >> trigger_before_insert.sql")
    os.system("sed -i 'y/#/'\\''/' trigger_before_insert.sql")

def _create_trigger_before_update(vUser, vDB, vTable, vColumn, vName):
    os.system("echo 'DELIMITER $$' > trigger_before_update.sql")
    os.system("echo 'create trigger " + vDB + ".before_" + vTable + "_" + vColumn + "_update before update on " + vDB + "." + vTable + " for each row' >> trigger_before_update.sql")
    os.system("echo 'begin' >> trigger_before_update.sql")
    os.system("echo '  DECLARE v_msg VARCHAR(255);' >> trigger_before_update.sql")
    os.system("echo '  if (substring_index(replace(user(),#@#,# #),# #,1) <> #" + vUser + "#) then' >> trigger_before_update.sql")
    os.system("echo '     set v_msg = #Blocked. Table is encrypted and you are not the key_id owner#; ' >> trigger_before_update.sql")
    os.system("echo '     SIGNAL SQLSTATE #45000# SET MESSAGE_TEXT = v_msg;' >> trigger_before_update.sql")
    os.system("echo '  end if; ' >> trigger_before_update.sql")
    os.system("echo '   if (OLD." + vColumn + " <> hex(aes_encrypt(NEW." + vColumn + ", hex(keyDB.get_shared_key_" + vUser + "_" + vName + "()))) and OLD." + vColumn + " <> NEW." + vColumn + ") then' >> trigger_before_update.sql")
    os.system("echo '       set NEW." + vColumn + "=hex(aes_encrypt(NEW." + vColumn + ", hex(keyDB.get_shared_key_" + vUser + "_" + vName + "())));' >> trigger_before_update.sql")
    os.system("echo '   Else ' >> trigger_before_update.sql")
    os.system("echo '       set NEW." + vColumn + "=OLD." + vColumn + ";' >> trigger_before_update.sql")
    os.system("echo '   end if;' >> trigger_before_update.sql")
    os.system("echo 'end$$' >> trigger_before_update.sql")
    os.system("echo 'DELIMITER ;' >> trigger_before_update.sql")
    os.system("sed -i 'y/#/'\\''/' trigger_before_update.sql")	

def _check_for_encrypt(vUser, vDB, vTable, vColumn, vName):
    # check if table column is encrypted
    vCheck = _run_sql("select count(1) from keyDB.table_column_encryption where schema_name='" + vDB + "' and table_name='" + vTable + "' and column_name='" + vColumn + "'", False)
    if vCheck[0][0] == 0:
       # check if any column is already encrypted using other user
       vCheck = _run_sql("select count(1) from keyDB.table_column_encryption where schema_name='" + vDB + "' and table_name='" + vTable + "' and user_name<>'" + vUser + "'", False) 
       if vCheck[0][0] == 0:
          return True
       else:
          print("\nBlocked. Some table columns already encrypted using other user\n")
          return False      
    else:
       print("\nBlocked. Table column is already encrypted\n")
       return False


@plugin_function("table_encryption.encrypt_table_column")
def encrypt_table_column():
    if _strict_root():
       vUser, host, port = _user_identity()
       vUser = shell.prompt("Please provide user_name: ")
       vPassword = shell.prompt('Please provide user password: ',{"type":"password"})
       x=shell.get_session()
       try:
          y=shell.open_session(vUser + "@localhost:" + str(port), vPassword)
          shell.set_session(y)
       except:
          shell.set_session(x)
          print("\nUnable to connect using " + vUser + " \n")
          return
       vName = shell.prompt("Please provide key_id: ")
       vDB = shell.prompt("Please provide DB NAME: ")
       vTable = shell.prompt("Please provide TABLE NAME: ")
       vColumn = shell.prompt("Please provide COLUMN NAME: ")
       shell.set_session(x)
       if _check_object(vUser, vDB, vTable, vColumn, vName) and _check_for_encrypt(vUser, vDB, vTable, vColumn, vName):
         try:
          shell.set_session(y)
          _run_sql("alter table " + vDB + "." + vTable + " add column " + vColumn + "_encrypted varchar(100) after " + vColumn + ", algorithm=inplace;", False)
          _run_sql("update " + vDB + "." + vTable + " set " + vColumn + "_encrypted=hex(aes_encrypt(" + vColumn + ", hex(keyDB.get_shared_key_" + vUser + "_" + vName + "())));", False)
          _run_sql("alter table " + vDB + "." + vTable + " drop column " + vColumn + ", algorithm=inplace;", False)
          _run_sql("alter table " + vDB + "." + vTable + " rename column " + vColumn + "_encrypted to " + vColumn + ", algorithm=inplace;", False)
          shell.set_session(x)
          _run_sql("grant super on *.* to " + vUser + "@'%'", False)
          _create_trigger_before_insert(vUser, vDB, vTable, vColumn, vName)
          os.system("mysql -u" + vUser + " -p" + vPassword + " -h127.0.0.1 -P" + str(port) + " -e 'source trigger_before_insert.sql'")
          os.system("rm trigger_before_insert.sql")
          _create_trigger_before_update(vUser, vDB, vTable, vColumn, vName)
          os.system("mysql -u" + vUser + " -p" + vPassword + " -h127.0.0.1 -P" + str(port) + " -e 'source trigger_before_update.sql'")
          os.system("rm trigger_before_update.sql")
          _run_sql("revoke super on *.* from " + vUser + "@'%'", False)
          _run_sql("insert into keyDB.table_column_encryption (schema_name, table_name, column_name, user_name, key_id, encryption_date) values ('" + vDB + "','" + vTable + "','" + vColumn + "','" + vUser + "','" + vName + "', sysdate())", False)
         except:
          print("\nUnable to continue, check table user privileges\n")
         shell.set_session(x)

def _check_key_id(vUser, vDB, vTable, vColumn):
    vCheck = _run_sql("select count(1) from keyDB.table_column_encryption where schema_name='" + vDB + "' and table_name='" + vTable + "' and column_name='" + vColumn + "' and user_name='" + vUser + "'", False)
    if vCheck[0][0] == 0:
         print("\nTable column encryption using this user not found\n")
         return "False"
    else:
         vCheck = _run_sql("select key_id from keyDB.table_column_encryption where schema_name='" + vDB + "' and table_name='" + vTable + "' and column_name='" + vColumn + "' and user_name='" + vUser + "'", False)
         return vCheck[0][0]          


@plugin_function("table_encryption.decrypt_table_column")
def decrypt_table_column():
    if _strict_root():
       vUser, host, port = _user_identity()
       vUser = shell.prompt("Please provide user_name: ")
       vPassword = shell.prompt('Please provide user password: ',{"type":"password"})
       x=shell.get_session()
       try:
          y=shell.open_session(vUser + "@localhost:" + str(port), vPassword)
          shell.set_session(y)
       except:
          shell.set_session(x)
          print("\nUnable to connect using " + vUser + " \n")
          return
       vDB = shell.prompt("Please provide DB NAME: ")
       vTable = shell.prompt("Please provide TABLE NAME: ")
       vColumn = shell.prompt("Please provide COLUMN NAME: ")
       shell.set_session(x)
       vName = _check_key_id(vUser, vDB, vTable, vColumn)
       shell.set_session(y)
       if _check_object(vUser, vDB, vTable, vColumn, vName) and vName != "False":
         try:
            _run_sql("alter table " + vDB + "." + vTable + " add column " + vColumn + "_decrypt varchar(100) after " + vColumn + ", algorithm=inplace;", False)
            _run_sql("update " + vDB + "." + vTable + " set " + vColumn + "_decrypt=aes_decrypt(unhex(" + vColumn + "), hex(keyDB.get_shared_key_" + vUser + "_" + vName + "()));", False)
            _run_sql("alter table " + vDB + "." + vTable + " drop column " + vColumn + ", algorithm=inplace;", False)
            _run_sql("alter table " + vDB + "." + vTable + " rename column " + vColumn + "_decrypt to " + vColumn + ", algorithm=inplace;", False)
            shell.set_session(x)
            _run_sql("drop trigger " + vDB + ".before_" + vTable + "_" + vColumn + "_insert", False)
            _run_sql("drop trigger " + vDB + ".before_" + vTable + "_" + vColumn + "_update", False)
            _run_sql("delete from keyDB.table_column_encryption where schema_name='" + vDB + "' and table_name='" + vTable + "' and column_name='" + vColumn + "'", False) 
         except:
            print("\nUnable to continue, check table user privileges\n")
            shell.set_session(x)
```
