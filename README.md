# 🛡️ Cybersecurity Home Lab — Fase 1: Perímetro y Segmentación de Red

## 📌 Descripción General
Despliegue e instalación de un cortafuegos/enrutador **pfSense CE 2.9.0** virtualizado en **Oracle VM VirtualBox** sobre un equipo anfitrión con procesador AMD Ryzen 7 9700X y 32 GB de RAM.

El objetivo principal de esta fase es aislar completamente el tráfico del laboratorio de ciberseguridad de la red doméstica física, garantizando un entorno seguro para pruebas de Red Team y Blue Team.

---

## 🌐 Topología y Segmentación de Red

| Interfaz | Adaptador VirtualBox | Interfaz SO | IP / Máscara | Función |
| :--- | :--- | :--- | :--- | :--- |
| **WAN** | Adaptador Puente (*Bridge*) | `em0` | `192.168.0.177/24` (DHCP) | Conexión externa a Internet |
| **LAN** | Red Interna (`lab-aislado`) | `em1` | `192.168.1.1/24` (Estática) | Gateway de la red aislada de víctimas/atacantes |

---

## ⚙️ Pasos de Despliegue Realizados

1. **Configuración del Hipervisor:**
   - Creación de la máquina virtual `pfSense-Router` con 1 GB de RAM y 1 vCPU.
   - Configuración de doble tarjeta de red:
     - **Adaptador 1:** Puente (*Realtek PCIe 2.5GbE Controller*).
     - **Adaptador 2:** Red Interna (`lab-aislado`) con Modo Promiscuo configurado en *Permitir Todo*.

2. **Instalación del Sistema Operativo:**
   - Arranque mediante ISO oficial de pfSense CE v2.9.0.
   - Instalación y particionado en el disco virtual `ada0`.
   - Desmontaje del medio óptico virtual (.ISO) para evitar bucles de arranque.

3. **Verificación de la Consola de Administración:**
   - Asignación exitosa de direcciones IP y verificación de la consola interactiva de FreeBSD.

```text
FreeBSD/amd64 (pfSense.home.arpa) (ttyv0)
Netgate Device ID: 5d6d59ffc31c76ca64f9

*** Welcome to pfSense 2.9.0-RELEASE (amd64) on pfSense ***

  WAN (wan) -> em0 -> v4/DHCP4: 192.168.0.177/24
  LAN (lan) -> em1 -> v4:       192.168.1.1/24
