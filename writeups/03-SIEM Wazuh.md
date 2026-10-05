# 🛡️ Fase 3: Despliegue de SIEM/XDR Wazuh e Integración de Endpoints

## 📌 1. Ficha Técnica del Servidor SIEM

| Parámetro / Componente | Configuración Asignada | Descripción |
| :--- | :--- | :--- |
| **Nombre de la VM** | `SIEM-Wazuh` | Servidor central de monitorización |
| **Imagen / Sistema** | `wazuh-4.14.8.ova` | All-in-One (Manager, Indexer, Dashboard) |
| **Dirección IP Estática** | `192.168.1.20/24` | Interfaz `eth0` |
| **Puerta de Enlace** | `192.168.1.1` | pfSense |
| **Servidor DNS** | `192.168.1.10` | DC01 (`lab.local`) |
| **Puertos de Agentes** | `1514/TCP-UDP`, `1515/TCP` | Conexión y registro de agentes |
| **Acceso Web Dashboard** | `https://192.168.1.20` | Credenciales iniciales: `admin` / `admin` |

---

## 🛠️ 2. Importación y Configuración de Red en `SIEM-Wazuh`

1. **Importación de la Appliance:**
   - Archivo: `wazuh-4.14.8.ova` importado en VirtualBox.
   - Recursos: 4 GB RAM, 2 vCPUs.
   - Adaptador de Red: Conectado a la **Red Interna** asignada a la LAN de `pfSense`.

2. **Asignación de IP Estática y Persistencia de Red:**
   Dado que la imagen mínima de Wazuh no incluye `nmtui` ni `nmcli`, se aplicó la IP de forma inmediata en la interfaz `eth0` y se escribió la configuración permanente en el script de red:

```bash
   # Asignación inmediata en memoria
   sudo ip addr add 192.168.1.20/24 dev eth0
   sudo ip route add default via 192.168.1.1

   # Configuración de red permanente
   sudo bash -c 'cat <<EOF> /etc/sysconfig/network-scripts/ifcfg-eth0
   DEVICE=eth0
   TYPE=Ethernet
   ONBOOT=yes
   BOOTPROTO=none
   IPADDR=192.168.1.20
   NETMASK=255.255.255.0
   GATEWAY=192.168.1.1
   DNS1=192.168.1.10
   EOF'
```

3. **Verificación de Conectividad:**
    
    - Ejecución de `ip a` confirmando la dirección `inet 192.168.1.20/24` en `eth0`.
        

## 💻 3. Enrolamiento de Agentes en Endpoints Windows

El despliegue de los agentes de Wazuh se realizó mediante **PowerShell (Administrador)** en cada máquina del dominio `lab.local`.

### 3.1. Enrolamiento en Controlador de Dominio (`DC01`)

PowerShell

```
# 1. Descargar el instalador oficial MSI
Invoke-WebRequest -Uri "[https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.8-1.msi](https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.8-1.msi)" -OutFile "$env:tmp\wazuh-agent.msi"

# 2. Instalar y vincular con el servidor Wazuh (192.168.1.20)
Start-Process msiexec.exe -ArgumentList '/i', "$env:tmp\wazuh-agent.msi", '/q', 'WAZUH_MANAGER="192.168.1.20"', 'WAZUH_REGISTRATION_SERVER="192.168.1.20"', 'WAZUH_AGENT_NAME="DC01"' -Wait

# 3. Iniciar el servicio del agente
NET START WazuhSvc
```

### 3.2. Enrolamiento en Estación de Trabajo (`Win-11-Client`)

PowerShell

```
# 1. Descargar el instalador oficial MSI
Invoke-WebRequest -Uri "[https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.8-1.msi](https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.8-1.msi)" -OutFile "$env:tmp\wazuh-agent.msi"

# 2. Instalar y vincular con el servidor Wazuh (192.168.1.20)
Start-Process msiexec.exe -ArgumentList '/i', "$env:tmp\wazuh-agent.msi", '/q', 'WAZUH_MANAGER="192.168.1.20"', 'WAZUH_REGISTRATION_SERVER="192.168.1.20"', 'WAZUH_AGENT_NAME="Win-11-Client"' -Wait

# 3. Iniciar el servicio del agente
NET START WazuhSvc
```

## 🚨 4. Resolución de Incidentes y Troubleshooting

- **Error: `nmtui: command not found` / `nmcli: command not found`**
    
    - _Causa:_ La distribución minimalista basada en la OVA de Wazuh no incluye la suite NetworkManager por defecto.
        
    - _Solución:_ Asignación directa mediante la suite `iproute2` (`ip addr`) y creación manual del fichero `/etc/sysconfig/network-scripts/ifcfg-eth0`.
        
- **Error: `[Permission Denied]` al editar con `vi` en consola**
    
    - _Causa:_ Error de sintaxis en la ruta del archivo (`/etc/sysconfig/network/scripts/ifcfg-eth0` en lugar de `network-scripts`).
        
    - _Solución:_ Generación automatizada del fichero mediante un bloque `cat <<EOF` ejecutado bajo `sudo bash -c`.
        
- **Error: `El nombre de servicio no es válido` al ejecutar `NET START Wazuh`**
    
    - _Causa 1:_ Faltaba la extensión `.msi` en el parámetro `-OutFile` al descargar el agente, provocando la falla silenciosa del instalador.
        
    - _Causa 2:_ Ejecución asíncrona de `msiexec.exe` intentando iniciar el servicio antes de que finalizara la instalación.
        
    - _Causa 3:_ El nombre del servicio registrado en Windows es **`WazuhSvc`** y no `Wazuh`.
        
    - _Solución:_ Inclusión del parámetro `-Wait` en `Start-Process`, especificación de la extensión `.msi` y uso del nombre de servicio correcto (`NET START WazuhSvc`).
        
- **Error de instalación bloqueada en `Win-11-Client`**
    
    - _Causa:_ La sesión activa en el cliente pertenecía al usuario estándar `jdoe`, por lo que PowerShell carecía de privilegios de elevación para registrar servicios del sistema.
        
    - _Solución:_ Apertura explícita de **PowerShell como Administrador** autenticando con credenciales elevadas del dominio (`LAB\Administrador`).
        

## ✅ 5. Verificación Final e Integración

1. **Acceso al Dashboard Web:**
    
    - Navegación desde `Win-11-Client` hacia `https://192.168.1.20`.
        
    - Autenticación exitosa con usuario `admin`.
        
2. **Estado de los Agentes:**
    
    - Confirmada la conexión en la pestaña **Wazuh > Agents**.
        
    - **`DC01`** (`192.168.1.10`) 🟢 **Active**
        
    - **`Win-11-Client`** (`192.168.1.x`) 🟢 **Active**