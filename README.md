# Práctica 6-3: Alta Disponibilidad (HA) en Proxmox con Contenedores LXC

## Requisitos previos
- Proxmox VE instalado (cluster con nodos: Marisma001, Marisma002, Marisma003)
- Template LXC de Debian/Ubuntu descargado (ej. `ubuntu-24.04-standard_24.04-2_amd64.tar.zst`)
- Red configurada en Proxmox

> **Nota:** Keepalived requiere contenedores **privilegiados** para funcionar (necesita acceso a capacidades de red como `CAP_NET_ADMIN` y `CAP_NET_RAW`).

---

## Paso 1: Crear dos contenedores web en Proxmox

Crea dos contenedores LXC con la misma configuración:

| CT | Nombre | IP | Rol |
|---|---|---|---|
| CT 101 | web1 | 192.168.1.101 | Servidor principal |
| CT 102 | web2 | 192.168.1.102 | Servidor secundario |

```bash
# Descargar template si no lo tienes
pveam update
pveam available | grep ubuntu

# Desde el nodo principal de Proxmox, crear los contenedores
pct create 101 template-name --storage local --memory 512 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.1.101/24,gw=192.168.1.1 \
  --ostype ubuntu --unprivileged 0 --features nesting=1

pct create 102 template-name --storage local --memory 512 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.1.102/24,gw=192.168.1.1 \
  --ostype ubuntu --unprivileged 0 --features nesting=1

# Iniciar contenedores
pct start 101
pct start 102

# Acceder
pct enter 101
```

> `--unprivileged 0` es necesario porque Keepalived necesita capacidades de red.

### Configurar servicio web (Apache) en ambos contenedores

En cada contenedor (vía `pct enter` o SSH):

```bash
# Instalar Apache
apt update && apt install apache2 -y

# Crear página de identificación
echo "<h1>Servidor Web $(hostname)</h1>" > /var/www/html/index.html
systemctl enable apache2 && systemctl restart apache2
```

Comprobar que funciona:
```bash
curl 127.0.0.1
```

---

## Paso 2: Crear contenedor HAProxy (balanceador de carga)

| CT | Nombre | IP | Rol |
|---|---|---|---|
| CT 103 | haproxy | 192.168.1.200 | Balanceador de carga |

```bash
pct create 103 template-name --storage local --memory 256 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.1.200/24,gw=192.168.1.1 \
  --ostype ubuntu --unprivileged 1

pct start 103

# Instalar HAProxy
pct enter 103
apt update && apt install haproxy -y
```

### Configurar HAProxy (`/etc/haproxy/haproxy.cfg`)

```cfg
global
    log /dev/log local0
    maxconn 4096
    user haproxy
    group haproxy

defaults
    log global
    mode http
    option httplog
    option dontlognull
    retries 3
    timeout connect 5000
    timeout client 50000
    timeout server 50000

frontend web_frontend
    bind *:80
    stats uri /haproxy?stats
    default_backend web_servers

backend web_servers
    balance roundrobin
    server web1 192.168.1.101:80 check
    server web2 192.168.1.102:80 check
```

```bash
systemctl restart haproxy
```

---

## Paso 3: Verificar balanceo

Comprueba que HAProxy balancea correctamente:

```bash
# Desde cualquier máquina
curl 192.168.1.200
# Debe alternar entre "Servidor Web web1" y "Servidor Web web2"
```

---

## Paso 4: Instalar y configurar Keepalived

Keepalived necesita que los contenedores web1 y web2 tengan ciertas capacidades. Si usaste `--unprivileged 0` al crearlos, ya funciona. Si no, edita `/etc/pve/lxc/101.conf`:

```
lxc.cap.drop:
lxc.cgroup.devices.allow: c 10:118 rwm
```

### Instalar Keepalived en web1 y web2

```bash
pct enter 101  # o 102
apt install keepalived -y
```

### Configurar Keepalived en web1 (MASTER) - `/etc/keepalived/keepalived.conf`

