#!/bin/bash
set -e
echo "--- Starting configuration ----------"
sudo apt update && sudo apt upgrade -y

maingroup="www-data"
mainusr="wpusr_root"

# Firewall und OpenSSH
sudo apt install -y openssh-server ufw
sudo ufw reset
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw --force enable

cd /var
sudo mkdir www
cd www
sudo mkdir html


#echo phase---------------------addusr abgeschlossen 

#Nginx installieren & Firewall anpassen
sudo apt install htop
sudo apt install neovim 
sudo apt install net-tools

# PHP PPA hinzufügen & installieren
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update
sudo apt install -y php8.3-fpm php8.3-common php8.3-mysql \
php8.3-xml php8.3-intl php8.3-curl php8.3-gd \
php8.3-imagick php8.3-cli php8.3-dev php8.3-imap \
php8.3-mbstring php8.3-opcache php8.3-redis \
php8.3-soap php8.3-zip 

# Jetzt PHP-Konfig anpassen (erst jetzt sind die Dateien da)
file_line_check() {
keyword="$1"
file="$2"
new_line="$3"

if grep -q "$keyword" "$file"; then
    sudo sed -i "/$keyword/c\\$new_line" "$file"
fi
}

file=( "/etc/php/8.3/fpm/pool.d/www.conf" "/etc/php/8.3/fpm/php.ini" )

keywords=( "user =" "group =" "listen.owner =" "listen.group =" "upload_max_filesize =" "post_max_size =" )

new_lines=( "user = www-data" "group = www-data" "listen.owner = www-data" "listen.group = www-data" "upload_max_filesize = 20M" "post_max_size = 80M" )

for ((f=0; f<2; f++)); do
filename="${file[$f]}"
if [[ "$filename" == *php.ini ]]; then
    for ((k=0; k<2; k++)); do
    file_line_check "${keywords[4+$k]}" "$filename" "${new_lines[4+$k]}"
    done
else
    for ((k=0; k<4; k++)); do
    file_line_check "${keywords[$k]}" "$filename" "${new_lines[$k]}"
    done
fi
done

sudo systemctl restart php8.3-fpm

# Nginx config neu laden
sudo apt-get remove --purge -y nginx nginx-common nginx-full
sudo rm -rf /etc/nginx /var/log/nginx /var/cache/nginx /usr/local/nginx /usr/local/src/nginx-*
sudo rm -f /etc/systemd/system/nginx.service
sudo systemctl daemon-reload

# 1. System vorbereiten: Build-Tools und libs
sudo apt update
sudo apt install -y build-essential gcc make cmake git libpcre3 libpcre3-dev zlib1g zlib1g-dev libssl-dev

# 2. Quellen herunterladen
#echo "_--------------------------------------------------------------------"
cd /usr/local/src
sudo wget http://nginx.org/download/nginx-1.27.0.tar.gz
sudo tar zxvf nginx-1.27.0.tar.gz
echo "_---------------------111111111111111111111111111111-----------------------------------------------"
sudo git clone https://github.com/google/ngx_brotli.git
cd ngx_brotli
sudo git submodule update --init
#echo "_-----------------------22222222222222222222222222222222222---------------------------------------------"

sudo mkdir /usr/local/src/ngx_brotli/deps/brotli/build
cd /usr/local/src/ngx_brotli/deps/brotli/build
sudo cmake ..
sudo make
#echo "_--------------------------3333333333333333333333333333333333-----------------------------------------"
# 3. Nginx konfigurieren mit Brotli Modul
cd /usr/local/src/nginx-1.27.0
sudo ./configure --with-http_ssl_module --with-http_v2_module --add-dynamic-module=/usr/local/src/ngx_brotli --with-cc-opt="-I/usr/local/src/ngx_brotli/deps/brotli/include" --with-ld-opt="-L/usr/local/src/ngx_brotli/deps/brotli/build"

# 4. Module bauen
sudo make modules

sudo mkdir /usr/local/nginx
sudo mkdir /usr/local/nginx/modules/

# 5. Module ins nginxf Verzeichnis kopieren
sudo cp objs/ngx_http_brotli_filter_module.so /usr/local/nginx/modules/
sudo cp objs/ngx_http_brotli_static_module.so /usr/local/nginx/modules/

sudo make install 

# 6. Rechte setzen für nginx Verzeichnisse (Logs etc.)
sudo mkdir -p /usr/local/nginx/logs
sudo chown -R www-data:www-data /usr/local/nginx

#7
sudo tee /etc/systemd/system/nginx.service > /dev/null <<EOF
[Unit]
Description=NGINX Webserver
After=network.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/bin/kill -s QUIT \$MAINPID

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable nginx
sudo systemctl start nginx

sudo git clone https://github.com/Alex-bltr/nginx-configuration.git

cd nginx-configuration
cd nginx

sudo mv cache /usr/local/nginx/conf
sudo mv global /usr/local/nginx/conf
sudo mv restrictions /usr/local/nginx/conf
sudo mv snippets /usr/local/nginx/conf
sudo rm /usr/local/nginx/conf/nginx.conf

sudo mv nginx.conf /usr/local/nginx/conf

cd ~
sudo rm -rf nginx-configuration

# WP-CLI installieren
sudo curl -f -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar || { echo "WP-CLI Download failed"; exit 1; }
sudo chmod +x wp-cli.phar
sudo mv wp-cli.phar /usr/local/bin/wp

# MySQL Server installieren
sudo apt install -y mysql-server

# mysql_secure_installation ist interaktiv! Falls automatisieren, extra Script nötig
sudo mysql_secure_installation

# MySQL Datenbank und Benutzer anlegen
sudo mysql -e "CREATE DATABASE wpdbmain CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_520_ci;"

read -s -p "Datenbank-Passwort für $mainusr: " dbpass
echo

sudo mysql -e "CREATE USER '$mainusr'@'localhost' IDENTIFIED BY '$dbpass';"
sudo mysql -e "GRANT ALL PRIVILEGES ON wpdbmain.* TO '$mainusr'@'localhost';"
sudo mysql -e "FLUSH PRIVILEGES;"

cd /var/www/html
sudo wp core download --allow-root
sudo chown -R www-data:www-data /var/www/html

cd /var
cd cache
sudo mkdir nginx
cd nginx
sudo mkdir fastcgi_cache
systemctl stop nginx
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
sudo certbot certonly --standalone -d scaveo.de
systemctl start nginx

timedatectl set-timezone Europe/Berlin



curl -s https://install.crowdsec.net | sudo sh
apt install crowdsec -y
sudo cscli console enroll cmcqf8o30000fjs02nmbr8o8i



echo "/usr/local/nginx/sbin/nginx -s reload"



echo "Server config successful"
