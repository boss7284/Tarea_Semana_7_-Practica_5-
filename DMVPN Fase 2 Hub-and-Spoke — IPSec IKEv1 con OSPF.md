# DMVPN Fase 2 Hub-and-Spoke — IPSec IKEv1 con OSPF

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
- [Verificación](#-verificación)
- [Diferencia Fase 2 vs Fase 3](#-diferencia-fase-2-vs-fase-3)

---

##  Descripción General

Este laboratorio implementa una **VPN DMVPN (Dynamic Multipoint VPN) Fase 2** con **1 Hub y 2 Spokes**, usando **IPSec IKEv1** en modo transport y **OSPF** como protocolo de enrutamiento. DMVPN combina **mGRE** (multipoint GRE), **NHRP** (Next Hop Resolution Protocol) e **IPSec** para crear una arquitectura Hub-and-Spoke escalable.

### Características de DMVPN Fase 2
- Los Spokes se **registran dinámicamente** en el Hub via NHRP
- El tráfico Spoke-a-Spoke crea **túneles directos** (sin pasar por el Hub)
- El Hub **resuelve la IP NBMA** del Spoke destino y la comunica al Spoke origen
- IPSec en **modo transport** (GRE ya encapsula — no se necesita doble encapsulamiento)
- OSPF `point-to-multipoint` sobre el túnel mGRE

> ** NOTA IMPORTANTE:** Usar `ip mtu 1400` + `ip ospf mtu-ignore` en Tunnel0 para evitar problemas de MTU con OSPF en IOU.

> ** ADVERTENCIA:** Estas configuraciones están diseñadas **EXCLUSIVAMENTE** para fines educativos en entornos simulados como **GNS3**.

---

##  Topología del Laboratorio

![image alt](https://github.com/boss7284/d.u.m.p2/blob/dfef801587d2544f41c55d284f5cf5c5c95251a0/Screenshot%202026-03-05%20000430.png)

### Diagrama de Red

```
                         +----------+
                         | IOU6 HUB |
                         | 200.0.0.2|
                         | 10.0.0.1 |
                         +----+-----+
                              | e0/0
                         +----+-----+
                         | IOU5 ISP |
                         +----+-----+
               e0/0 (200.0.0.5) /   \ (200.0.0.9) e0/1
                                /     \
                    +----------+       +----------+
                    | IOU1     |       | IOU2     |
                    | SPOKE1   |       | SPOKE2   |
                    | 200.0.0.6|       |200.0.0.10|
                    | 10.0.0.2 |       | 10.0.0.3 |
                    +----+-----+       +-----+----+
                         | e0/1             | e0/1
                    +----+-----+       +-----+----+
                    | IOU3 SW  |       | IOU4 SW  |
                    +----+-----+       +-----+----+
                         |                   |
                        PC1                 PC2
                   10.10.2.100         10.10.3.100

Subred DMVPN (mGRE): 10.0.0.0/24
ISP hacia HUB: e0/2 = 200.0.0.1/30
```

### Configuración de Red

| Dispositivo | Interfaz | IP | Descripción |
|---|---|---|---|
| IOU6 HUB | e0/0 | 200.0.0.2/30 | WAN hacia ISP |
| IOU6 HUB | Tunnel0 | 10.0.0.1/24 | mGRE DMVPN |
| IOU5 ISP | e0/2 | 200.0.0.1/30 | hacia HUB |
| IOU5 ISP | e0/0 | 200.0.0.5/30 | hacia SPOKE1 |
| IOU5 ISP | e0/1 | 200.0.0.9/30 | hacia SPOKE2 |
| IOU1 SPOKE1 | e0/0 | 200.0.0.6/30 | WAN |
| IOU1 SPOKE1 | e0/1 | 10.10.2.1/24 | LAN |
| IOU1 SPOKE1 | Tunnel0 | 10.0.0.2/24 | mGRE |
| IOU2 SPOKE2 | e0/0 | 200.0.0.10/30 | WAN |
| IOU2 SPOKE2 | e0/1 | 10.10.3.1/24 | LAN |
| IOU2 SPOKE2 | Tunnel0 | 10.0.0.3/24 | mGRE |
| PC1 | e0 | 10.10.2.100/24 | GW 10.10.2.1 |
| PC2 | e0 | 10.10.3.100/24 | GW 10.10.3.1 |

---

##  Parámetros

| Parámetro | Valor |
|---|---|
| Versión IKE | IKEv1 |
| PSK | `Cisco12345` (wildcard — cualquier peer) |
| Cifrado IPSec | AES-256 |
| Hash IPSec | SHA-256 |
| Modo IPSec | **Transport** (no tunnel) |
| Protocolo Túnel | mGRE (multipoint GRE) |
| NHRP Network-ID | 1 |
| NHRP Auth | `nhrpkey` |
| Subred DMVPN | 10.0.0.0/24 |
| Enrutamiento | OSPF Área 0 `point-to-multipoint` |
| MTU Tunnel | 1400 |

---

## ⚙️ Configuración por Dispositivo

### PC1 (VPCS)
```
ip 10.10.2.100 255.255.255.0 10.10.2.1
save
```

### PC2 (VPCS)
```
ip 10.10.3.100 255.255.255.0 10.10.3.1
save
```

### IOU3 — Switch LAN-A (SPOKE1)
```cisco
hostname SW-A

interface e0/0
 switchport mode access
 switchport access vlan 1
 no shutdown

interface e0/1
 switchport mode access
 switchport access vlan 1
 no shutdown
```

### IOU4 — Switch LAN-B (SPOKE2)
```cisco
hostname SW-B

interface e0/0
 switchport mode access
 switchport access vlan 1
 no shutdown

interface e0/1
 switchport mode access
 switchport access vlan 1
 no shutdown
```

### IOU5 — ISP
```cisco
hostname R-ISP

! ── Interfaz hacia HUB ──────────────────────────────────────
interface e0/2
 description hacia-HUB
 ip address 200.0.0.1 255.255.255.252
 no shutdown

! ── Interfaz hacia SPOKE1 ───────────────────────────────────
interface e0/0
 description hacia-SPOKE1
 ip address 200.0.0.5 255.255.255.252
 no shutdown

! ── Interfaz hacia SPOKE2 ───────────────────────────────────
interface e0/1
 description hacia-SPOKE2
 ip address 200.0.0.9 255.255.255.252
 no shutdown

! ── Rutas estáticas ─────────────────────────────────────────
ip route 10.10.2.0 255.255.255.0 200.0.0.6
ip route 10.10.3.0 255.255.255.0 200.0.0.10
```

### IOU6 — HUB (DMVPN Fase 2 IKEv1)
```cisco
hostname HUB-F2

! ════════════════════════════════════════════════════════════
!  INTERFACES FÍSICAS
! ════════════════════════════════════════════════════════════
interface e0/0
 description WAN-ISP
 ip address 200.0.0.2 255.255.255.252
 no shutdown

! ════════════════════════════════════════════════════════════
!  RUTA DEFAULT
! ════════════════════════════════════════════════════════════
ip route 0.0.0.0 0.0.0.0 200.0.0.1

! ════════════════════════════════════════════════════════════
!  IKEv1 — PSK wildcard para aceptar cualquier spoke
! ════════════════════════════════════════════════════════════
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key Cisco12345 address 0.0.0.0 0.0.0.0

! ════════════════════════════════════════════════════════════
!  IPSEC — modo TRANSPORT (GRE ya encapsula)
! ════════════════════════════════════════════════════════════
crypto ipsec transform-set TS-DMVPN esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile PROF-DMVPN
 set transform-set TS-DMVPN

! ════════════════════════════════════════════════════════════
!  TÚNEL mGRE — HUB
!  ip nhrp map multicast dynamic = acepta registros dinámicos
!  tunnel mode gre multipoint = mGRE, múltiples peers
!  OSPF point-to-multipoint = correcto para Hub-and-Spoke
! ════════════════════════════════════════════════════════════
interface Tunnel0
 description DMVPN-HUB-F2
 ip address 10.0.0.1 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip nhrp authentication nhrpkey
 ip nhrp network-id 1
 ip nhrp map multicast dynamic
 ip ospf network point-to-multipoint
 ip ospf mtu-ignore
 tunnel source e0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile PROF-DMVPN
 no shutdown

! ════════════════════════════════════════════════════════════
!  OSPF
! ════════════════════════════════════════════════════════════
router ospf 1
 router-id 1.1.1.1
 network 10.0.0.0 0.0.0.255 area 0
```

### IOU1 — SPOKE1 (DMVPN Fase 2 IKEv1)
```cisco
hostname SPOKE1-F2

! ════════════════════════════════════════════════════════════
!  INTERFACES FÍSICAS
! ════════════════════════════════════════════════════════════
interface e0/0
 description WAN-ISP
 ip address 200.0.0.6 255.255.255.252
 no shutdown

interface e0/1
 description LAN-A
 ip address 10.10.2.1 255.255.255.0
 no shutdown

! ════════════════════════════════════════════════════════════
!  RUTA DEFAULT
! ════════════════════════════════════════════════════════════
ip route 0.0.0.0 0.0.0.0 200.0.0.5

! ════════════════════════════════════════════════════════════
!  IKEv1
! ════════════════════════════════════════════════════════════
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key Cisco12345 address 0.0.0.0 0.0.0.0

crypto ipsec transform-set TS-DMVPN esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile PROF-DMVPN
 set transform-set TS-DMVPN

! ════════════════════════════════════════════════════════════
!  TÚNEL mGRE — SPOKE1
!  ip nhrp nhs = NHS del HUB (Next Hop Server)
!  ip nhrp map = IP túnel HUB → IP NBMA del HUB
!  ip nhrp map multicast = enviar multicast OSPF al HUB
! ════════════════════════════════════════════════════════════
interface Tunnel0
 description DMVPN-SPOKE1-F2
 ip address 10.0.0.2 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip nhrp authentication nhrpkey
 ip nhrp network-id 1
 ip nhrp nhs 10.0.0.1
 ip nhrp map 10.0.0.1 200.0.0.2
 ip nhrp map multicast 200.0.0.2
 ip ospf network point-to-multipoint
 ip ospf mtu-ignore
 tunnel source e0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile PROF-DMVPN
 no shutdown

! ════════════════════════════════════════════════════════════
!  OSPF
! ════════════════════════════════════════════════════════════
router ospf 1
 router-id 2.2.2.2
 network 10.0.0.0 0.0.0.255 area 0
 network 10.10.2.0 0.0.0.255 area 0
```

### IOU2 — SPOKE2 (DMVPN Fase 2 IKEv1)
```cisco
hostname SPOKE2-F2

! ════════════════════════════════════════════════════════════
!  INTERFACES FÍSICAS
! ════════════════════════════════════════════════════════════
interface e0/0
 description WAN-ISP
 ip address 200.0.0.10 255.255.255.252
 no shutdown

interface e0/1
 description LAN-B
 ip address 10.10.3.1 255.255.255.0
 no shutdown

! ════════════════════════════════════════════════════════════
!  RUTA DEFAULT
! ════════════════════════════════════════════════════════════
ip route 0.0.0.0 0.0.0.0 200.0.0.9

! ════════════════════════════════════════════════════════════
!  IKEv1
! ════════════════════════════════════════════════════════════
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key Cisco12345 address 0.0.0.0 0.0.0.0

crypto ipsec transform-set TS-DMVPN esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile PROF-DMVPN
 set transform-set TS-DMVPN

! ════════════════════════════════════════════════════════════
!  TÚNEL mGRE — SPOKE2
! ════════════════════════════════════════════════════════════
interface Tunnel0
 description DMVPN-SPOKE2-F2
 ip address 10.0.0.3 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip nhrp authentication nhrpkey
 ip nhrp network-id 1
 ip nhrp nhs 10.0.0.1
 ip nhrp map 10.0.0.1 200.0.0.2
 ip nhrp map multicast 200.0.0.2
 ip ospf network point-to-multipoint
 ip ospf mtu-ignore
 tunnel source e0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile PROF-DMVPN
 no shutdown

! ════════════════════════════════════════════════════════════
!  OSPF
! ════════════════════════════════════════════════════════════
router ospf 1
 router-id 3.3.3.3
 network 10.0.0.0 0.0.0.255 area 0
 network 10.10.3.0 0.0.0.255 area 0
```

---

##  Verificación

### Ver estado DMVPN en el HUB
```cisco
show dmvpn
```
Salida esperada:
```
Interface: Tunnel0, IPv4 NHRP Details
Type:Hub, NHRP Peers:2,
  Peer NBMA Addr    Peer Tunnel Add  State   UpDn Tm
  200.0.0.6         10.0.0.2         UP      00:05:00
  200.0.0.10        10.0.0.3         UP      00:05:00
```

### Ver registros NHRP en el HUB
```cisco
show ip nhrp
```

### Ver vecinos OSPF
```cisco
show ip ospf neighbor
```
Salida esperada:
```
Neighbor ID  State  Address    Interface
2.2.2.2      FULL/  10.0.0.2   Tunnel0
3.3.3.3      FULL/  10.0.0.3   Tunnel0
```

### Ver rutas OSPF en SPOKE1
```cisco
show ip route ospf
```
Salida esperada:
```
O    10.0.0.1/32   via 10.0.0.1, Tunnel0
O    10.0.0.3/32   via 10.0.0.1, Tunnel0
O    10.10.3.0/24  via 10.0.0.1, Tunnel0
```

### Ver SA IKEv1
```cisco
show crypto isakmp sa
```

### Ver paquetes cifrados
```cisco
show crypto ipsec sa | include encaps|decaps
```

### Limpiar NHRP para demostración
```cisco
clear ip nhrp
```

### Ping spoke a spoke
```
PC1> ping 10.10.3.100
```

---

##  Diferencia Fase 2 vs Fase 3

| Aspecto | Fase 2 (este lab) | Fase 3 |
|---|---|---|
| HUB extra | ninguno | `ip nhrp redirect` |
| SPOKE extra | ninguno | `ip nhrp shortcut` |
| Resolución S-S | NHRP Resolution Reply | NHRP Redirect + Shortcut Route |
| Summarización HUB |  No permitida |  Permitida |
| Enrutamiento | OSPF `point-to-multipoint` | EIGRP |
| IKE | IKEv1 | IKEv2 |


---

##  Requisitos

- GNS3
- Cisco IOU L3 (imagen 15.4.1T o superior)
- Cisco IOU L2 (para switches)
- VPCS

---

> ** ADVERTENCIA:** Estas configuraciones están diseñadas **EXCLUSIVAMENTE** para fines educativos en entornos simulados como **GNS3**.

---

*Autor: Roberto de Jesus — Matrícula: 2023-0348*
*Fecha: 2025*
