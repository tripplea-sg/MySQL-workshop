# TDE with InnoDB Cluster
Securing connection 
```
# check PRIMARY node, let say on 3306
mysql -uroot -h::1 
GRANT GROUP_REPLICATION_STREAM ON *.* TO `gradmin`@`%`
set persist group_replication_ip_allowlist='127.0.0.1';
set persist require_secure_transport=on;
exit

mysql -uroot -h::1 -P3307
set persist group_replication_ip_allowlist='127.0.0.1';
set persist require_secure_transport=on;
exit

mysql -uroot -h::1 -P3308
set persist group_replication_ip_allowlist='127.0.0.1';
set persist require_secure_transport=on;
exit
```
Add the following to the my.cnf of 3306
```
early-plugin-load=keyring_encrypted_file.so
keyring_encrypted_file_data=/home/opc/db/3306/data/keyring
keyring_encrypted_file_password=password
```
Add the following to the my.cnf of 3307
```
early-plugin-load=keyring_encrypted_file.so
keyring_encrypted_file_data=/home/opc/db/3307/data/keyring
keyring_encrypted_file_password=password
```
Add the following to the my.cnf of 3308
```
early-plugin-load=keyring_encrypted_file.so
keyring_encrypted_file_data=/home/opc/db/3308/data/keyring
keyring_encrypted_file_password=password
```
Rolling restart:
```
mysql -uroot -h::1 -e -P3306 "restart"

# check cluster status, repeat until 3306 available
mysqlsh gradmin:grpass@localhost:3307 -- cluster status
mysql -uroot -h::1 -e -P3307 "restart"

# check cluster status, repeat until 3307 available
mysqlsh gradmin:grpass@localhost:3306 -- cluster status
mysql -uroot -h::1 -e -P3308 "restart"
mysqlsh gradmin:grpass@localhost:3307 -- cluster setPrimaryInstance localhost:3306
```
Install keyring udf
```
mysql -uroot -h127.0.0.1 -e "INSTALL PLUGIN keyring_udf SONAME 'keyring_udf.so'"
mysql -uroot -h127.0.0.1 -P3307 -e "set global super_read_only=off; INSTALL PLUGIN keyring_udf SONAME 'keyring_udf.so'; set global super_read_only=on"
mysql -uroot -h127.0.0.1 -P3308 -e "set global super_read_only=off; INSTALL PLUGIN keyring_udf SONAME 'keyring_udf.so'; set global super_read_only=on"

# login to 3306
mysql -uroot -h127.0.0.1

CREATE FUNCTION keyring_key_generate RETURNS INTEGER
  SONAME 'keyring_udf.so';
CREATE FUNCTION keyring_key_fetch RETURNS STRING
  SONAME 'keyring_udf.so';
CREATE FUNCTION keyring_key_length_fetch RETURNS INTEGER
  SONAME 'keyring_udf.so';
CREATE FUNCTION keyring_key_type_fetch RETURNS STRING
  SONAME 'keyring_udf.so';
CREATE FUNCTION keyring_key_store RETURNS INTEGER
  SONAME 'keyring_udf.so';
CREATE FUNCTION keyring_key_remove RETURNS INTEGER
  SONAME 'keyring_udf.so';

create user apps@'%' identified by 'apps';
grant all privileges on *.* to apps@'%';
grant execute on *.* to apps@'%';
exit

# login as apps
mysql -uapps -papps -h::1
SELECT keyring_key_generate('MyKey', 'AES', 32);
exit
```
Copy keyring encrupted file from 3306 to 3307 and 3308
```
cp /home/opc/db/3306/data/keyring /home/opc/db/3307/data/keyring
cp /home/opc/db/3306/data/keyring /home/opc/db/3308/data/keyring
```
Restart 3307 and 3308
```
mysql -uroot -h::1 -e -P3307 "restart"

# check cluster status, repeat until 3307 available
mysqlsh gradmin:grpass@localhost:3306 -- cluster status
mysql -uroot -h::1 -e -P3308 "restart"
```
## Encrypt a table
Download world_x.sql
```
wget https://downloads.mysql.com/docs/world_x-db.zip
```
Unzip world_x.sql
```
unzip world_x-db.zip
```
Upload world_x to database
```
mysql -uroot -h127.0.0.1 -e "source /home/opc/world_x-db/world_x.sql"
```
See content of table world_x.city without login to mysql (some values are in clear text):
```
strings -a /home/opc/db/3306/data/world_x/city.ibd
strings -a /home/opc/db/3307/data/world_x/city.ibd
strings -a /home/opc/db/3308/data/world_x/city.ibd
```
Login to mysql
```
mysql -uroot -h127.0.0.1 
```
Encrypt table world_x.city:
```
alter table world_x.city encryption='Y';
exit;
```
Check content of table world_x.city without login to mysql (all values are encrypted):
```
strings -a /home/opc/db/3306/data/world_x/city.ibd
strings -a /home/opc/db/3307/data/world_x/city.ibd
strings -a /home/opc/db/3308/data/world_x/city.ibd
```
