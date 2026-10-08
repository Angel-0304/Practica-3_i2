# Practica-3_i2
Infraestructura 2

# Práctica 3 – Infraestructura 2: Jump Server con VPN de acceso remoto

**ITLA – Instituto Tecnológico de Las Américas**
**Estudiante:** Angel Alcántara · **Matrícula:** 2024-2356 · **Carrera:** Ciberseguridad
**Docente:** Jonathan Esteban Rondón Corniel · **Fecha:** 8 de octubre de 2026

🎥 **Video de la demostración:** [Tarea Semana 4 (Practica 3)](https://itlaedudo-my.sharepoint.com/:f:/g/personal/20242356_itla_edu_do/IgBIrjfN_D2pRaQ0YzuoF4zeAZHWqsh6jn8hGo0eqXDlMN4?e=GjFpqC)
📄 **Configuraciones completas:** [AngelAlcantara_20242356_P3_i2.txt](AngelAlcantara_20242356_P3_i2.txt)

---

## Contenido

1. [Objetivo](#1-objetivo)
2. [Topología y direccionamiento](#2-topología-y-direccionamiento)
3. [Equipo de red (R-USER, SW-Usuarios e ISP)](#3-equipo-de-red-r-user-sw-usuarios-e-isp)
4. [FortiGate](#4-fortigate)
5. [Web Server](#5-web-server)
6. [Jump Server](#6-jump-server)
7. [Cliente (PC-Kali)](#7-cliente-pc-kali)
8. [Pruebas](#8-pruebas)
9. [Problemas que salieron y cómo se resolvieron](#9-problemas-que-salieron-y-cómo-se-resolvieron)
10. [Running-config de los equipos](#10-running-config-de-los-equipos)

---

## 1. Objetivo

La idea de esta infraestructura es que los usuarios no lleguen directo a los servidores. El usuario se conecta por una **VPN de acceso remoto** al FortiGate, y esa VPN **solo le da acceso al Jump Server**. Desde el Jump Server, a través de **RemoteApps publicadas en el RD Web Client**, se trabaja con el Web Server, y el Jump Server solo puede llegar a él por **HTTPS, RDP y SSH**.

Hay dos usuarios con permisos diferentes:

| Usuario | Grupo | Lo que ve en el Jump Server | Restricción |
|---|---|---|---|
| Carlos Pérez (`cperez`) | Sin privilegios | Solo la web (Sistema de Caja) | Política **explícita** que le bloquea el SSH |
| María Rodríguez (`mrodriguez`) | Con privilegios | Web, PuTTY y Escritorio Remoto | — |

Toda la configuración del FortiGate se hizo por la GUI (por consola solo el acceso de gestión inicial).

---

## 2. Topología y direccionamiento

![Topología en PNETLab](img/01-topologia.png)

| Segmento | Red | Gateway | Equipos |
|---|---|---|---|
| WAN R-USER | 20.24.23.0/30 | 20.24.23.1 (ISP) | R-USER .2 |
| WAN FortiGate | 20.24.56.0/30 | 20.24.56.1 (ISP) | FG-Server port1 .2 |
| VLAN 10 – Usuarios | 10.23.56.0/25 | 10.23.56.1 (R-USER) | DHCP .10 a .100 |
| LAN-WEB (port4) | 10.23.56.128/29 | 10.23.56.129 | Web-Server .130 |
| LAN-JUMP (port2) | 10.23.56.136/29 | 10.23.56.137 | Jump Server .138 |
| Pool de la VPN | 10.23.56.200 – .210 | — | Una IP /32 por cliente |
| Gestión FortiGate (port3) | 192.168.128.0/24 | — | 192.168.128.141 (Cloud0) |

| Equipo | Imagen |
|---|---|
| FG-Server | FortiGate VM64-KVM v6.4.0 (licencia de evaluación) |
| R-USER / ISP / SW-Usuarios | Cisco IOL |
| Web-Server | Ubuntu Server 20.04 |
| Jump Server | Windows Server 2022 Standard Evaluation (Desktop Experience) |
| PC-Kali | Kali Linux 2026.2 |

> La licencia de evaluación del FortiGate limita la VPN a **DES/MD5** y a un **máximo de 5 políticas**. Se usaron 4.

---

## 3. Equipo de red (R-USER, SW-Usuarios e ISP)

- **SW-Usuarios:** VLAN 10, trunk hacia el R-USER solo con la VLAN 10 y el puerto del PC en modo access. Los puertos sin uso están apagados.
- **R-USER:** router-on-a-stick con la subinterfaz `e0/1.10`, servidor DHCP para la VLAN 10, NAT hacia el ISP y `ip tcp adjust-mss 1360` para que no haya problemas de tamaño con la VPN.
- **ISP:** solo tiene las dos WAN y un Loopback que simula Internet. No conoce ninguna red 10.23.56.x, así que al Jump Server solo se llega por la VPN.

Las running-config completas están en la [sección 10](#10-running-config-de-los-equipos).

---

## 4. FortiGate

### 4.1 Interfaces y ruta

| Puerto | Alias | IP | Acceso |
|---|---|---|---|
| port1 | WAN | 20.24.56.2/30 | PING |
| port2 | LAN-JUMP | 10.23.56.137/29 | PING |
| port3 | (gestión) | 192.168.128.141/24 | PING, HTTPS, SSH, HTTP |
| port4 | LAN-WEB | 10.23.56.129/29 | PING |

Ruta por defecto `0.0.0.0/0` hacia `20.24.56.1` por el port1.

![WAN port1](img/02-fg-port1-wan.png)
![LAN-WEB port4](img/03-fg-port4-lan-web.png)
![LAN-JUMP port2](img/04-fg-port2-lan-jump.png)
![Ping al Web Server](img/05-fg-ping-web-server.png)

### 4.2 Usuarios y grupos

| Usuario | Grupo |
|---|---|
| cperez | VPN-Basico |
| mrodriguez | VPN-Privilegiado |

![Usuarios](img/06-fg-usuarios.png)
![Grupos](img/07-fg-grupos.png)

### 4.3 VPN IPsec de acceso remoto (VPN-JUMP)

Se creó con **VPN > IPsec Wizard > Remote Access** y después se pasó a túnel personalizado para ajustarla:

| Parámetro | Valor |
|---|---|
| Tipo | Dial-up, IKEv1 modo agresivo |
| Autenticación | Pre-shared key + XAuth |
| Grupo XAuth | **Inherit from policy** (cada política decide qué grupo entra) |
| Phase 1 / Phase 2 | DES / MD5, DH grupo 5, PFS activado |
| Pool de IPs | 10.23.56.200 – 10.23.56.210 (/32) |
| Split tunnel | Solo `Jump-Server` (10.23.56.138/32) |

Con el split tunnel, al cliente solo se le instala la ruta hacia el Jump Server. Aunque los selectores de la Phase 2 son `0.0.0.0/0`, lo que puede pasar por el túnel lo controlan las políticas.

![Túnel VPN-JUMP](img/08-fg-tunel-vpn.png)
![Phase 2](img/09-fg-phase2.png)

### 4.4 Políticas

| # | Nombre | Origen → Destino | Grupo | Servicio | Acción |
|---|---|---|---|---|---|
| 1 | Deny-SSH-Basico | VPN-JUMP → Jump-Server | VPN-Basico | SSH | **DENY** (con log) |
| 2 | VPN-Basico-a-Jump | VPN-JUMP → Jump-Server | VPN-Basico | HTTPS | ACCEPT |
| 3 | VPN-Priv-a-Jump | VPN-JUMP → Jump-Server | VPN-Privilegiado | HTTPS, RDP | ACCEPT |
| 4 | Jump-a-Web | Jump-Server → Web-Server | — | HTTPS, RDP, SSH | ACCEPT |
| — | Implicit Deny | any → any | — | ALL | DENY (con log) |

- Ninguna política tiene NAT.
- La **Deny-SSH-Basico** está de primera para que el bloqueo del SSH del usuario sin privilegios sea explícito y quede registrado en el log.
- No hay ninguna política del Jump Server hacia Internet ni de la VPN hacia el Web Server. Eso lo bloquea la Implicit Deny.

![Políticas](img/10-fg-politicas.png)

---

## 5. Web Server

Ubuntu Server 20.04 con IP fija `10.23.56.130/29` y gateway `10.23.56.129`. Para instalar los paquetes se conectó un momento a Cloud0 y después se devolvió al port4.

| Servicio | Detalle |
|---|---|
| HTTPS | Apache con certificado autofirmado, página "Sistema de Caja" |
| SSH | OpenSSH, sin login de root |
| RDP | xrdp con escritorio XFCE (`allowed_users=anybody` en `/etc/X11/Xwrapper.config`) |

---

## 6. Jump Server

Windows Server 2022 con IP `10.23.56.138/29`. Durante la instalación tuvo una segunda tarjeta en Cloud0 para descargar lo necesario, y al final se quitó.

**Lo que se configuró:**

- **Active Directory y DNS:** dominio `itla.local` (NetBIOS `ITLA`). El DNS solo escucha en la `.138`.
- **Usuarios del dominio:** `cperez` en el grupo **RDS-Basico** y `mrodriguez` en **RDS-Privilegiado**.
- **Programas:** Firefox (para la web) y PuTTY. El cliente RDP (`mstsc`) ya viene con Windows.
- **Remote Desktop Services (Quick Start):** colección `QuickSessionCollection`, solo para los dos grupos.
- **RemoteApps:**

| RemoteApp | Programa | Grupos |
|---|---|---|
| Sistema de Caja (Web) | Firefox → `https://10.23.56.130` | RDS-Basico, RDS-Privilegiado |
| PuTTY (SSH) | putty.exe | RDS-Privilegiado |
| Escritorio Remoto (RDP) | mstsc.exe | RDS-Privilegiado |

- **Licencias:** modo **Por Usuario** (el Web Client no funciona por dispositivo).
- **Certificado:** autofirmado para `JUMP-SERVER.itla.local` / `10.23.56.138`, asignado a todos los roles de RDS.
- **RD Web Client (HTML5)** en `https://jump-server.itla.local/RDWeb/webclient`.
- **RD Gateway:** con el gateway todo el acceso del cliente va por **HTTPS (443)**, así que la política de la VPN no necesita más puertos.

![Red del Jump Server](img/11-jump-red.png)
![DNS de itla.local](img/12-jump-dns.png)
![Despliegue de RDS](img/13-jump-rds.png)
![RemoteApps por grupo y licencias](img/14-jump-remoteapps.png)
![RD Web Client publicado](img/15-jump-webclient.png)

---

## 7. Cliente (PC-Kali)

Kali en la VLAN 10, recibe IP por DHCP del R-USER (`10.23.56.11/25`). El cliente VPN es **strongSwan 6.0.1** con dos conexiones, una por usuario, que comparten la misma configuración base:

```
conn base
    keyexchange=ikev1
    aggressive=yes
    ike=des-md5-modp1536!
    esp=des-md5-modp1536!
    left=%defaultroute
    leftauth=psk
    leftauth2=xauth
    leftsourceip=%config
    right=20.24.56.2
    rightid=%any
    rightauth=psk
    rightsubnet=10.23.56.138/32
    auto=ignore

conn cperez
    also=base
    leftid=@cperez
    xauth_identity=cperez
    auto=add

conn mrodriguez
    also=base
    leftid=@mrodriguez
    xauth_identity=mrodriguez
    auto=add
```

---

## 8. Pruebas

### 8.1 Usuario sin privilegios (cperez)

La VPN conecta, recibe la `10.23.56.200` y el túnel solo lleva tráfico hacia el Jump Server (`10.23.56.200/32 === 10.23.56.138/32`).

![VPN de cperez](img/18-kali-vpn-cperez.png)

| Prueba | Resultado |
|---|---|
| HTTPS al Web Client | `200` |
| SSH al Jump Server (22) | timed out → **Deny-SSH-Basico** |
| RDP al Jump Server (3389) | timed out → Implicit Deny |

![Pruebas de cperez](img/19-kali-pruebas-cperez.png)

En el Web Client solo le aparece **Sistema de Caja (Web)**. Al abrirlo, el Firefox del Jump Server llega al Web Server por HTTPS (el aviso es por el certificado autofirmado del Apache).

![Web Client de cperez](img/16-webclient-cperez.png)
![Sistema de Caja desde cperez](img/20-caja-cperez.png)

El log del FortiGate muestra el bloqueo con la política **Deny-SSH-Basico (3)**. Debajo también se ve el Jump Server intentando salir a Internet y siendo bloqueado por la Implicit Deny.

![Log del bloqueo SSH](img/21-log-deny-ssh.png)

### 8.2 Usuario con privilegios (mrodriguez)

El 3389 abre y el 22 sigue cerrado, porque al Jump Server solo se entra por HTTPS y RDP.

![VPN de mrodriguez](img/22-kali-vpn-mrodriguez.png)

En el Web Client le aparecen las tres apps:

![Web Client de mrodriguez](img/17-webclient-mrodriguez.png)

| App | Destino | Resultado |
|---|---|---|
| Sistema de Caja (Web) | https://10.23.56.130 | Carga la página |
| PuTTY (SSH) | angel@10.23.56.130 | Entra al Web Server |
| Escritorio Remoto (RDP) | 10.23.56.130 | Escritorio XFCE del Web Server |

![Sistema de Caja](img/23-caja-mrodriguez.png)
![PuTTY al Web Server](img/24-putty-web-server.png)
![RDP al Web Server](img/25-rdp-web-server.png)

### 8.3 Túnel en el FortiGate

![Monitor IPsec](img/26-ipsec-monitor.png)

---

## 9. Problemas que salieron y cómo se resolvieron

| Problema | Causa | Solución |
|---|---|---|
| El cliente no completaba la VPN (`calculated HASH does not match`) | La PSK no era igual en el Kali y en el FortiGate | Se volvió a escribir la PSK en los dos lados |
| strongSwan no podía mandar usuario y clave por XAuth | La versión 6.1 de Kali ya no trae el plugin `xauth-generic` | Se instaló la 6.0.1 de Debian trixie y se dejó en `apt-mark hold` |
| El Web Client mostraba las apps, pero se desconectaba al abrirlas | Sin RD Gateway, la sesión usa el puerto 3392 y fallaba la validación del certificado | Se agregó el rol **RD Gateway**; todo pasa por HTTPS 443 |
| Después de reiniciar, el Web Client decía "No pudimos conectar" | El **Connection Broker** (Tssdis) no arrancaba con el sistema, por ser también controlador de dominio | Se inició y se dejó con arranque retrasado junto con la base WID |
| El RDP de Administrator era rechazado (0x3 / 0x7) | Administrator no está en los grupos de la colección | Para administrar se entra con `mstsc /admin` |
| xrdp volvía al login después de poner la clave | Ubuntu solo deja abrir Xorg desde la consola física | `allowed_users=anybody` en `Xwrapper.config` |
| Firefox de mrodriguez daba "La conexión ha caducado" | Se intentó por HTTP (80), que no está permitido | Se entra con `https://` (el 80 bloqueado demuestra que la política funciona) |

---

## 10. Running-config de los equipos

> Las claves aparecen como `ENC <oculto>` o `<PASSWORD>`.

<details>
<summary><b>ISP</b></summary>

```
ISP#show running-config
Building configuration...

Current configuration : 1302 bytes
!
version 15.4
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
!
hostname ISP
!
boot-start-marker
boot-end-marker
!
no aaa new-model
mmi polling-interval 60
no mmi auto-configure
no mmi pvc
mmi snmp-timeout 180
!
no ip domain lookup
ip cef
no ipv6 cef
!
multilink bundle-name authenticated
!
redundancy
!
interface Loopback0
 description Simula Internet
 ip address 8.8.8.8 255.255.255.255
!
interface Ethernet0/0
 description Enlace hacia R-USER e0/0
 ip address 20.24.23.1 255.255.255.252
!
interface Ethernet0/1
 description Enlace hacia FG-Server port1
 ip address 20.24.56.1 255.255.255.252
!
interface Ethernet0/2
 no ip address
 shutdown
!
interface Ethernet0/3
 no ip address
 shutdown
!
interface Ethernet1/0
 no ip address
 shutdown
!
interface Ethernet1/1
 no ip address
 shutdown
!
interface Ethernet1/2
 no ip address
 shutdown
!
interface Ethernet1/3
 no ip address
 shutdown
!
ip forward-protocol nd
!
no ip http server
no ip http secure-server
!
control-plane
!
banner motd ^CISP - Lab Jump Server - Matricula 2024-2356^C
!
line con 0
 exec-timeout 0 0
 logging synchronous
line aux 0
line vty 0 4
 login
 transport input none
!
end

ISP#show ip interface brief
Interface                  IP-Address      OK? Method Status                Protocol
Ethernet0/0                20.24.23.1      YES NVRAM  up                    up
Ethernet0/1                20.24.56.1      YES NVRAM  up                    up
Ethernet0/2                unassigned      YES NVRAM  administratively down down
Ethernet0/3                unassigned      YES NVRAM  administratively down down
Ethernet1/0                unassigned      YES NVRAM  administratively down down
Ethernet1/1                unassigned      YES NVRAM  administratively down down
Ethernet1/2                unassigned      YES NVRAM  administratively down down
Ethernet1/3                unassigned      YES NVRAM  administratively down down
Loopback0                  8.8.8.8         YES NVRAM  up                    up

ISP#show ip route
Gateway of last resort is not set

      8.0.0.0/32 is subnetted, 1 subnets
C        8.8.8.8 is directly connected, Loopback0
      20.0.0.0/8 is variably subnetted, 4 subnets, 2 masks
C        20.24.23.0/30 is directly connected, Ethernet0/0
L        20.24.23.1/32 is directly connected, Ethernet0/0
C        20.24.56.0/30 is directly connected, Ethernet0/1
L        20.24.56.1/32 is directly connected, Ethernet0/1
```
</details>

<details>
<summary><b>SW-Usuarios</b></summary>

```
SW-Usuarios#show running-config
Building configuration...

Current configuration : 1144 bytes
!
! Last configuration change at 13:14:06 UTC Thu Oct 8 2026
!
version 15.2
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
service compress-config
!
hostname SW-Usuarios
!
boot-start-marker
boot-end-marker
!
no aaa new-model
!
no ip domain-lookup
ip cef
no ipv6 cef
!
spanning-tree mode rapid-pvst
spanning-tree extend system-id
!
vlan internal allocation policy ascending
!
interface Ethernet0/0
 description Trunk hacia R-USER e0/1
 switchport trunk allowed vlan 10
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
interface Ethernet0/1
 description Access hacia PC-Kali
 switchport access vlan 10
 switchport mode access
 spanning-tree portfast edge
!
interface Ethernet0/2
 description No usado
 shutdown
!
interface Ethernet0/3
 description No usado
 shutdown
!
ip forward-protocol nd
!
no ip http server
no ip http secure-server
!
control-plane
!
banner motd ^CSW-Usuarios - Lab Jump Server - Matricula 2024-2356^C
!
line con 0
 exec-timeout 0 0
 logging synchronous
line aux 0
line vty 0 4
 login
!
end

SW-Usuarios#show vlan brief

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Et0/2, Et0/3
10   USUARIOS                         active    Et0/1
1002 fddi-default                     act/unsup
1003 token-ring-default               act/unsup
1004 fddinet-default                  act/unsup
1005 trnet-default                    act/unsup

SW-Usuarios#show interfaces trunk

Port        Mode             Encapsulation  Status        Native vlan
Et0/0       on               802.1q         trunking      1

Port        Vlans allowed on trunk
Et0/0       10

Port        Vlans allowed and active in management domain
Et0/0       10

Port        Vlans in spanning tree forwarding state and not pruned
Et0/0       10
```
</details>

<details>
<summary><b>R-USER</b></summary>

```
R-USER#show running-config
Building configuration...

Current configuration : 1623 bytes
!
version 15.4
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
!
hostname R-USER
!
boot-start-marker
boot-end-marker
!
no aaa new-model
mmi polling-interval 60
no mmi auto-configure
no mmi pvc
mmi snmp-timeout 180
!
ip dhcp excluded-address 10.23.56.1 10.23.56.9
ip dhcp excluded-address 10.23.56.101 10.23.56.127
!
ip dhcp pool VLAN10-USUARIOS
 network 10.23.56.0 255.255.255.128
 default-router 10.23.56.1
 dns-server 8.8.8.8
!
no ip domain lookup
ip cef
no ipv6 cef
!
multilink bundle-name authenticated
!
redundancy
!
interface Ethernet0/0
 description WAN hacia ISP
 ip address 20.24.23.2 255.255.255.252
 ip nat outside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
!
interface Ethernet0/1
 description Trunk hacia SW-Usuarios
 no ip address
!
interface Ethernet0/1.10
 description VLAN 10 - Usuarios
 encapsulation dot1Q 10
 ip address 10.23.56.1 255.255.255.128
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
!
interface Ethernet0/2
 no ip address
 shutdown
!
interface Ethernet0/3
 no ip address
 shutdown
!
ip forward-protocol nd
!
no ip http server
no ip http secure-server
ip nat inside source list NAT-LAN interface Ethernet0/0 overload
ip route 0.0.0.0 0.0.0.0 20.24.23.1
!
ip access-list extended NAT-LAN
 permit ip 10.23.56.0 0.0.0.127 any
!
control-plane
!
banner motd ^CR-USER - Lab Jump Server - Matricula 2024-2356^C
!
line con 0
 exec-timeout 0 0
 logging synchronous
line aux 0
line vty 0 4
 login
 transport input none
!
end

R-USER#show ip interface brief
Interface                  IP-Address      OK? Method Status                Protocol
Ethernet0/0                20.24.23.2      YES NVRAM  up                    up
Ethernet0/1                unassigned      YES NVRAM  up                    up
Ethernet0/1.10             10.23.56.1      YES NVRAM  up                    up
Ethernet0/2                unassigned      YES NVRAM  administratively down down
Ethernet0/3                unassigned      YES NVRAM  administratively down down
NVI0                       20.24.23.2      YES unset  up                    up
R-USER#show ip route
Gateway of last resort is 20.24.23.1 to network 0.0.0.0

S*    0.0.0.0/0 [1/0] via 20.24.23.1
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.23.56.0/25 is directly connected, Ethernet0/1.10
L        10.23.56.1/32 is directly connected, Ethernet0/1.10
      20.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        20.24.23.0/30 is directly connected, Ethernet0/0
L        20.24.23.2/32 is directly connected, Ethernet0/0
R-USER#show ip dhcp binding
Bindings from all pools not associated with VRF:
IP address          Client-ID/              Lease expiration        Type
                    Hardware address/
                    User name
10.23.56.10         0150.5f10.0047.00       Oct 09 2026 01:15 PM    Automatic
10.23.56.11         0150.57d9.0047.00       Oct 09 2026 03:15 PM    Automatic

R-USER#show ip nat translations
Pro Inside global      Inside local       Outside local      Outside global
udp 20.24.23.2:4500    10.23.56.11:4500   20.24.56.2:4500    20.24.56.2:4500
udp 20.24.23.2:34011   10.23.56.11:34011  208.91.112.53:53   208.91.112.53:53
(... consultas DNS del cliente omitidas)
```
</details>

<details>
<summary><b>FG-Server (FortiGate)</b></summary>

```
FG-Server # show system global
config system global
    set alias "FortiGate-VM64-KVM"
    set hostname "FG-Server"
    set timezone 77
end

FG-Server # show system interface
config system interface
    edit "port1"
        set vdom "root"
        set ip 20.24.56.2 255.255.255.252
        set allowaccess ping
        set type physical
        set alias "WAN"
        set lldp-reception enable
        set role wan
        set snmp-index 1
    next
    edit "port2"
        set vdom "root"
        set ip 10.23.56.137 255.255.255.248
        set allowaccess ping
        set type physical
        set alias "LAN-JUMP"
        set device-identification enable
        set lldp-transmission enable
        set role lan
        set snmp-index 2
    next
    edit "port3"
        set vdom "root"
        set ip 192.168.128.141 255.255.255.0
        set allowaccess ping https ssh http
        set type physical
        set snmp-index 3
    next
    edit "port4"
        set vdom "root"
        set ip 10.23.56.129 255.255.255.248
        set allowaccess ping
        set type physical
        set alias "LAN-WEB"
        set device-identification enable
        set lldp-transmission enable
        set role lan
        set snmp-index 4
    next
    edit "ssl.root"
        set vdom "root"
        set type tunnel
        set alias "SSL VPN interface"
        set snmp-index 5
    next
    edit "fortilink"
        set vdom "root"
        set fortilink enable
        set ip 169.254.1.1 255.255.255.0
        set allowaccess ping fabric
        set type aggregate
        set lldp-reception enable
        set lldp-transmission enable
        set snmp-index 6
    next
    edit "VPN-JUMP"
        set vdom "root"
        set type tunnel
        set snmp-index 7
        set interface "port1"
    next
end

FG-Server # show router static
config router static
    edit 1
        set gateway 20.24.56.1
        set device "port1"
    next
end

FG-Server # show firewall address
config firewall address
    edit "none"
        set uuid 7f0f9ef2-c275-51f1-faeb-61089f35edad
        set subnet 0.0.0.0 255.255.255.255
    next
    edit "login.microsoftonline.com"
        set uuid 7f0fa33e-c275-51f1-efa7-95e7fa861576
        set type fqdn
        set fqdn "login.microsoftonline.com"
    next
    edit "login.microsoft.com"
        set uuid 7f0fa6d6-c275-51f1-cb6b-9eb9048995e6
        set type fqdn
        set fqdn "login.microsoft.com"
    next
    edit "login.windows.net"
        set uuid 7f0fa99c-c275-51f1-3a8c-2286b0329b73
        set type fqdn
        set fqdn "login.windows.net"
    next
    edit "gmail.com"
        set uuid 7f0fac4e-c275-51f1-5cfe-a3b125737cab
        set type fqdn
        set fqdn "gmail.com"
    next
    edit "wildcard.google.com"
        set uuid 7f0faef6-c275-51f1-0224-250cdccfc537
        set type fqdn
        set fqdn "*.google.com"
    next
    edit "wildcard.dropbox.com"
        set uuid 7f0fb540-c275-51f1-5dcc-62c5b9d140cd
        set type fqdn
        set fqdn "*.dropbox.com"
    next
    edit "all"
        set uuid 7f11bd90-c275-51f1-7246-ae79afec1dba
    next
    edit "FIREWALL_AUTH_PORTAL_ADDRESS"
        set uuid 7f11bf02-c275-51f1-c599-d1c377240aa2
    next
    edit "FABRIC_DEVICE"
        set uuid 7f11c02e-c275-51f1-89cb-9e448a7b2743
        set comment "IPv4 addresses of Fabric Devices."
    next
    edit "SSLVPN_TUNNEL_ADDR1"
        set uuid 7f122456-c275-51f1-b039-9ee4c7b79c80
        set type iprange
        set associated-interface "ssl.root"
        set start-ip 10.212.134.200
        set end-ip 10.212.134.210
    next
    edit "port4 address"
        set uuid 799be2a0-c2a7-51f1-74a8-35d199bd78e9
        set type interface-subnet
        set subnet 10.23.56.129 255.255.255.248
        set interface "port4"
    next
    edit "Jump-Server"
        set uuid bf6d3246-c32b-51f1-d934-62ae5fbbcda4
        set subnet 10.23.56.138 255.255.255.255
    next
    edit "Web-Server"
        set uuid ca6319f4-c32b-51f1-bb30-cda983557662
        set subnet 10.23.56.130 255.255.255.255
    next
    edit "VPN-JUMP_range"
        set uuid 9149ec1e-c32c-51f1-23fc-873cef99bf20
        set type iprange
        set comment "VPN: VPN-JUMP (Created by VPN wizard)"
        set start-ip 10.23.56.200
        set end-ip 10.23.56.210
    next
end

FG-Server # show firewall addrgrp
config firewall addrgrp
    edit "G Suite"
        set uuid 7f0fbc34-c275-51f1-92fd-a5853f7d4bfd
        set member "gmail.com" "wildcard.google.com"
    next
    edit "Microsoft Office 365"
        set uuid 7f0fc242-c275-51f1-b4eb-b2a5b83326d8
        set member "login.microsoftonline.com" "login.microsoft.com" "login.windows.net"
    next
    edit "VPN-JUMP_split"
        set uuid 91434008-c32c-51f1-7e15-f30034191a4d
        set member "Jump-Server"
        set comment "VPN: VPN-JUMP (Created by VPN wizard)"
    next
end

FG-Server # show user local
config user local
    edit "guest"
        set type password
        set passwd ENC <oculto>
    next
    edit "cperez"
        set type password
        set passwd-time 2026-10-08 11:21:55
        set passwd ENC <oculto>
    next
    edit "mrodriguez"
        set type password
        set passwd-time 2026-10-08 11:22:12
        set passwd ENC <oculto>
    next
end

FG-Server # show user group
config user group
    edit "SSO_Guest_Users"
    next
    edit "Guest-group"
        set member "guest"
    next
    edit "VPN-Basico"
        set member "cperez"
    next
    edit "VPN-Privilegiado"
        set member "mrodriguez"
    next
end

FG-Server # show vpn ipsec phase1-interface
config vpn ipsec phase1-interface
    edit "VPN-JUMP"
        set type dynamic
        set interface "port1"
        set mode aggressive
        set peertype any
        set net-device disable
        set mode-cfg enable
        set proposal des-md5
        set comments "VPN: VPN-JUMP (Created by VPN wizard)"
        set dhgrp 5
        set xauthtype auto
        set ipv4-start-ip 10.23.56.200
        set ipv4-end-ip 10.23.56.210
        set dns-mode auto
        set ipv4-split-include "VPN-JUMP_split"
        set psksecret ENC <oculto>
    next
end

FG-Server # show vpn ipsec phase2-interface
config vpn ipsec phase2-interface
    edit "VPN-JUMP"
        set phase1name "VPN-JUMP"
        set proposal des-md5
        set dhgrp 5
        set keepalive enable
        set comments "VPN: VPN-JUMP (Created by VPN wizard)"
    next
end

FG-Server # show firewall policy
config firewall policy
    edit 3
        set name "Deny-SSH-Basico"
        set uuid 6ee766f4-c32e-51f1-e63a-01508e665658
        set srcintf "VPN-JUMP"
        set dstintf "port2"
        set srcaddr "VPN-JUMP_range"
        set dstaddr "Jump-Server"
        set schedule "always"
        set service "SSH"
        set logtraffic all
        set groups "VPN-Basico"
        set comments "VPN: VPN-JUMP (Created by VPN wizard) (Copy of VPN-Priv-a-Jump)"
    next
    edit 4
        set name "Jump-a-Web"
        set uuid ba5f8c1a-c32e-51f1-53d9-732a27fd64a3
        set srcintf "port2"
        set dstintf "port4"
        set srcaddr "Jump-Server"
        set dstaddr "Web-Server"
        set action accept
        set schedule "always"
        set service "RDP" "SSH" "HTTPS"
        set logtraffic all
        set comments "VPN: VPN-JUMP (Created by VPN wizard) (Copy of VPN-Priv-a-Jump) (Copy of Deny-SSH-Basico)"
    next
    edit 2
        set name "VPN-Basico-a-Jump"
        set uuid ab1e1a10-c32d-51f1-778a-32cf8a06fd9f
        set srcintf "VPN-JUMP"
        set dstintf "port2"
        set srcaddr "VPN-JUMP_range"
        set dstaddr "Jump-Server"
        set action accept
        set schedule "always"
        set service "HTTPS"
        set logtraffic all
        set groups "VPN-Basico"
        set comments "VPN: VPN-JUMP (Created by VPN wizard) (Copy of VPN-Priv-a-Jump) (Reverse of VPN-Priv-a-Jump)"
    next
    edit 1
        set name "VPN-Priv-a-Jump"
        set uuid 914b2c00-c32c-51f1-fde7-87f63d7f9f91
        set srcintf "VPN-JUMP"
        set dstintf "port2"
        set srcaddr "VPN-JUMP_range"
        set dstaddr "Jump-Server"
        set action accept
        set schedule "always"
        set service "HTTPS" "RDP"
        set logtraffic all
        set groups "VPN-Privilegiado"
        set comments "VPN: VPN-JUMP (Created by VPN wizard)"
    next
end
```
</details>

<details>
<summary><b>Web Server, Jump Server y PC-Kali</b></summary>

Los comandos completos de estos tres equipos (netplan, Apache, xrdp, PowerShell del Jump Server y strongSwan) están en el [TXT de configuraciones](AngelAlcantara_20242356_P3_i2.txt).
</details>