```cfg
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1234
    }
    virtual_ipaddress {
        192.168.1.100
    }
    track_script {
        chk_apache
    }
}
```

### Configurar Keepalived en web2 (BACKUP) - `/etc/keepalived/keepalived.conf`

```cfg
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1234
    }
    virtual_ipaddress {
        192.168.1.100
    }
    track_script {
        chk_apache
    }
}
```

### Script de monitoreo (`/etc/keepalived/check_apache.sh`)

```bash
#!/bin/bash
if systemctl is-active --quiet apache2; then
    exit 0
else
    exit 1
fi
```

```bash
chmod +x /etc/keepalived/check_apache.sh
systemctl restart keepalived
```

---

## Paso 5: Probar failover

1. **Comprobar VIP**: `ip a | grep 192.168.1.100` (debe aparecer en web1)
2. **Simular caída del web1**: `systemctl stop apache2` en web1
3. **Verificar que VIP migra a web2**: `ip a | grep 192.168.1.100` (ahora en web2)
4. **Acceder vía VIP**: `curl 192.168.1.100` (debe responder web2)
5. **Restaurar web1**: `systemctl start apache2` y verificar que VIP vuelve a web1

---

## Paso 6: Migración automática en cluster Proxmox

Configurar el cluster Proxmox para migración automática de contenedores con prioridades.

### 6.1 Crear el cluster Proxmox

```bash
# En Marisma001 (nodo maestro)
pvecm create NOMBRE_CLUSTER

# En Marisma002 y Marisma003 (nodos secundarios)
pvecm add IP_DEL_MAESTRO
```

### 6.2 Configurar HA para los contenedores

```bash
# Añadir contenedores al gestor HA (usar ct: en vez de vm:)
ha-manager add ct:101 --state started
ha-manager add ct:102 --state started
ha-manager add ct:103 --state started
```

### 6.3 Configurar grupos de recursos HA

Por línea de comandos en un nodo del cluster:

```bash
pvesh create /cluster/ha/groups --group ha_group \
  --nodes "marisma002,marisma001,marisma003" \
  --type ordered --noed 1
```

O desde la interfaz web:
- Datacenter → HA → Groups → Crear grupo
- Nombre: `ha_group`
- Nodos ordenados por prioridad: `Marisma002, Marisma001, Marisma003`
- Noed activado

### 6.4 Asignar contenedores al grupo HA

```bash
pvesh set /cluster/ha/resources/ct:101 --group ha_group
pvesh set /cluster/ha/resources/ct:102 --group ha_group
pvesh set /cluster/ha/resources/ct:103 --group ha_group
```

### 6.5 Verificar configuración HA

```bash
# Ver grupos
ha-manager config

# Ver estado
ha-manager status

# Ver recursos
ha-manager list
```

### 6.6 Probar migración automática

**Opción A - Migración manual controlada:**
```bash
pvesh create /cluster/ha/resources/ct:101/migrate --node marisma001
```

**Opción B - Simular caída de nodo:**
```bash
# En el nodo a probar (ej. Marisma002)
systemctl stop pve-cluster
# o directamente:
poweroff
```

Los contenedores deben migrar automáticamente a Marisma001 (siguiente prioridad).

---

## Verificación final

```bash
# Estado del cluster
pvecm status

# Estado HA
ha-manager status

# Comprobar que los contenedores están corriendo
pct list
```

---

## Notas

- La prioridad definida es: **Marisma002** (1ª) → **Marisma001** (2ª) → **Marisma003** (3ª)
- Los contenedores son **privilegiados** (`--unprivileged 0`) para que Keepalived funcione
- En HA Manager se usa `ct:ID` en lugar de `vm:ID` para contenedores LXC
- Keepalived usa VRRP para la IP virtual flotante entre web1 y web2
- HAProxy balancea en round-robin entre ambos servidores web
- Proxmox HA Manager migra contenedores automáticamente si un nodo del cluster cae
