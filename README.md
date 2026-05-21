# Práctica 6-3: Alta Disponibilidad (HA) en Proxmox

## Requisitos previos
- Proxmox VE instalado (cluster con nodos: Marisma001, Marisma002, Marisma003)
- Plantilla/ISO de Debian/Ubuntu para crear VMs
- Red configurada en Proxmox

---

## Paso 1: Crear dos servidores virtuales en Proxmox

Crea dos VMs en Proxmox con la misma configuración:

| VM | Nombre | IP | Rol |
|---|---|---|---|
| VM 101 | web1 | 192.168.1.101 | Servidor principal |
| VM 102 | web2 | 192.168.1.102 | Servidor secundario |

```bash
# Desde el nodo principal de Proxmox, crea las VMs
qm create 101 --name web1 --memory 512 --net0 virtio,bridge=vmbr0
qm create 102 --name web2 --memory 512 --net0 virtio,bridge=vmbr0
```

### Configurar servicio web (Apache/Nginx) en ambas VMs

En cada VM:

```bash
# Instalar Apache
apt update && apt install apache2 -y

# Crear página de identificación
echo "<h1>Servidor Web $(hostname)</h1>" > /var/www/html/index.html
systemctl enable apache2 && systemctl restart apache2
```

---

## Paso 2: Instalar y configurar HAProxy (balanceador de carga)

Crea una tercera VM para HAProxy:

| VM | Nombre | IP | Rol |
|---|---|---|---|
| VM 103 | haproxy | 192.168.1.200 | Balanceador de carga |

```bash
# Instalar HAProxy
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

En las VMs web1 y web2:

```bash
apt install keepalived -y
```

### Configurar Keepalived en web1 (maestro) - `/etc/keepalived/keepalived.conf`

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

### Configurar Keepalived en web2 (backup) - `/etc/keepalived/keepalived.conf`

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

Configurar el cluster Proxmox para migración automática con prioridades.

### 6.1 Crear el cluster Proxmox

```bash
# En Marisma001 (nodo maestro)
pvecm create NOMBRE_CLUSTER

# En Marisma002 y Marisma003 (nodos secundarios)
pvecm add IP_DEL_MAESTRO
```

### 6.2 Configurar HA para las VMs

```bash
# Añadir VMs al gestor HA de Proxmox
ha-manager add vm:101 --state started
ha-manager add vm:102 --state started
ha-manager add vm:103 --state started
```

### 6.3 Configurar grupos de recursos HA

En la interfaz web de Proxmox:
- Datacenter → HA → Groups → Crear grupo
- Nombre: `ha_group`
- Nodos ordenados por prioridad: `Marisma002, Marisma001, Marisma003`
- Noed (no hay restricción)

O por línea de comandos en un nodo del cluster:

```bash
pvesh create /cluster/ha/groups --group ha_group --nodes "marisma002,marisma001,marisma003" --type ordered --noed 1
```

### 6.4 Asignar VMs al grupo HA

```bash
# Asignar cada VM al grupo HA
pvesh set /cluster/ha/resources/vm:101 --group ha_group
pvesh set /cluster/ha/resources/vm:102 --group ha_group
pvesh set /cluster/ha/resources/vm:103 --group ha_group
```

### 6.5 Verificar configuración HA

```bash
# Ver grupos
ha-manager config

# Ver estado
ha-manager status

# Ver recursos
ha-manager resources
```

### 6.6 Probar migración automática

```bash
# Simular caída de Marisma002 (nodo con prioridad 1)
# Desde otro nodo o desde Proxmox:
pvesh create /cluster/ha/resources/vm:101/migrate --node marisma001
```

O directamente apagar un nodo del cluster para ver la migración automática:

```bash
# En el nodo a probar (ej. Marisma002)
systemctl stop pve-cluster
# o directamente:
poweroff
```

Las VMs deben migrar automáticamente a Marisma001 (siguiente prioridad).

---

## Verificación final

```bash
# Estado del cluster
pvecm status

# Estado HA
ha-manager status

# Comprobar que las VMs están corriendo
qm list
```

---

## Notas

- La prioridad definida es: **Marisma002** (1ª) → **Marisma001** (2ª) → **Marisma003** (3ª)
- Keepalived usa VRRP para la IP virtual flotante entre web1 y web2
- HAProxy balancea en round-robin entre ambos servidores web
- Proxmox HA Manager migra VMs automáticamente si un nodo del cluster cae
