# Enterprise Encryption with InnoDB Cluster
Install plugin
```
# login to 3306
mysql -uroot -h::1 --skip-binary-as-hex

CREATE FUNCTION asymmetric_decrypt RETURNS STRING SONAME 'openssl_udf.so';
CREATE FUNCTION asymmetric_derive RETURNS STRING SONAME 'openssl_udf.so';
CREATE FUNCTION asymmetric_encrypt RETURNS STRING SONAME 'openssl_udf.so';
CREATE FUNCTION asymmetric_sign RETURNS STRING SONAME 'openssl_udf.so';
CREATE FUNCTION asymmetric_verify RETURNS INTEGER SONAME 'openssl_udf.so';
CREATE FUNCTION create_asymmetric_priv_key RETURNS STRING SONAME 'openssl_udf.so';
CREATE FUNCTION create_asymmetric_pub_key RETURNS STRING SONAME 'openssl_udf.so';
CREATE FUNCTION create_dh_parameters RETURNS STRING SONAME 'openssl_udf.so';
CREATE FUNCTION create_digest RETURNS STRING SONAME 'openssl_udf.so';

exit;
```
Set generate invisible primary key
```
mysql -uroot -h::1 -e "set persist sql_generate_invisible_primary_key=1;"
mysql -uroot -h::1 -P3307 -e "set persist sql_generate_invisible_primary_key=1;"
mysql -uroot -h::1 -P3308 -e "set persist sql_generate_invisible_primary_key=1;"
```
Login to 3306 as apps and encrypt table
```
mysql -uapps -papps -h::1 --skip-binary-as-hex
create table world_x.city_info_encrypted as select id, name, countrycode, district, hex(aes_encrypt(info, hex(keyring_key_fetch('MyKey')))) info from world_x.city;
exit;
```
Query table on 3306
```
mysql -uapps -papps -h::1 --skip-binary-as-hex -e "select id, name, countrycode, district, aes_decrypt(unhex(info), hex(keyring_key_fetch('MyKey'))) from world_x.city_info_encrypted;"
```
Query table on 3307
```
mysql -uapps -papps -h::1 --skip-binary-as-hex -e "select id, name, countrycode, district, aes_decrypt(unhex(info), hex(keyring_key_fetch('MyKey'))) from world_x.city_info_encrypted;"
```
Query table on 3308
```
mysql -uapps -papps -h::1 --skip-binary-as-hex -e "select id, name, countrycode, district, aes_decrypt(unhex(info), hex(keyring_key_fetch('MyKey'))) from world_x.city_info_encrypted;"
```
Install Query Rewrite
```
# connect to primary node
CREATE DATABASE IF NOT EXISTS query_rewrite;

CREATE TABLE IF NOT EXISTS query_rewrite.rewrite_rules (
  id INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
  pattern VARCHAR(5000) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL,
  pattern_database VARCHAR(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin,
  replacement VARCHAR(5000) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL,
  enabled ENUM('YES', 'NO') CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL
    DEFAULT 'YES',
  message VARCHAR(1000) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin,
  pattern_digest VARCHAR(64),
  normalized_pattern VARCHAR(100)
) DEFAULT CHARSET = utf8mb4 ENGINE = INNODB;

INSTALL PLUGIN rewriter SONAME 'rewriter.so';

# connect to each other node
set global super_read_only=off;
INSTALL PLUGIN rewriter SONAME 'rewriter.so';
set global super_read_only=on;

# connect to primary node
DELIMITER //

CREATE PROCEDURE query_rewrite.flush_rewrite_rules()
BEGIN
  DECLARE message_text VARCHAR(100);
  COMMIT;
  SELECT load_rewrite_rules() INTO message_text;
  IF NOT message_text IS NULL THEN
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = message_text;
  END IF;
END //

DELIMITER ;
```
