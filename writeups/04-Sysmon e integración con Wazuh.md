
# 🕵️ Fase 4: Telemetría Avanzada Endpoint con Sysmon e Integración en Wazuh

## 📌 1. Objetivos y Ficha Técnica

El objetivo principal de esta fase es incrementar drásticamente la visibilidad del **Blue Team** en los endpoints mediante el despliegue de **Microsoft Sysmon (System Monitor)**. Sysmon permite capturar eventos detallados que el visor de eventos estándar de Windows omite, tales como la línea de comandos completa, los procesos padre, la creación de archivos y modificaciones en el registro.

| Componente | Detalle |
| :--- | :--- |
| **Herramienta Endpoint** | Microsoft Sysinternals Sysmon64 (v15.22) |
| **Plantilla de Reglas** | SwiftOnSecurity Config (`sysmonconfig-export.xml`) |
| **Canal de Registros** | `Microsoft-Windows-Sysmon/Operational` |
| **Endpoints Involucrados** | `DC01` (`192.168.1.10`) y `Win-11-Client` (`192.168.1.x`) |
| **Colector de Eventos** | Agente de Wazuh (`WazuhSvc`) |

---

## 🛠️ 2. Resolución de Incidentes Pre-Instalación (Troubleshooting DNS en DC01)

### 🚨 Incidente: Fallo de resolución DNS (`RCODE_SERVER_FAILURE`)

Al intentar descargar Sysmon64 desde `DC01`, el comando `Invoke-WebRequest` falló con el error:
> `Invoke-WebRequest : No se puede resolver el nombre remoto: 'live.sysinternals.com'`

#### Causa Raíz
`DC01` actuaba como servidor DNS primario del dominio `lab.local`, pero no tenía configurados reenviadores DNS (*Forwarders*) válidos hacia Internet o el servicio de resolución de `pfSense` (`192.168.1.1`) no estaba reenviando las consultas externas de forma efectiva.

#### Remediación Aplicada
Se configuraron reenviadores públicos redundantes (`1.1.1.1` y `8.8.8.8`) directamente en el servicio DNS de Active Directory en `DC01` ejecutando en PowerShell (Administrador):

```powershell
# Configurar reenviadores DNS públicos en DC01
Set-DnsServerForwarder -IPAddress "1.1.1.1", "8.8.8.8" -PassThru

# Limpiar caché de resolución local
Clear-DnsClientCache

# Verificación de resolución de nombres hacia Internet
Resolve-DnsName live.sysinternals.com
```

---

## 🚀 3. Despliegue de Sysmon64 y Configuración SwiftOnSecurity

Se ejecutó el siguiente procedimiento automatizado en **`DC01`** y **`Win-11-Client`** desde una consola de **PowerShell como Administrador**:

```powershell
# 1. Crear directorio de trabajo estandarizado
New-Item -ItemType Directory -Force -Path "C:\Tools\Sysmon"
Set-Location "C:\Tools\Sysmon"

# 2. Descargar ejecutable Sysmon64 oficial
Invoke-WebRequest -Uri "https://live.sysinternals.com/Sysmon64.exe" -OutFile "Sysmon64.exe"

# 3. Descargar reglas optimizadas de SwiftOnSecurity
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "sysmonconfig.xml"

# 4. Instalar el driver y servicio de Sysmon
.\Sysmon64.exe -i sysmonconfig.xml -accepteula
```

---

## ⚙️ 4. Ingesta del Canal de Sysmon en el Agente de Wazuh

Para enviar la telemetría recolectada por Sysmon hacia el Manager de Wazuh (`192.168.1.20`), se modificó el archivo `ossec.conf` en cada endpoint.

### Script de Inserción XML en `ossec.conf`:

```powershell
$OssecConf = "C:\Program Files (x86)\ossec-agent\ossec.conf"

$SysmonConfig = @"
  <!-- Ingesta de Telemetria Avanzada Sysmon -->
  <localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>
</ossec_config>
"@

$content = Get-Content $OssecConf -Raw
if ($content -notlike "*Microsoft-Windows-Sysmon/Operational*") {
    $content = $content -replace '</ossec_config>', $SysmonConfig
    Set-Content -Path $OssecConf -Value $content -Encoding UTF8
    Write-Host "Configuración de Sysmon insertada exitosamente." -ForegroundColor Green
} else {
    Write-Host "La configuración de Sysmon ya estaba presente." -ForegroundColor Yellow
}

# Reiniciar el agente para aplicar cambios
Restart-Service WazuhSvc
```

---

## 📊 5. Estructuración del Dashboard e Interpretación de Alertas

En la sección **Discover** del Dashboard de Wazuh, se filtró la vista con la consulta:

```text
data.win.system.providerName : "Microsoft-Windows-Sysmon"
```

### Configuración de Vista Estilo SOC

Se añadieron las siguientes columnas clave desde el panel *Available fields* para facilitar la auditoría y análisis rápido:

1. `agent.name`: Identificador de la máquina origen (`DC01` / `Win-11-Client`).
2. `data.win.system.eventID`: Código numérico del tipo de evento.
3. `data.win.eventdata.image`: Ruta completa del ejecutable lanzado.
4. `data.win.eventdata.commandLine`: Comando exacto y argumentos pasados al ejecutable.

---

### 📖 Guía Rápida de Event IDs Importantes en Threat Hunting

| Event ID | Nombre del Evento | Contexto de Análisis |
| :---: | :--- | :--- |
| **`1`** | **Process Creation** | Registra cada programa abierto con su línea de comandos y ejecutable padre. Permite detectar uso no autorizado de `net.exe`, `whoami.exe` o `powershell.exe`. |
| **`3`** | **Network Connection** | Registra conexiones TCP/UDP establecidas por un binario. Permite identificar conexiones a Command & Control (C2). |
| **`7`** | **Image Loaded** | Registra librerías DLL cargadas por procesos. Útil para detectar ataques de *DLL Hijacking*. |
| **`11`** | **File Create** | Registra cuándo y qué programa creó un archivo en disco (ej. scripts o artefactos maliciosos). |
| **`12 / 13`** | **Registry Event** | Registra modificaciones de claves de Registro (ej. claves de persistencia *Run* o *RunOnce*). |

---

## ✅ 6. Estado Actual del Laboratorio

- [x] Agentes Wazuh activos en `DC01` y `Win-11-Client`.
- [x] Sysmon v15.22 operativo con la regla de configuración de *SwiftOnSecurity*.
- [x] Ingesta activa y validada en tiempo real en el Dashboard de Wazuh.
- [ ] **Próximo Paso (Fase 5):** Ejecución del ataque Kerberoasting contra la cuenta de servicio `svc_mssql` y creación/validación de reglas de detección para Event ID 4769.