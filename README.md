# WB5-mysql-mariadb

He creado una bdd con el password root.

> Pregunta 1 : Vaya para este servicio parece un poco justo 151, las conexiones sigueen entrando y esta afectando a producción. Prueba a subir las conexiones máximas a 201 sin editar ningun archivo de configuración ni reiniciar el servicio.

Como usuario root de la bdd:
```
set global max_connections=201;
Query OK, 0 rows affected (0.001 sec)
		
MariaDB [(none)]> show variables like 'max_connections';
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections | 201   |
+-----------------+-------+
1 row in set (0.002 sec)
```

> Pregunta 2 : Crea un usuario de mysql separado para el desarrollador usando la metodologia mostrada anteriormente pero solo con permisos de lectura (SELECT). 
> Descarga un gestor de base de datos tipo dbeaver, configura una conexión remota usando tunel ssh con el nuevo usuario y comprueba que todo funciona correctamente (haciendo un select y un delete). Deberás crear una tabla e introducir datos con el usuario root antes.

Creamos el usuario de sistema developer con la pass developer:
```
useradd -s /bin/false developer
passwd developer
		
grant SELECT on test.* to 'developer'@'localhost' identified by 'supersecurepassword';
```		
		
> Pregunta 3 : Parece que la persona que instaló el mysql, no apuntó la contraseña de root y se ha perdido. Busca una forma de resetear el pw de root de mysql si no la tubieramos y no pudieramos loguear en el promt con root.
	
Con esta solución simple se puede resetear con el promt
https://blogs.ua.es/jpm33/2015/02/18/resetear-el-password-del-root-en-mysql/ 
Y esta en caso de que no podamos acceder:
  
