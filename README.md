![image](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/3d84a856-acea-49f1-832e-f40975698ae6)
# M300_ST20b
# Inhaltsverzeichnis
1. [Einführung](#einleitung)
	1. [Systemumgebung](#Systemumgebung)
3. [VAGRANT File](#VAGRANT)
	1. [Thema 1](#thema1)
	1. [Thema 2](#thema2)
	1. [Thema 3](#thema3)	
4. [Schluß](#skript)
# Einführung


Im Modul 300 geht es darum, eine Service Umgebung völlig automatisiert zu erstellen. Dies bietet viele Vorteile und spart im End-Effekt viel Zeit. Ich habe mich hierbei bei einem Kollegen von der Parallelklasse inspiriert. Also beschloss ich es Ihm ähnlich zu machen, dabei hatte ich immer das Ziel im Hintergrund nicht jemanden zu kopieren und nicht eigene Ideen einzubringen. Meine Idee war also, Eine Ubuntu VM mit einer Umgebung bestehend aus einem Webserver worauf eine Monitoring Software läuft. Diese Website ist auch je nach Zeit beliebig erweiterbar mit zb. einer Datenbank. 

# Systemumgebung
Um sich die Umgebung besser Vorstellen zu können: 

![M300_Systemumgebung](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/e01cc04c-293e-4a71-a8c4-2adc107539fe)

![M300_Systemumgebung](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/9a771739-2d7b-4b1e-9677-7870d9ace3bb)

Um auf die erstellte Umgebung Zugreifen zu können, kann ich auf dem Host-System https://localhost:3346 öffnen. 

Damit die Umgebung aber überhaupt funktioniert, muss man Zuerst im Gitbash das entsprechende Skript ausführen. 

Dafür wechsel ich in mein persönliches verzeichnis mit dem Vagrant file.
```
$ cd ~/Desktop/Modul-to-Upload/Modul\ 300/myvagrant/
```
Und lasse die Umgebung nach dem Vagrant file erstellen. 
```
$ Vagrant up
```

Nach etwa 5 min ist die Umgebung kreirt und man kann im Gitbash per SSH zugreifen. 
```
$ Vagrant ssh
```

Weitere wichtige Befehle sind: 
```
$ Vagrant status    #Status der VM
$ Vagrant halt      #pausieren der VM
$ Vagrant destory   #Löschen der VM
```

# VAGRANT File
```
#Erstellung VM 
time_zone = "Europe/Zurich"

Vagrant.configure("2") do |config| # Konfiguration der Vagrant-Box
  config.vm.box = "generic/ubuntu1804" # Verwendet die Ubuntu 18.04-Box
  config.vm.network "forwarded_port", guest: 80, host: 3446, auto_correct: true # Weiterleitungsport von 80 auf 3446
  config.ssh.forward_agent = true
  config.vm.synced_folder ".", "/vagrant", type: "virtualbox", # Synchronisiere das aktuelle Verzeichnis mit dem /vagrant-Verzeichnis in der VM
  owner: "www-data", group: "www-data" # Gruppen und Benutzer die Folder festlegen

  config.vm.provider "virtualbox" do |vb| # Provider Konfiguration
    vb.name = "M300_LB02_Leandro" # VM Name
    vb.memory = 3072 # RAM in MB
    vb.cpus = 3 # Anzahl Prozessoren
    vb.customize ["guestproperty", "set", :id, "/VirtualBox/GuestAdd/VBoxService/--timesync-set-threshold", 10000]
    vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    vb.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
  end


config.vm.provision "shell" do |s|
    s.args = [time_zone]
    s.inline = <<-SHELL  

#Software und Sicherheits Konfiguration

deploy_start_time=$(date +%s)

sudo timedatectl set-timezone Europe/Zurich
apt-get update -q # Updated ganzes System

#Firewall 
sudo apt-get install -y ufw 
sudo ufw --force enable
sudo ufw allow from 10.0.2.2 to any port 22   # Port 22 (SSH) nur für den Host (wo die VM laufen) öffnen 
sudo ufw allow 80    # Port 80 (HTTP) öffnen für alle
sudo ufw status
sudo systemctl start ssh



#Sonstige Programme
apt-get install -q -y vim git # Vim und Git installieren ins /var/www
apt-get install -q -y apache2 # Installieren weiterer Software
apt-get install -q -y php7.2 libapache2-mod-php7.2
apt-get install -q -y php7.2-curl php7.2-gd php7.2-mbstring php7.2-mysql php7.2-xml php7.2-zip php7.2-bz2 php7.2-intl
apt-get install -q -y mariadb-server mariadb-client
apt-get install -y ldap-utils
a2enmod rewrite headers
ldapsearch -x -LLL -H ldap://your-ldap-server -D "cn=admin,dc=example,dc=com" -w yourpassword -b "ou=users,dc=example,dc=com" "(objectclass=*)" cn email
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod proxy_balancer
sudo a2enmod lbmethod_byrequests
sudo systemctl restart apache2

#Erstellung  Benutzer Leandro & User Root
sudo useradd -m -s /bin/bash leandro  
echo 'leandro:Admin1234!' | sudo chpasswd  

#Erstellung neuer ROOT User
sudo useradd -m -s /bin/bash leandro-root
echo 'leandro-root:Admin1234!' | sudo chpasswd
sudo usermod -aG sudo leandro-root

# Dateiberechtigungen festlegen
sudo chown -R leandro:www-data /var/www/html
sudo chmod -R 755 /var/www/html 

# SSH-Zugriff für den Benutzer aktivieren
sudo mkdir /home/leandro/.ssh  
sudo cp /root/.ssh/authorized_keys /home/leandro/.ssh/  
sudo chown -R leandro:leandro /home/leandro/.ssh 
sudo chmod 700 /home/leandro/.ssh 
sudo chmod 600 /home/leandro/.ssh/authorized_keys

# Optional: SSH-Zugriff für den Benutzer über Passwort aktivieren
sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config  # Passwortauthentifizierung aktivieren
sudo systemctl restart sshd  


# Vagrant-Ordner als Apache-Root-Ordner festlegen und lokal speichern
dir='/vagrant/www'
if [ ! -d "$dir" ]; then
  mkdir "$dir"
fi
if [ ! -L /var/www/html ]; then
  rm -rf /var/www/html
  ln -fs "$dir" /var/www/html
fi
cd "$dir"

file='/etc/apache2/sites-available/dev.conf'
if [ ! -f "$file" ]; then
  SITE_CONF=$(cat <<EOF
<Directory /var/www/html>
  AllowOverride All
  Options +Indexes -MultiViews +FollowSymLinks
  AddDefaultCharset utf-8
  SetEnv ENVIRONMENT "development"
  php_flag display_errors On
  EnableSendfile Off
</Directory>
EOF
)
  echo "$SITE_CONF" > "$file"
fi
a2ensite dev
sudo touch /etc/apache2/sites-available/reverse-proxy.conf
sudo bash -c 'cat > /etc/apache2/sites-available/reverse-proxy.conf << EOL
<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass / http://127.0.0.1:3446/
ProxyPassReverse / http://127.0.0.1:3446/
ServerName reverse-proxy.example.com
</VirtualHost>
EOL'

# Vhost aktivieren
sudo a2ensite reverse-proxy.conf

# Apache neustarten
sudo service apache2 restart

# Composer installieren
EXPECTED_SIGNATURE="$(wget -q -O - https://composer.github.io/installer.sig)"
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
ACTUAL_SIGNATURE="$(php -r "echo hash_file('SHA384', 'composer-setup.php');")"
if [ "$EXPECTED_SIGNATURE" != "$ACTUAL_SIGNATURE" ]; then
  >&2 echo 'ERROR: Invalid installer signature (Composer)'
  rm composer-setup.php
else
  php composer-setup.php --quiet
  rm composer-setup.php
  mv composer.phar /usr/local/bin/composer
  chmod +x /usr/local/bin/composer
  sudo -H -u vagrant bash -c 'composer global require hirak/prestissimo'
fi

# phpinfo script 
file='phpinfo.php'
if [ ! -f "$file" ]; then
  echo '<?php phpinfo();' > "$file"
fi
# OPcache gui script herunterladen von Webseite
file='opcache.php'
if [ ! -f "$file" ]; then
  wget -nv -O "$file" https://raw.githubusercontent.com/amnuts/opcache-gui/master/index.php
fi
# Adminer script herunterladen von Webseite
file='adminer.php'
if [ ! -f "$file" ]; then
  wget -nv -O "$file" http://www.adminer.org/latest.php
  wget -nv https://raw.githubusercontent.com/vrana/adminer/master/designs/pepa-linha/adminer.css
fi
# Installiren von node, npm, bower, yarn und Webseiteninhalt anzeigen
apt-get install curl
curl -sL https://deb.nodesource.com/setup_10.x | sudo -E bash -
apt-get install -y nodejs
npm install -g bower
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
sudo apt-get update && sudo apt-get install yarn

#Entfernen des Standard Users
#sudo userdel -r vagrant

#Erstellung Abgeschlossen
echo "-------------------------------------------------------------"
echo "
Umgebung wurde erfolgreich generiert!
Die User "leandro" und "leandro-root" wurden erstellt."

sudo apt-get -y -q install bc

#Ausgabe benötigte Zeit 
deploy_end_time=$(date +%s)
      deploy_duration=$((deploy_end_time - deploy_start_time))
      minutes=$((deploy_duration / 60 + 1 ))
      seconds=$((deploy_duration % 60))
      echo "Deploy-Zeit: $minutes Minuten $seconds Sekunden"

SHELL
  end
end
```
#Aufgretene Probleme
-SSH Connection falsche IP 
-Ausgehende Verbindung Rule blockiert installation von Programmen
<img width="328" alt="image" src="https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/4b2a0eec-c785-46f4-bd5b-6d3fe4bff553">

Ubuntu Client 
User: leaben
pw: Admin1234!
Hostname: M300-VM-Ubuntu
Domain_Name: myguest.virtualbox.org
computername: leaben-virtualbox
