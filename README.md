# 🛡️ Enterprise Cybersecurity & Threat Detection Homelab (`lab.local`)

Este repositorio contiene la documentación técnica, diagramas de arquitectura, procedimientos de despliegue y bitácoras de troubleshooting para un laboratorio de ciberseguridad orientado a **Blue Team, Threat Hunting y Detección de Amenazas**.

El entorno simula una red corporativa con **Active Directory Domain Services (AD DS)** integrada con un **SIEM/XDR (Wazuh)** para la ingesta, análisis y correlación de eventos de seguridad en tiempo real.

---

## 📐 Arquitectura de Red y Topología

```text
                     [ Internet ]
                          │
                          ▼
              ┌───────────────────────┐
              │   pfSense (Gateway)   │
              │     192.168.1.1       │
              └───────────┬───────────┘
                          │
         ┌────────────────┴────────────────┐
         │       Red Interna (LAN)         │
         │        192.168.1.0/24           │
         └────┬───────────────┬────────────┘
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
                         │  Win-11-Client   │
                         │ (Estación jdoe)  │
                         │  192.168.1.x     │
                         └──────────────────┘
```
## 🖥️ Matriz de Infraestructura y Nodos

|**Nodo**|**Sistema Operativo**|**Rol / Servicio**|**IP Estática**|**Estado**|
|---|---|---|---|---|
|`pfSense`|FreeBSD / pfSense|Firewall, Gateway, Router LAN|`192.168.1.1`|🟢 Activo|
|`DC01`|Windows Server 2022|Domain Controller (`lab.local`), DNS, AD DS|`192.168.1.10`|🟢 Activo|
|`Win-11-Client`|Windows 11 Enterprise|Estación de Trabajo (`jdoe`)|`DHCP / Static`|🟢 Activo|
|`SIEM-Wazuh`|AlmaLinux / Wazuh OVA|SIEM/XDR Manager, Indexer, Dashboard|`192.168.1.20`|🟢 Activo|

## 🚀 Fases del Proyecto y Hoja de Ruta

- [x] **Fase 1: Segmentación de Red y Gateway**
    
    - Despliegue de `pfSense` en VirtualBox.
        
    - Configuración del direccionamiento IP y aislamiento de la Red Interna.
        
- [x] **Fase 2: Infraestructura de Active Directory**
    
    - Promoción de `DC01` como controlador del dominio `lab.local`.
        
    - Estructuración de Unidades Organizativas (`LAB_Corp`) y creación de cuentas (`jdoe`, `svc_mssql`).
        
    - Unición de `Win-11-Client` al dominio y validación de inicio de sesión.
        
    - Prueba de persistencia local y remediación de hijacking de `sethc.exe`.
        
- [x] **Fase 3: Despliegue de SIEM/XDR (Wazuh)**
    
    - Importación y asignación de recursos a `SIEM-Wazuh` (`4.14.8`).
        
    - Configuración de persistencia de red estática (`192.168.1.20`).
        
    - Despliegue y registro de agentes en `DC01` y `Win-11-Client` mediante `WazuhSvc`.
        
    - Validación de conectividad y estado **Active** en el Dashboard.
        
- [ ] **Fase 4: Telemetría Avanzada (Sysmon) [Siguiente Paso]**
    
    - Instalación de Sysmon64 con plantilla de reglas de _SwiftOnSecurity_.
        
    - Configuración del canal `Microsoft-Windows-Sysmon/Operational` en `ossec.conf`.
        
- [ ] **Fase 5: Simulación Ofensiva y Threat Hunting**
    
    - Simulación de ataque Kerberoasting contra la cuenta de servicio `svc_mssql`.
        
    - Análisis de logs de eventos Event ID 4769 y alertas personalizadas en Wazuh.
        

## 📂 Estructura del Repositorio

```Plaintext
.
├── README.md
└── writeups/
    ├── phase-1-network-pfsense.md
    ├── phase-2-active-directory-setup.md
    └── phase-3-wazuh-siem-deployment.md
```
