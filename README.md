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
2. En el menú superior, ir a:  
   **Archivo > Abrir**
3. Navegar hasta la carpeta donde se extrajo Metasploitable.
4. Seleccionar el archivo:
  Metasploitable.vmx
5. Pulsar en **Abrir**.

VMware cargará la configuración de la máquina virtual y, tras esto, **la máquina quedará lista para usarse**.

En este punto, Metasploitable 2 ya está importada correctamente en VMware y preparada para su configuración y arranque.
