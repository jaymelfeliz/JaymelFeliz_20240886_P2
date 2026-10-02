# Laboratorio FortiGate Site-to-Site IPsec VPN (GNS3)

> **Nota:** El enlace/video demostrativo en vivo de la implementación y pruebas debe colocarse directamente en este encabezado.

## 🎥 Video Demostrativo

*Haz clic en la imagen superior para ver la demostración completa del despliegue, configuración de políticas, DHCP y pruebas de conectividad bidireccional.*

## 📌 Propósito del Laboratorio

El propósito fundamental de este laboratorio es diseñar, implementar y validar un **túnel VPN IPsec Site-to-Site con IKEv2** utilizando dos cortafuegos **FortiGate (FortiOS v7.x)** en un entorno de simulación **GNS3**.

### Objetivos Clave:

1. **Conectividad Segura:** Interconectar de forma cifrada la red de Usuarios (Sitio A) con la red de Servidores (Sitio B) atravesando un segmento WAN inseguro (`192.168.56.0/24`).

2. **Políticas de Seguridad sin NAT:** Configurar políticas de inspección bidireccionales en FortiOS excluyendo NAT para preservar las direcciones IP de origen/destino originales.

3. **Servicios de Red Dinámicos:** Asignar dirección IP, máscara y Default Gateway al Servidor Ubuntu remota mediante el servidor DHCP de FortiGate-B.

4. **Verificación y Aislamiento:** Validar el cifrado del tráfico mediante comandos de diagnóstico CLI (`sniffer`, `ike debug`, `ping-options`) y comprobar que el tráfico fuera del túnel sea rechazado.

## 📐 Diagrama de Topología

```
flowchart LR
    subgraph SitioA ["Sitio A (LAN Usuarios)"]
        VPCS["VPCS Client\n10.8.86.2/25"] ---|port2| FGT_A["FortiGate-A\nGW: 10.8.86.1/25"]
    end

    subgraph WAN ["Red WAN / Enlace Inseguro"]
        FGT_A ---|port1: 192.168.56.50| Tunnel(("Túnel IPsec\n'VPN-Nueva'\nAES128-SHA256"))---|port1: 192.168.56.60| FGT_B
    end

    subgraph SitioB ["Sitio B (LAN Servidores)"]
        FGT_B["FortiGate-B\nGW: 10.8.86.129/28\n(DHCP Server)"] ---|port2| Ubuntu["Servidor Ubuntu\n10.8.86.130/28"]
    end

    classDef fgt fill:#f96,stroke:#333,stroke-width:2px;
    classDef node fill:#6cf,stroke:#333,stroke-width:1px;
    class FGT_A,FGT_B fgt;
    class VPCS,Ubuntu node;

```

## 📑 Tabla de Parámetros de Red

| Dispositivo | Interfaz | Dirección IP / Máscara | Función / Descripción | 
 | ----- | ----- | ----- | ----- | 
| **VPCS** | `eth0` | `10.8.86.2/25` | Cliente de la red de usuarios (GW: `10.8.86.1`) | 
| **FortiGate-A** | `port1` (WAN) | `192.168.56.50/24` | IPsec Gateway Local | 
| **FortiGate-A** | `port2` (LAN) | `10.8.86.1/25` | Gateway predeterminado Red Usuarios | 
| **FortiGate-B** | `port1` (WAN) | `192.168.56.60/24` | IPsec Gateway Remoto | 
| **FortiGate-B** | `port2` (LAN) | `10.8.86.129/28` | Gateway Servidores + DHCP Server | 
| **Servidor Ubuntu** | `eth0` | `10.8.86.130/28` | Servidor de Destino (Asignación DHCP) | 

## 📷 Evidencias de Configuración y Pruebas (Imágenes)

### 1. Estado del Túnel IPsec en FortiOS

 *Figura 1: Estado activo (Up/Verde) del túnel IPsec en el monitor de FortiGate-A y FortiGate-B.*

### 2. Configuración de Políticas de Firewall (Sin NAT)

 *Figura 2: Reglas de entrada y salida permitiendo el tráfico LAN <-> VPN con NAT desactivado.*