1. Edit the configuration file using: $ sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
2. Add skip-grant-tables under [mysqld] block and save the changes.
3. Restart MySQL service using: sudo service mysql restart
4. Check MySQL service status: sudo service mysql status
 ![image](https://user-images.githubusercontent.com/65896169/126289745-e1e76b57-c18c-4065-8e28-eb9841f7c604.png)
5. Login to mysql with: $ mysql -u root
6. And change the root password:
		mysql> FLUSH PRIVILEGES;
		mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'newpassword';
7. Revert back the MySQL configuration file changes by removing skip-grant-tables line or commenting it with a # (hash).
8. Finally restart the MySQL service and you are good to go.

## Cluster bdd

Para comprobar que una réplica funciona correctamente nos vamos a fijar sobretodo en Slave_IO_Running y además en los Seconds_Behind_Master que no suban demasiado.

> Pregunta 4 : Inserta varias rows de información en la tabla equipo. Comprueba que replica todo correctamente. Crea otra tabla replica y también inserta varias rows. De nueve vuelve a comprobar que replica correctamente.

`insert into equipo values (4,"kike",5,"negro");`
![image](https://user-images.githubusercontent.com/65896169/129566592-d3aa7d81-ce3a-4070-9126-7ba78287a9b2.png)

`CREATE TABLE test.replica ( id INT NOT NULL AUTO_INCREMENT, nombre VARCHAR(50), numero INT, color VARCHAR(25), PRIMARY KEY(id));`
![image](https://user-images.githubusercontent.com/65896169/129567357-459829c1-0e6d-452e-80a8-d4ea9249d25b.png)

> Pregunta 5 : La replicación ahora funciona pero estamos en la nube, hay que securizar las comunicaciones. Configura la replica para usar ssl MASTER_SSL=1.

Creamos los certificados, para ello usamos el siguiente script:
```
mkdir /etc/mysql/certs && cd /etc/mysql/certs

# Create CA certificate
echo "CREANDO CERTIFICADO DE CA"
openssl genrsa 2048 > ca-key.pem
openssl req -new -x509 -nodes -days 3600 \
        -key ca-key.pem -out ca.pem

# Create server certificate, remove passphrase, and sign it
# server-cert.pem = public key, server-key.pem = private key
echo "CREANDO CERTIFICADO DE SERVIDOR"
openssl req -newkey rsa:2048 -days 3600 \
        -nodes -keyout server-key.pem -out server-req.pem
openssl rsa -in server-key.pem -out server-key.pem
openssl x509 -req -in server-req.pem -days 3600 \
        -CA ca.pem -CAkey ca-key.pem -set_serial 01 -out server-cert.pem

# Create client certificate, remove passphrase, and sign it
# client-cert.pem = public key, client-key.pem = private key
echo "CREANDO CERTIFICADO DE CLIENTE"
openssl req -newkey rsa:2048 -days 3600 \
        -nodes -keyout client-key.pem -out client-req.pem
openssl rsa -in client-key.pem -out client-key.pem
openssl x509 -req -in client-req.pem -days 3600 \
        -CA ca.pem -CAkey ca-key.pem -set_serial 01 -out client-cert.pem

echo "VERIFICANDO CERTIFICADOS"
openssl verify -CAfile ca.pem server-cert.pem client-cert.pem
```
Y modificamos el fichero /etc/mysql/mariadb.conf.d/50-server.cnf para añadir las rutas en el master:
```
[mysqld]
ssl-ca = /etc/mysql/certs/ca.pem
ssl-cert = /etc/mysql/certs/server-cert.pem
ssl-key = /etc/mysql/certs/server-key.pem
```
Copiamos los certificados generados en el slave en la misma ruta y modificamos /etc/mysql/mariadb.conf.d/50-client.cnf:
```
[client]
ssl-ca=/etc/mysql/certs/ca.pem
ssl-cert=/etc/mysql/certs/client-cert.pem
ssl-key=/etc/mysql/certs/client-key.pem
```

Reiniciamos mysql en master y en el slave hacemos lo siguiente:
```
mysql -uroot -p
STOP SLAVE;
CHANGE MASTER TO MASTER_SSL=1;
START SLAVE;
```

Y verificamos que está correctamente replicando:
```
SHOW SLAVE STATUS\G
*************************** 1. row ***************************
                Slave_IO_State: Waiting for master to send event
                   Master_Host: 65.21.49.147
                   Master_User: replica
                   Master_Port: 3306
                 Connect_Retry: 60
               Master_Log_File: mysql-bin.000011
           Read_Master_Log_Pos: 342
                Relay_Log_File: mysql-bin.000006
                 Relay_Log_Pos: 555
         Relay_Master_Log_File: mysql-bin.000011
              Slave_IO_Running: Yes
             Slave_SQL_Running: Yes
               Replicate_Do_DB:
           Replicate_Ignore_DB:
            Replicate_Do_Table:
        Replicate_Ignore_Table:
       Replicate_Wild_Do_Table:
   Replicate_Wild_Ignore_Table:
                    Last_Errno: 0
                    Last_Error:
                  Skip_Counter: 0
           Exec_Master_Log_Pos: 342
               Relay_Log_Space: 1243
               Until_Condition: None
                Until_Log_File:
                 Until_Log_Pos: 0
            Master_SSL_Allowed: Yes
            Master_SSL_CA_File:
            Master_SSL_CA_Path:
               Master_SSL_Cert:
             Master_SSL_Cipher:
                Master_SSL_Key:
         Seconds_Behind_Master: 0
 Master_SSL_Verify_Server_Cert: No
                 Last_IO_Errno: 0
                 Last_IO_Error:
                Last_SQL_Errno: 0
                Last_SQL_Error:
   Replicate_Ignore_Server_Ids:
              Master_Server_Id: 1
                Master_SSL_Crl:
            Master_SSL_Crlpath:
                    Using_Gtid: No
                   Gtid_IO_Pos:
       Replicate_Do_Domain_Ids:
   Replicate_Ignore_Domain_Ids:
                 Parallel_Mode: conservative
                     SQL_Delay: 0
           SQL_Remaining_Delay: NULL
       Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
              Slave_DDL_Groups: 1
Slave_Non_Transactional_Groups: 0
    Slave_Transactional_Groups: 2
```
![image](https://user-images.githubusercontent.com/65896169/129724499-badc6537-3a33-45a9-8a20-aecc4b5c5f26.png)

## Galera

Iniciamos los 3 nodos:

![image](https://user-images.githubusercontent.com/65896169/129787276-b0a08854-499c-4609-93b3-1d6c02fc1cd3.png)
![image](https://user-images.githubusercontent.com/65896169/129788147-99c5233f-94e5-4623-8669-c6e02123c6ce.png)

> Pregunta 6 : Que oción usaremos de configuración para escoger de que nodo va a hacer la sincronización inicial un nodo en concreto?

Incluimos la opción `wsrep_sst_donor` y ponemos el orden de preferencia de sincronización que queremos:
```wsrep_sst_donor="galera-omega,"```

> Pregunta 7 : Para el servicio de mysql en el nodo Delta. A continuación configura adecuadamente Omega como nodo que nos va entregar los datos. Borra el contenido de /var/lib/mysql/* y a continuación enciende el servicio de mysql. Comprueba que sincronización proviene de Omega y no afecta al servicio de Bravo. Envía capturas con las evidencias.

Hemos parado el nodo delta y borrada la información y hemos configurado que sincronice del nodo omega:
```service mysql stop
rm -rf /var/lib/mysql/*
```

Al arrancar de nuevo podemos apreciar como ha elegido el nodo omega para la sincronización:

```
2021-08-18 10:35:02 0 [Note] WSREP: Member 0.0 (galera-delta) requested state transfer from 'galera-omega,'. Selected 2.0 (galera-omega)(SYNCED) as donor.
2021-08-18 10:35:02 0 [Note] WSREP: Shifting PRIMARY -> JOINER (TO: 310)
2021-08-18 10:35:02 1 [Note] WSREP: Requesting state transfer: success, donor: 2
```

Si lo ejecutamos varias veces vemos que siempre elige omega.
Si paramos el servicio en omega y volvemos a ejecutar, vemos que intenta sincronizar desde omega, pero al no estar disponible elige a bravo:

```
2021-08-18 10:46:11 0 [Note] WSREP: Member 1.0 (galera-delta) requested state transfer from 'galera-omega,'. Selected 0.0 (galera-bravo)(SYNCED) as donor.
2021-08-18 10:46:11 1 [Note] WSREP: Requesting state transfer: success, donor: 0
```
