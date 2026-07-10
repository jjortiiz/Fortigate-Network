# Implementación de Red usando FortiGate (GUI)

## Información Académica
* **Institución:** Instituto Tecnológico de las Américas (ITLA)
* **Nombres:** Juan Francisco Javier Ortiz
* **Matrícula:** 2025-0861
* **Materia:** Seguridad de Redes
* **Docente:** Jonathan Rondon
* **Carrera:** Seguridad Informática
* **Fecha:** Junio 21, 2025

---

## 1. Objetivo
El objetivo principal de este laboratorio es implementar, configurar y asegurar una infraestructura de red perimetral utilizando un firewall FortiGate (FTG-SR) gestionado totalmente a través de su interfaz gráfica (GUI). Se busca proporcionar acceso seguro a Internet para los usuarios internos, aislar y proteger la zona de servidores (DMZ) aplicando políticas estrictas de control de acceso, y desplegar perfiles de seguridad avanzados (UTM/NGFW) como control de aplicaciones, filtrado web, IPS y WAF para mitigar amenazas y bloquear tráfico no autorizado.

---

## 2. Topología
La topología implementada en el laboratorio simula un entorno perimetral conectado a un proveedor de Internet (NUBE ISP), interconectando una zona LAN de usuarios mediante un switch de distribución (SW-1) y una zona de servidores que contiene un servidor web (WINDOWS SERVER).

### 2.1 Tabla de Direccionamiento

| Dispositivo | Interfaz | IP | Gateway | Máscara | Descripción |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **ISP** | - | - | 192.168.5.2 | 255.255.255.0 | Proveedor de Internet |
| **FTG-SR** | port1 | 192.168.5.150 | 192.168.5.2 | 255.255.255.0 | Conexión con ISP (WAN) |
| **FTG-SR** | port2 | 8.61.10.1 | - | 255.255.255.128 | Gateway LAN de Usuarios |
| **FTG-SR** | port3 | 8.61.10.129 | - | 255.255.255.240 | Gateway LAN de Servidores |
| **SW-1** | e0/0 | - | - | - | Switch para distribución de equipos |
| **WINDOWS SERVER** | e0 | 8.61.10.130 | 8.61.10.129 | 255.255.255.240 | Servidor WEB |
| **PC-01** | e0 | 8.61.10.5 | 8.61.10.1 | 255.255.255.128 | Equipo cliente con IP por DHCP |
| **PC-02** | e0 | 8.61.10.6 | 8.61.10.1 | 255.255.255.128 | Equipo cliente con IP por DHCP |
| **PC-03 Linux** | e0 | 8.61.10.7 | 8.61.10.1 | 255.255.255.128 | Equipo cliente con IP por DHCP |
| **VPC** | e0 | 8.61.10.8 | 8.61.10.1 | 255.255.255.128 | Equipo cliente con IP por DHCP |

---

## 3. Configuración y Políticas de Seguridad

### 3.1 Interfaces de Red y Enrutamiento
* Se configuró la asignación correcta de direcciones IP en las interfaces físicas del FortiGate (`port1` para WAN, `port2` para Usuarios y `port3` para Servidores).
* Activación del servicio **DHCP Server** en el `port2` para la asignación dinámica de direccionamiento a los clientes internos (`PC-01`, `PC-02`, `PC-03 Linux` y `VPC`).
* Validación y establecimiento de una **Ruta Estática** (`0.0.0.0/0`) apuntando al gateway `192.168.5.2` a través de la interfaz `port1` (WAN).
* Configuración de una política de firewall específica que **SOLO permite tráfico HTTP** originado en la LAN de Usuarios con destino a la zona de Servidores.

### 3.2 Perfiles de Seguridad (UTM / NGFW)

#### Filtro Web (Web Filter)
* Creación del perfil `bloqueo_dominios_itla` bajo la modalidad *Filter-based*.
* Configuración de un **Static URL Filter** para bloquear explícitamente el dominio de `itla.edu.do`, subdominios y la plataforma `youtube.com` mediante el uso de reglas de tipo *Wildcard* en acción *Block*.

#### Control de Aplicaciones (Application Control)
* Implementación de sensores destinados a la restricción de redes sociales y plataformas multimedia.
* Configuración de bloqueos específicos (*Application and Filter Overrides*) haciendo énfasis en firmas comunes como:
  * **Facebook** (incluyendo sub-firmas como Facebook App, Chat y Login)
  * **Twitter / X**
  * **Instagram**
* Se configuró un filtro restrictivo prioritario para mitigar el uso de llamadas y videollamadas de voz sobre IP (**WhatsApp VoIPCall**).

#### Política de Seguridad DoS y Bloqueo de Escaneos
* Configuración de umbrales estrictos dentro de las políticas de firewall DoS frente a anomalías de nivel de capa 3 y 4 (*L4 Anomalies*).
* Inclusión de reglas de mitigación con acción *Block* para las firmas:
  * `tcp_syn_flood` (Umbral: 2000)
  * `tcp_port_scan` (Umbral: 1000)
  * `tcp_src_session` y `tcp_dst_session` (Umbrales: 5000)
* Bloqueo verificado de manera efectiva impidiendo el reconocimiento y escaneo de puertos de la infraestructura mediante herramientas de auditoría externa como **Nmap**.

#### Web Application Firewall (WAF)
* Despliegue del perfil `WAF Server` protegiendo de forma dedicada el segmento del Servidor Web (`WINDOWS SERVER`).
* Activación y puesta en marcha en modo *Block* de firmas de severidad alta para la contención de ataques web automatizados y manuales, tales como inyección SQL (**SQL Injection**) y explotación de vulnerabilidades conocidas (**Known Exploits**).

---

## 4. Comprobaciones de Funcionamiento

### 4.1 Verificación de Bloqueo de Redes Sociales (Instagram)
Al intentar acceder a la dirección `http://instagram.com` desde un host interno, la conexión es interceptada de manera exitosa por el FortiGate mostrando el aviso correspondiente:
* **Mensaje:** *Application Blocked. You have attempted to use an application that violates your Internet usage policy.*
* **Aplicación Identificada:** Instagram
* **Categoría:** Social Media

### 4.2 Verificación de Filtro Web (itla.edu.do)
Al realizar una petición HTTP hacia el dominio local restringido `http://itla.edu.do`, la herramienta perimetral bloquea la carga de la página mediante las firmas locales:
* **Mensaje:** *Web Page Blocked. The page you have requested has been blocked because the URL is banned.*
* **Filtro Origen:** *Local URLfilter Block*

### 4.3 Verificación Anti-Escaneos (Nmap)
Al ejecutar una inspección y escaneo de puertos mediante `nmap -sS -P0 8.61.10.130` desde una máquina de auditoría (Kali Linux) hacia el entorno interno, los mecanismos de prevención perimetral bloquean los sondeos de red evitando la materialización y recolección de información sensible de los activos:
* **Resultado del escaneo:** *Note: Host seems down. If it is really up, but blocking our ping probes, try -Pn. Nmap done: 1 IP address (0 hosts up) scanned.*

### 4.4 Verificación de Protección WAF (Mitigación SQLi)
Al simular un ataque web mediante la inyección de sentencias lógicas maliciosas en los parámetros de la URL (`http://8.61.10.130/?id=1 OR 1=1`), el módulo de Firewall de Aplicaciones Web detiene inmediatamente la transferencia de datos:
* **Mensaje:** *Web Application Firewall. This transfer is blocked by a Web Application Firewall.*
* **Event ID:** 30000040
* **Event Type:** Signature (SQL Injection)
