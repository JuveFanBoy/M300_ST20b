![image](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/cd85f277-cf42-4e8a-bb90-78d0f354c3aa)
# M300 Plattformübergreifende Dienste in ein Netzwerk integrieren
# Inhaltsverzeichnis
1. [Einführung](#Einführung)
	1. [Systemumgebung](#Systemumgebung)
3. [VAGRANT-File](#VAGRANT-File)
	1. [Thema 1](#thema1)
	1. [Thema 2](#thema2)
	1. [Thema 3](#thema3)	
4. [Testing](#Testing)
	1. [Deploy](#Deploy_der_Umgebung)
	1. [Startseite](#Startseite)
	1. [Firewall](#Firewall)
	1. [Reverse Proxy](#Reverse_Proxy)
	1. [Adminer](#Adminer)
	1. [OPcache](#OPcache)
  	1. [Docker](#Docker)
  	1. [Vaultwarden](#Vaultwarden)	
5. [Wissenzuwachs](#Wissenszuwachs)
6. [Reflexion](#Reflexion)

# Einführung


Im Modul 300 geht es darum, eine Service Umgebung völlig automatisiert zu erstellen. Dies bietet viele Vorteile und spart im End-Effekt viel Zeit. Meine Idee war es mithilfe des Vagrant-Files ein Ubuntu Client generieren zu lassen, worauf ein Apache Webserver mit links zu zweier Service wie Datenbank & Monitoring Seiten.

# Systemumgebung
Um sich die Umgebung besser Vorstellen zu können: 

![M300_Systemumgebung](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/8fb43627-5b87-4c8d-b107-dd010828d000)


Um auf die erstellte Umgebung Zugreifen zu können, kann ich auf dem Host-System https://localhost:3346 öffnen. 

Damit die Umgebung aber überhaupt funktioniert, muss man Zuerst im Gitbash das entsprechende Skript ausführen. 

Dafür wechsel ich in mein persönliches verzeichnis mit dem Vagrant file.
```
$ cd ~/Desktop/Modul-to-Upload/Modul\ 300/myvagrant/
```
Und lasse die Umgebung mithilfe des Vagrant-Files erstellen. 
```
$ Vagrant up
```

Nach etwa 5 min ist die Umgebung kreirt und man kann im Gitbash per SSH zugreifen. 
```
$ Vagrant ssh
```

Wichtige Vagrant Commands: 
```
$ Vagrant status    	#Status der VM
$ Vagrant halt      	#Pausieren der VM
$ Vagrant destory   	#Löschen der VM
$ sudo su -p        	#Wechsel zu "Root" User
$ sudo su <user>    	#Wechsel zu beliebigem User
$ curl <url>		#Antwort einer Website abragen
```
Wichtige Docker Commands: 
```
$ sudo docker ps -a 	#Laufende Docker Prozesse anzeigen
$ docker stop <name>	#Stoppen eines vorhandenen Container
$ docker rm <name>  	#Entfernen eines vorhandenen Container
$ Vagrant destory   	#Löschen der VM
$ sudo su -p        	#Wechsel zu "Root" User
$ sudo su <user>    	#Wechsel zu beliebigem User
```


# VAGRANT-File
```
#Erstellung VM 
time_zone = "Europe/Zurich"

Vagrant.configure("2") do |config| # Konfiguration der Vagrant-Box
  config.vm.box = "generic/ubuntu1804" # Verwendet die Ubuntu 18.04-Box
  config.vm.network "forwarded_port", guest: 3446, host: 3446, auto_correct: true # Weiterleitungsport von 80 auf 3446
  config.vm.network "forwarded_port", guest: 80, host: 3445, auto_correct: true # Weiterleitungsport von 80 auf 3446

  config.ssh.forward_agent = true
  config.vm.synced_folder ".", "/vagrant", type: "virtualbox", # Synchronisiere das aktuelle Verzeichnis mit dem /vagrant-Verzeichnis in der VM
  owner: "www-data", group: "www-data" # Gruppen und Benutzer die Folder festlegen

  config.vm.provider "virtualbox" do |vb| # Provider Konfiguration
    vb.name = "M300_LB03_Leandro" # VM Name
    vb.memory = 3072 # RAM in MB
    vb.cpus = 4 # Anzahl Prozessoren
    vb.customize ["guestproperty", "set", :id, "/VirtualBox/GuestAdd/VBoxService/--timesync-set-threshold", 10000]
    vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    vb.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
  end

config.vm.provision "shell" do |s|
    s.args = [time_zone]
    s.inline = <<-SHELL  

#Erstellung Zeitstempel für Zeittracking
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

#Erstellung Benutzer Leandro 
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
    ServerName reverse-proxy.example.com

    ProxyPreserveHost On

    # Weiterleitung zu Vaultwarden auf Port 3445
    ProxyPass /vaultwarden http://localhost:3445
    ProxyPassReverse /vaultwarden http://localhost:3445

    # Weiterleitung zu anderen Diensten auf Port 80
    ProxyPass /opcache http://localhost
    ProxyPassReverse /opcache http://localhost
    ProxyPass /adminer http://localhost
    ProxyPassReverse /adminer http://localhost
    ProxyPass /phpadmin http://localhost
    ProxyPassReverse /phpadmin http://localhost
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

#Login für Adminer fürs GUI
sudo mysql <<-EOF
  create user 'leandro'@'localhost' identified by 'Admin1234!';
  Grant all privileges on *.* to 'leandro'@'localhost';
  Flush privileges;
EOF

echo "-----------------------------------------------------------------
DOCKER WIRD INSTALLIERT!
-----------------------------------------------------------------"
# Docker Installation
sudo apt-get install -q -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update -q
sudo apt-get install -q -y docker-ce docker-ce-cli containerd.io

# Docker Compose Installation
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Docker Benutzergruppe hinzufügen
sudo usermod -aG docker leandro

# VAULTWARDEN SERVICE INSTALLATION
sudo mkdir /opt/vaultwarden


cd /opt/vaultwarden
# Vaultwarden Container starten
sudo touch /opt/vaultwarden/docker-compose.yaml
sudo echo "version: '3' 
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    ports:
      - 3446:80
    environment:
      - ROCKET_PORT=80
      - ROCKET_WORKERS=10
    volumes:
      - ./data:/data"| sudo tee /opt/vaultwarden/docker-compose.yaml

cat /opt/vaultwarden/docker-compose.yaml
cd /opt/vaultwarden && sudo docker-compose up -d

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
# Testing

# Deploy_der_Umgebung 
Nachdem man die Umgebung mit "Vagrant up" gestartet hat, bekommt man diesen kurzen Überblick und die Meldung das die Umgebung nun Verfügbar ist. 

![image](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/3feb31cc-ebdc-4c11-a784-2ef827fa5564)

# Firewall
Im Vagrant-File habe ich konfiguration, dass nur meine eigene Client SSH Zugriff auf die VM bekommt, dazu habe ich den Port 80 geöffnet, sodass ich von meinem Client auf dem Port 3346 auf den Port 80 umgeleitet Zugriff auf den Apache Server bekomme. 
Ob die im Vagrant-File hinterlegte Konfiguration auch ausgeführt wurde, überprüfen wir mit:

```$ sudo ufw status```

![image](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/3d52aa1a-e37b-4743-bde6-97d27e3381b2)

# Reverse_Proxy
Auch den Reverse Proxy können wir überprüfen mit: 


```$ curl -i localhost:3346```
![image](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/7f5749ae-e0c1-406e-a79d-198afb1f67a8)


Das bedeutet, dass die Verbindung erfolgreich hergestellt wurde und der Server eine positive Antwort (HTTP-Statuscode 200 OK) zurückgegeben hat. Außerdem zeigt der Servername "Apache/2.4.29 (Ubuntu)" an, dass der verwendete Server die Apache-Version 2.4.29 unter Ubuntu ist.

# Startseite

Um auf die generierte Umgebung zugreifen zu können öffnet man den Localhost auf einem gewissen Port: 
![image](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/bb5130c6-4f38-49d0-9cc4-a2c1d2a4151d)


Für den Apache Server mit der Datenbank und dem Ressource Monitoring Service öffnet man http://localhost:3445
![image](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/13641ac1-b6ec-4a77-ae86-58e247418cc7)

Um den Vaultwarden Service, der auf Docker basiert zu öffnen, nutz man den Port: 3446 http://localhost:3446
![image](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/0c42d8cf-50a8-414c-8e0d-e495c4fcdbcf)


# Adminer
Bei der Umgebung, habe ich eine MYSQL Datenbank mit Adminer erstellen lassen. bei öffnen der Seite erscheint erstmal eine Login Seite

![image](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/b8c147a1-26ac-4bf4-b51d-2e3611d37449)

Wenn man sich mit den im VAGRANT-File definierten Anmeldedaten erfolgreich eingelogt hat, sieht man den ganzen Datenbank inhalt. 

![image](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/0aa3d36c-228b-4d50-b6b5-dd19f50c75c1)

# OPcache
Um immer ein Auge auf den Leistungs-Verbrauch zu haben, kann man die zweite Seite öffnen, eine Ressource Monitoring Page die mit dem Service OPcache erstellt wurde. 

![image](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/1e4b4827-9a06-41ae-8a01-89eb74dd964a)


# Docker

![image](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/1ac1aba0-a76f-403e-8b0b-3002a233b71e)

Mein Ziel war es, die vollständige LB03 in einem einzelnen File zu haben, welches alles automatisiert installiert. Darum entschied ich mich Docker auf der bereits vorhanden Ubuntu VM zu installieren. 
Nach ein paar testing versuchen, hat sich Docker auf der Ubuntu VM installieren lassen. 

```
# Docker Installation
sudo apt-get install -q -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update -q
sudo apt-get install -q -y docker-ce docker-ce-cli containerd.io

# Docker Compose Installation
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Docker Benutzergruppe hinzufügen
sudo usermod -aG docker leandro
```


# Vaultwarden 

![image](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/7b2fa039-cc9d-41e2-ae81-092ed0dca623)

Um seine Passwörter sicher bei sich Localgehostet, und doch immer Zugriffsbereit zu haben, habe ich den Vaultwarden Passwort Manger eingerichtet. Da ich Bitwarden, welches die extern gehostete Version des Passwort Manager darstellt schon privat nutzte, schien mir dies für die perfekte möglichkeit zu versuchen alles Lokal zu verlagern. Auf einem GitHub Repository, fand ich schliesslich das Image: 

```
version: '3' 
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    ports:
      - 3446:80
    environment:
      - ROCKET_PORT=80
      - ROCKET_WORKERS=10
    volumes:
      - ./data:/data
```
Wenn ich den Service nun starte, sehe ich ihn unter den Docker Prozessen: 
![image](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/14c23009-b3a8-4eaa-83d2-7c49570b2118)

Wenn ich mich nun Anmelde, habe ich diese Startseite vor mir. Aktuell hat Sie noch keine Passwort Einträge. Ausserdem, weisst sie grosse ähnlichkeit mit Bitwarden auf: 
![image](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/d2992cfc-bf04-4221-9d8b-3b011a58deed)



# Endstand 
Das Skript erstellt ein Umbuntu Client worauf diverse Dienste laufen. Es wird ein Apache Service kreirt der auf https://localhost:3445 läuft. Darauf findet man eine Übersicht von zum einen ein Monitoring Service und ein Adminer SQL Datenbank. Ergänzt werden diese Dienste mit der Applikation Docker. Diese wird ebenfalls auf dem Ubuntu Client installiert worauf wiederum von einem .yaml File "Vaultwarden" als Docker Service gehostet wird. Diesen kann man per https://localhost:3446 erreichen

# Aufgretene_Probleme
1. Während der LB03 ist insgesamt nur ein grösseres Problem aufgetreten. Dieses jedoch, hat mich richtig Zeit gekostet. Es geht Darum, dass das Image nicht richtig erkannt wurde. In der 1. Zeile sollte:
```
version: '3' 
```
stehen. Jedoch hat das Ruby File die Apostroph bei jedem Versuch entfernt. Das Suspekte daran, dass wenn man genau diesen Command Manuell ausgeführt hat, sind die Apostrophs geblieben: 
![image](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/a3dec2d3-8441-4a2a-95c7-7e25df39b579)

Nach vielen versuchen und Fixing, habe ich schliesslich eine ganz andere Methode verwendet, bei dem die '' blieben.: 
![image](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/8c749772-91e0-4c20-9a9d-c173a005ccde)

# Wissenszuwachs
* Docker praktisch kennengelernt
* Docker Konfiguration per Skript
* Portforwarding (Reverse Proxy) viel tiefer verstanden
* .yaml File struktur wier aufgefrischt



# Reflexion

Tag 1

Der erste Tag der LB03 hat heute erst um 16:00 für mich gestartet, da ich bis hierhin noch nicht 100% zufrieden mit meiner LB02 war, weshalb ich daran noch Arbeitete.  Da uns Herr Rohr schon zu begin des Nachmittags in das Modul eingeführt und die Aufgabe genauer erklärt war mir klar was ich zu tun hatte. Daraufhin habe ich Docker heruntergeladen und versucht irgendein Docker Container darauf zu installieren. Nach einigen Versuchen suchte ich im Internet, ob es möglich sei, Docker auf Ubuntu zu installieren. Ich hatte zwar keine Zeit mehr weiteres zu probieren, jedoch hatte ich eine Antwort auf meine Frage und habe mir gewisse Ziele für die nächste Lektion vorgenommen.

Tag 2

Als mir diese Frage beantwortet wurde, fing in direkt damit an, meine schon Vorhandene Maschine  zu erweitern. Ich habe Docker zuerst Manuell auf der Maschine aufgesetzt und als die Installation funktionierte, entschied ich mich für ein Service. Da ich privat schon einmal versucht habe Vaultwarden (die Lokale Version von Bitwarden) aufzusetzten und leider daran gescheitert bin, nahm ich mir dieses Ziel erneut vor. Nach Nachforschungen und einigen Stunden, habe ich Vaultwarden in den letzten 5 Minuten nun zum laufen gebracht. Was mir bei der Manuellen installation die grösste Schwierigkeit darstellte, war das ganze Port Forwarding, da ich zuerst nicht verstand wie ich alles weiterleiten sollte. 

Tag 3

Heute war es soweit. Ich habe mir vorgenommen heute 1 Woche vor Abgabe die LB03 abzuschliessen. In der Zeit von letzter Woche bis heute, habe ich alles von letzter Woche ins Ruby abgeschrieben. Soweit funktionierte alles, bis auf den angegebenen error mit dem Version '3'. Also entschied ich nmich nicht von zuhause aus zu arbeiten, sondern in der Schule zu bleiben und den Bug weiter zu fixen. Da ich dies schon min. 1-2h zuhause probierte, machte ich mir nicht viel Hoffnung, da manuell ja alles funktionierte. Danach entschied ich mich, dass gesamte erstellen des .yaml File zu löschen und es neu zu erstellen. Dies funktionierte erstaulicher weise gut. Der Bug war also gefixt und das Skript funktionierte komplett automatisiert. 
