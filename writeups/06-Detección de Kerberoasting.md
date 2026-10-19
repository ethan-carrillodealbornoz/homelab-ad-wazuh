# Hito de Detección: Identificación de Ataques de Kerberoasting mediante Reglas Custom en Wazuh SIEM

## 1. Resumen Ejecutivo

En esta fase del proyecto de laboratorio de Active Directory y SIEM, se ha logrado la detección activa en tiempo real de ataques de **Kerberoasting** (solicitud no autorizada de tickets Kerberos TGS encryptados en RC4 para cuentas de servicio). Mediante el uso de la herramienta `impacket-GetUserSPNs` desde un nodo atacante Kali Linux, se desencadenaron los eventos correspondientes en el Controlador de Dominio (`DC01`), los cuales fueron procesados, parseados y alertados con nivel crítico por el servidor central de **Wazuh SIEM**.

## 2. Arquitectura del Laboratorio

| **Componente**    | **Rol / Función**                  | **IP / Nombre**                  |
| ----------------- | ---------------------------------- | -------------------------------- |
| **DC01**          | Domain Controller (Windows Server) | `192.168.1.10` / `lab.local`     |
| **Wazuh Manager** | Servidor SIEM y Analizador de Logs | `192.168.1.102` / `wazuh-server` |
| **Kali Linux**    | Estación Atacante (Impacket)       | Variable en la red local         |

## 3. Configuración Técnico-Analítica

### 3.1. Ajustes en el Agente de Windows (`DC01`)

Para permitir la transmisión completa de eventos de auditoría de seguridad Kerberos sin omisiones ni filtros restrictivos, se actualizó la configuración del archivo `ossec.conf` (`C:\Program Files (x86)\ossec-agent\ossec.conf`):

```HTML
<localfile>
  <location>Security</location>
  <log_format>eventchannel</log_format>
</localfile>
```

> **Nota de reinicio**: Para aplicar los cambios de configuración en Windows, el servicio debe ser reiniciado invocando el nombre exacto de la instancia del servicio:

```Powershell
Restart-Service WazuhSvc
```

### 3.2. Regla Personalizada en Wazuh Manager

Se añadió la siguiente regla de detección personalizada en `/var/ossec/etc/rules/local_rules.xml` en el servidor central de Wazuh:

```HTML
<group name="windows, security, kerberoasting">
  <rule id="100301" level="12">
    <if_sid>60103</if_sid>
    <field name="win.system.eventID">^4769$</field>
    <field name="win.eventdata.TicketOptions" type="pcre2">^0x408100..$</field>
    <field name="win.eventdata.TicketEncryptionType" type="pcre2">^0x17$</field>
    <options>no_full_log</options>
    <description>Posible ataque de Kerberoasting detectado: Solicitud de ticket TGS con cifrado RC4 (0x17).</description>
    <group>authentication_failures, attacks, credential_access</group>
  </rule>
</group>
```

#### Parámetros Técnicos de la Regla:

- **Rule ID**: `100301` (Nivel 12 - Severidad Alta/Crítica).
    
- **`if_sid` (60103)**: Hereda de la regla base de eventos de seguridad de Windows.
    
- **`EventID 4769`**: Solicitud de Ticket de Servicio Kerberos (TGS).
    
- **`TicketEncryptionType 0x17`**: Filtra el uso de cifrado débil RC4-HMAC, estándar en ataques de Kerberoasting.
    
- **`TicketOptions ^0x408100..$`**: Evalúa las opciones de solicitud de ticket típicas generadas por herramientas de extracción masiva como Impacket.
    

## 4. Ejecución del Ataque (Simulación)

Desde la máquina **Kali Linux**, se lanzó la solicitud de tickets SPN apuntando al Controlador de Dominio:

```Powershell
impacket-GetUserSPNs lab.local/testuser:'U2QcZ2mt*9MP' -dc-ip 192.168.1.10 -request
```

**Resultado de la ejecución**: Extracción exitosa de los hashes Kerberos para cuentas de servicio (`svc_mssql`, `svc_sql`).

## 5. Guía de Troubleshooting y Resolución de Errores

Durante el desarrollo e implementación del escenario se identificaron y resolvieron los siguientes inconvenientes técnicos:

### Inconveniente 1: Filtro de Logs Restrictivo en el Agente de Windows

- **Síntoma**: Los eventos `4769` se registraban en el Visor de Eventos de Windows, pero no aparecían en el Dashboard de Wazuh.
    
- **Causa raíz**: El archivo `ossec.conf` contenía una etiqueta `<query>` con cláusulas `EventID != ...` que bloqueaban la transmisión del canal de Seguridad.
    
- **Solución**: Se eliminó el bloque `<query>` dentro del tag `<localfile>` para transmitir el canal `Security` completo.
    

### Inconveniente 2: Error de Desincronización Horaria (`KRB_AP_ERR_SKEW`)

- **Síntoma**: Al ejecutar Impacket se recibía el error: `Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)`.
    
- **Causa raíz**: Desfase de más de 5 minutos entre el reloj del sistema Kali Linux y el Controlador de Dominio.
    
- **Solución**: Sincronización del reloj mediante comandos de red:
    
    ```Bash
    sudo rdate -n 192.168.1.10
    # O alternativa NTP:
    sudo timedatectl set-ntp true
    ```
    

### Inconveniente 3: Nombre Incorrecto del Servicio al Reiniciar el Agente en PowerShell

- **Síntoma**: Ejecutar `Restart-Service Wazuh` no actualizaba la configuración en tiempo real, manteniendo el agente bloqueado o desfasado.
    
- **Causa raíz**: En Windows Server, el nombre interno del servicio de Wazuh es `WazuhSvc`.
    
- **Solución**: Invocación explícita del servicio correcto:
    
    ```PowerShell
    Restart-Service WazuhSvc
    ```
    

## 6. Verificación de Resultados

Tras la aplicación de las soluciones de troubleshooting, la reejecución de la prueba confirmó la captura inmediata en el Dashboard de Wazuh:

- **Alerta Generada**: `100301`
    
- **Nivel**: `12`
    
- **Descripción**: _Posible ataque de Kerberoasting detectado: Solicitud de ticket TGS con cifrado RC4 (0x17)._
    
- **Agente de Origen**: `DC01`
    
