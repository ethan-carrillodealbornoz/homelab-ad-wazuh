
# 🏰 Active Directory Domain Services — Guía Completa de Despliegue e Integración

## 📌 1. Ficha Técnica e Infraestructura del Dominio

| Parámetro / Función | Dirección IP / Nombre | Descripción |
| :--- | :--- | :--- |
| **Dominio Principal** | `lab.local` / `LAB` | Bosque raíz del laboratorio |
| **Domain Controller (DC01)** | `192.168.1.10/24` | Windows Server 2022 (AD DS, DNS, Kerberos) |
| **Puerta de Enlace (Gateway)** | `192.168.1.1` | pfSense |
| **DNS Primario** | `192.168.1.10` | Redirigido a `DC01` |
| **Cliente Workstation** | `Win-11-Client` | Windows 11 Pro unido al dominio |
| **Estructura de OUs** | `LAB_Corp` | Sub-OUs: `Users`, `Admins`, `ServiceAccounts` |
| **Usuarios Creados** | `jdoe`, `svc_mssql` | Usuario estándar y cuenta de servicio con SPN |

---

## 🛠️ 2. Configuración Inicial y Promoción de `DC01`

1. **Direccionamiento IP Estático en `DC01` (`ncpa.cpl`):**
   - Dirección IP: `192.168.1.10`
   - Máscara de subred: `255.255.255.0`
   - Puerta de enlace predeterminada: `192.168.1.1`
   - Servidor DNS preferido: `127.0.0.1` (o `192.168.1.10`)

2. **Instalación del Rol AD DS y Promoción:**
   - En el **Administrador del Servidor**, agregar el rol **Servicios de dominio de Active Directory (AD DS)**.
   - Promocionar el servidor a controlador de dominio seleccionando **Agregar un nuevo bosque**.
   - Nombre del dominio raíz: `lab.local`.
   - Reiniciar el servidor al finalizar la promoción.

---

## 🔓 3. Procedimiento de Recuperación de Acceso (`sethc.exe Hijacking`)

Ante bloqueos de inicio de sesión por discrepancias en el mapa de teclado de VirtualBox o contraseñas olvidadas durante el primer arranque, se aplicó una técnica de escalada física mediante Windows Recovery Environment (WinRE):

1. Arrancar `DC01` desde la ISO de Windows Server y seleccionar **Reparar el equipo** > **Solucionar problemas** > **Símbolo del sistema**.
2. Identificar la unidad del sistema operativo y reemplazar la herramienta de accesibilidad Teclas pegajosas (*Sticky Keys*):
```cmd
   cd /d D:\Windows\System32
   ren sethc.exe sethc.exe.bak
   copy cmd.exe sethc.exe
```

3. Reiniciar la VM normalmente (`wpeutil reboot`).
    
4. En la pantalla de inicio de sesión de Windows, presionar **MAYÚS 5 veces** para abrir una ventana de `cmd.exe` con privilegios `NT AUTHORITY\SYSTEM`.
    
5. Restablecer la contraseña del administrador del dominio:
    
    
    
    ```DOS
    net user Administrador <NuevaContraseña123!> /domain
    ```
    

## 💻 4. Configuración y Unión de `Win-11-Client` al Dominio

1. **Alineación de Red e IP/DNS:**
    
    - En `Win-11-Client`, abrir `ncpa.cpl`.
        
    - Establecer como **Servidor DNS preferido**: `192.168.1.10` (`DC01`).
        
    - Comprobar conectividad y resolución con:
        
        DOS
        
        ```Bash
        ping lab.local
        ```
        
2. **Unión al Dominio:**
    
    - Presionar `Win + R`, ejecutar `sysdm.cpl`.
        
    - En la pestaña _Nombre de equipo_, hacer clic en **Cambiar...**.
        
    - Seleccionar **Dominio** e introducir `lab.local`.
        
    - Autenticar con credenciales de administrador: `LAB\Administrador`.
        
    - Reiniciar el equipo para aplicar los cambios.
        

## 📜 5. Población y Automatización de Active Directory (PowerShell)

