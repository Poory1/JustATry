# Server Aufsetzen
- [SSL Zertifikat und SSO Login](#1---ssl-zertifikat-und-sso-login)  
- [Git einrichten](#2---git-einrichten)
- [Installieren der Datenbank](#3---installieren-der-datenbank)
- [Einrichten der Datenbank](#4---einrichten-der-datenbank)
- [NodeJs und Npm installieren](#5---nodejs-und-npm-installieren)
- [Package vorbereiten](#6---package-vorbereiten)
- [Webserver aufsetzen](#7---webserver-aufsetzen)
- [Vorgehen nach Serverausfall](#8---vorgehen-nach-serverausfall)

## 1 - SSL Zertifikat und SSO Login
Zu aller erst, bevor jeder weiterer Abschnitt in Angriff genommen werden kann, müssen zwei Dinge unternommen werden.
  1. Es muss ein SSL Zertifikat vom HRZ beantragt werden. Am schnellsten geht dies, wenn Sie Thomas Krügl fragen, falls dieser auch für ihren Server verantwortlich ist. Er lädt dieses dann auf ihren Server hoch.
  2. Es muss der SSO Login für ihre URL beantragt werden. Dafür schreiben Sie bitte dem HRZ (service@hrz.tu-darmstadt.de) eine Email, in der sie folgende Dinge angeben: Name der URL, die notwendigen Attribute (in unserem Fall wären das **cn** (die TU-ID), **fullName** (den vollen Namen des Studierenden/TU-Mitglied),  **mail** (also die Maildresse des Einloggenden) und es muss eine Person genannt werden, die für alles verantwortlich ist.

## 2 - Git einrichten
Bevor wir mit der Einrichtung aller Dinge beginnen, müssen ein paar Funktionalitäten installiert werden. Dafür führen Sie bitte Folgendes auf dem Server aus.  
 
    $ sudo apt-get install tmux curl wget git gedit vim
 
Legen Sie im Home Verzeichnis ihres Servers einen Ordner mit dem Namen .ssh an, falls dieser noch nicht vorhanden ist. Überprüfen durch `ls-la`.  
 
    $ mkdir .ssh
 
Gehen sie nun in den erstellten Ordner.  
 
    $ cd .ssh  
 
  Erstellen Sie einen neuen Ssh Key und geben Sie ihm den Namen ims\_rsa. Drücken Sie zweimal Enter, um kein Passwort für den Key zu setzen. 
  
    $ ssh-keygen -b 4096 -t rsa
  
Erstellen sie eine config Datei im .ssh Ordner.  
 
    $ touch config
 
Benutzen Sie einen Texteditor Ihrer Wahl (z.B. `nano config`) und fügen sie folgendes in die config Datei ein:  

```
Host git.rwth-aachen.de  
    HostName git.rwth-aachen.de  
    User git  
    Port 22  
    IdentityFile ~.ssh/ims_rsa  
````
Jetzt gehen Sie bitte in Gitlab **git.rwth-aachen.de** rechts oben auf User Settings, wählen sie links Ssh-Key aus fügen sie den Inhalt aus ims_rsa.pub in das Textfeld ein. (Tipp: Um den Key zu kopieren einfach die ims_rsa Datei mit `gedit ims_rsa` öffnen und der Inhalt lässt sich leicht per str+c kopieren). Drücken Sie Add Key, um den Ssh Key ihrem Gitlab Projekt hinzuzufügen.

Testen Sie ihre config, indem Sie folgendes in ihr Terminal eingeben.

    $ ssh -T git@git.rwth-aachen.de
    
Der Output sollte "Welcome to Gitlab, @XXXX XXXX!" sein.

Anschließend klonen Sie das Projekt in das Home Verzeichnis ihres Servers.

    $ git clone git@git.rwth-aachen.de:mark.baierl/webapp-ims.git
    
## 3 - Installieren der Datenbank
Bei der Datenbank handelt es sich um Postgresql.  
Genauere Informationen finden Sie unter https://www.postgresql.org/about/  
Installieren Sie Postgresql.  

    $ sudo apt-get install postgresql postgresql-contrib

Falls der Befehl nicht funktionieren sollte, führen Sie bitte folgende Befehle in diesem Kapitel aus. Wenn alles geklappt hat, springen Sie zum nächsten Abschnitt.  

Importieren sie den Signing Key von Postgresql.

    $ wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
    
Fügen Sie das Repository hinzu. 

     $ sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ buster-pgdg main" | /etc/apt/sources.list.d/pgdg.list'
     
Updaten Sie ihr System.

    $ sudo apt-get update
    
Versuchen Sie anschließend erneut Postgresql zu installieren.

    $ sudo apt-get install postgresql postgresql-contrib

## 4 - Einrichten der Datenbank

Loggen Sie sich als Superuser in der Datenbank ein.
   
    $ sudo -u postgres psql postgres
   
Erstellen Sie dort den User webapp_ims und geben Sie ihm ein vernünftiges Passwort. Noten Sie sich dieses. Sie werden es später noch brauchen.

    $ create user webapp_ims encrypted password 'chooseASafePassword';
    
Erstellen sie eine neue Datenbank mit dem Namen webapp_ims und weisen Sie ihr den User webapp_ims zu.

    $ create database webapp_ims owner webapp_ims;
    
Verlassen sie nun die Datenbank indem sie **\q** schreiben. Prüfen Sie nun, ob Sie sich in die Datenbank einloggen können.

    $ psql -U webapp_ims webapp_ims

Falls der Fehler: “psql: Fehler: konnte nicht mit Server verbinden: FATAL:  Peer-Authentifizierung für Benutzer webapp_ims fehlgeschlagen" auftritt, gehen Sie in das Verzeichnis /etc/postgresql/11/main/pg_hba.conf (Die Versionsnummer 11 vom postgresql muss im Pfad ggf. durch ihre Versionsnummer ersetzt werden) und ersetzen Sie dort in Zeile 85 und 90 "peer" durch "md5".

```
$ cd /etc/postgresql/11/main
$ sudo nano pg_hba.con
```

Starten Sie nun Postgresql neu und versuchen Sie sich erneut in die Datenbank einzuloggen.

```
$ sudo systemctl restart postgresql
$ psql -U webapp_ims webapp_ims
```

Unter gewissen Umständen funktioniert der Neustart der Datenbank nicht. Probieren Sie in diesem Fall:

    $ /etc/init.d/postgresql reload

Damit haben Sie nun ihre Datenbank erfolgreich eingerichtet.

## 5 - Nodejs und Npm installieren

Installieren Sie den Node Version Manager. Mit ihm ist es möglich zwischen verschiedenen Node Versionen zu wechseln und die dazu passende Npm Version zu benutzen. 

```
$ curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.35.2/install.sh | bash 
$ source ~/.nvm/nvm.sh
```

**Öffnen Sie nun eine neue Konsole oder starten Sie ihr System neu** und installieren Sie NodeJs Version 12.

    $ nvm install 12
    
Wenn sie nun `node -v` eingeben, sollte v12.16.1 und bei `npm -v` 6.13.4  ausgegeben werden. Wichtig hierbei ist, dass sie NodeJs 12 und Npm Version 6 verwenden.

## 6 - Package vorbereiten

Gehen sie in das Oberverzeichnis vom Package "webapp-ims" `cd webapp-ims`, welches Sie vorhin in ihr Home Verzeichnis geklont haben. Dort führen Sie das helper-script aus, um die node_modules zu installieren. Dabei können ein paar Fehlermeldungen auftreten. Dies ist aber okay und kann einen Augenblick dauern. 

    $ ./npm-install-helper.bash
    
Gehen Sie in das Verzeichnis /webapp-ims/packages/utils und stellen sie das Package für andere Packages zur Verfügung.

    $ npm run build
    
Gehen sie in /webapp-ims/packages/migrations und kopieren Sie die settings.default.js und benennen Sie sie in settings.js um.

    $ cp settings.default.js settings.js

Anschließend öffnen Sie die settings.js (`nano settings.js`) und ändern sie den Username zu "webapp_ims", ändern Sie das Passwort zu dem, welches Sie vorhin beim einrichten der Datenbank vergeben haben und ändern Sie den Namen der Datenbank zu webapp_ims.

Gehen sie in /webapp-ims/packages/server und kopieren Sie die settings.default.js und benennen Sie sie in settings.js um.

    $ cp settings.default.js settings.js
    
Anschließend öffnen Sie die settings.js (`nano settings.js`) und ändern sie den Username zu "webapp_ims", ändern Sie das Passwort zu dem, welches Sie vorhin beim einrichten der Datenbank vergeben haben und ändern Sie den Namen der Datenbank zu webapp_ims.  
Unter Origin müssen Sie die korrekte URL angeben, unter der die Webapplikation erreichbar sein soll. 
Geben Sie außerdem den korrekten Pfad zu ihrem SSL Zertifikat an. Zuletzt müssen sie noch das secret auf require('crypto').randomBytes(64).toString('hex') setzen und darauf achten, dass cookie: secure: true gesetzt ist.  
Den ganzen Prozess, den Sie im Server mit der settings.default.js gemacht haben, müssen Sie nun auch mit der settings.default.js in /webapp-ims/packages/server/test/api/ machen. 

In /webapp-ims/packages/server/logs müssen die beiden Dateien error.log und info.log erstellt werden.

```
$ touch error.log
$ touch info.log 
```

In /webapp-ims/packages/utils/src muss die URL in der Datei constants.js zu Ihrer geändert werden. Dies passiert in dem Return Statement unter der TODO Anzeige.
Im wahrscheinlichsten Fall muss die URL "https://vm1.rbg.informatik.tu-darmstadt.de" durch ihre ersetzt werden. Wichtig ist aber, dass "${API_PREFIX}" erhalten bleibt.
**Achten Sie auf die richtigen Anführungszeichen.**

Anschließend gehen sie in /webapp-ims/packages/migrations und führen sie folgende drei Befehle aus.

```
$ npm run migrate:undo
$ npm run migrate:undo
$ npm run migrate 
```

Navigieren Sie zu /webapp-ims/packages/utils und führen sie folgenden Befehl aus.

     $ npm run build
     
Navigieren Sie zu /webapp-ims/packages/client und führen sie folgenden Befehl aus. Dieser builded die index.html für den Webserver.

    $ npm run build
    
Öffnen Sie mit tmux eine neue Session, gehen sie dort in /webapp-ims/packages/server und starten Sie den Server für die Backend Kommunikation. In Zukunft wird dieser immer in dieser Session laufen.

```
$ tmux new -s server
$ npm run start:prod
```

Die Tmux Session können Sie verlassen, indem Sie Strg+b und dann d drücken. Um wieder in die Session zurück zu kommen, müssen sie folgendes schreiben.

    $ tmux attach -t server

Der Server kann durch folgenden Befehl wieder gestoppt werden, falls notwendig: `npm run stop:prod`

## 7 - Webserver aufsetzen

Installieren Sie Nginx als Software für den Webserver.

    $ sudo apt-get install nginx

Gehen sie in das Verzeichnis /etc/nginx/sites-available. Fügen sie dort ihre eigenen Nginx Settings hinzu.

    $ sudo nano webapp-ims
    
Dort muss folgendes eingefügt werden. Alle Stellen, die mit zwei Sternen markiert sind, müssen durch ihre individuellen Einstellungen ersetzt werden.

```
server {
    listen 80;
    server_name **vm1.rbg.informatik.tu-darmstadt.de**;
    return 301 https://**vm1.rbg.informatik.tu-darmstadt.de**;
}


server {
    listen 443 ssl default_server;
    listen [::]:443 ssl;
    server_name **vm1.rbg.informatik.tu-darmstadt.de**;
    root **/home/sysop/webapp-ims/packages/client/build**;
    index index.html;
    
    acces_log /var/log/nginx/webapp-ims.log;
    error_log /var/log/nginx/webapp-ims.error.log;
    
    ssl certificate /home/sysop/CA/cert.pem;
    ssl_certificate_key /home/sysop/CA/key.pem;
    
    include /etc/nginx/include/ssl_options.conf;
    
    location / {
        try_files $uri /index.html=404;
    }
    
    location /api {
        proxy_pass https://localhost:8080;
        include /etc/nginx/includes/proxy_options.conf;
    }
    
    location /admin-api {
        proxy_pass htpps://localhost:8090;
        include /etc/nginx/includes/proxy_options.conf;
    }
}
```

Beim Servernamen muss der Name ihrer URL oder die IP verwendet werden. Unter root muss der Pfad zu ihrer index.html Datei angegeben werden, die Sie vorhin gebuilded haben.  
Anschließend muss noch eine Verknüpfung zu den unter nginx verfügbaren Seiten hergestellt werden. Dazu gehen sie in /etc/nginx/sites-enabled und führen sie folgendes aus.

```
$ sudo rm -r default
$ sudo ln -s /etc/nginx/sites-available/webapp-ims /etc/nginx/sites-enabled/webapp-ims
```

Anschließend müssen noch die passenden SSL Optionen und Proxy Optionen in Nginx eingepflegt werden. Gehen sie dafür ins Homeverzeichnis.

    $ sudo cp -R webapp-ims/infrastructure/nginx/includes /etc/nginx
    
Abschließend muss noch nginx neugestartet werden und danach sollte die Webanwendung unter ihrer URL verfügbar sein.

    $ sudo systemctl restart nginx

## 8 - Vorgehen nach Serverausfall

Zuerst einmal sollten sie zur Sicherheit nginx neu starten.

$ sudo systemctl restart nginx

Danach müssen Sie eine neue tmux session starten und in dieser den internen Server neu starten.

```
$tmux new -s server
$ cd /webapp/packages/server
$ npm run start:prod
```

Damit sollte hoffentlich alles funktionieren.
Bei Fragen oder Problemen wenden Sie sich an lukas_schraven@web.de

