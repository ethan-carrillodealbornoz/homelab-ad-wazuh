## 📋 Overview

Este documento detalla la fase ofensiva (_Red Teaming_) ejecutada contra un entorno de laboratorio de Active Directory (`lab.local`). La simulación abarca desde el uso de una cuenta con privilegios estándar hasta la extracción y descifrado offline de tickets Kerberos (Kerberoasting), validación de credenciales mediante SMB, y reconocimiento automatizado de la estructura del dominio.

## 🏗️ Lab Environment Details

- **Domain Controller (DC01):** `192.168.1.10`
    
- **Domain Name:** `lab.local`
    
- **Initial Low-Privilege Account:** `testuser` (Credentiales obtenidas previamente)
    
- **Target Service Account:** `svc_sql` (Cuenta con SPN habilitado)
    
- **Compromised Password:** `Password123!`
    

## 🛠️ Phase 1: Kerberoasting & Offline Hash Cracking

El ataque de Kerberoasting aprovecha el funcionamiento del protocolo Kerberos, permitiendo a cualquier usuario autenticado en el dominio solicitar un ticket TGS (_Ticket Granting Service_) para cualquier cuenta asociada a un SPN (_Service Principal Name_), para posteriormente intentar descifrarlo offline.

### 1. Extracción de Tickets TGS

Utilizando las credenciales del usuario base, se solicitan y exportan los hashes correspondientes a las cuentas de servicio del dominio.

### 2. Descifrado Offline con Hashcat

Una vez obtenido el hash del servicio `svc_sql`, se procedió a realizar un ataque de diccionario utilizando reglas de mutación avanzadas (`best66.rule`) para adivinar contraseñas corporativas complejas que no se encuentran en diccionarios planos.

- **Comando ejecutado:**
    
    
    
    ```Bash
    hashcat -m 13100 -a 0 hash.txt /home/kali/rockyou.txt -r /usr/share/hashcat/rules/best66.rule
    ```
    
- **Resultado obtenido:**
    
    - **Hash Mode:** `13100` (Kerberos 5, etype 23, TGS-REP)
        
    - **Contraseña Descifrada:** `Password123!`
        

## 🔍 Phase 2: Credential Validation & SMB Enumeration

Una vez obtenida la contraseña en texto plano de la cuenta de servicio (`svc_sql`), el siguiente paso consistió en validar su validez y examinar los recursos de red accesibles.

### 1. Comprobación de Acceso y Recursos Compartidos

Se empleó la herramienta `netexec` para verificar la autenticación contra el servicio SMB del Controlador de Dominio.

- **Comando ejecutado:**
    
    
    
    ```Bash
    netexec smb 192.168.1.10 -u svc_sql -p 'Password123!' --shares
    ```
    
- **Hallazgos:** Se confirmó el acceso válido al dominio y se mapearon los recursos estándar (`IPC$`, `NETLOGON`, `SYSVOL`).
    

## ✍️ Phase 3: Post-Exploitation (Proof of Concept)

Para demostrar una consecuencia tangible de la intrusión, se procedió a interactuar con los recursos compartidos del servidor para dejar una evidencia física de la brecha.

### 1. Creación del archivo de evidencia local

En la máquina atacante (Kali Linux), se generó un archivo de texto con el mensaje de prueba:



```Bash
echo "Purple Teaming Simulation - Access Achieved via Kerberoasting" > pwned.txt
```

### 2. Carga del archivo vía SMB

Utilizando el cliente interactivo de Impacket, se estableció conexión con las credenciales comprometidas para subir el archivo de prueba a un recurso compartido con permisos de escritura:



```Bash
impacket-smbclient lab.local/svc_sql:'Password123!'@192.168.1.10
```

Una vez dentro de la sesión interactiva:



```Plaintext
put pwned.txt
```

## 🗺️ Phase 4: Active Directory Reconnaissance (BloodHound)

Con un acceso autenticado dentro del dominio, se recolectó la topología completa de Active Directory para evaluar relaciones de confianza, control de acceso (ACLs) y posibles rutas de escalada de privilegios.

### 1. Configuración de Resolución DNS

Para que las herramientas de AD resuelvan correctamente los registros del dominio, se configuró el servidor DNS de la máquina atacante para apuntar directamente al Controlador de Dominio (`192.168.1.10`) en `/etc/resolv.conf`.

### 2. Recolección de Datos Automatizada

Se ejecutó `bloodhound-python` especificando el FQDN del Controlador de Dominio y el servidor DNS dedicado:

- **Comando ejecutado:**
    
    
    
    ```Bash
    bloodhound-python -u svc_sql -p 'Password123!' -d lab.local -dc DC01.lab.local -ns 192.168.1.10 -c All
    ```
    
- **Resultado:** Generación exitosa de los archivos JSON (`_users.json`, `_groups.json`, `_computers.json`, etc.) que contienen el mapa estructural del dominio para su posterior análisis.