Para automatizar la creación de la estructura corporativa de Unidades Organizativas (OUs), usuarios estándar y cuentas de servicio vulnerables (_Kerberoasting_), se ejecuta el siguiente script en **`DC01`**:

```PowerShell
# 1. Cargar el módulo de Active Directory
Import-Module ActiveDirectory

# 2. Crear la estructura de Unidades Organizativas (OUs)
New-ADOrganizationalUnit -Name "LAB_Corp" -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Users" -Path "OU=LAB_Corp,DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Admins" -Path "OU=LAB_Corp,DC=lab,DC=local"
New-ADOrganizationalUnit -Name "ServiceAccounts" -Path "OU=LAB_Corp,DC=lab,DC=local"

# 3. Crear un usuario estándar (John Doe)
$passwordUsuario = ConvertTo-SecureString "UserPass123!" -AsPlainText -Force
New-ADUser -Name "John Doe" -GivenName "John" -Surname "Doe" `
    -SamAccountName "jdoe" -UserPrincipalName "jdoe@lab.local" `
    -Path "OU=Users,OU=LAB_Corp,DC=lab,DC=local" `
    -AccountPassword $passwordUsuario -Enabled $true -ChangePasswordAtLogon $false

# 4. Crear una Cuenta de Servicio vulnerable a Kerberoasting
$passwordSvc = ConvertTo-SecureString "ServicePass123!" -AsPlainText -Force
New-ADUser -Name "MSSQL Service Account" -SamAccountName "svc_mssql" `
    -UserPrincipalName "svc_mssql@lab.local" `
    -Path "OU=ServiceAccounts,OU=LAB_Corp,DC=lab,DC=local" `
    -AccountPassword $passwordSvc -Enabled $true -ChangePasswordAtLogon$false

# 5. Asignar Service Principal Name (SPN) para habilitar Kerberoasting
Set-ADUser -Identity "svc_mssql" -ServicePrincipalNames @{Add="MSSQLSvc/DC01.lab.local:1433"}
```

## 🚨 6. Troubleshooting y Solución de Errores Comunes

- **Error: `New-ADOrganizationalUnit : El término 'New-ADOrganizationalUnit' no se reconoce...`**
    
    - _Causa:_ El script se ejecutó en una máquina sin las herramientas RSAT instaladas (ej. `Win-11-Client`) o el módulo no estaba cargado.
        
    - _Solución:_ Ejecutar en `DC01` cargando el módulo mediante `Import-Module ActiveDirectory` o instalando la característica con `Add-WindowsFeature RSAT-AD-PowerShell`.
        
- **Error: `InvalidArgument: ParameterBindingException - NamedParameterNotFound` en `-ChangePasswordAtLogon`**
    
    - _Causa:_ Error tipográfico por falta de espacio (`-ChangePasswordAtLogon$false` en lugar de `-ChangePasswordAtLogon $false`).
        
    - _Solución:_ Asegurar la separación correcta entre el nombre del parámetro y `$false`.
        

## ✅ 7. Verificación Final e Inicio de Sesión de Dominio

1. **Inicio de sesión desde `Win-11-Client`:**
    
    - En la pantalla de bloqueo, seleccionar **Otro usuario**.
        
    - Credenciales: `jdoe` / `UserPass123!`.
        
    - Comprobar que en la parte inferior de la pantalla indique _"Iniciar sesión en: LAB"_.
        
2. **Confirmación desde la Consola de Comandos (`cmd.exe`):**
    



```DOS
C:\Users\jdoe> whoami
lab\jdoe

C:\Users\jdoe> net user jdoe /domain
Se procesará la solicitud en un controlador de dominio del dominio lab.local.

Nombre de usuario                   jdoe
Nombre completo                     John Doe
Cuenta activa                       Sí
La cuenta expira                    Nunca
...
Miembros del grupo global           *Usuarios del dominio
Se ha completado el comando correctamente.
```

3. **Verificación visual en `DC01`:**
    
    - Abrir `dsa.msc` (_Usuarios y equipos de Active Directory_) y verificar la presencia de la OU `LAB_Corp` con sus sub-OUs y los usuarios `jdoe` y `svc_mssql`.