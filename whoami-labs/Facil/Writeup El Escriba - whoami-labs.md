
## Información General

|Campo|Detalle|
|---|---|
|Máquina|El Escriba|
|Plataforma|whoami-labs|
|Sistema Operativo|Linux (Debian 13, "trixie")|
|Dirección IP|172.17.0.2|
|Servicios expuestos|22/tcp (SSH), 80/tcp (HTTP)|
|Dificultad|Fácil|
|Vector de acceso inicial|Credenciales en texto plano (Base64) filtradas en el código fuente HTML|
|Vector de escalada|Capability `cap_dac_override` en `vim.tiny`|

## Resumen del Ataque

La máquina expone un panel de infraestructura ("Infra Dashboard") en el puerto 80 que, a primera vista, solo muestra métricas ficticias (CPU, RAM, sesiones activas). Sin embargo, el código fuente de la página contiene un bloque de JavaScript con un comentario que hace referencia a una auditoría pendiente y una variable `legacy_auth` codificada en Base64. Al decodificarla se obtienen credenciales válidas de un usuario del sistema, lo que permite el acceso por SSH. Una vez dentro, la enumeración de binarios con capabilities revela que `vim.tiny` tiene asignada la capability `cap_dac_override`, lo que permite leer archivos protegidos incluyendo la flag de root sin necesidad de privilegios adicionales ni de un binario SUID clásico.

## Técnicas Usadas

- Escaneo de puertos y detección de servicios/versiones con Nmap
- Análisis de código fuente HTML/JS en busca de comentarios y datos sensibles
- Decodificación Base64 (CyberChef - Magic)
- Reutilización de credenciales para acceso SSH
- Enumeración de binarios SUID (`find -perm -4000`)
- Enumeración de Linux Capabilities (`getcap -r /`)
- Abuso de `cap_dac_override` en `vim.tiny` para lectura arbitraria de archivos

## Desarrollo

### 1. Escaneo inicial de puertos

Se lanza un escaneo rápido de todos los puertos TCP para identificar la superficie de ataque:

```
nmap -p- -sS --min-rate 5000 -n -vvv -Pn -oN ports 172.17.0.2
```

![[IMG-20260723160522314.png]]

Solo dos puertos abiertos: SSH (22) y HTTP (80).

### 2. Enumeración de servicios y versiones

```
nmap -p 22,80 -sC -sV -oN allports 172.17.0.2  
```

![[IMG-20260723160522438.png]]

No se detectan scripts NSE con vulnerabilidades obvias. Se procede a inspeccionar el servicio web manualmente.

### 3. Revisión del panel web

```
http://172.17.0.2/
```

![[IMG-20260723160522623.png]]

El dashboard en sí no aporta nada explotable a nivel visual: son métricas estáticas sin interactividad ni formularios. Se descarta la vía de inyección/parámetros y se revisa el código fuente.

### 4. Análisis del código fuente

Al inspeccionar el HTML se encuentra un bloque `<script>` con un objeto de configuración interno:

![[IMG-20260723160522742.png]]

El comentario "TODO: Move these variables to a secure vault before the Q4 audit" confirma que `legacy_auth` es un dato sensible que nunca se migró a un almacén seguro.

![[IMG-20260723160522845.png]]

### 5. Decodificación de las credenciales

Se toma el valor de `legacy_auth` y se decodifica en CyberChef usando la receta "Magic":

```
Legacy_auth: c3R1ZGVudDpzdHVkZW50MTIz
```

```
https://gchq.github.io/CyberChef/ > magic
```

![[IMG-20260723160522950.png]]

El resultado revela un par de credenciales en formato `usuario:contraseña`:

```
student:student123
```

### 6. Acceso por SSH

Se limpia la entrada previa de `known_hosts` (por reutilización de IP en el laboratorio) y se conecta con las credenciales obtenidas:

```
ssh-keygen -f '/home/kali/.ssh/known_hosts' -R '172.17.0.2'
ssh student@172.17.0.2 
```

```
student@1b46120a5485:~$ whoami
```

![[IMG-20260723160523048.png]]

Acceso inicial confirmado como el usuario `student`.

### 7. Enumeración de usuarios del sistema

```
student@1b46120a5485:~$ grep bash /etc/passwd
```

![[IMG-20260723160523141.png]]

Solo dos usuarios con shell interactiva: `root` y `student`.

### 8. Intento de escalada vía sudo (descartado)

```
student@1b46120a5485:~$ sudo -l
```

![[IMG-20260723160523265.png]]

`sudo` ni siquiera está instalado en el sistema. Se descarta esta vía y se continúa con la búsqueda de binarios SUID.

### 9. Enumeración de binarios SUID (sin hallazgos explotables)

```
student@1b46120a5485:~$ find / -perm -4000 -type f 2>/dev/null
```

![[IMG-20260723160523363.png]]

Todos los binarios SUID encontrados son estándar del sistema y no presentan una vía de escalada evidente (no hay `GTFOBins` aprovechables entre ellos en este contexto). Se descarta esta línea y se pivota hacia Linux Capabilities.

### 10. Enumeración de Linux Capabilities

```
student@1b46120a5485:~$ getcap -r / 2>/dev/null
```

![[IMG-20260723160523474.png]]

`vim.tiny` tiene asignada la capability `cap_dac_override`, que permite saltarse las comprobaciones de permisos de lectura/escritura sobre el sistema de ficheros. Esto habilita la lectura de cualquier archivo, incluidos los pertenecientes a `root`.

### 11. Lectura de la flag de root

```
/usr/bin/vim.tiny /root/root.txt
```

![[IMG-20260723160523585.png]]

### 12. Flag de usuario

```
student@1b46120a5485:~$ cat user.txt 
Capabilities{e1_escriba_passwd}
```

## Flags

```
Capabilities{e1_escriba_passwd}
Capabilities{e1_escriba_v1m}
```

## Lecciones Aprendidas

- Los comentarios de desarrollo dejados en producción (`TODO: Move these variables to a secure vault`) son en sí mismos una fuente de inteligencia: confirman que un dato es sensible y que su exposición es un descuido conocido, no accidental.
- Codificar credenciales en Base64 no es cifrado ni protección: es ofuscación trivialmente reversible, y su presencia en el DOM cliente equivale a exponerlas en texto plano.
- La ausencia de `sudo` o de binarios SUID explotables no significa que el sistema esté bien asegurado: las Linux Capabilities son un vector de escalada igual de válido y a menudo menos vigilado en las auditorías.
- Asignar `cap_dac_override` a un editor de texto (`vim.tiny`) es funcionalmente equivalente a darle SUID root para lectura/escritura de archivos, y debe tratarse con el mismo nivel de criticidad.

## Medidas de Mitigación

- Eliminar cualquier dato de autenticación (aunque esté codificado) del código fuente accesible desde el cliente; mover configuraciones sensibles a variables de entorno o un vault gestionado en el backend.
- Establecer revisiones de código (linting/CI) que detecten y bloqueen el commit de comentarios tipo `TODO`/`FIXME` que referencien credenciales o migraciones de seguridad pendientes.
- Auditar periódicamente las capabilities asignadas a binarios del sistema con `getcap -r /` y retirar `cap_dac_override`, `cap_dac_read_search` u otras capabilities peligrosas de cualquier binario que no las requiera explícitamente para su función.
- Aplicar el principio de mínimo privilegio: si un binario necesita permisos elevados para un caso de uso puntual, preferir wrappers específicos y auditables en lugar de otorgar capabilities amplias de forma permanente.







