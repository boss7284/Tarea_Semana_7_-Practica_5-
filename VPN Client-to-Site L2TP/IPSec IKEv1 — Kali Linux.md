# VPN Client-to-Site L2TP/IPSec IKEv1 — Kali Linux

**Asignatura:** Seguridad en Redes

**Estudiante:** Roberto de Jesus

**Matrícula:** 2023-0348

**Profesor:** Jonathan Esteban Rondón

**Fecha:** 2025

**Link del video:** [https://youtu.be/N-pc-nD8Tuc](https://youtu.be/N-pc-nD8Tuc)]

---

## Tabla de Contenidos
- [Descripción General](#-descripción-general)
- [Topología del Laboratorio](#-topología-del-laboratorio)
- [Parámetros](#-parámetros)
- [Configuración por Dispositivo](#-configuración-por-dispositivo)
- [Configuración del Cliente Kali Linux](#-configuración-del-cliente-kali-linux)
- [Verificación](#-verificación)
- [Flujo de Conexión](#-flujo-de-conexión)

---

##  Descripción General

Este laboratorio implementa una **VPN Client-to-Site con L2TP/IPSec IKEv1**. Un cliente **Kali Linux** se conecta remotamente a la red corporativa a través de Internet (ISP). L2TP proporciona el túnel de capa 2 y autentica al usuario con **MS-CHAPv2**. IPSec cifra el canal L2TP usando **IKEv1 con PSK**. El servidor Cisco actúa como **LNS (L2TP Network Server)** y asigna al cliente una IP del pool corporativo.

### ¿Cómo funciona?

1. Kali inicia **IKE Fase 1** con el LNS (IKEv1, UDP 500)
2. Se establece **IPSec SA en modo Transport** (no Tunnel — L2TP ya encapsula)
3. Kali inicia **sesión L2TP** sobre UDP 1701 (cifrado por IPSec)
4. Se negocia **PPP con MS-CHAPv2** (autenticación usuario/contraseña)
5. El LNS asigna una IP del pool `192.168.200.x` a la interfaz `ppp0` de Kali
6. Kali puede acceder a la red `10.10.1.0/24`

> ** ADVERTENCIA:** Estas configuraciones están diseñadas **EXCLUSIVAMENTE** para fines educativos en entornos simulados como **GNS3**.

---

##  Topología del Laboratorio

![image alt](https://github.com/boss7284/d.u.m.p2/blob/a38f651cd85e20c324666e7d4cd853c60b898eab/imagen%20gns3%20finall2xtp.jpeg)
### Diagrama de Red

```
+------------------+     +----------+     +------------------+     +----------+     +----------+
|  Kali Linux      |     | IOU-ISP  |     |  IOU1 LNS        |     | IOU2 SW  |     |   VPCS   |
|  Cliente VPN     +-----+ Router   +-----+  Servidor L2TP   +-----+          +-----+   PC     |
|  IP: 200.0.0.6   |     | ISP      |     |  WAN: 200.0.0.2  |     | LAN      |     |10.10.1.100
|  VPN: 192.168    |     |          |     |  LAN: 10.10.1.1  |     |          |     |          |
|  .200.x (ppp0)   |     |          |     |  Pool: 192.168   |     |          |     |          |
+------------------+     +----------+     |  .200.10-.50     |     +----------+     +----------+
                                          +------------------+
```

### Configuración de Red

| Dispositivo | Interfaz | IP | Descripción |
|---|---|---|---|
| Kali Linux | eth0 | 200.0.0.6/30 | WAN hacia ISP |
| Kali Linux | ppp0 | 192.168.200.x | IP asignada por VPN |
| IOU-ISP | e0/0 | 200.0.0.5/30 | hacia Kali |
| IOU-ISP | e0/1 | 200.0.0.1/30 | hacia LNS |
| IOU1 LNS | e0/0 | 200.0.0.2/30 | WAN |
| IOU1 LNS | e0/1 | 10.10.1.1/24 | LAN corporativa |
| IOU2 SW | — | sin IP | Switch LAN |
| VPCS PC | e0 | 10.10.1.100/24 | GW 10.10.1.1 |
| Pool VPN | — | 192.168.200.10 – 192.168.200.50 | IPs para clientes |

---

##  Parámetros

| Parámetro | Valor |
|---|---|
| Versión IKE | IKEv1 |
| Autenticación IPSec | Pre-Shared Key |
| Clave PSK | `IPSecL2TP` |
| Cifrado IKEv1 | AES-256 |
| Hash IKEv1 | SHA |
| DH Group | Group 2 |
| Modo IPSec | **Transport** (L2TP encapsula) |
| Protocolo Túnel | L2TP (UDP 1701) |
| Autenticación PPP | MS-CHAPv2 |
| Usuario VPN | `vpnuser` |
| Contraseña VPN | `VpnPass123` |
| Pool IP clientes | 192.168.200.10 – 192.168.200.50 |
| Virtual-Template | Virtual-Template1 |

---

##  Configuración por Dispositivo

### VPCS — PC de la LAN corporativa

```
ip 10.10.1.100 255.255.255.0 10.10.1.1
save
```

### IOU2 — Switch LAN

```cisco
hostname SW-LAN

interface e0/0
 switchport mode access
 switchport access vlan 1
 no shutdown

interface e0/1
 switchport mode access
 switchport access vlan 1
 no shutdown
```

### IOU-ISP — Router ISP

```cisco
hostname R-ISP

! ── Interfaz hacia Kali ─────────────────────────────────────
interface e0/0
 description hacia-Kali
 ip address 200.0.0.5 255.255.255.252
 no shutdown

! ── Interfaz hacia LNS ──────────────────────────────────────
interface e0/1
 description hacia-LNS
 ip address 200.0.0.1 255.255.255.252
 no shutdown

! ── Rutas estáticas ─────────────────────────────────────────
ip route 10.10.1.0 255.255.255.0 200.0.0.2
ip route 192.168.200.0 255.255.255.0 200.0.0.2
```

### IOU1 — Servidor L2TP/IPSec LNS

```cisco
hostname R1-LNS

! ════════════════════════════════════════════════════════════
!  INTERFACES
! ════════════════════════════════════════════════════════════
interface e0/0
 description WAN-ISP
 ip address 200.0.0.2 255.255.255.252
 no shutdown

interface e0/1
 description LAN-Corporativa
 ip address 10.10.1.1 255.255.255.0
 no shutdown

! ════════════════════════════════════════════════════════════
!  RUTA DEFAULT
! ════════════════════════════════════════════════════════════
ip route 0.0.0.0 0.0.0.0 200.0.0.1

! ════════════════════════════════════════════════════════════
!  AAA — autenticación de usuarios VPN
! ════════════════════════════════════════════════════════════
aaa new-model
aaa authentication ppp VPN-AUTH local
aaa authorization network VPN-AUTH local

username vpnuser password VpnPass123

! ════════════════════════════════════════════════════════════
!  POOL DE IPs para clientes VPN
! ════════════════════════════════════════════════════════════
ip local pool POOL-VPN 192.168.200.10 192.168.200.50

! ════════════════════════════════════════════════════════════
!  IKEv1 — protege el canal L2TP
!  Crypto map dinámico: acepta cualquier cliente
! ════════════════════════════════════════════════════════════
crypto isakmp policy 10
 encr aes 256
 hash sha
 authentication pre-share
 group 2
 lifetime 86400

crypto isakmp key IPSecL2TP address 0.0.0.0 0.0.0.0

crypto isakmp nat keepalive 20

! ════════════════════════════════════════════════════════════
!  IPSEC — modo TRANSPORT para L2TP
!  (L2TP encapsula, no se necesita modo tunnel)
! ════════════════════════════════════════════════════════════
crypto ipsec transform-set TS-L2TP esp-aes 256 esp-sha-hmac
 mode transport

crypto dynamic-map DMAP-L2TP 10
 set transform-set TS-L2TP

crypto map CMAP-L2TP 65535 ipsec-isakmp dynamic DMAP-L2TP

interface e0/0
 crypto map CMAP-L2TP

! ════════════════════════════════════════════════════════════
!  VPDN — habilita L2TP
! ════════════════════════════════════════════════════════════
vpdn enable

vpdn-group L2TP-SERVER
 accept-dialin
  protocol l2tp
  virtual-template 1
 no l2tp tunnel authentication

! ════════════════════════════════════════════════════════════
!  VIRTUAL-TEMPLATE — interfaz creada por cada cliente
!  ip unnumbered = usa la IP de e0/1 como local
!  peer default ip address pool = asigna IP del pool
! ════════════════════════════════════════════════════════════
interface Virtual-Template1
 ip unnumbered e0/1
 peer default ip address pool POOL-VPN
 ppp authentication ms-chap-v2 VPN-AUTH
 ppp authorization VPN-AUTH
 no shutdown
```

---

##  Configuración del Cliente Kali Linux

### Paso 1 — Asignar IP a Kali

```bash
sudo ip addr add 200.0.0.6/30 dev eth0
sudo ip link set eth0 up
sudo ip route add default via 200.0.0.5
```

Verificar:
```bash
ip addr show eth0
ip route show
```

### Paso 2 — Instalar paquetes

```bash
sudo apt update
sudo apt install strongswan xl2tpd -y
```

### Paso 3 — Configurar IPSec

```bash
sudo nano /etc/ipsec.conf
```

```ini
config setup
    charondebug="ike 1, knl 1, cfg 0"

conn L2TP-PSK
    authby=secret
    auto=add
    keyexchange=ikev1
    type=transport
    left=%defaultroute
    leftprotoport=17/1701
    right=200.0.0.2
    rightprotoport=17/1701
    ike=aes256-sha1-modp1024!
    esp=aes256-sha1!
```

### Paso 4 — Configurar PSK

```bash
sudo nano /etc/ipsec.secrets
```

```
: PSK "IPSecL2TP"
```

### Paso 5 — Configurar L2TP

```bash
sudo nano /etc/xl2tpd/xl2tpd.conf
```

```ini
[lac vpn-corporativa]
lns = 200.0.0.2
ppp debug = yes
pppoptfile = /etc/ppp/options.l2tpd.client
length bit = yes
```

### Paso 6 — Configurar PPP

```bash
sudo nano /etc/ppp/options.l2tpd.client
```

```
ipcp-accept-local
ipcp-accept-remote
refuse-eap
require-mschap-v2
noccp
noauth
mtu 1280
mru 1280
noipdefault
defaultroute
usepeerdns
connect-delay 5000
name vpnuser
password VpnPass123
```

### Paso 7 — Conectar

```bash
# Iniciar servicios
sudo ipsec restart
sudo systemctl restart xl2tpd

# Levantar tunnel IPSec
sudo ipsec up L2TP-PSK

# Levantar tunnel L2TP
echo "c vpn-corporativa" | sudo tee /var/run/xl2tpd/l2tp-control

# Esperar 3 segundos
sleep 3

# Ver interfaz ppp0 con IP asignada
ip addr show ppp0
```

---

##  Verificación

### En Kali — verificar conexión IPSec
```bash
sudo ipsec up L2TP-PSK
```
Salida esperada:
```
IKE_SA L2TP-PSK[1] established between 200.0.0.6 ... 200.0.0.2
CHILD_SA L2TP-PSK established with SPIs ...
connection 'L2TP-PSK' established successfully 
```

### En Kali — verificar interfaz ppp0
```bash
ip addr show ppp0
```
Salida esperada:
```
ppp0: <POINTOPOINT,MULTICAST,NOARP,UP> mtu 1280
    inet 192.168.200.10 peer 10.10.1.1/32
```

### En Kali — ping a la LAN corporativa
```bash
ping 10.10.1.100 -c 5
```

### En el router LNS — verificar SA IKEv1
```cisco
show crypto isakmp sa
```
Salida esperada:
```
dst             src             state     conn-id status
200.0.0.2       200.0.0.6       QM_IDLE      1001 ACTIVE
```

### En el router LNS — verificar paquetes cifrados
```cisco
show crypto ipsec sa | include encaps|decaps
```

### En el router LNS — verificar pool usado
```cisco
show ip local pool POOL-VPN
```
Salida esperada:
```
Pool           Begin           End             Free  In use
POOL-VPN       192.168.200.10  192.168.200.50  40    1
```

### En el router LNS — verificar interfaz virtual creada
```cisco
show interfaces Virtual-Access1
```

---

##  Flujo de Conexión

```
Kali                                    R1-LNS
  │                                         │
  ├──── IKE_SA_INIT (UDP 500) ─────────────>│  Negociar AES-256 SHA DH2
  │<─── IKE Resp ───────────────────────────│
  │                                         │
  ├──── IKE_AUTH PSK "IPSecL2TP" ──────────>│  Autenticar con PSK
  │<─── IPSec SA Transport ─────────────────│  Canal cifrado ✅
  │                                         │
  ├──── L2TP SCCRQ (UDP 1701 cifrado) ─────>│  Iniciar sesión L2TP
  │<─── L2TP SCCCN ─────────────────────────│
  │                                         │
  ├──── PPP LCP ────────────────────────────>│  Negociar PPP
  ├──── PPP MS-CHAPv2 (vpnuser/VpnPass123) ->│  Autenticar usuario
  │<─── IP: 192.168.200.10 ─────────────────│  IP asignada del pool ✅
  │                                         │
  └═════ Acceso a 10.10.1.0/24 ════════════╝
```

---

##  Requisitos

- GNS3
- Cisco IOU L3 (imagen 15.4.1T o superior)
- Cisco IOU L2 (para switches)
- VPCS
- Kali Linux QEMU
- strongswan
- xl2tpd

---

> ** ADVERTENCIA:** Estas configuraciones están diseñadas **EXCLUSIVAMENTE** para fines educativos en entornos simulados como **GNS3**.

---

*Autor: Roberto de Jesus — Matrícula: 2023-0348*
*Fecha: 2025*
