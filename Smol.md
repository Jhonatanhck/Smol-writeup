Reconocimiento
![[Pasted image 20260102191827.png]]
Tenemos el puerto 22 y 80, al parecer el puerto 80 nos esta redirigiendo a una pagina web llamada smol.thm y como nuestro sistema no sabe que es smol.thm lo tenemos que escribir en el /etc/hosts 

![[Pasted image 20260102191945.png]]
Aqui tenemos la pagina web 

![[Pasted image 20260102192008.png]]
con wappalyzer nos muesstra que tenemos un wordpress corriendo 

![[Pasted image 20260102192343.png]]
con gobuster encontre estos directorios pero no hay nada interesante dentro de ellos asi que voy a hacer uso de otra herramienta llamada wpscan

![[Pasted image 20260102192612.png]]
con wpscan encontre este plugin que es vulnerable a LFI

![[Pasted image 20260102193146.png]]
dentro de esa ruta podemos encontrar este archivo .php que puede que nos permita tener LFI

![[Pasted image 20260102193917.png]]
Investiigando un poco encontre que poniendo esta query en la url podemos ver los datos de wp-config

![[Pasted image 20260102194001.png]]
y efectivamente, aqui tenemos las credenciales

![[Pasted image 20260102194105.png]]
y asi estamos dentro del wp-admin


![[Pasted image 20260102194503.png]]
encontre esta pista que parece que hay una vulnerabilidad con el plugin hello dolly

![[Pasted image 20260102194926.png]]
usando nuestra vulnerabilidad LFI que use anteriormente la utilice para ver el codigo del plugin hello dolly y revisando encontre esta cadena en base64 que es esto

![[Pasted image 20260102195028.png]]
bisicamente podemos ejecutar comandos

![[Pasted image 20260102195143.png]]
volvi al dashboard y poniendo ?cmd=id arriba podemos ver que nos pone www-data asi que tenemos RCE 

![[Pasted image 20260102200311.png]]
Yo voy a utilizar este payload urlencoded para conseguir mi reverse shell

![[Pasted image 20260102195323.png]]
nos ponemos en escucha

![[Pasted image 20260102200349.png]]
y ya estamos dentro

![[Pasted image 20260102200539.png]]
Dentro del directorio home tenemos varios usuarios 

![[Pasted image 20260102201245.png]]
Con las credenciales que encontramos anteriormente pude entrar en la base de datos de wp

![[Pasted image 20260102201459.png]]
aqui tenemos las credenciales de los usuarios 

![[Pasted image 20260102202124.png]]
guarde los usuarios en este formato para crackearlos con john

![[Pasted image 20260102202446.png]]
aqui john encontro la contrasena para el usuario diego asi que la voy a probar inmediatamente

![[Pasted image 20260102202526.png]]
y ya estamos dentro de diego

![[Pasted image 20260102203405.png]]
dentro del directorio de think vemos que tiene un directorio .ssh que tal vez tenga el id_rsa para poder transformarnos como think

![[Pasted image 20260102203456.png]]
Y efectivamente aqui tenemos el id_rsa

![[Pasted image 20260102203558.png]]
Y ya estamos dentro de think

![[Pasted image 20260102205842.png]]
revisando esta ruta que vi en linpeas parece que hay una regla escrita 
![[Pasted image 20260102210010.png]]
esto es lo que hace esa regla, basicamente es, Si el usuario think intenta convertirse en gege, déjalo pasar inmediatamente sin pedir contraseña

![[Pasted image 20260102205412.png]]
y asi con su - gege pude transformarme en gege

![[Pasted image 20260102210116.png]]
ahora puedo trabajar con este archivo que habia visto antes me lo voy llevar a mi maquina para examinarlo 

![[Pasted image 20260102210333.png]]
asi que voy a intentar con zip2john para ver si puedo descifrar la contrasena 

![[Pasted image 20260102210637.png]]
y aqui john nos acaba de dar la contrasena para el archivo

![[Pasted image 20260102210801.png]]
aqui tenemos el directorio que descomprimi

![[Pasted image 20260102211241.png]]
en este archivo encontramos las credenciales para convertirnos en xavi

![[Pasted image 20260102211325.png]]
ya estamos dentro de xavi 

![[Pasted image 20260102211405.png]]
y estando como xavi me fijo que puedo ejecutar cualquier comando como root

![[Pasted image 20260102211458.png]]
ahora al ejecutar este comando nos transformaremos en root

![[Pasted image 20260102211525.png]]
y ya somos root, PWNED

![[Pasted image 20260102211624.png]]
Aqui esta la flag

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












