#!/bin/bash

# Exit on command errors and treat unset variables as an error
set -eu

app=$YNH_APP_INSTANCE_NAME

# Retrieve arguments
domain=$YNH_APP_ARG_DOMAIN
path=$YNH_APP_ARG_PATH
admin=$YNH_APP_ARG_ADMIN
#language=$YNH_APP_ARG_LANGUAGE

# Source YunoHost helpers
. /usr/share/yunohost/helpers

# Save app settings
ynh_app_setting_set "$app" admin "$admin"
#ynh_app_setting_set "$app" language "$language"

# Check domain/path availability
sudo yunohost app checkurl "${domain}${path}" -a "$app" \
    || ynh_die "Path not available: ${domain}${path}"

# install jetty 	
sudo apt-get install jetty8 libjetty-extra -y -qq

## activer le démarrage de jetty
sudo sed sed -i "s/NO_START=1/NO_START=0/g" /etc/default/jetty8

# installation des dépendances pour la compilation de guacamole
SRC_DIR="~/gacamole_src"
mkdir $SRC_DIR

fonction check_dep () {
if ! ynh_package_is_installed $DEP 
  then
  echo "$DEP" >> $SRC_DIR/src_dep.lst
fi
}

if [ -f $SRC_DIR/src_dep.lst ]
	then 
	rm -rf $SRC_DIR/src_dep.lst
fi
DEP=libcairo2-dev check_dep
DEP=libpng12-dev check_dep
DEP=libossp-uuid--dev check_dep
DEP=libfreerdp-dev check_dep
DEP=libpango1.0-dev check_dep
DEP=libssh2-1-dev check_dep
DEP=libtelnet-dev check_dep
DEP=libvncserver-dev check_dep
DEP=libpulse-dev check_dep
DEP=libssl-dev check_dep
DEP=libvorbis-dev check_dep
DEP=libwebp-dev check_dep

if [ -f $SRC_DIR/src_dep.lst ]
	then 
		sudo apt-get -y install `cat $SRC_DIR/src_dep.lst` -y -qq
fi

wget -O $SRC_DIR/guacamole-0.9.9.war http://downloads.sourceforge.net/project/guacamole/current/binary/guacamole-0.9.9.war
wget -O $SRC_DIR/guacamole-server-0.9.9.tar.gz http://sourceforge.net/projects/guacamole/files/current/source/guacamole-server-0.9.9.tar.gz 
wget -O $SRC_DIR/guacamole-auth-jdbc-0.9.9.tar.gz http://sourceforge.net/projects/guacamole/files/current/extensions/guacamole-auth-jdbc-0.9.9.tar.gz
wget -O $SRC_DIR/mysql-connector-java-5.1.38.tar.gz http://dev.mysql.com/get/Downloads/Connector/j/mysql-connector-java-5.1.38.tar.gz
cd $SRC_DIR
tar -xzf guacamole-server-0.9.9.tar.gz
tar -xzf guacamole-auth-jdbc-0.9.9.tar.gz
tar -xzf mysql-connector-java-5.1.38.tar.gz
sudo mkdir -p /etc/guacamole/{lib,extensions}
sudo mkdir -p /etc/guacamole/lib
cd guacamole-server-0.9.9
./configure --with-init-dir=/etc/init.d
make
sudo make install
sudo ldconfig
sudo systemctl enable guacd
sudo ln -s /etc/guacamole/guacamole.war /var/lib/jetty8/webapps/
sudo cp mysql-connector-java-5.1.38/mysql-connector-java-5.1.38-bin.jar /etc/guacamole/lib/
sudo cp guacamole-auth-jdbc-0.9.9/mysql/guacamole-auth-jdbc-mysql-0.9.9.jar /etc/guacamole/extensions/
sudo echo "mysql-hostname: localhost" >> /etc/guacamole/guacamole.properties
sudo echo "mysql-port: 3306" >> /etc/guacamole/guacamole.properties
sudo echo "mysql-database: guacamole" >> /etc/guacamole/guacamole.properties
sudo echo "mysql-username: guacamole" >> /etc/guacamole/guacamole.properties

## manual initialisation SQL (remove in the futur)
#mysql -u root -pMYSQLROOTPASSWORD
#create database guacamole_db;
#create user 'guacamole_user'@'localhost' identified by 'PASSWORD';
#GRANT SELECT,INSERT,UPDATE,DELETE ON guacamole_db.* TO 'guacamole_user'@'localhost';
#flush privileges;
#quit
#cat guacamole-auth-jdbc-0.9.9/mysql/schema/*.sql | mysql -u root -pMYSQLROOTPASSWORD guacamole_db


# This is where you will want to change "PASSWORD"
echo "mysql-password: PASSWORD" >> /etc/guacamole/guacamole.properties
rm -rf /usr/share/jetty8/.guacamole
ln -s /etc/guacamole /usr/share/jetty8/.guacamole

# Restart Tomcat Service
service jetty8 restart

# Copy source files
final_path=/opt/$app
sudo mkdir -p $final_path


# Set permissions to app files
# you may need to make some file and/or directory writeable by www-data (nginx user)
sudo chown -R root:root $final_path

# If your app use a MySQL database you can use these lines to bootstrap
# a database, an associated user and save the password in app settings.
#
# # Generate MySQL password and create database
dbuser=$app
dbname=$app
dbpass=$(ynh_string_random 12)
ynh_app_setting_set "$app" mysqlpwd "$dbpass"
ynh_mysql_create_db "$dbname" "$dbuser" "$dbpass"
#
# # Load initial SQL into the new database
ynh_mysql_connect_as "$dbuser" "$dbpass" "$dbname" < "$SRC_DIR/guacamole-auth-jdbc-0.9.9/mysql/schema/*.sql"

# Modify Nginx configuration file and copy it to Nginx conf directory
sed -i "s@YNH_WWW_PATH@$path@g" ../conf/nginx.conf
sed -i "s@YNH_WWW_ALIAS@$final_path/@g" ../conf/nginx.conf
# If a dedicated php-fpm process is used:
# Don't forget to modify ../conf/nginx.conf accordingly or your app will not work!
#
# sudo sed -i "s@YNH_WWW_APP@$app@g" ../conf/nginx.conf
sudo cp ../conf/nginx.conf /etc/nginx/conf.d/$domain.d/$app.conf

# clean
if [ -f $SRC_DIR/src_dep.lst ]
	then 
		sudo apt-get purge `cat $SRC_DIR/src_dep.lst` -y -qq
		sudo apt-get autoremove --purge -y -qq
fi
rm -rf $SRC_DIR

sudo service nginx reload
