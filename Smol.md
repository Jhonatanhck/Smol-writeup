
# Writeup: Smol (TryHackMe)

**Autor:** JhonatanHck  
**Fecha:** 03/01/2026  
**Categoría:** Linux / Web / PrivEsc

---

## 1. Reconocimiento y Enumeración

Iniciamos con un escaneo de puertos para identificar servicios activos.

![Escaneo de puertos](images/Pasted%20image%2020260102191827.png)

Detectamos los puertos **22 (SSH)** y **80 (HTTP)** abiertos. Al intentar acceder al puerto 80, el servidor nos redirige al dominio `smol.thm`. Como nuestro sistema no resuelve este dominio por defecto, debemos añadirlo a nuestro archivo `/etc/hosts`.

![Edición de hosts](images/Pasted%20image%2020260102191945.png)

Una vez configurado, podemos visualizar la página web:

![Página web principal](images/Pasted%20image%2020260102192008.png)

Utilizando **Wappalyzer**, confirmamos que el sitio está corriendo sobre **WordPress**.

![Wappalyzer](images/Pasted%20image%2020260102192343.png)

Realicé una enumeración de directorios con **Gobuster**, pero no arrojó resultados críticos. Procedí a enumerar vulnerabilidades específicas de WordPress utilizando **WPScan**.

![WPScan result](images/Pasted%20image%2020260102192612.png)

WPScan detectó un plugin instalado que es vulnerable a **LFI (Local File Inclusion)**.

---

## 2. Explotación Web (LFI a RCE)

Navegando a la ruta del plugin vulnerable, encontramos un archivo `.php` que permite la inclusión de archivos locales del servidor.

![Vulnerabilidad LFI](images/Pasted%20image%2020260102193146.png)

Investigando vectores de ataque, encontré que utilizando wrappers de PHP en la query de la URL, es posible extraer el contenido de archivos sensibles como `wp-config.php`.

![Query LFI](images/Pasted%20image%2020260102193917.png)

La inyección fue exitosa y logramos leer las credenciales de la base de datos dentro del archivo de configuración.

![Credenciales wp-config](images/Pasted%20image%2020260102194001.png)

Con estas credenciales, logramos acceder al panel de administración `wp-admin`.

![Dashboard WordPress](images/Pasted%20image%2020260102194105.png)

Dentro del panel, encontré una pista que sugiere una vulnerabilidad en el plugin **Hello Dolly**.

![Pista Hello Dolly](images/Pasted%20image%2020260102194503.png)

Usando la vulnerabilidad LFI anterior, leí el código fuente del plugin Hello Dolly y encontré una cadena en base64 sospechosa.

![Código base64](images/Pasted%20image%2020260102194926.png)

Al decodificarla, confirmamos que el código permite la ejecución de comandos arbitrarios.

![Decodificación](images/Pasted%20image%2020260102195028.png)

Para probarlo, volví al dashboard e inyecté `?cmd=id` en la URL. El sistema respondió con el usuario `www-data`, confirmando que tenemos **RCE (Remote Code Execution)**.

![Prueba de RCE](images/Pasted%20image%2020260102195143.png)

---

## 3. Acceso Inicial (Reverse Shell)

Para conseguir una reverse shell estable, utilicé el siguiente payload URL-encoded:

![Payload Reverse Shell](images/Pasted%20image%2020260102200311.png)

Preparamos el listener con Netcat:

![Netcat listening](images/Pasted%20image%2020260102195323.png)

Ejecutamos el payload en el navegador y obtenemos acceso a la terminal.

![Shell obtenida](images/Pasted%20image%2020260102200349.png)

---

## 4. Movimiento Lateral

Revisando el directorio `/home`, identificamos varios usuarios en el sistema.

![Usuarios home](images/Pasted%20image%2020260102200539.png)

Reutilizando las credenciales encontradas previamente en el `wp-config.php`, accedí a la base de datos MySQL de WordPress.

![Acceso MySQL](images/Pasted%20image%2020260102201245.png)

Extraje los hashes de las contraseñas de los usuarios registrados.

![Hashes usuarios](images/Pasted%20image%2020260102201459.png)

Guardé los usuarios y hashes en un formato apto para ser procesados por **John the Ripper**.

![Formato hash](images/Pasted%20image%2020260102202124.png)

John logró crackear la contraseña para el usuario **diego**.

![John the Ripper result](images/Pasted%20image%2020260102202446.png)

Con esta contraseña, migramos de usuario `www-data` a **diego**.

![Login diego](images/Pasted%20image%2020260102202526.png)

### Escalando a usuario 'think'

