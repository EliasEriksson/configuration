# Postgres 14 install and configuration

## install
```bash
apt install postgresql
```

## logging in as superuser
```bash
su postgres -c psql
```

## creating a user with database
login as superuser then run
```sql
CREATE DATABASE "elias-eriksson";
CREATE USER "elias-eriksson" WITH ENCRYPTED PASSWORD 'elias-eriksson';
GRANT ALL PRIVILEGES ON DATABASE "elias-eriksson" to "elias-eriksson";
```
