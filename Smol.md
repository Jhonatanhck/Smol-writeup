# Writeup: Máquina Smol

## 1. Reconocimiento y Enumeración

Iniciamos con un escaneo de puertos.

<img src="Images/Pasted image 20260102191827.png" alt="Escaneo de puertos">

Detectamos los puertos **22 (SSH)** y **80 (HTTP)** abiertos. Al parecer, el puerto 80 nos está redirigiendo a un dominio llamado `smol.thm`. Como nuestro sistema no sabe resolver `smol.thm`, debemos añadirlo a nuestro archivo `/etc/hosts`.

<img src="Images/Pasted image 20260102191945.png" alt="Edicion hosts">

Una vez configurado, podemos acceder a la página web:

<img src="Images/Pasted image 20260102192008.png" alt="Web principal">

Utilizando **Wappalyzer**, confirmamos que hay un **WordPress** corriendo en el servidor.

<img src="Images/Pasted image 20260102192343.png" alt="Wappalyzer">

Con **Gobuster** enumeré algunos directorios, pero no encontré nada interesante, así que procedí a usar otra herramienta específica para este CMS llamada **WPScan**.

<img src="Images/Pasted image 20260102192612.png" alt="WPScan">

WPScan encontró un plugin que es vulnerable a **LFI (Local File Inclusion)**.

## 2. Explotación Web (LFI)

Dentro de la ruta del plugin vulnerable, encontramos un archivo `.php` que parece permitir la inclusión de archivos locales.

<img src="Images/Pasted image 20260102193146.png" alt="LFI Vulnerable">

Investigando un poco, encontré que inyectando una query específica en la URL (usando wrappers de PHP), podemos visualizar el contenido del archivo `wp-config.php`.

<img src="Images/Pasted image 20260102193917.png" alt="LFI Config">

Efectivamente, logramos leer el archivo y obtener las credenciales de la base de datos.

<img src="Images/Pasted image 20260102194001.png" alt="Credenciales DB">

Con estas credenciales, logramos iniciar sesión en el panel `wp-admin`.

<img src="Images/Pasted image 20260102194105.png" alt="WP Admin">

Dentro del dashboard, encontré una pista que sugiere que hay una vulnerabilidad en el plugin **Hello Dolly**.

<img src="Images/Pasted image 20260102194503.png" alt="Pista Hello Dolly">

Usando la vulnerabilidad LFI anterior, leí el código fuente del plugin Hello Dolly. Revisando el código, encontré una cadena en **base64** sospechosa.

<img src="Images/Pasted image 20260102194926.png" alt="Codigo Base64">

Al decodificarla, vemos que básicamente nos permite ejecutar comandos en el sistema.

<img src="Images/Pasted image 20260102195028.png" alt="Decodificado">

Volví al dashboard y probé la ejecución de comandos añadiendo `?cmd=id` en la URL. El servidor respondió con `www-data`, confirmando que tenemos **RCE (Remote Code Execution)**.

<img src="Images/Pasted image 20260102195143.png" alt="RCE Confirmado">

## 3. Acceso Inicial (Reverse Shell)

Para ganar acceso al sistema, voy a utilizar este payload URL-encoded para conseguir una Reverse Shell.

<img src="Images/Pasted image 20260102200311.png" alt="Payload">

Nos ponemos en escucha con Netcat en nuestra máquina atacante:

<img src="Images/Pasted image 20260102195323.png" alt="Netcat Listen">

Ejecutamos el payload y ya estamos dentro del sistema.

<img src="Images/Pasted image 20260102200349.png" alt="Acceso Shell">

## 4. Movimiento Lateral

Dentro del directorio `/home` observamos que existen varios usuarios.

<img src="Images/Pasted image 20260102200539.png" alt="Usuarios Home">

Reutilizando las credenciales que encontramos anteriormente en el `wp-config.php`, pude entrar en la base de datos de WordPress.

<img src="Images/Pasted image 20260102201245.png" alt="MySQL">

Aquí extraje los hashes de las contraseñas de los usuarios.

<img src="Images/Pasted image 20260102201459.png" alt="Hashes">

Guardé los usuarios en un archivo con el formato adecuado para crackearlos con **John the Ripper**.

<img src="Images/Pasted image 20260102202124.png" alt="Formato John">

John encontró la contraseña para el usuario **diego**, así que la voy a probar inmediatamente.

<img src="Images/Pasted image 20260102202446.png" alt="John Cracked">

Logramos acceder como el usuario diego.

<img src="Images/Pasted image 20260102202526.png" alt="Login Diego">

### Usuario Think

Enumerando el sistema, vemos que el usuario **think** tiene un directorio `.ssh` visible. Es posible que contenga su clave privada `id_rsa`.

<img src="Images/Pasted image 20260102203405.png" alt="SSH Folder">

Efectivamente, aquí tenemos el `id_rsa`.

<img src="Images/Pasted image 20260102203456.png" alt="ID RSA">

Usamos la clave para conectarnos y ya estamos dentro como **think**.

<img src="Images/Pasted image 20260102203558.png" alt="Login Think">

### Usuario Gege

Revisando una ruta que vi en el output de LinPEAS, parece que hay una regla de seguridad escrita.

<img src="Images/Pasted image 20260102205842.png" alt="Ruta Regla">

Esto es lo que hace esa regla: básicamente, *"Si el usuario think intenta convertirse en gege, déjalo pasar inmediatamente sin pedir contraseña"*.

<img src="Images/Pasted image 20260102210010.png" alt="Regla Cat">

Ejecutando `su - gege` pude transformarme en el usuario **gege**.

<img src="Images/Pasted image 20260102205412.png" alt="Login Gege">

## 5. Escalada de Privilegios (Root)

Ahora puedo trabajar con un archivo comprimido que había visto antes. Me lo voy a llevar a mi máquina local para examinarlo.

<img src="Images/Pasted image 20260102210116.png" alt="Transferencia">

Voy a intentar usar `zip2john` para obtener el hash y ver si puedo descifrar la contraseña del ZIP.

<img src="Images/Pasted image 20260102210333.png" alt="Zip2John">

John nos acaba de dar la contraseña para el archivo.

<img src="Images/Pasted image 20260102210637.png" alt="Pass Zip">

Descomprimo el archivo y reviso el directorio resultante.

<img src="Images/Pasted image 20260102210801.png" alt="Unzip">

En este archivo encontramos las credenciales para convertirnos en el usuario **xavi**.

<img src="Images/Pasted image 20260102211241.png" alt="Creds Xavi">

Ya estamos dentro como **xavi**.

<img src="Images/Pasted image 20260102211325.png" alt="Login Xavi">

Estando como xavi, reviso mis permisos (`sudo -l`) y me fijo que puedo ejecutar cualquier comando como root.

<img src="Images/Pasted image 20260102211405.png" alt="Sudo L">

Ahora, al ejecutar `sudo su` (o el comando permitido), nos transformaremos en root.

<img src="Images/Pasted image 20260102211458.png" alt="Sudo Su">

Y ya somos **root**. PWNED!

<img src="Images/Pasted image 20260102211525.png" alt="Root">

Aquí está la flag:

<img src="Images/Pasted image 20260102211624.png" alt="Flag">

