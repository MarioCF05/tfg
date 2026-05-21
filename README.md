# Práctica 6-3: Alta Disponibilidad (HA) en Proxmox con Contenedores LXC

## Requisitos previos
- Proxmox VE instalado (cluster con nodos: Marisma001, Marisma002, Marisma003)
- Template LXC de Ubuntu descargado: `ubuntu-24.04-standard_24.04-2_amd64.tar.zst`
- Red: `192.168.8.0/24`

> **Nota:** Keepalived requiere contenedores **privilegiados** para funcionar (necesita acceso a capacidades de red como `CAP_NET_ADMIN` y `CAP_NET_RAW`).

---

## Paso 1: Crear los contenedores web

| CT | Nombre | IP | Rol |
|---|---|---|---|
| CT 201 | mario-server1 | 192.168.8.101 | Servidor principal |
| CT 202 | mario-server2 | 192.168.8.102 | Servidor secundario |

```bash
# Ver templates disponibles
pveam update
pveam available | grep ubuntu

# Crear contenedores
pct create 201 local:vztmpl/ubuntu-24.04-standard_24.04-2_amd64.tar.zst \
  --storage local-lvm --memory 512 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.8.101/24,gw=192.168.8.1 \
  --hostname mario-server1 --ostype ubuntu --unprivileged 0 --features nesting=1

pct create 202 local:vztmpl/ubuntu-24.04-standard_24.04-2_amd64.tar.zst \
  --storage local-lvm --memory 512 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.8.102/24,gw=192.168.8.1 \
  --hostname mario-server2 --ostype ubuntu --unprivileged 0 --features nesting=1

# Iniciar
pct start 201
pct start 202

# Acceder
pct enter 201
```

> `--unprivileged 0` es necesario porque Keepalived necesita capacidades de red.

### Configurar Apache en ambos contenedores

```bash
# Dentro de cada contenedor (pct enter 201 y 202)
apt update && apt install apache2 -y
echo "<h1>Servidor Web $(hostname)</h1>" > /var/www/html/index.html
systemctl enable apache2 && systemctl restart apache2
```

Comprobar:
```bash
curl 127.0.0.1
```

---

## Paso 2: Crear contenedor HAProxy

| CT | Nombre | IP | Rol |
|---|---|---|---|
| CT 204 | haproxy | 192.168.8.200 | Balanceador |

```bash
pct create 204 local:vztmpl/ubuntu-24.04-standard_24.04-2_amd64.tar.zst \
  --storage local-lvm --memory 256 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.8.200/24,gw=192.168.8.1 \
  --hostname haproxy --ostype ubuntu --unprivileged 1

pct start 204
pct enter 204
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
    server mario-server1 192.168.8.101:80 check
    server mario-server2 192.168.8.102:80 check
```

```bash
systemctl restart haproxy
```

---

## Paso 3: Verificar balanceo

```bash
curl 192.168.8.200
# Alterna entre "Servidor Web mario-server1" y "Servidor Web mario-server2"
```

---

## Paso 4: Instalar y configurar Keepalived

Keepalived necesita contenedores privilegiados (`--unprivileged 0` ya lo aplica). Si no funciona, edita `/etc/pve/lxc/201.conf` (y `202.conf`):

```
lxc.cap.drop:
lxc.cgroup.devices.allow: c 10:118 rwm
```

### Instalar

```bash
pct enter 201
apt install keepalived -y
# Lo mismo en 202
```

### Configurar en mario-server1 (MASTER) - `/etc/keepalived/keepalived.conf`

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
        192.168.8.100
    }
    track_script {
        chk_apache
    }
}
```

### Configurar en mario-server2 (BACKUP) - `/etc/keepalived/keepalived.conf`

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
        192.168.8.100
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

1. **VIP**: `ip a | grep 192.168.8.100` (debe estar en mario-server1)
2. **Caída**: `systemctl stop apache2` en mario-server1
3. **Migración**: `ip a | grep 192.168.8.100` (ahora en mario-server2)
4. **Acceso**: `curl 192.168.8.100` (responde mario-server2)
5. **Restaurar**: `systemctl start apache2` en mario-server1 (VIP vuelve)

---

## Paso 6: Migración automática en cluster Proxmox

### 6.1 Crear cluster

```bash
# En Marisma001
pvecm create NOMBRE_CLUSTER

# En Marisma002 y Marisma003
pvecm add IP_MARISMA001
```

### 6.2 Añadir contenedores al HA Manager

```bash
ha-manager add ct:201 --state started
ha-manager add ct:202 --state started
ha-manager add ct:204 --state started
```

### 6.3 Crear grupo HA

```bash
pvesh create /cluster/ha/groups --group ha_group \
  --nodes "marisma002,marisma001,marisma003" \
  --type ordered --noed 1
```

### 6.4 Asignar contenedores al grupo

```bash
pvesh set /cluster/ha/resources/ct:201 --group ha_group
pvesh set /cluster/ha/resources/ct:202 --group ha_group
pvesh set /cluster/ha/resources/ct:204 --group ha_group
```

### 6.5 Verificar

```bash
ha-manager config
ha-manager status
ha-manager list
```

### 6.6 Probar migración

```bash
# Opción A - Migración manual
pvesh create /cluster/ha/resources/ct:201/migrate --node marisma001

# Opción B - Caída de nodo
systemctl stop pve-cluster  # en Marisma002
```

---

## Verificación final

```bash
pvecm status
ha-manager status
pct list
```

---

## Notas

- Prioridad cluster: **Marisma002** → **Marisma001** → **Marisma003**
- Contenedores privilegiados (`--unprivileged 0`) para Keepalived
- Usar `ct:ID` en HA Manager (no `vm:ID`)
- Keepalived usa VRRP con VIP `192.168.8.100`
- HAProxy balancea en round-robin entre mario-server1 y mario-server2
