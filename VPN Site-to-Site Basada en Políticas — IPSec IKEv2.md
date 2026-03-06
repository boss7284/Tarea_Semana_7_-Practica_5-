# VPN Site-to-Site Basada en Políticas — IPSec IKEv2

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
- [Parámetros IPSec IKEv2](#-parámetros-ipsec-ikev2)
- [Configuración por Dispositivo](#-configuración-por-dispositivo)
- [Verificación](#-verificación)
- [Diferencia IKEv1 vs IKEv2](#-diferencia-ikev1-vs-ikev2)

---

##  Descripción General

Este laboratorio implementa una **VPN Site-to-Site basada en políticas** con **IPSec IKEv2**. IKEv2 es el sucesor de IKEv1, más eficiente al requerir solo **4 mensajes** para establecer la SA. Usa **IKEv2 Keyring** y **IKEv2 Profile** en lugar del `crypto isakmp key` y `crypto isakmp policy`. El Crypto Map agrega `set ikev2-profile`.

### Mejoras de IKEv2 sobre IKEv1
- **4 mensajes** (vs 6 de IKEv1 Main Mode)
- **MOBIKE** — movilidad de IPs durante la sesión
- Mejor manejo de **NAT-T**
- Soporte nativo de **EAP**

> ** ADVERTENCIA:** Estas configuraciones están diseñadas **EXCLUSIVAMENTE** para fines educativos en entornos simulados como **GNS3**.

---

##  Topología del Laboratorio

![image alt](https://github.com/boss7284/d.u.m.p2/blob/dfef801587d2544f41c55d284f5cf5c5c95251a0/Screenshot%202026-03-03%20224513.png)

### Diagrama de Red

```
+--------+  +---------+  +----------+  +---------+  +----------+  +---------+  +--------+
|  PC1   |  | IOU3    |  |  IOU1 R1 |  |  R-ISP  |  |  IOU2 R2 |  | IOU4    |  |  PC2   |
| VPCS   +--+ SW LAN-A+--+  Peer A  +--+         +--+  Peer B  +--+ SW LAN-B+--+ VPCS   |
+--------+  +---------+  +----------+  +---------+  +----------+  +---------+  +--------+
10.10.1.100             200.0.0.2/30   200.0.0.1   200.0.0.6/30               10.10.2.100
                        10.10.1.1/24   200.0.0.5   10.10.2.1/24
```

### Configuración de Red

#### IOU1 — R1
- WAN e0/0: `200.0.0.2/30`
- LAN e0/1: `10.10.1.1/24`
- Peer remoto: `200.0.0.6`

#### R-ISP
- e0/1: `200.0.0.1/30`
- e0/0: `200.0.0.5/30`

#### IOU2 — R2
- WAN e0/0: `200.0.0.6/30`
- LAN e0/1: `10.10.2.1/24`

#### Dispositivos Finales

| Dispositivo | IP | Gateway |
|---|---|---|
| PC1 | 10.10.1.100/24 | 10.10.1.1 |
| PC2 | 10.10.2.100/24 | 10.10.2.1 |

---

##  Parámetros IPSec IKEv2

| Parámetro | Valor |
|---|---|
| Versión IKE | IKEv2 |
| Mensajes negociación | 4 (IKE_SA_INIT + IKE_AUTH) |
| Autenticación | Pre-Shared Key |
| Clave PSK | `Cisco12345` |
| Cifrado IKEv2 | AES-CBC-256 |
| Integridad | SHA-256 |
| DH Group | Group 14 |
| Cifrado IPSec | ESP-AES-256 |
| Hash IPSec | ESP-SHA256-HMAC |
| Modo IPSec | Tunnel |
| PFS | Group 14 |
| Selección de tráfico | Crypto ACL |

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


### IOU1 — R1 Peer A (IKEv2 Policy-Based)

```cisco
hostname R1-IKEv2

! ════════════════════════════════════════════════════════════
!  INTERFACES
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
!  IKEv2 — Proposal + Policy + Keyring + Profile
!  Reemplaza completamente al "crypto isakmp" de IKEv1
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
crypto ipsec transform-set TS-IKEv2 esp-aes 256 esp-sha256-hmac
 mode tunnel

! ════════════════════════════════════════════════════════════
!  CRYPTO ACL — igual que IKEv1
! ════════════════════════════════════════════════════════════
ip access-list extended ACL-VPN-IKEv2
 permit ip 10.10.1.0 0.0.0.255 10.10.2.0 0.0.0.255

! ════════════════════════════════════════════════════════════
!  CRYPTO MAP — DIFERENCIA vs IKEv1: set ikev2-profile
! ════════════════════════════════════════════════════════════
crypto map CMAP-IKEv2 10 ipsec-isakmp
 set peer 200.0.0.6
 set transform-set TS-IKEv2
 set pfs group14
 set ikev2-profile PROF-IKEv2
 match address ACL-VPN-IKEv2

interface e0/0
 crypto map CMAP-IKEv2
```

### IOU2 — R2 Peer B (IKEv2 Policy-Based)

```cisco
hostname R2-IKEv2

! ════════════════════════════════════════════════════════════
!  INTERFACES
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
crypto ipsec transform-set TS-IKEv2 esp-aes 256 esp-sha256-hmac
 mode tunnel

! ════════════════════════════════════════════════════════════
!  CRYPTO ACL (inverso a R1)
! ════════════════════════════════════════════════════════════
ip access-list extended ACL-VPN-IKEv2
 permit ip 10.10.2.0 0.0.0.255 10.10.1.0 0.0.0.255

! ════════════════════════════════════════════════════════════
!  CRYPTO MAP
! ════════════════════════════════════════════════════════════
crypto map CMAP-IKEv2 10 ipsec-isakmp
 set peer 200.0.0.2
 set transform-set TS-IKEv2
 set pfs group14
 set ikev2-profile PROF-IKEv2
 match address ACL-VPN-IKEv2

interface e0/0
 crypto map CMAP-IKEv2
```

---

##  Verificación

### Generar tráfico desde PC1
```
PC1> ping 10.10.2.100
```

### Verificar SA IKEv2 Fase 1
```cisco
show crypto ikev2 sa
```
Salida esperada:
```
Tunnel-id  Local          Remote         Status
1          200.0.0.2/500  200.0.0.6/500  READY
  Encr: AES-CBC, keysize: 256, Hash: SHA256, DH Grp:14, Auth: PSK
```

### Verificar SA IKEv2 con detalle
```cisco
show crypto ikev2 sa detail
```

### Verificar sesiones IKEv2
```cisco
show crypto ikev2 session
```

### Verificar IPSec SA
```cisco
show crypto ipsec sa
```

### Verificar paquetes cifrados
```cisco
show crypto ipsec sa | include encaps|decaps
```

### Verificar crypto map
```cisco
show crypto map
```

---

##  Diferencia IKEv1 vs IKEv2

| Aspecto | IKEv1 | IKEv2 |
|---|---|---|
| Mensajes Fase 1 | 6 (Main Mode) | 4 siempre |
| Configurar política | `crypto isakmp policy` | `crypto ikev2 proposal` + `policy` |
| Configurar PSK | `crypto isakmp key` | `crypto ikev2 keyring` |
| Perfil de peer | No existe | `crypto ikev2 profile` |
| Crypto Map extra | ninguno | `set ikev2-profile` |
| Verificar SA | `show crypto isakmp sa` | `show crypto ikev2 sa` |
| Estado OK | `QM_IDLE` | `READY` |


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