### 3. Pruebas de Ping y Sniffer de Paquetes

 *Figura 3: Captura del comando `diagnose sniffer packet` confirmando el tráfico de entrada por la LAN y salida encapsulada por la VPN.*

## 📜 Scripts y Comandos Utilizados

Los comandos exactos aplicados en la CLI de FortiOS y Ubuntu durante el despliegue se encuentran almacenados en la carpeta [`/scripts`](./scripts/):

### 1. Script de Configuración FortiGate-A (`scripts/fortigate_a.conf`)

```
config vpn ipsec phase1-interface
    edit "VPN-Nueva"
        set interface "port1"
        set peertype any
        set net-device disable
        set proposal aes128-sha256
        set remote-gw 192.168.56.60
        set psksecret Secreta123
    next
end

config vpn ipsec phase2-interface
    edit "VPN-Nueva-P2"
        set phase1name "VPN-Nueva"
        set proposal aes128-sha256
        set src-subnet 10.8.86.0 255.255.255.128
        set dst-subnet 10.8.86.128 255.255.255.240
    next
end

config router static
    edit 1
        set dst 10.8.86.128 255.255.255.240
        set device "VPN-Nueva"
    next
end

```

### 2. Script de Configuración FortiGate-B (`scripts/fortigate_b.conf`)

```
config vpn ipsec phase1-interface
    edit "VPN-Nueva"
        set interface "port1"
        set peertype any
        set net-device disable
        set proposal aes128-sha256
        set remote-gw 192.168.56.50
        set psksecret Secreta123
    next
end

config vpn ipsec phase2-interface
    edit "VPN-Nueva-P2"
        set phase1name "VPN-Nueva"
        set proposal aes128-sha256
        set src-subnet 10.8.86.128 255.255.255.240
        set dst-subnet 10.8.86.0 255.255.255.128
    next
end

config system dhcp server
    edit 1
        set default-gateway 10.8.86.129
        set netmask 255.255.255.240
        set interface "port2"
        config ip-range
            edit 1
                set start-ip 10.8.86.130
                set end-ip 10.8.86.142
            next
        end
    next
end

config router static
    edit 1
        set dst 10.8.86.0 255.255.255.128
        set device "VPN-Nueva"
    next
end

```

### 3. Comandos de Verificación en Servidor Ubuntu

```
# Solicitar IP por DHCP desde el FortiGate-B
sudo dhclient eth0

# Verificar IP asignada y gateway predeterminado
ip a
ip route

# Desactivar firewall interno de Ubuntu para permitir sondas
sudo ufw disable

```

## 📁 Estructura del Repositorio

```
├── README.md                      <- Documentación Principal
├── docs/
│   └── images/                    <- Diagramas y Capturas de Pantalla
│       ├── 01_ipsec_tunnel_up.png
│       ├── 02_firewall_policies.png
│       └── 03_icmp_sniffer_test.png
├── configs/                       <- Running-Configs Completos de FortiOS
│   ├── FortiGate-A-running.conf
│   └── FortiGate-B-running.conf
└── scripts/                       <- CLI Scripts de Despliegue Rápido
    ├── fortigate_a.conf
    ├── fortigate_b.conf
    └── ubuntu_setup.sh

```

## 💾 Running-Configs

Los respaldos (*Running Configurations*) completos extraídos mediante el comando `show full-configuration` se encuentran ubicados dentro de la carpeta `/configs/`:

* [Running Configuration FortiGate-A](./configs/FortiGate-A-running.conf)

* [Running Configuration FortiGate-B](./configs/FortiGate-B-running.conf)

## 🧪 Comandos de Diagnóstico Útiles

```
# Debug de negociación IKE / IPsec en tiempo real
diagnose debug application ike -1
diagnose debug enable

# Sniffer de tráfico ICMP filtrado por host
diagnose sniffer packet any "host 10.8.86.130" 4 10 a

# Probar ping forzando la interfaz de origen (LAN)
execute ping-options source 10.8.86.1
execute ping 10.8.86.130

```