Enumerando el sistema, encontramos que el usuario **think** tiene su directorio `.ssh` con permisos de lectura.

![Directorio think](images/Pasted%20image%2020260102203405.png)

Dentro encontramos su clave privada `id_rsa`.

![id_rsa think](images/Pasted%20image%2020260102203456.png)

Usamos la clave para conectarnos por SSH como **think**.

![Login think](images/Pasted%20image%2020260102203558.png)

### Escalando a usuario 'gege'

Revisando una ruta inusual señalada por LinPEAS, descubrí una configuración de seguridad específica.

![Ruta sospechosa](images/Pasted%20image%2020260102205842.png)

El archivo indica una regla: *"Si el usuario think intenta convertirse en gege, déjalo pasar inmediatamente sin pedir contraseña"*.

![Regla cat](images/Pasted%20image%2020260102210010.png)

Ejecutamos `su - gege` y logramos acceder sin password.

![Login gege](images/Pasted%20image%2020260102205412.png)

---

## 5. Escalada de Privilegios (Root)

Como usuario `gege`, tuve acceso a un archivo comprimido sospechoso. Lo transferí a mi máquina local para analizarlo.

![Transferencia archivo](images/Pasted%20image%2020260102210116.png)

Utilicé `zip2john` para extraer el hash del archivo ZIP.

![zip2john](images/Pasted%20image%2020260102210333.png)

John the Ripper nos entregó la contraseña del archivo comprimido.

![Password zip](images/Pasted%20image%2020260102210637.png)

Al descomprimirlo, encontramos un archivo de texto con credenciales para el usuario **xavi**.

![Credenciales xavi](images/Pasted%20image%2020260102211241.png)

Accedemos como xavi.

![Login xavi](images/Pasted%20image%2020260102211325.png)

Finalmente, revisando los permisos de sudo (`sudo -l`), descubrí que xavi tiene permisos para ejecutar cualquier comando como root (`ALL=(ALL:ALL) ALL`).

![Sudoers xavi](images/Pasted%20image%2020260102211405.png)

Ejecutamos `sudo su` para obtener una shell como root.

![Escalada root](images/Pasted%20image%2020260102211525.png)

**¡Máquina PWNED!**

![Flag](images/Pasted%20image%2020260102211624.png)

# Que aprendi

### Resumen de Aprendizaje: Máquina Smol (TryHackMe)

#### 1. Enumeración y CMS (WordPress)

- **Enumeración de Plugins:** No basta con saber que es WordPress. Es crítico listar los plugins y sus versiones.
    
- **Peligro de Plugins Obsoletos:** El plugin `jsmol2wp` demostró cómo un componente de terceros puede comprometer todo el sitio a través de una vulnerabilidad **LFI** (Local File Inclusion).
    

#### 2. Explotación Web (LFI a Credenciales)

- **Lectura de Archivos de Configuración:** El LFI no solo sirve para conseguir RCE directo. La técnica clave aquí fue leer el archivo `wp-config.php` para extraer credenciales de base de datos en texto plano.
    
- **Wrapper Base64 de PHP:** Aprendiste a usar `php://filter/convert.base64-encode/resource=archivo` para "bypassear" la ejecución del código PHP y poder leer el código fuente.
    

#### 3. Análisis de Backdoors y Obfuscación

- **Persistencia en Plugins "Inocentes":** La máquina enseñó que los atacantes (o creadores de CTF) pueden esconder puertas traseras en plugins por defecto como **Hello Dolly** (`hello.php`).
    
- **Deobfuscación Básica:** Identificaste patrones maliciosos como `eval(base64_decode(...))` y aprendiste a decodificar el payload para entender que el parámetro `cmd` permitía la ejecución remota de comandos (RCE).
    

#### 4. Movimiento Lateral y Linux Internals (PAM)

- **Reutilización de Contraseñas:** Un clásico. La contraseña de la base de datos (extraída de `wp-config.php`) fue reutilizada por un usuario del sistema (`think`).
    
- **Configuración de PAM (`/etc/pam.d/su`):** **(Lección Estrella ⭐)**
    
    - Aprendiste que Linux controla la autenticación con módulos PAM.
        
    - Viste cómo una regla `auth sufficient pam_succeed_if.so` puede crear una relación de confianza, permitiendo que un usuario (`think`) se transforme en otro (`gege`) sin contraseña, ignorando la seguridad estándar de `su`.
        

#### 5. Hacking de Contraseñas

- **Identificación de Hashes:** Reconocimiento de hashes `phpass` (WordPress).
    
- **Herramientas:** Uso de **John the Ripper** para crackear hashes volcados de la base de datos.












