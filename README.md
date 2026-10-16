# Práctica: Oracle 23ai Free, PostgreSQL y MariaDB en una VM

Abel · <fecha> · <módulo>

Instalé tres gestores de bases de datos en la misma VM, creé una base y un usuario en cada uno y comprobé la conexión. Aquí está lo que hice, en orden.

## 1. VM y versiones instaladas

La VM se llama `SGBD_LinuxMint_Abelmm` y está en VirtualBox. Es un Linux Mint de 64 bits con 4 GB de RAM, 2 procesadores y un disco VDI que empecé con 25 GB y amplié a 150 GB. Tiene paginación anidada y paravirtualización KVM. La red es un adaptador Intel PRO/1000 MT en NAT, y la IP de la VM es <IP>.

![Configuración de la VM](evidencias/00-vm-config.png)

La captura muestra la RAM, las CPU, el disco y la red asignados a la VM.

| SGBD | Versión | Cómo lo comprobé |
|---|---|---|
| Oracle Database 23ai Free | 23.5.0.0 | `sudo podman images` |
| PostgreSQL | <versión> | `psql --version` |
| MariaDB | <versión> | `mariadb --version` |

## 2. Pasos y comandos ejecutados

### Oracle (Podman)

Instalé Podman y descargué la imagen oficial:

```bash
sudo apt update
sudo apt install podman
sudo podman pull container-registry.oracle.com/database/free:23.5.0.0
sudo podman images
```

Creé un volumen con nombre para que los datos sobrevivan si se recrea el contenedor, y lancé el contenedor publicando el 1521:

```bash
sudo podman volume create oracle_23ai_datos
sudo podman run -d --name cont-oracle -p 1521:1521 --restart=always -v oracle_23ai_datos:/opt/oracle/oradata:U container-registry.oracle.com/database/free:23.5.0.0
sudo podman ps
sudo podman ps -a
sudo podman logs cont-oracle
```

Esperé a que `podman ps` lo mostrara como *healthy*. Los logs terminaron con `DATABASE IS READY TO USE!`.

![Contenedor Oracle healthy](evidencias/01-oracle-ps.png)

La imagen genera una contraseña de administración si no se pasa `ORACLE_PWD`, así que la consulté en los logs y la cambié después. Entré como sysdba y creé el usuario dentro de la PDB `FREEPDB1`:

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

```bash
sudo apt update
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
sudo apt update
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

## 3. Comprobación de estado y conexiones

Para el estado de los servicios usé `systemctl status` sobre `podman-restart.service` (que hace que el contenedor de Oracle vuelva tras reiniciar la VM), `postgresql` y `mariadb`, además de `sudo podman ps` para el contenedor.

![Estado de los servicios](evidencias/05-systemctl.png)

La captura muestra los tres servicios en estado <active>.

Después me conecté con el cliente de cada gestor:

```bash
sudo podman exec -it cont-oracle sqlplus orauser@//localhost:1521/FREEPDB1
psql -h localhost -U pguser -d dbpg -W
mariadb -u muser -p dbm
```

En Oracle ejecuté `SHOW USER` ya dentro de la sesión:

![Oracle: SHOW USER](evidencias/02-oracle-showuser.png)

Demuestra que `orauser` se autentica en `FREEPDB1` (sale `USER is "ORAUSER"`).

![PostgreSQL](evidencias/03-postgres.png)

Demuestra que `pguser` entra en `dbpg` y que el servidor responde con su versión.

![MariaDB](evidencias/04-mariadb.png)

Demuestra que `muser` entra en `dbm` y que el servidor responde con su versión.

## 4. Configuración de red


La VM está en NAT, así que para acceder desde fuera  cambié el adaptador a puente / configuré reenvío de puertos.

- PostgreSQL: `listen_addresses = '<IP_VM>'` en `postgresql.conf` y la regla `host dbpg pguser <red>/24 scram-sha-256` en `pg_hba.conf`.
- MariaDB: `bind-address = <IP_VM>` y un usuario `'muser'@'<IP_CLIENTE>'` con permisos solo sobre `dbm.*`.
- UFW: abrí el 5432 solo para mi red y el 3306 solo para la IP del cliente.
- Pruebas: `nc -zv <IP_VM> <puerto>` y después login con `psql -h` y `mariadb -h`, porque un puerto abierto no prueba la autenticación.

![Conexión remota](evidencias/06-remoto.png)

## 5. Problemas encontrados

**Disco lleno al descargar la imagen de Oracle.** Al hacer el `podman pull` salió este error:
