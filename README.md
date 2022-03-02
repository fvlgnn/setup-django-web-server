# Setup Django on Web Server

Install and setup Django with PostgreSQL or SQLite, Nginx with Gunicorn or Apache with wsgi on virtual environment.


## Essential Setup / Requirement


### Setup

- Setup Region and Local Zone on OS
- `sudo apt install python3-pip python3-dev python3-venv`
- `sudo apt install curl git sqlite3 build-essential`

### Firewall (optional)

- `sudo ufw allow 8000`
- `sudo ufw delete allow 8000`
- `sudo ufw allow 'Nginx Full'`


### Workspace

- `mkdir ~/dummyapp`
- `cd ~/dummyapp`


### Virtual Environment

- `sudo -H pip3 install --upgrade pip`
- `sudo -H pip3 install virtualenv`
- `virtualenv venv`
- `source venv/bin/activate`


### HowTo: Nginx - Gunicorn ([EN](nginx-gunicorn/README.md) - [IT](nginx-gunicorn/README.it.md))

For install Django on Nginx with Gunicorn ([english howto](nginx-gunicorn/README.md) - [italian howto](nginx-gunicorn/README.it.md)) for multiple site and specific virtual environment view folder [nginx-gunicorn](nginx-gunicorn/) 


### HowTo: Apache - mod_wsgi ([EN](apache-wsgi/README.md) - [IT](apache-wsgi/README.it.md))

For install Django on Apache with WSGI ([english howto](apache-wsgi/README.md) - [italian howto](apache-wsgi/README.it.md)) for multiple site and specific virtual environment view folder [apache-wsgi](apache-wsgi/) 



### PostgreSQL

- `sudo apt install libpq-dev postgresql postgresql-contrib`
- verify 
    - `client_encoding TO 'UTF-8'`
    - `timezone TO 'UTC or YOUR TIMEZONE'`
    - `default_transaction_isolation TO 'read committed'`
- `sudo systemctl status postgresql`
- `sudo systemctl start postgresql` or `sudo pg_ctlcluster 12 main start`
- `sudo systemctl enable postgresql`
- `sudo -u postgres psql`

```sql
CREATE DATABASE dummydb;
CREATE USER dummyuser WITH PASSWORD 'dummypassword';
GRANT ALL PRIVILEGES ON DATABASE dummydb TO dummyuser;
```

- `\q`

