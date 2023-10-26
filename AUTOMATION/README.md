# Automation
## Create required directory
Create ftp target directory
```
mkdir -p /shared/mysql_customer_orders/delivery_orders
```
Create script directory
```
mkdir /home/opc/script
```
## Put all automation script and template
Create lakehouse template /home/opc/script/ddl_template.txt using command: "vi /home/opc/script/ddl_template.txt" <\br>
Switch to editing mode by press ESC-i <\br>
Copy and paste the following content into ddl_template.txt
```
SET @db_list = '["v_database_name"]';

SET @dl_tables = '[{
"db_name": "v_database_name",
"tables": [{
    "table_name": "v_table_name",
    "dialect": 
       {
       "format": "csv",
       "field_delimiter": "\\t",
       "record_delimiter": "\\n"
       },
    "file": [{"region": "v_region", "namespace": "v_namespace", "bucket": "v_bucket","prefix": "v_text_prefix/"}]
}] }]';
SET @options = JSON_OBJECT('mode', 'normal',  'policy', 'disable_unsupported_columns',  'external_tables', CAST(@dl_tables AS JSON));
CALL sys.heatwave_load(@db_list, @options);
```
Create /home/opc/script/auto_ddl.sh with the following content, please change database server_ip, database_user, and database_port parameter accordingly
```
#!/bin/bash

server_ip="hw"
database_user="admin"
database_port="3306"

database_name=`mysqlsh $database_user@$server_ip --sql -e "select database_name from lakehouse_metadata.ddl_master where dir_name='$1'" | grep -v database_name`

table_name=`mysqlsh $database_user@$server_ip --sql -e "select table_name from lakehouse_metadata.ddl_master where dir_name='$1'" | grep -v table_name`

c=`echo "select count(1) is_exist from information_schema.TABLES where table_name='$table_name' and table_schema='$database_name';"`
x=`mysqlsh $database_user@$server_ip:$database_port --sql -e "$c" | grep -v is_exist` 

if [ $x == "0" ]; then
	c=`echo "select text_format from lakehouse_metadata.ddl_master where database_name='$database_name' and table_name='$table_name'"`
	x=`mysqlsh $database_user@$server_ip:$database_port --sql -e "$c"`
	text_format=`echo $x | awk '{print $2}'`

	c=`echo "select field_delimiter from lakehouse_metadata.ddl_master where database_name='$database_name' and table_name='$table_name'"`
	x=`mysqlsh $database_user@$server_ip:$database_port --sql -e "$c"`
	field_delimiter=`echo $x | awk '{print $2}'`

	c=`echo "select record_delimiter from lakehouse_metadata.ddl_master where database_name='$database_name' and table_name='$table_name'"`
	x=`mysqlsh $database_user@$server_ip:$database_port --sql -e "$c"`
	record_delimiter=`echo $x | awk '{print $2}'`

	c=`echo "select region from lakehouse_metadata.ddl_master where database_name='$database_name' and table_name='$table_name'"`
	x=`mysqlsh $database_user@$server_ip:$database_port --sql -e "$c"`
	region=`echo $x | awk '{print $2}'`

	c=`echo "select namespace from lakehouse_metadata.ddl_master where database_name='$database_name' and table_name='$table_name'"`
	x=`mysqlsh $database_user@$server_ip:$database_port --sql -e "$c"`
namespace=`echo $x | awk '{print $2}'`

	c=`echo "select bucket from lakehouse_metadata.ddl_master where database_name='$database_name' and table_name='$table_name'"`
	x=`mysqlsh $database_user@$server_ip:$database_port --sql -e "$c"`
bucket=`echo $x | awk '{print $2}'`

	c=`echo "select text_prefix from lakehouse_metadata.ddl_master where database_name='$database_name' and table_name='$table_name'"`
	x=`mysqlsh $database_user@$server_ip:$database_port --sql -e "$c"`
	text_prefix=`echo $x | awk '{print $2}'`


	cat ddl_template.txt | sed "s/v_database_name/$database_name/g" | sed "s/v_table_name/$table_name/g" | sed "s/v_text_format/$text_format/g" | sed "s/v_field_delimiter/$field_delimiter/g" | sed "s/v_record_delimiter/$record_delimiter/g" | sed "s/v_region/$region/g" | sed "s/v_bucket/$bucket/g" | sed "s/v_text_prefix/$text_prefix/g" | sed "s/v_namespace/$namespace/g" > create.sql
	mysqlsh $database_user@$server_ip:$database_port --sql -e "source create.sql"
	rm create.sql

	echo "alter table $database_name.$table_name secondary_unload;" > change_column.sql

	mysqlsh $database_user@$server_ip:$database_port --sql -e "select concat('alter table ',a.table_schema,'.',a.table_name,' RENAME COLUMN ', a.column_name, ' to ', col_name, ';') sql_text  from information_schema.columns a, lakehouse_metadata.ddl_master b, lakehouse_metadata.ddl_detail c where a.table_schema='mysql_customer_orders' and a.table_name='delivery_orders' and a.table_schema=b.database_name and a.table_name=b.table_name and b.i=c.i and c.col_sq=a.ordinal_position;" | grep -v sql_text >> change_column.sql

	echo "alter table $database_name.$table_name secondary_load;" >> change_column.sql

	mysqlsh $database_user@$server_ip:$database_port --sql -e "source change_column.sql"
	rm change_column.sql
else
        echo "Refresh table $database_name.$table_name..."
        c=`echo "alter table $database_name.$table_name secondary_unload; alter table $database_name.$table_name secondary_load;"`
        mysqlsh $database_user@$server_ip:$database_port --sql -e "$c"
fi
```
Create /home/opc/script/auto_detect with the following content:
```
#!/bin/bash

TARGET=/shared/mysql_customer_orders/delivery_orders
PROCESSED=/home/opc/object_storage/test
WORKING_DIR=/home/opc/script

while true; do
	diff <(find $TARGET/ -exec basename {} \; | sort) <(find $PROCESSED/ -exec basename {} \; | sort) | grep ".csv" | awk '{print $2}' > $WORKING_DIR/list_working_file.lst
	if [ `cat $WORKING_DIR/list_working_file.lst | wc -l` -gt 0 ]; then
		echo "Found file difference ..."
		while read p; do	
			if [ ! -f $PROCESSED/$p ]; then
				echo "Checking $TARGET/$p .."
                		x=`ls -altr $TARGET/$p | grep $p | awk '{print $5}'`
                		y=0
				while [ $x != $y ]; do
					echo "Wait for uploading $p completed .."
					sleep 20
					y=$x
					x=`ls -altr $TARGET/$p | grep $p | awk '{print $5}'`
				done
				echo "Process uploading $p is completed, upload to Object Storage"
                		cp $TARGET/$p $PROCESSED/$p
				echo "Process uploading $p to Object Storage is completed, load to HeatWave"
				$WORKING_DIR/auto_ddl.sh $PROCESSED/	
				echo "Process loading to MySQL HeatWave is completed for file $p"
        		fi
		done <$WORKING_DIR/list_working_file.lst
	fi
	rm $WORKING_DIR/list_working_file.lst
done;
```
## Install database schema for lakehouse metadata
Login to MySQL using MySQL Shell
```
mysqlsh <user>@<host>:<port> --sql
```
Make sure you enter the right password and choose "Y" when asked to save password. </br>
Create database schema:
```
create database lakehouse_metadata;
```
Create table ddl master
```
CREATE TABLE `ddl_master` (
  `i` int NOT NULL AUTO_INCREMENT,
  `database_name` varchar(30) DEFAULT NULL,
  `table_name` varchar(20) DEFAULT NULL,
  `text_format` varchar(10) DEFAULT NULL,
  `field_delimiter` varchar(10) DEFAULT NULL,
  `record_delimiter` varchar(10) DEFAULT NULL,
  `region` varchar(20) DEFAULT NULL,
  `namespace` varchar(50) DEFAULT NULL,
  `bucket` varchar(10) DEFAULT NULL,
  `text_prefix` varchar(10) DEFAULT NULL,
  `dir_name` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`i`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```
Create table ddl_detail
```
CREATE TABLE `ddl_detail` (
  `i` int DEFAULT NULL,
  `col_sq` int DEFAULT NULL,
  `col_name` varchar(20) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```
Populate table accordingly.
## Run automation
Execute the following script on backgroumd mode:
```
/home/opc/script/auto_detect.sh &
```
