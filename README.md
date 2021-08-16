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

## Galera
> Pregunta 6 : Que oción usaremos de configuración para escoger de que nodo va a hacer la sincronización inicial un nodo en concreto?
> Pregunta 7 : Para el servicio de mysql en el nodo Delta. A continuación configura adecuadamente Omega como nodo que nos va entregar los datos. Borra el contenido de /var/lib/mysql/* y a continuación enciende el servicio de mysql. Comprueba que sincronización proviene de Omega y no afecta al servicio de Bravo. Envía capturas con las evidencias.
