![image](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/cd85f277-cf42-4e8a-bb90-78d0f354c3aa)
# M300_ST20b
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
5. [Wissenzuwachs](#Wissenszuwachs)
6. [Reflexion](#Reflexion)

# Einführung


Im Modul 300 geht es darum, eine Service Umgebung völlig automatisiert zu erstellen. Dies bietet viele Vorteile und spart im End-Effekt viel Zeit. Meine Idee war es mithilfe des Vagrant-Files ein Ubuntu Client generieren zu lassen, worauf ein Apache Webserver mit links zu zweier Service wie Datenbank & Monitoring Seiten.

# Systemumgebung
Um sich die Umgebung besser Vorstellen zu können: 

![M300_Systemumgebung](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/9a771739-2d7b-4b1e-9677-7870d9ace3bb)

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

Weitere wichtige Befehle sind: 
```
$ Vagrant status    #Status der VM
$ Vagrant halt      #pausieren der VM
$ Vagrant destory   #Löschen der VM
$ sudo su -p        #Wechsel zu "Root" User
$ sudo su <user>    #Wechsel zu beliebigem User
```

# VAGRANT-File
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

#Login für Adminer fürs GUI
sudo mysql <<-EOF
  create user 'leandro'@'localhost' identified by 'Admin1234!';
  Grant all privileges on *.* to 'leandro'@'localhost';
  Flush privileges;
EOF

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

Um auf die generierte Umgebung zugreifen zu können öffnet man http://localhost:3346

![image](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/13641ac1-b6ec-4a77-ae86-58e247418cc7)

# Adminer
Bei der Umgebung, habe ich eine MYSQL Datenbank mit Adminer erstellen lassen. bei öffnen der Seite erscheint erstmal eine Login Seite

![image](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/b8c147a1-26ac-4bf4-b51d-2e3611d37449)

Wenn man sich mit den im VAGRANT-File definierten Anmeldedaten erfolgreich eingelogt hat, sieht man den ganzen Datenbank inhalt. 

![image](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/0aa3d36c-228b-4d50-b6b5-dd19f50c75c1)

# OPcache
Um immer ein Auge auf den Leistungs-Verbrauch zu haben, kann man die zweite Seite öffnen, eine Ressource Monitoring Page die mit dem Service OPcache erstellt wurde. 

![image](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/1e4b4827-9a06-41ae-8a01-89eb74dd964a)

# Endstand 
Das Skript erstellt ein Umbuntu Client worauf diverse Dienste laufen. Es wird ein Apache Service kreirt der auf https://localhost:3446 läuft. Darauf findet man eine Übersicht von zum einen ein Monitoring Service und ein Adminer SQL Datenbank. Zudem werden ein normaler User und ein Root User erstellt. Im hintergrund werden zusätzlich einige Security Features wie Firewall Rules & Reverse Proxy. Am Ende wird an der Konsole die benötigte Deploy Zeit ausgegeben und man kann ich auf die Umgebung verbinden. 

# Aufgretene_Probleme
1. SSH Connection, da falsche IP in der SSH Rule Angegeben
Zu  beginn, war ich nicht sicher welche IP ich in der SSH Rule angeben musste. Nach gewissen rechechier-arbeiten, habe ich herausgefunden welche IP ich allowen soll.
```sudo ufw allow from 10.0.2.2 to any port 22```

2. Ausgehende Verbindung Rule blockiert installation von Programmen
Beim durchgehen und konfigurieren der Firewall Rules, traten auf einmal Fehler bei der Installation von Services Auf. Daraufhin ging ich erstmalig davon aus, dass gewisse downloadslinks oder commands geändert haben. Nach langem testing und rumprobieren, kam ich zu dem Entschluss das es an diesen Rules läge. Ein gutes beispiel wenn man den wald vor lauter bäumen nicht sieht. 
```
$ sudo ufw deny out to any
$ sudo ufw allow out 22/tcp 
```
![image](https://github.com/JuveFanBoy/M300_ST20b/assets/60262192/e4ae43c6-2be1-4508-a9ad-f45a5e5d7e13)

# Wissenszuwachs
* Vagrant kennengelernt
* VM Konfiguration per Skript
* Portforwarding Zugriff per Host
* Firewall und Reverse Proxy auf Ubuntu Maschine
* Automatisierung einer Umgebung per Skript



# Reflexion

Tag 1

Der erste Tag war ein eher Theoretischer Tag. Herr Roht, hat uns die VM Nutzung der Zukunft vorgestellt. Um dies auch selber nutzen zu können, haben wir VAGRANT installiert und Virtual Box, da diese eine offene Schnittstelle hat. Danach haben wir angefangen mit Gitlab Aufträge zu lösen. Dies fand ich schon cool, da es die Vorbereitung auf eine automatisierte Umgebung war. Im allgemeinen bin ich schon sehr fasziniert von Skripten und es macht mir auch sehr viel Spass. Die Aufträge des Nachmittag gelangen mir gut und ich freute mich auf die Zukünftigen Tage. 

Tag 2

Heute wurde wir das erste mal in die LB02 eingeführt. Herr Rohr erklärte uns, was genau das Ziel war und wie wir es erreichen könnten. Dies gefiel mir gut, da er uns viele Freiheiten liess für eigene Vorschläge und Kreativität. Danach began ich das erste mal mich ins Vagrant Thema einzulesen. mir gelang es die erste Basic Konfiguration der VM zu erstellen mit custom VM-Name und auch zugeteilte Ressourcen. Danach tüftelte ich weiter am SSH Zugriff, der leider immer noch nicht ganz funktionierte. 

Tag 3

Nach 2 Wochen Pause, da Pfingsten dazwischen lag, brauchte ich mich noch ein bisschen einzulese bis ich wieder voll dabei war. Da ich in der Freizeit auch noch ein bisschen Zeit reinsteckte, musste ich lediglich den Wissenstand vor ein paar Tagen wieder aufholen. Im Unterricht sprachen wir über das Thema Sicherheit. Hierfür waren vor allem, Firewall Regeln und Reverse-Proxy wichtig. Nach dem interessanten Input wandte ich mich wieder meinem Skript zu. Ich baute die Firewall Regeln ein und plötzlich funktionierte der SSH Zugriff nicht mehr. Mir war bewusst, dass es an den Firewall Rules läge, jedoch war ich mir nicht bewusst welche IP ich bei der ALLOW SSH Rule eingeben sollte. 

Tag 4

Nach einigem Nachforschen, habe ich nun endlich die Richtige IP gefunden und der SSH Zugriff steht wieder. Zudem, hatte ich ein zweites grosses Problem was die Installation von nötigen Applikations-Paketen blockierte. Dies konnte ich unterdessen auch fixen, indem ich eine Zeile löschte, die das blockieren von unnötigen Ports vorsieht. Der rest vom Tag konnten wir weiter an unseren Skripts arbeiten. Ich installierte OPcache auf meinem Apache Server für das Monitoring und baute weitere Features ein.

Tag 5

Heute war der Abgabeterming. Da ich noch sehr viel zuhause arbeitete und nun alles vollständig funktionierte, hatte ich keine Zweifel. Ich dokumentierte also noch den Rest der Zeit die heute zur verfügung stand. Danach gab ich ein bisschen später um ca 16:00 ab und war zufrieden mit meiner LB02. Ich bin gespannt auf die nächste LB. 
