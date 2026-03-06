# VPN Site-to-Site Basada en Enrutamiento — IPSec IKEv1 (VTI)

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

---

##  Descripción General

Este laboratorio implementa una **VPN Site-to-Site basada en enrutamiento (Route-Based)** usando **IPSec IKEv1** con **Virtual Tunnel Interface (VTI)**. El tráfico se cifra según la tabla de enrutamiento — cualquier tráfico enrutado hacia `Tunnel0` es cifrado automáticamente. No requiere Crypto ACL. OSPF corre sobre el túnel para intercambio dinámico de rutas.

### Diferencia clave con Policy-Based
- No requiere Crypto ACL
- Soporta enrutamiento dinámico (OSPF, EIGRP, BGP)
- La interfaz Tunnel0 es visible y monitoreable
- Usa `tunnel mode ipsec ipv4` en lugar de crypto map en interfaz física

> ** ADVERTENCIA:** Estas configuraciones están diseñadas **EXCLUSIVAMENTE** para fines educativos en entornos simulados como **GNS3**.

---

## 🖧 Topología del Laboratorio

### Diagrama de Red

```
+--------+  +---------+  +----------+  +---------+  +----------+  +---------+  +--------+
|  PC1   |  | IOU3    |  |  IOU1 R1 |  |  R-ISP  |  |  IOU2 R2 |  | IOU4    |  |  PC2   |
| VPCS   +--+ SW LAN-A+--+          +--+         +--+          +--+ SW LAN-B+--+ VPCS   |
+--------+  +---------+  +----------+  +---------+  +----------+  +---------+  +--------+
10.10.1.100             Tunnel0(VTI):              Tunnel0(VTI):               10.10.2.100
                        172.16.0.1/30              172.16.0.2/30
```

### Configuración de Red

#### IOU1 — R1
- WAN e0/0: `200.0.0.2/30`
- LAN e0/1: `10.10.1.1/24`
- Tunnel0 VTI: `172.16.0.1/30`

#### R-ISP
- e0/1 hacia R1: `200.0.0.1/30`
- e0/0 hacia R2: `200.0.0.5/30`

#### IOU2 — R2
- WAN e0/0: `200.0.0.6/30`
- LAN e0/1: `10.10.2.1/24`
- Tunnel0 VTI: `172.16.0.2/30`

#### Dispositivos Finales

| Dispositivo | IP | Gateway |
|---|---|---|
| PC1 | 10.10.1.100/24 | 10.10.1.1 |
| PC2 | 10.10.2.100/24 | 10.10.2.1 |

---

##  Parámetros

| Parámetro | Valor |
|---|---|
| Versión IKE | IKEv1 |
| Autenticación | Pre-Shared Key |
| Clave PSK | `Cisco12345` |
| Cifrado Fase 1 | AES-256 |
| Hash Fase 1 | SHA-256 |
| DH Group | Group 14 |
| Cifrado IPSec | ESP-AES-256 |
| Hash IPSec | ESP-SHA256-HMAC |
| Modo IPSec | Tunnel |
| Tipo Tunnel | `tunnel mode ipsec ipv4` (VTI) |
| Subred VTI | 172.16.0.0/30 |
| Enrutamiento | OSPF Área 0 |
| Crypto ACL | **No requerida** |

---

## ⚙️ Configuración por Dispositivo

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


### IOU1 — R1 (IKEv1 Route-Based VTI)

```cisco
hostname R1-IKEv1-VTI

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
crypto ipsec transform-set TS-VTI esp-aes 256 esp-sha256-hmac
 mode tunnel

! ════════════════════════════════════════════════════════════
!  IPSEC PROFILE — reemplaza al crypto map
!  Se aplica directamente en Tunnel0, NO requiere Crypto ACL
! ════════════════════════════════════════════════════════════
crypto ipsec profile PROF-VTI
 set transform-set TS-VTI
 set pfs group14

! ════════════════════════════════════════════════════════════
!  INTERFAZ TUNNEL VTI
!  tunnel mode ipsec ipv4 = VTI nativo (no GRE)
!  tunnel protection = aplica el IPSec Profile
! ════════════════════════════════════════════════════════════
interface Tunnel0
 description VTI-IKEv1-a-R2
 ip address 172.16.0.1 255.255.255.252
 tunnel source e0/0
 tunnel destination 200.0.0.6
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROF-VTI
 no shutdown

! ════════════════════════════════════════════════════════════
!  OSPF SOBRE EL TÚNEL — no requiere Crypto ACL
! ════════════════════════════════════════════════════════════
router ospf 1
 network 10.10.1.0 0.0.0.255 area 0
 network 172.16.0.0 0.0.0.3 area 0
```

### IOU2 — R2 (IKEv1 Route-Based VTI)

```cisco
hostname R2-IKEv1-VTI

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
crypto ipsec transform-set TS-VTI esp-aes 256 esp-sha256-hmac
 mode tunnel

! ════════════════════════════════════════════════════════════
!  IPSEC PROFILE
! ════════════════════════════════════════════════════════════
crypto ipsec profile PROF-VTI
 set transform-set TS-VTI
 set pfs group14

! ════════════════════════════════════════════════════════════
!  INTERFAZ TUNNEL VTI
! ════════════════════════════════════════════════════════════
interface Tunnel0
 description VTI-IKEv1-a-R1
 ip address 172.16.0.2 255.255.255.252
 tunnel source e0/0
 tunnel destination 200.0.0.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROF-VTI
 no shutdown

! ════════════════════════════════════════════════════════════
!  OSPF SOBRE EL TÚNEL
! ════════════════════════════════════════════════════════════
router ospf 1
 network 10.10.2.0 0.0.0.255 area 0
 network 172.16.0.0 0.0.0.3 area 0
```

---

##  Verificación

### Verificar interfaz Tunnel0 levantada
```cisco
show interfaces Tunnel0
```
Salida esperada:
```
Tunnel0 is up, line protocol is up
  Internet address is 172.16.0.1/30
  Tunnel source 200.0.0.2, destination 200.0.0.6
  Tunnel protocol/transport IPSEC/IP
```

### Verificar vecinos OSPF sobre el túnel
```cisco
show ip ospf neighbor
```
Salida esperada:
```
Neighbor ID  State  Address      Interface
200.0.0.6    FULL/  172.16.0.2   Tunnel0
```

### Verificar rutas OSPF
```cisco
show ip route ospf
```
Salida esperada:
```
O    10.10.2.0/24 [110/1001] via 172.16.0.2, Tunnel0
```

### Verificar ISAKMP SA
```cisco
show crypto isakmp sa
```

### Verificar IPSec Profile
```cisco
show crypto ipsec profile
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
