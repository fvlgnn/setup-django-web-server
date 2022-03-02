# HowTo Django servito da Nginx e Gunicorn con Supervisor

Per questa guida supponiamo di utilizzare un sistema operativo Debian based (Debian, Ubuntu, Raspberry Pi OS, Mint, ecc.) comunque la guida è applicabile a tutti gli altri sistemi operativi Linux, quello che cambia sarà il Package Manager (apt, yum, pkg, ecc.). 

Python Django viene eseguito tramite **virtual environment** quindi tutte le librerie, i pacchetti e gli applicativi necessari allo scopo del progetto python sono installati ed eseguiti all'interno della cartella di progetto, senza compromettere i software originale del web-server.

In questo articolo non viene trattata la parte relativa al database, pertanto viene utilizzato come database SQLite, che è il DB di default di una web-app django. In produzione è consigliato utilizzare PostgreSQL ma questa operazione richiede un articolo a parte.


## Installazione software propedeutico

Sul server deve essere installato del software propedeutico per la fruizione della web-app Django. Pertanto se non è già installato dovrà essere installato Nginx, Supervisor che serve a verificare il corretto funzionamento di Gunicorn, Python e il package manager pip, il software Python per la creazione delle Virtual Environments, SQLite o PostgreSQL, software per sviluppatori opzionali quali Git, cURL, Certbot (Let's Encrypt certificati SSL per HTTPS), editor di testo tipo nano, vi, pico, ecc.

Utilizzare privilegi di `root` o `sudo`.

- `apt install python3 python3-pip python3-venv python3-dev`
- `apt install nginx supervisor`
- `apt install sqlite3`

Opzionali

- `apt install curl git nano build-essential` - Tool Sviluppo
- `apt install python3-certbot-nginx` - Certbot
- `apt install libpq-dev postgresql postgresql-contrib` - PostgreSQL


## Cartella di progetto 

Creare la cartella che ospiterà l'intero progetto. Essendo una web-app, il webserver fornisce già una cartella `/var/www`. Assegnare alla cartella di progetto i privilegi di lettura/scrittura/esecuzione adeguati all'utente di sviluppo. Supporre che la cartella di progetto sia denominata `django-webapp`

Utilizzare privilegi di `root` o `sudo`.

- `mkdir /var/www/django-webapp`
- `chown www-data:users /var/www/django-webapp` - _www-data_ è l'utente utilizzato dal webserver, _users_ è il gruppo assegnato di default a tutti gli utenti di un OS Linux, con tutta probabilità l'utente utilizzato ne fa parte.
- `chmod g+w /var/www/django-webapp` - Vengono assegnati privilegi di scrittura al gruppo _users_ e quindi all'utente utilizzato per eseguire le prossime operazioni. Senza questo passaggio con tutta probabilità non sarebbe procedere oltre.


## Inizializzazione ambiente Python

Viene inizializzato l'ambiente Python utilizzando la **Virtual Environment** che conterrà tutte le librerie, i pacchetti e gli applicativi utili al progetto.

- `cd /var/www/django-webapp`
- `python3 -m venv venv` - Crea la Virtual Environment dentro la cartella `venv` in alternativa può essere creata direttamente nella radice della cartella di progetto `python3 -m venv .` dove il `.` significa _qui_. Per il resto dell'articolo supporre che la cartella utilizzata per contenere le variabili d'ambiente sia `venv`.
- `source ./venv/bin/activate` - Variabile d'ambiente attivata, pertanto ogni operazione di installazione e configurazione di _componenti_ Python viene isolata all'interno di tale ambiente. Per disattivare le variabili d'ambiente eseguire il comando `deactivate`.
- `pip install --upgrade pip` - Aggiornamento di _pip_, opzionale ma raccomandato.


**TIPS:** Se viene utilizzato _git_, configurare con attenzione il `.gitignore` pe evitare di portare sul repository cartelle inutili delle variabili d'ambiente (`venv`, `bin`, `lib`, `share`, `run`, `include`, ecc.).


### Inizializzazione web-app Django

Nell'articolo vengono riportati i passaggi fondamentali per inizializzare un applicazione Django. L'articolo non tratta eventuali configurazioni e personalizzazioni, per queste è bene avvalersi di tutorial o articoli specifici all'utilizzo di Django Framework.

- `pip install Django` - Installa Django Framework. Qualora sia presente un file di pacchetti Python eseguire `pip install -r requirements.txt` mentre per salvare nel file i pacchetti attualmente installati, eseguire `pip freeze > requirements.txt`
- Creazione del core della web-app. Supporre che il nome della web-app sia `mysite`
	- `django-admin startproject mysite .` - web-app creata da zero. 
	- `git clone git@github.com:digitalocean/sample-django.git` - web-app scaricata da repository git. Applicazione Django base di _DigitalOcean_.

#### Comandi utili django

- `python manage.py startapp app` - Crea l'applicazione con la logica Django _MTV: Model, Template, View_ 
- `python manage.py makemigrations` - Crea le migrazioni dai modelli e verifica i componenti.
- `python manage.py migrate` - Effettua le migrazioni. (Nel caso di tabelle mancanti nel database `python manage.py migrate --run-syncdb`).
- `python manage.py createsuperuser` - Crea l'utente di amministrazione.
- `python manage.py collectstatic` - Crea i file statici per gli assets grafici _(css, js, img, fonts, ecc.)_.
- `python manage.py runserver` - Avvia l'applicazione con il web-server interno. Attenzione eseguire solo in fase di sviluppo e per verificare anomalie.


## Gunicorn, installazione e configurazione

Gunicorn è un Web Server Gateway Interface (WSGI) Python.

- `pip install gunicorn` - Installa Gunicorn
- `gunicorn --bind 0.0.0.0:8000 mysite.wsgi` - Avvia l'web-app Django servita da Gunicorn selezionado il corretto file WSGI.
    - Navigare la web-app utilizzando l'IP del server (assicurarsi che in console non siano presenti errori e che nel file `mysite/settings.py` sia abilitato l'accesso all'host utilizzato).
    - Terminare il wes-server tramite `CTRL+C`.
- (Opzionale) Verificare che nella cartella di progetto sia presente la cartella `run` con all'interno il file `gunicorn.sock` (`run/gunicorn.sock`). 


Creare un file eseguibile denominato `gunicorn_start` all'interno della cartella `venv/bin/`. Questo file è è lo script Gunicorn dove sono presenti parametri necessari al corretto funzionamento di Gunicorn per essere servito da Supervisor. 

- `touch ./venv/bin/gunicorn_start` - Crea il file
- `chmod 775 ./venv/bin/gunicorn_start` - Imposta permessi di esecuzione sul file
- `nano ./venv/bin/gunicorn_start` - Aggiornare il file utilizzando come bozza, da modificare a seconda delle esigenze il seguente codice.


```bash
#!/bin/bash

NAME="django-webapp"                                        # Nome dell'applicazione
DJANGODIR=/var/www/django/django-webapp                     # Cartella progetto Django
SOCKFILE=/var/www/django/django-webapp/run/gunicorn.sock    # Socket utilizzato per la comunicazione
USER=www-data                                               # Utente utilizzato per l'esecuzione (www-data è l'utente di default di nginx)
GROUP=www-data                                              # Gruppo utilizzato per l'esecuzione (potrebbe essere utilizzato users)
NUM_WORKERS=3                                               # Numero di processi generati da Gunicorn
DJANGO_SETTINGS_MODULE=mysite.settings                      # File settings utilizzato da Django
DJANGO_WSGI_MODULE=mysite.wsgi                              # File WSGI utilizzato da Django

echo "Starting $NAME as `whoami`"

# Attivazione variabile d'ambiente
cd $DJANGODIR
source ./venv/bin/activate
export DJANGO_SETTINGS_MODULE=$DJANGO_SETTINGS_MODULE
export PYTHONPATH=$DJANGODIR:$PYTHONPATH

# Crea la cartella run se non esiste
RUNDIR=$(dirname $SOCKFILE)
test -d $RUNDIR || mkdir -p $RUNDIR

# Avvia la web-app Django tramite Gunicorn
# Configurazione impostata per essere eseguita sotto Supervisor. Non usare --daemon
exec ./venv/bin/gunicorn ${DJANGO_WSGI_MODULE}:application \
  --name $NAME \
  --workers $NUM_WORKERS \
  --user=$USER --group=$GROUP \
  --bind=unix:$SOCKFILE \
  --log-level=warning \
  --log-file=-
```

- Testare il file lanciando il comando `./venv/bin/gunicorn_start` (CTRL+C per uscire successivamente).
- Verificare la presenza della cartella `run` con all'interno il file socket `gunicorn.sock` (`run/gunicorn.sock`). 


Disattivare la **Virtual Environment** con il comando `deactivate`.


## Supervisor

Questo applicativo verifica e gestisce il comportamento di Gunicorn e del ciclo di vita della web-app Django come configurato nel file script utilizzato (`gunicorn_start`). Supervisor prevalentemente si occupa di scrivere messaggi di log relativi a Gunicorn legato alla mantenimento della web-ap Django e si occupa di verificarne il comportamento, ne gestisce l'avvio, effettua il riavvio della web-app in caso di arresti anomali, riavvio del server, ecc.

Per abilitare Supervisor sulla web-app django è necessaria la creazione di un file di configurazione e abilitarne l'utilizzo e avviare il servizio.

Utilizzare privilegi di `root` o `sudo` e disattivare la variabile d'ambiente  con il comando `deactivate` se ancora abilitata.

- `touch /etc/supervisor/conf.d/django-webapp.conf`
- `nano /etc/supervisor/conf.d/django-webapp.conf` - Aggiornare il file utilizzando come bozza, da modificare a seconda delle esigenze il seguente codice.

```
[program:django-webapp]
command = /var/www/django-webapp/venv/bin/gunicorn_start                ; comando da eseguire per avviare la web-app
user = www-data                                                         ; Utente utilizzato per l'esecuzione (www-data è l'utente di default di nginx)
stdout_logfile = /var/www/django-webapp/logs/gunicorn_supervisor.log   	; Percorso dove scrivere i messaggi di log
redirect_stderr = true                                              	; Salva stderr nel file di log
```

Abilitare Supervisor sulla web-app con i seguenti comandi

- `supervisorctl reread` - Effettua lettura dei file di configurazione di Supervisor nella cartella `/etc/supervisor/conf.d`.
- `supervisorctl update` - Aggiorna eventuali modifiche sui file di configurazione.
- `supervisorctl start django-webapp` - Supervisor avvia la supervisione sulla la web-app _django-webapp_.


Comandi utili di Supervisor

- `supervisorctl status` - Verifica lo stato dei programmi gestiti.
- `supervisorctl restart nome-programma` - Arresta la supervisione sul programma indicato.
- `supervisorctl tail nome-programma` - Stampa in console l'output della supervisione sul programma indicato.


## Configurazione Nginx

Di seguito viene riportata i passaggi per la configurazione di Nginx utilizzando un _vhost_ specifico per la web-app.

Utilizzare privilegi di `root` o `sudo`.

- Creare il file di configurazione per la django-webapp. 
	- `touch  /etc/nginx/sites-available/django-webapp`
- Abilitare la configurazione della web-app su Nginx.
	- `ln -s /etc/nginx/sites-available/django-webapp /etc/nginx/sites-enabled/`
- Modificare la configurazione `django-webapp`
	- `nano /etc/nginx/sites-available/django-webapp`
	- Utilizzare come spunto le seguenti configurazioni facendo attenzione ai puntamenti dei parametri di _upstream_, _location_ e _alias_. Vedi [Struttura finale della cartella di progetto](#struttura-finale-della-cartella-di-progetto).


```
upstream django-webapp {
  server unix:/var/www/django-webapp/run/gunicorn.sock fail_timeout=0;
}

server {
	listen 80 ;
	listen [::]:80 ;

	server_name django-webapp.tuo-dominio.it;
	
	client_max_body_size 32M;
	
	access_log /var/www/django-webapp/logs/nginx_access.log;
	error_log /var/www/django-webapp/logs/nginx_error.log error;

    location = /favicon.ico {
		alias /var/www/django-webapp/assets/favicon.ico;
		log_not_found off;
		access_log off;
	}
	
	location = /robots.txt {
        alias /var/www/django-webapp/assets/robots.txt;
    }
    
    location /static/ {
        alias /var/www/django-webapp/static/;
    }
	
	location /assets/ {
        alias /var/www/django-webapp/assets/;
    }

    location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_redirect off;
        if (!-f $request_filename) {
            proxy_pass http://django-webapp;
            break;
        }
    }
}
```

- `nginx -t` - Verifica le configurazioni di Nginx e stampa eventuali errori 


## Rifiniture

Prima di riavviare Nginx e rendere effettive le modifiche è necessario eseguire alcuni passaggi opzionali ma che potrebbero essere propedeutici al corretto funzionamento della web-app.

Utilizzare privilegi di `root` o `sudo`.

- Assicurazione che i privilegi sui file nella cartella di progetto siano correttamente configurati.
	- `chown www-data:users -R /var/www/django-webapp`
	- `chmod g+w -R /var/www/django-webapp`
- `systemctl restart nginx` - Riavviare Nginx.


Se tutto è andato per il meglio, navigando su `http://django-webapp.tuo-dominio.it` la web-app django dovrebbe essere fruibile.


## Struttura finale della cartella di progetto

Esempio della struttura riportante i principali componenti della cartella di progetto `django-webapp`. Da tenere presente per la configurazione del virtual host di Nginx, della configurazione di Gunicorn e di Supervisor.

```
/var/www/django-webapp/
├── manage.py
├── app
├── assets
|   ├── favicon.ico
|   └── robots.txt
├── logs
|   ├── nginx_access.log 
|   ├── nginx_error.log 
|   └── gunicorn_supervisor.log.py
├── mysite
|   ├── settings.py
|   └── wsgi.py
├── static
├── run
|   └── gunicorn.sock
└── venv
    └── bin
        ├── activate
        ├── gunicorn
        ├── gunicorn_start
        ├── pip
        ├── django-admin
        └── python
```


## Risoluzione dei problemi

Molti problemi in questo tipo di operazioni sono legati ai permessi sui file, puntamenti errati e ad eventuali errori di sintassi nei file.


## Riferimenti

- [https://django-project-skeleton.readthedocs.io/en/latest/nginx_vhost.html]()
- [https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-20-04]()
- [https://michal.karzynski.pl/blog/2013/06/09/django-nginx-gunicorn-virtualenv-supervisor/]()
- [https://michal.karzynski.pl/blog/2013/10/29/serving-multiple-django-applications-with-nginx-gunicorn-supervisor/]()
- [https://medium.com/@mijlalawan/deploying-multiple-django-apps-on-a-vps-with-gunicorn-and-nginx-e47edc6bbf60]()
