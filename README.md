# Práctica: instalar tres SGBD

Abel · 16/10/2026 · SGBD ASIR2

Esta práctica consiste en instalar Oracle 23ai Free, PostgreSQL y MariaDB en una misma máquina virtual, crear una base de datos y un usuario en cada uno y comprobar que se puede entrar. Lo hice siguiendo el orden del enunciado.

** Para la máquina virtual.** Usé VirtualBox con una VM llamada `SGBD_LinuxMint_Abelmm`, un Linux Mint de 64 bits con 4 GB de RAM y 2 procesadores. El disco VDI empezó en 25 GB y lo tuve que ampliar a 50 GB. La red está en NAT y la IP de la VM es <IP>.

![Configuración de la VM](evidencias/00-vm-config.png)

*Configuración de la VM en VirtualBox.*

**Versiones instaladas:**

| SGBD | Versión | Cómo lo comprobé |
|---|---|---|
| Oracle Database 23ai Free | 23.5.0.0 | `sudo podman images` |
| PostgreSQL | 16.15 | `psql --version` |
| MariaDB | <versión> | `mariadb --version` |

## 1. Oracle Database 23ai Free con Podman

Primero instalé Podman y bajé la imagen de Oracle:

```bash
sudo apt update
sudo apt install podman
sudo podman pull container-registry.oracle.com/database/free:23.5.0.0
sudo podman images
```

Después creé el volumen con nombre, para que los datos no se pierdan si se borra el contenedor, y lancé el contenedor publicando el puerto 1521:

```bash
sudo podman volume create oracle_23ai_datos
sudo podman run -d --name cont-oracle -p 1521:1521 --restart=always -v oracle_23ai_datos:/opt/oracle/oradata:U container-registry.oracle.com/database/free:23.5.0.0
sudo podman ps
sudo podman ps -a
sudo podman logs cont-oracle
```

Tardó un rato en arrancar. Fui repitiendo `podman ps` hasta que salió *healthy*, y en los logs apareció `DATABASE IS READY TO USE!`.

![Contenedor en healthy](evidencias/01-oracle-ps.png)

*`podman ps` con el contenedor ya sano.*

Como no puse `ORACLE_PWD`, la imagen generó una contraseña de administración. La miré en los logs y luego la cambié por una mía.

Para crear el usuario entré como sysdba y trabajé dentro de la PDB `FREEPDB1`:

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

Y comprobé que `orauser` podía entrar:

```bash
sudo podman exec -it cont-oracle sqlplus orauser@//localhost:1521/FREEPDB1
```

Una vez dentro ejecuté `SHOW USER` para confirmar con qué usuario estaba conectado.

![Sesión como orauser](evidencias/02-oracle-showuser.png)

*Sesión abierta en `FREEPDB1`; `SHOW USER` devuelve `ORAUSER`.*

## 2. PostgreSQL

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib php-pgsql
sudo systemctl start postgresql
sudo systemctl enable postgresql
sudo -i -u postgres
psql
```

Ya en `psql` creé la base y el usuario:

```sql
CREATE DATABASE dbpg;
CREATE USER pguser WITH PASSWORD '<clave>';
GRANT ALL PRIVILEGES ON DATABASE dbpg TO pguser;
\q
```

Salí del usuario `postgres` con `exit` y probé a entrar con `pguser`:

```bash
psql -h localhost -U pguser -d dbpg -W
```

![Conexión a PostgreSQL](evidencias/03-postgres.png)

*Conectado como `pguser` a `dbpg`; con `SELECT version();` sale la versión.*

## 3. MariaDB

```bash
sudo apt update
sudo apt install mariadb-server mariadb-client
sudo systemctl enable --now mariadb
sudo mariadb
```

Dentro de MariaDB:

```sql
CREATE DATABASE dbm CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'muser'@'localhost' IDENTIFIED BY '<clave>';
GRANT ALL PRIVILEGES ON dbm.* TO 'muser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Y la comprobación:

```bash
mariadb -u
