
# Writeup: Máquina Smol

## 1. Reconocimiento y Enumeración

Iniciamos con un escaneo de puertos.
![Escaneo de puertos con Nmap](images/Pasted%20image%2020260102191827.png)

Detectamos los puertos **22 (SSH)** y **80 (HTTP)** abiertos. Al parecer, el puerto 80 nos está redirigiendo a un dominio llamado `smol.thm`. Como nuestro sistema no sabe resolver `smol.thm`, debemos añadirlo a nuestro archivo `/etc/hosts`.

![Edición de /etc/hosts](images/Pasted%20image%2020260102191945.png)

Una vez configurado, podemos acceder a la página web:
![Página web principal](images/Pasted%20image%2020260102192008.png)

Utilizando **Wappalyzer**, confirmamos que hay un **WordPress** corriendo en el servidor.
![Wappalyzer detectando WordPress](images/Pasted%20image%2020260102192343.png)

Con **Gobuster** enumeré algunos directorios, pero no encontré nada interesante, así que procedí a usar otra herramienta específica para este CMS llamada **WPScan**.

![Resultado de WPScan](images/Pasted%20image%2020260102192612.png)

WPScan encontró un plugin que es vulnerable a **LFI (Local File Inclusion)**.

## 2. Explotación Web (LFI)

Dentro de la ruta del plugin vulnerable, encontramos un archivo `.php` que parece permitir la inclusión de archivos locales.
![Archivo vulnerable](images/Pasted%20image%2020260102193146.png)

Investigando un poco, encontré que inyectando una query específica en la URL (usando wrappers de PHP), podemos visualizar el contenido del archivo `wp-config.php`.

![Explotación LFI para leer config](images/Pasted%20image%2020260102193917.png)

Efectivamente, logramos leer el archivo y obtener las credenciales de la base de datos.
![Credenciales encontradas](images/Pasted%20image%2020260102194001.png)

Con estas credenciales, logramos iniciar sesión en el panel `wp-admin`.
![Dashboard de WordPress](images/Pasted%20image%2020260102194105.png)

Dentro del dashboard, encontré una pista que sugiere que hay una vulnerabilidad en el plugin **Hello Dolly**.
![Pista sobre Hello Dolly](images/Pasted%20image%2020260102194503.png)

Usando la vulnerabilidad LFI anterior, leí el código fuente del plugin Hello Dolly. Revisando el código, encontré una cadena en **base64** sospechosa.
![Código base64 en plugin](images/Pasted%20image%2020260102194926.png)

Al decodificarla, vemos que básicamente nos permite ejecutar comandos en el sistema.
![Decodificación de la cadena](images/Pasted%20image%2020260102195028.png)

Volví al dashboard y probé la ejecución de comandos añadiendo `?cmd=id` en la URL. El servidor respondió con `www-data`, confirmando que tenemos **RCE (Remote Code Execution)**.
![Confirmación de RCE](images/Pasted%20image%2020260102195143.png)

## 3. Acceso Inicial (Reverse Shell)

Para ganar acceso al sistema, voy a utilizar este payload URL-encoded para conseguir una Reverse Shell.
![Payload Reverse Shell](images/Pasted%20image%2020260102200311.png)

Nos ponemos en escucha con Netcat en nuestra máquina atacante:
![Netcat escuchando](images/Pasted%20image%2020260102195323.png)

Ejecutamos el payload y ya estamos dentro del sistema.
![Acceso obtenido](images/Pasted%20image%2020260102200349.png)

## 4. Movimiento Lateral

Dentro del directorio `/home` observamos que existen varios usuarios.
![Usuarios en /home](images/Pasted%20image%2020260102200539.png)

Reutilizando las credenciales que encontramos anteriormente en el `wp-config.php`, pude entrar en la base de datos de WordPress.
![Acceso a MySQL](images/Pasted%20image%2020260102201245.png)

Aquí extraje los hashes de las contraseñas de los usuarios.
![Hashes de usuarios](images/Pasted%20image%2020260102201459.png)

Guardé los usuarios en un archivo con el formato adecuado para crackearlos con **John the Ripper**.
![Formato para John](images/Pasted%20image%2020260102202124.png)

John encontró la contraseña para el usuario **diego**, así que la voy a probar inmediatamente.
![John crackeando password](images/Pasted%20image%2020260102202446.png)

Logramos acceder como el usuario diego.
![Login como diego](images/Pasted%20image%2020260102202526.png)

### Usuario Think

Enumerando el sistema, vemos que el usuario **think** tiene un directorio `.ssh` visible. Es posible que contenga su clave privada `id_rsa`.
![Directorio .ssh de think](images/Pasted%20image%2020260102203405.png)

Efectivamente, aquí tenemos el `id_rsa`.
![Clave privada RSA](images/Pasted%20image%2020260102203456.png)

Usamos la clave para conectarnos y ya estamos dentro como **think**.
![Login como think](images/Pasted%20image%2020260102203558.png)

### Usuario Gege

Revisando una ruta que vi en el output de LinPEAS, parece que hay una regla de seguridad escrita.
![Ruta sospechosa](images/Pasted%20image%2020260102205842.png)

Esto es lo que hace esa regla: básicamente, *"Si el usuario think intenta convertirse en gege, déjalo pasar inmediatamente sin pedir contraseña"*.
![Regla de permisos](images/Pasted%20image%2020260102210010.png)

Ejecutando `su - gege` pude transformarme en el usuario **gege**.
![Login como gege](images/Pasted%20image%2020260102205412.png)

## 5. Escalada de Privilegios (Root)

Ahora puedo trabajar con un archivo comprimido que había visto antes. Me lo voy a llevar a mi máquina local para examinarlo.
![Transferencia de archivo](images/Pasted%20image%2020260102210116.png)

Voy a intentar usar `zip2john` para obtener el hash y ver si puedo descifrar la contraseña del ZIP.
![Ejecución de zip2john](images/Pasted%20image%2020260102210333.png)

John nos acaba de dar la contraseña para el archivo.
![Password del zip crackeada](images/Pasted%20image%2020260102210637.png)

Descomprimo el archivo y reviso el directorio resultante.
![Directorio descomprimido](images/Pasted%20image%2020260102210801.png)

En este archivo encontramos las credenciales para convertirnos en el usuario **xavi**.
![Credenciales de xavi](images/Pasted%20image%2020260102211241.png)

Ya estamos dentro como **xavi**.
![Login como xavi](images/Pasted%20image%2020260102211325.png)

Estando como xavi, reviso mis permisos (`sudo -l`) y me fijo que puedo ejecutar cualquier comando como root.
![Permisos sudo](images/Pasted%20image%2020260102211405.png)

Ahora, al ejecutar `sudo su` (o el comando permitido), nos transformaremos en root.
![Escalada a root](images/Pasted%20image%2020260102211458.png)

Y ya somos **root**. PWNED!
![Root access](images/Pasted%20image%2020260102211525.png)

Aquí está la flag:
![Root Flag](images/Pasted%20image%2020260102211624.png)






