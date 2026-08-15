# CTF Writeups

![Writeups](https://img.shields.io/badge/writeups-11-blue)
![Idioma](https://img.shields.io/badge/idioma-español-red)
![Estado](https://img.shields.io/badge/estado-activo-brightgreen)
![Licencia](https://img.shields.io/badge/licencia-MIT-green)

Writeups técnicos de máquinas y retos CTF resueltos en distintas plataformas, documentados paso a paso: reconocimiento, explotación, escalada de privilegios y lecciones aprendidas. Cada writeup refleja el proceso real, incluyendo caminos fallidos y descartados, no solo la ruta exitosa final.

**Metodología:** ejecuto, verifico y documento cada paso con salida de comandos real antes de avanzar al siguiente.

---

## Índice de Writeups

| Máquina | Plataforma | Dificultad | Técnicas clave |
|---|---|---|---|
| [Msfvenom](./whoami-labs/msfvenom/) | whoami-labs | - | Java RMI (CVE-2011-3556) |
| [PostgreSQL](./whoami-labs/postgresql/) | whoami-labs | - | Credenciales débiles, `COPY FROM PROGRAM` (CVE-2019-9193) |
| [El Escriba](./whoami-labs/el-escriba/) | whoami-labs | - | Credenciales Base64 filtradas, capability `cap_dac_override` |
| [El Archivero](./whoami-labs/el-archivero/) | whoami-labs | - | Directory listing, capability `cap_dac_read_search` en `tar` |
| [El Heredero](./whoami-labs/el-heredero/) | whoami-labs | - | Clave SSH filtrada, capability `cap_chown` |
| [Jinteki](./thepwnlab/jinteki/) | ThePwnLab | - | FTP anónimo, cifrado AES-256-CBC, `sudo ALL:ALL` |
| [El Baifo](./thehackerslabs/el-baifo/) | TheHackersLabs | - | Redis sin auth, MQTT leak, cron script editable por grupo |
| [Bocata de Calamares](./thehackerslabs/bocata-de-calamares/) | TheHackersLabs | - | SQLi login bypass, LFI, `sqlmap`, GTFOBins (`find`) |
| [Fruits](./thehackerslabs/fruits/) | TheHackersLabs | - | LFI vía fuzzing (`ffuf`), bruteforce SSH, GTFOBins (`find`) |
| [Casa Paco](./thehackerslabs/casa-paco/) | TheHackersLabs | - | Command injection, cron world-writable, SUID bash |
| [Goiko](./thehackerslabs/goiko/) | TheHackersLabs | - | Null session SMB, cracking offline (`john`/`hashcat`), PATH hijacking |

---

## Plataformas

Writeups organizados por plataforma de origen:

- **[whoami-labs](./whoami-labs/)**
- **[TheHackersLabs](./thehackerslabs/)**
- **[ThePwnLab](./thepwnlab/)**
- **[HackTheBox](./hackthebox/)**
- **[BBLabs](./bblabs/)**

## Estructura de cada writeup

Cada máquina sigue una plantilla fija:

1. **Información General** — nombre, plataforma, dificultad, IP, fecha
2. **Resumen del Ataque**
3. **Técnicas Usadas**
4. **Desarrollo** — pasos numerados con salida real de comandos
5. **Lecciones Aprendidas**
6. **Medidas de Mitigación**

## Sobre el autor

Writeups mantenidos por [@elc0ket](https://github.com/elc0ket), practicante de ciberseguridad ofensiva enfocado en pentesting. Ver también [ethical-hacking-notes](https://github.com/elc0ket/ethical-hacking-notes) para apuntes y cheatsheets por módulo.

Perfil verificado: [cyber-profile.com/u/jmsoriano](https://cyber-profile.com/u/jmsoriano)

## Licencia

Contenido con fines educativos. Uso libre bajo licencia [MIT](./LICENSE), salvo que se indique lo contrario en un writeup concreto.
