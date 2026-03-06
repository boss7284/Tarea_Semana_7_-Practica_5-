# VPN Site-to-Site Basada en Enrutamiento — IPSec IKEv2 (VTI)

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

Este laboratorio implementa una **VPN Site-to-Site basada en enrutamiento (Route-Based)** usando **IPSec IKEv2** con **Virtual Tunnel Interface (VTI)**. El **IPSec Profile** referencia directamente el `ikev2-profile` mediante `set ikev2-profile`. No requiere Crypto ACL. OSPF corre sobre el túnel.

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
10.10.1.100             Tunnel0(VTI):              Tunnel0(VTI):               10.10.2.100
                        172.16.0.1/30              172.16.0.2/30
```

### Configuración de Red

#### IOU1 — R1
- WAN e0/0: `200.0.0.2/30`
- LAN e0/1: `10.10.1.1/24`
- Tunnel0 VTI: `172.16.0.1/30`

#### R-ISP
- e0/1: `200.0.0.1/30` / e0/0: `200.0.0.5/30`

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
| Versión IKE | IKEv2 |
| Autenticación | Pre-Shared Key |
| Clave PSK | `Cisco12345` |
| Cifrado IKEv2 | AES-CBC-256 |
| Integridad | SHA-256 |
| DH Group | Group 14 |
| Tipo Tunnel | `tunnel mode ipsec ipv4` (VTI) |
| Subred VTI | 172.16.0.0/30 |
| Enrutamiento | OSPF Área 0 |
| Crypto ACL | **No requerida** |

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


### R-ISP

```cisco
hostname R-ISP

interface e0/1
 description hacia-R1
 ip address 200.0.0.1 255.255.255.252
 no shutdown

interface e0/0
 description hacia-R2
 ip address 200.0.0.5 255.255.255.252
 no shutdown

ip route 10.10.1.0 255.255.255.0 200.0.0.2
ip route 10.10.2.0 255.255.255.0 200.0.0.6
```


### IOU1 — R1 (IKEv2 VTI)

```cisco
hostname R1-IKEv2-VTI

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

ip route 0.0.0.0 0.0.0.0 200.0.0.1

! ════════════════════════════════════════════════════════════
!  IKEv2
! ════════════════════════════════════════════════════════════
crypto ikev2 proposal PROP-IKEv2
 encryption aes-cbc-256
 integrity sha256
 group 14

crypto ikev2 policy POL-IKEv2
 proposal PROP-IKEv2

crypto ikev2 keyring KR-IKEv2
 peer R2
  address 200.0.0.6
  pre-shared-key Cisco12345

crypto ikev2 profile PROF-IKEv2
 match identity remote address 200.0.0.6 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KR-IKEv2
 lifetime 86400

! ════════════════════════════════════════════════════════════
!  TRANSFORM SET
! ════════════════════════════════════════════════════════════
crypto ipsec transform-set TS-VTI esp-aes 256 esp-sha256-hmac
 mode tunnel

! ════════════════════════════════════════════════════════════
!  IPSEC PROFILE — referencia ikev2-profile
!  DIFERENCIA vs IKEv1 VTI: set ikev2-profile
! ════════════════════════════════════════════════════════════
crypto ipsec profile PROF-VTI-IKEv2
 set transform-set TS-VTI
 set pfs group14
 set ikev2-profile PROF-IKEv2

! ════════════════════════════════════════════════════════════
!  INTERFAZ VTI
! ════════════════════════════════════════════════════════════
interface Tunnel0
 description IKEv2-VTI-a-R2
 ip address 172.16.0.1 255.255.255.252
 tunnel source e0/0
 tunnel destination 200.0.0.6
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROF-VTI-IKEv2
 no shutdown

! ════════════════════════════════════════════════════════════
!  OSPF
! ════════════════════════════════════════════════════════════
router ospf 1
 network 10.10.1.0 0.0.0.255 area 0
 network 172.16.0.0 0.0.0.3 area 0
```

### IOU2 — R2 (IKEv2 VTI)

```cisco
hostname R2-IKEv2-VTI

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

ip route 0.0.0.0 0.0.0.0 200.0.0.5

! ════════════════════════════════════════════════════════════
!  IKEv2
! ════════════════════════════════════════════════════════════
crypto ikev2 proposal PROP-IKEv2
 encryption aes-cbc-256
 integrity sha256
 group 14

crypto ikev2 policy POL-IKEv2
 proposal PROP-IKEv2

crypto ikev2 keyring KR-IKEv2
 peer R1
  address 200.0.0.2
  pre-shared-key Cisco12345

crypto ikev2 profile PROF-IKEv2
 match identity remote address 200.0.0.2 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KR-IKEv2
 lifetime 86400

! ════════════════════════════════════════════════════════════
!  TRANSFORM SET
! ════════════════════════════════════════════════════════════
crypto ipsec transform-set TS-VTI esp-aes 256 esp-sha256-hmac
 mode tunnel

! ════════════════════════════════════════════════════════════
!  IPSEC PROFILE
! ════════════════════════════════════════════════════════════
crypto ipsec profile PROF-VTI-IKEv2
 set transform-set TS-VTI
 set pfs group14
 set ikev2-profile PROF-IKEv2

! ════════════════════════════════════════════════════════════
!  INTERFAZ VTI
! ════════════════════════════════════════════════════════════
interface Tunnel0
 description IKEv2-VTI-a-R1
 ip address 172.16.0.2 255.255.255.252
 tunnel source e0/0
 tunnel destination 200.0.0.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROF-VTI-IKEv2
 no shutdown

! ════════════════════════════════════════════════════════════
!  OSPF
! ════════════════════════════════════════════════════════════
router ospf 1
 network 10.10.2.0 0.0.0.255 area 0
 network 172.16.0.0 0.0.0.3 area 0
```

---

##  Verificación

### Verificar Tunnel0
```cisco
show interfaces Tunnel0
```

### Verificar SA IKEv2
```cisco
show crypto ikev2 sa
```
Salida esperada:
```
Tunnel-id  Status
1          READY
```

### Verificar OSPF
```cisco
show ip ospf neighbor
show ip route ospf
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
