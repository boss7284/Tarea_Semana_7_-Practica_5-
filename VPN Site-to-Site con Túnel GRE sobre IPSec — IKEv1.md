# VPN Site-to-Site con Túnel GRE sobre IPSec — IKEv1

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
- [Flujo de Encapsulamiento](#-flujo-de-encapsulamiento)

---

##  Descripción General

Este laboratorio implementa una **VPN Site-to-Site con túnel GRE sobre IPSec IKEv1**. GRE (Generic Routing Encapsulation, protocolo IP 47) proporciona el túnel de capa 3 soportando multicast y enrutamiento dinámico. IPSec cifra todo el tráfico GRE. La **Crypto ACL protege el protocolo GRE** (`permit gre`) en lugar de las LANs directamente.

### Diferencia clave con VTI
- **VTI:** `tunnel mode ipsec ipv4` — IPSec nativo, crypto map NO en interfaz física
- **GRE/IPSec:** `tunnel mode gre ip` — GRE primero, crypto map en e0/0 cifra el GRE
- GRE agrega 24 bytes de overhead pero soporta multicast nativo

> ** ADVERTENCIA:** Estas configuraciones están diseñadas **EXCLUSIVAMENTE** para fines educativos en entornos simulados como **GNS3**.

---

##  Topología del Laboratorio

![image alt](https://github.com/boss7284/d.u.m.p2/blob/dfef801587d2544f41c55d284f5cf5c5c95251a0/Screenshot%202026-03-03%20224513.png)

### Diagrama de Red

```
+--------+  +---------+  +----------+  +---------+  +----------+  +---------+  +--------+
|  PC1   |  | IOU3    |  |  IOU1 R1 |  |  R-ISP  |  |  IOU2 R2 |  | IOU4    |  |  PC2   |
| VPCS   +--+ SW LAN-A+--+          +--+         +--+          +--+ SW LAN-B+--+ VPCS   |
+--------+  +---------+  +----------+  +---------+  +----------+  +---------+  +--------+
10.10.1.100             Tunnel0(GRE):              Tunnel0(GRE):               10.10.2.100
                        192.168.100.1/30           192.168.100.2/30

Encapsulamiento: [IP LAN] → [GRE Header 24B] → [ESP/IPSec] → [IP WAN]
```

### Configuración de Red

#### IOU1 — R1
- WAN e0/0: `200.0.0.2/30`
- LAN e0/1: `10.10.1.1/24`
- Tunnel0 GRE: `192.168.100.1/30`

#### R-ISP
- e0/1 hacia R1: `200.0.0.1/30`
- e0/0 hacia R2: `200.0.0.5/30`

#### IOU2 — R2
- WAN e0/0: `200.0.0.6/30`
- LAN e0/1: `10.10.2.1/24`
- Tunnel0 GRE: `192.168.100.2/30`

#### Dispositivos Finales

| Dispositivo | IP | Gateway |
|---|---|---|
| PC1 | 10.10.1.100/24 | 10.10.1.1 |
| PC2 | 10.10.2.100/24 | 10.10.2.1 |

---

##  Parámetros

| Parámetro | Valor |
|---|---|
| Protocolo Túnel | GRE (protocolo IP 47) |
| Versión IKE | IKEv1 |
| Autenticación | Pre-Shared Key |
| Clave PSK | `Cisco12345` |
| Cifrado Fase 1 | AES-256 |
| Hash Fase 1 | SHA-256 |
| DH Group | Group 14 |
| Cifrado IPSec | ESP-AES-256 |
| Modo IPSec | Tunnel |
| Crypto ACL | `permit gre host X host Y` |
| Subred GRE | 192.168.100.0/30 |
| Enrutamiento | OSPF Área 0 |

---

##  Configuración por Dispositivo

### PC1 (VPCS)
```
ip 10.10.1.100 255.255.255.0 10.10.1.1
save
```

### PC2 (VPCS)
```
ip 10.10.2.100 255.255.255.0 10.10.2.1
save
```

### IOU3 — Switch LAN-A

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

### IOU4 — Switch LAN-B

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


### R-ISP — Router ISP

```cisco
hostname R-ISP

! ── Interfaz hacia R1 ───────────────────────────────────────
interface e0/1
 description hacia-R1
 ip address 200.0.0.1 255.255.255.252
 no shutdown

! ── Interfaz hacia R2 ───────────────────────────────────────
interface e0/0
 description hacia-R2
 ip address 200.0.0.5 255.255.255.252
 no shutdown

! ── Rutas estáticas ─────────────────────────────────────────
ip route 10.10.1.0 255.255.255.0 200.0.0.2
ip route 10.10.2.0 255.255.255.0 200.0.0.6
```


### IOU1 — R1 (GRE over IPSec IKEv1)

```cisco
hostname R1-GRE-IKEv1

! ════════════════════════════════════════════════════════════
!  INTERFACES FÍSICAS
! ════════════════════════════════════════════════════════════
interface e0/1
 description LAN-A
 ip address 10.10.1.1 255.255.255.0
 no shutdown

interface e0/0
 description WAN-ISP
 ip address 200.0.0.2 255.255.255.252
 no shutdown

! ════════════════════════════════════════════════════════════
!  RUTA DEFAULT
! ════════════════════════════════════════════════════════════
ip route 0.0.0.0 0.0.0.0 200.0.0.1

! ════════════════════════════════════════════════════════════
!  FASE 1 — ISAKMP
! ════════════════════════════════════════════════════════════
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key Cisco12345 address 200.0.0.6

! ════════════════════════════════════════════════════════════
!  FASE 2 — TRANSFORM SET
! ════════════════════════════════════════════════════════════
crypto ipsec transform-set TS-GRE esp-aes 256 esp-sha256-hmac
 mode tunnel

! ════════════════════════════════════════════════════════════
!  CRYPTO ACL — protege el protocolo GRE (IP proto 47)
!  DIFERENCIA CLAVE vs Policy-Based: permit gre, no las LANs
! ════════════════════════════════════════════════════════════
ip access-list extended ACL-GRE
 permit gre host 200.0.0.2 host 200.0.0.6

! ════════════════════════════════════════════════════════════
!  CRYPTO MAP — en e0/0 (WAN física), no en el túnel
! ════════════════════════════════════════════════════════════
crypto map CMAP-GRE 10 ipsec-isakmp
 set peer 200.0.0.6
 set transform-set TS-GRE
 set pfs group14
 match address ACL-GRE

interface e0/0
 crypto map CMAP-GRE

! ════════════════════════════════════════════════════════════
!  TÚNEL GRE
!  tunnel mode gre ip = GRE estándar punto a punto
!  El crypto map en e0/0 cifra el tráfico GRE automáticamente
! ════════════════════════════════════════════════════════════
interface Tunnel0
 description GRE-sobre-IPSec-a-R2
 ip address 192.168.100.1 255.255.255.252
 tunnel source e0/0
 tunnel destination 200.0.0.6
 tunnel mode gre ip
 no shutdown

! ════════════════════════════════════════════════════════════
!  OSPF SOBRE EL TÚNEL GRE
! ════════════════════════════════════════════════════════════
router ospf 1
 network 10.10.1.0 0.0.0.255 area 0
 network 192.168.100.0 0.0.0.3 area 0
```

### IOU2 — R2 (GRE over IPSec IKEv1)

```cisco
hostname R2-GRE-IKEv1

! ════════════════════════════════════════════════════════════
!  INTERFACES FÍSICAS
! ════════════════════════════════════════════════════════════
interface e0/1
 description LAN-B
 ip address 10.10.2.1 255.255.255.0
 no shutdown

interface e0/0
 description WAN-ISP
 ip address 200.0.0.6 255.255.255.252
 no shutdown

! ════════════════════════════════════════════════════════════
!  RUTA DEFAULT
! ════════════════════════════════════════════════════════════
ip route 0.0.0.0 0.0.0.0 200.0.0.5

! ════════════════════════════════════════════════════════════
!  FASE 1 — ISAKMP
! ════════════════════════════════════════════════════════════
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key Cisco12345 address 200.0.0.2

! ════════════════════════════════════════════════════════════
!  FASE 2 — TRANSFORM SET
! ════════════════════════════════════════════════════════════
crypto ipsec transform-set TS-GRE esp-aes 256 esp-sha256-hmac
 mode tunnel

! ════════════════════════════════════════════════════════════
!  CRYPTO ACL (inverso a R1)
! ════════════════════════════════════════════════════════════
ip access-list extended ACL-GRE
 permit gre host 200.0.0.6 host 200.0.0.2

! ════════════════════════════════════════════════════════════
!  CRYPTO MAP
! ════════════════════════════════════════════════════════════
crypto map CMAP-GRE 10 ipsec-isakmp
 set peer 200.0.0.2
 set transform-set TS-GRE
 set pfs group14
 match address ACL-GRE

interface e0/0
 crypto map CMAP-GRE

! ════════════════════════════════════════════════════════════
!  TÚNEL GRE
! ════════════════════════════════════════════════════════════
interface Tunnel0
 description GRE-sobre-IPSec-a-R1
 ip address 192.168.100.2 255.255.255.252
 tunnel source e0/0
 tunnel destination 200.0.0.2
 tunnel mode gre ip
 no shutdown

! ════════════════════════════════════════════════════════════
!  OSPF
! ════════════════════════════════════════════════════════════
router ospf 1
 network 10.10.2.0 0.0.0.255 area 0
 network 192.168.100.0 0.0.0.3 area 0
```

---

##  Verificación

### Verificar interfaz GRE activa
```cisco
show interfaces Tunnel0
```
Salida esperada:
```
Tunnel0 is up, line protocol is up
  Tunnel protocol/transport GRE/IP
```

### Verificar vecinos OSPF
```cisco
show ip ospf neighbor
```

### Verificar rutas OSPF
```cisco
show ip route ospf
```

### Verificar IPSec — protocolo GRE (47) en la SA
```cisco
show crypto ipsec sa
```
Buscar en la salida:
```
local  ident (addr/mask/prot/port): (200.0.0.2/255.255.255.255/47/0)
remote ident (addr/mask/prot/port): (200.0.0.6/255.255.255.255/47/0)
```

### Verificar ISAKMP SA
```cisco
show crypto isakmp sa
```

### Verificar paquetes cifrados
```cisco
show crypto ipsec sa | include encaps|decaps
```

### Ping de prueba
```
PC1> ping 10.10.2.100
```

---

##  Flujo de Encapsulamiento

```
PC1 envía paquete a PC2:

  [IP: 10.10.1.100 → 10.10.2.100 | Datos]       ← paquete original
            ↓ GRE encapsula
  [IP: 200.0.0.2 → 200.0.0.6 | GRE | IP original]
            ↓ IPSec ESP cifra el GRE completo
  [IP: 200.0.0.2 → 200.0.0.6 | ESP CIFRADO]
            ↓ viaja por el ISP
  R2: IPSec descifra → GRE desencapsula → entrega a PC2
```


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
