# Practica_Instalar_3_SGBD
Repositorio para la practica 2.2 de SGBD
# Práctica: Oracle 23ai Free, PostgreSQL y MariaDB en una VM

Abel Martin Marchena · SGBD

Montar tres gestores de bases de datos en la misma máquina virtual, crear una base y un usuario en cada uno y comprobar que se puede entrar. Lo que sigue es lo que fui haciendo.

## 1. La VM y las versiones

La VM la hice en VirtualBox y se llama `SGBD_LinuxMint_Abelmm`. Es un Linux Mint de 64 bits con 4 GB de RAM, 2 procesadores y un disco VDI de 250 GB en SATA. La red es un adaptador Intel PRO/1000 MT en modo NAT

Se ve en la captura [00-vm-config.png](evidencias/00-vm-config.png), que muestra la configuración de la máquina en VirtualBox.

Versiones instaladas:

- Oracle Database 23ai Free: 23.5.0.0 (imagen de Podman)
- PostgreSQL: <versión> (`psql --version`)
- MariaDB: <versión> (`mariadb --version`)


## 2.COMANDOS

### Oracle (Podman)

Oracle es el único que va en contenedor. Instalé Podman, bajé la imagen y creé un volumen con nombre para que los datos no se pierdan si recreo el contenedor:

```bash
sudo apt update
sudo apt install podman
sudo podman pull container-registry.oracle.com/database/free:23.5.0.0
sudo podman volume create oracle_23ai_datos
sudo podman run -d --name cont-oracle -p 1521:1521 --restart=always \
  -v oracle_23ai_datos:/opt/oracle/oradata:U \
  container-registry.oracle.com/database/free:23.5.0.0
```

El primer arranque tarda. Fui mirando `sudo podman ps` hasta que salió *healthy*. La contraseña de administrador la genera la imagen si no se le pasa `ORACLE_PWD`, así que la saqué de `sudo podman logs cont-oracle`.

![Contenedor Oracle healthy](evidencias/01-oracle-ps.png)

Luego entré como sysdba y creé el usuario dentro de la PDB `FREEPDB1`, no en el contenedor raíz:

```bash
sudo podman exec -it cont-oracle sqlplus / as sysdba
```

```sql
SHOW PDBS
ALTER SESSION SET CONTAINER = FREEPDB1;
CREATE USER orauser IDENTIFIED BY "<clave>";
GRANT CREATE SESSION, CREATE TABLE TO orauser;
ALTER USER orauser QUOTA 100M ON USERS;
EXIT
```

### PostgreSQL

Este salió directo de los repositorios:

```bash
sudo apt install postgresql postgresql-contrib php-pgsql
sudo systemctl start postgresql
sudo systemctl enable postgresql
sudo -i -u postgres
psql
```

```sql
CREATE DATABASE dbpg;
CREATE USER pguser WITH PASSWORD '<clave>';
GRANT ALL PRIVILEGES ON DATABASE dbpg TO pguser;
\q
```

### MariaDB

```bash
sudo apt install mariadb-server mariadb-client
sudo systemctl enable --now mariadb
sudo mariadb
```

```sql
CREATE DATABASE dbm CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'muser'@'localhost' IDENTIFIED BY '<clave>';
GRANT ALL PRIVILEGES ON dbm.* TO 'muser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## 3. Cómo comprobé que funcionaba

Para el estado de los servicios usé `sudo podman ps` (Oracle), `systemctl status postgresql` y `systemctl status mariadb`. Los tres estaban <activos / healthy>.

Después entré con cada usuario desde su cliente:

```bash
sudo podman exec -it cont-oracle sqlplus orauser@//localhost:1521/FREEPDB1
psql -h localhost -U pguser -d dbpg -W
mariadb -u muser -p dbm
```

![Login de orauser](evidencias/02-oracle-orauser.png)

![Login de pguser](evidencias/03-postgres.png)

![Login de muser](evidencias/04-mariadb.png)

Las tres capturas muestran que cada usuario se autentica en su base y que el servidor responde con su versión.


## 4. Red
