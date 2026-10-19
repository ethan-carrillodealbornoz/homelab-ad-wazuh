# 🛡️ Enterprise Cybersecurity & Threat Detection Homelab (`lab.local`)

Este repositorio contiene la documentación técnica, diagramas de arquitectura, procedimientos de despliegue y bitácoras de troubleshooting para un laboratorio de ciberseguridad orientado a **Blue Team, Threat Hunting y Detección de Amenazas**.

El entorno simula una red corporativa con **Active Directory Domain Services (AD DS)** integrada con un **SIEM/XDR (Wazuh)** para la ingesta, análisis y correlación de eventos de seguridad en tiempo real.

## 📐 Arquitectura de Red y Topología

```
                     [ Internet ]
                          │
                          ▼
              ┌───────────────────────┐
              │   pfSense (Gateway)   │
              │      192.168.1.1      │
              └───────────┬───────────┘
                          │
          ┌───────────────┴────────────────┐
          │        Red Interna (LAN)       │
          │         192.168.1.0/24         │
          └────┬───────────────┬───────────┘
               │               │
               ▼               ▼
      ┌────────────────┐  ┌──────────────────┐
      │  SIEM-Wazuh    │  │       DC01       │
      │  (Wazuh 4.14.8)│  │ (Windows Server) │
      │  192.168.1.20  │  │   192.168.1.10   │
      └────────────────┘  └────────┬─────────┘
                                   │
                                   ▼
                          ┌──────────────────┐
                          │   Win-11-Client  │
                          │ (Estación jdoe)  │
                          │   192.168.1.x    │
                          └──────────────────┘
```

## 🖥️ Matriz de Infraestructura y Nodos

|   |   |   |   |   |
|---|---|---|---|---|
|**Nodo**|**Sistema Operativo**|**Rol / Servicio**|**IP Estática**|**Estado**|
|`pfSense`|FreeBSD / pfSense|Firewall, Gateway, Router LAN|`192.168.1.1`|🟢 Activo|
|`DC01`|Windows Server 2022|Domain Controller (`lab.local`), DNS, AD DS|`192.168.1.10`|🟢 Activo|
|`Win-11-Client`|Windows 11 Enterprise|Estación de Trabajo (`jdoe`)|`DHCP / Static`|🟢 Activo|
|`SIEM-Wazuh`|AlmaLinux / Wazuh OVA|SIEM/XDR Manager, Indexer, Dashboard|`192.168.1.20`|🟢 Activo|
|`Kali-Linux`|Kali Linux|Equipo de Auditoría / Simulación Ofensiva|`192.168.1.x`|🟢 Activo|

## 🚀 Fases del Proyecto y Hoja de Ruta

- [x] **Fase 1: Segmentación de Red y Gateway**
    
    - Despliegue de `pfSense` en VirtualBox.
        
    - Configuración del direccionamiento IP y aislamiento de la Red Interna.
        
- [x] **Fase 2: Infraestructura de Active Directory**
    
    - Promoción de `DC01` como controlador del dominio `lab.local`.
        
    - Estructuración de Unidades Organizativas (`LAB_Corp`) y creación de cuentas (`jdoe`, `svc_mssql`).
        
    - Unión de `Win-11-Client` al dominio y validación de inicio de sesión.
        
    - Prueba de persistencia local y remediación de hijacking de `sethc.exe`.
        
- [x] **Fase 3: Despliegue de SIEM/XDR (Wazuh)**
    
    - Importación y asignación de recursos a `SIEM-Wazuh` (`4.14.8`).
        
    - Configuración de persistencia de red estática (`192.168.1.20`).
        
    - Despliegue y registro de agentes en `DC01` y `Win-11-Client` mediante `WazuhSvc`.
        
    - Validación de conectividad y estado **Active** en el Dashboard.
        
- [x] **Fase 4: Simulación Ofensiva
    
    - Configuración de cuentas vulnerables con SPN (`svc_mssql`) en Active Directory.
        
    - Extracción de tickets TGS cifrados con RC4 (`0x17`) desde Kali Linux usando Impacket (`GetUserSPNs`).
          
- [x] **Fase 5: Detección de Kerberoasting (Event ID 4769)**
    
    - Apertura de canales de telemetría de eventos de seguridad en `ossec.conf`.
        
    - Creación y validación de regla personalizada `100301` (`level 12`) con sintaxis `pcre2`.
        
    - Troubleshooting completo: desinhibición de filtros en agente, sincronización de reloj Kerberos (`Clock skew`) y reinicio efectivo del servicio `WazuhSvc`.
    
- [ ] **Fase 6: Telemetría Avanzada (Sysmon) y Ampliación de Threat Hunting [Siguiente Paso]**
    
    - Instalación de Sysmon64 con plantilla de reglas de _SwiftOnSecurity_.
        
    - Configuración del canal `Microsoft-Windows-Sysmon/Operational` en `ossec.conf`.
        
    - Detección de técnicas adicionales de MITRE ATT&CK (AS-REP Roasting, Pass-the-Hash, Process Injection).
        

## 📂 Estructura del Repositorio

```
.
├── README.md
└── writeups/
    ├── 01-pfsense-deployment.md
    ├── 02-active-directory.md
    ├── 03-SIEM Wazuh.md
    ├── 04-Sysmon e integración con Wazuh.md
    ├── 05-AD Kerberoasting y Post-Explotación.md
    └── 06-Detección de Kerberoasting
```