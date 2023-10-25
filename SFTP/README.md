# Mouting Object Storage

## On OCI Console
In the Oracle Cloud Infrastructure Console, click the Profile icon in the top-right corner, and select User Settings </br>
Click Customer Secret Keys, and then click Generate Secret Key </br>
Give the key a meaningful name (for example, ftp), and then click Generate Secret Key </br>
Copy and save the secret key because it won't be shown again. </br>
The S3 credentials are created by using an access key and the secret key. </br>
The access key is displayed in the Customer Secret Keys area of the Console.

## On OCI Compute
Install s3fs
```
sudo yum install s3fs-fuse
```
Set credential
```
echo "{ACCESS_KEY_ID value}:{SECRET_ACCESS_KEY value}" > /home/opc/.passwd-s3fs
chmod 600 /home/opc/.passwd-s3fs 
```
Mount Object Storage Bucket
```
mkdir /home/opc/object_storage

sudo chmod +x /usr/bin/fusermount

s3fs [bucket] [destination directory] -o endpoint=[region] -o passwd_file=${HOME}/.passwd-s3fs -o url=https://[namespace].compat.objectstorage.[region].oraclecloud.com/ -onomultipart -o use_path_request_style 
```
For example:
```
s3fs test /home/opc/object_storage -o endpoint=ap-singapore-1 -o passwd_file=/home/opc/.passwd-s3fs -o url=https://mynamespace.compat.objectstorage.myregion.oraclecloud.com/ -onomultipart -o use_path_request_style 
```
## Test your Filezilla
Refer to this online reference: </br>
https://www.bettertechtips.com/how-to/use-scp-filezilla/
## Install inotify for automation
```
sudo yum install -y inotify-tools-3.14-9.el7.x86_64
```




