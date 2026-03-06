# VPN Site-to-Site Basada en Políticas — IPSec IKEv1

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
- [Parámetros IPSec IKEv1](#-parámetros-ipsec-ikev1)
- [Configuración por Dispositivo](#-configuración-por-dispositivo)
- [Verificación](#-verificación)
- [Diferencia Policy-Based vs Route-Based](#-diferencia-policy-based-vs-route-based)

---

##  Descripción General

Este laboratorio implementa una **VPN Site-to-Site punto a punto basada en políticas (Policy-Based)** utilizando **IPSec con IKEv1**. El tráfico que se cifra es únicamente el que coincide con una **Crypto ACL**, la cual define el tráfico "interesante". El túnel se levanta **on-demand** cuando hay tráfico que coincide.

### ¿Cómo funciona?
- PC1 envía tráfico hacia PC2
- R1 detecta que el tráfico coincide con la Crypto ACL
- R1 inicia negociación **IKE Fase 1** (ISAKMP SA) — 6 mensajes Main Mode
- Se negocia **IKE Fase 2** (IPSec SA) — Quick Mode
- El tráfico se cifra con ESP y viaja por el ISP
- R2 descifra y entrega a PC2

> ** NOTA:** Los primeros 3 pings mostrarán `timeout` — es normal, son los paquetes descartados durante la negociación IPSec (~2-3 seg). El 4to ping ya responde con el túnel establecido.

> ** ADVERTENCIA:** Estas configuraciones están diseñadas **EXCLUSIVAMENTE** para fines educativos en entornos simulados como **GNS3**.

---

## 🖧 Topología del Laboratorio

### Diagrama de Red

```
+--------+    +---------+    +----------+    +---------+    +----------+    +---------+    +--------+
|  PC1   |    | IOU3    |    |  IOU1 R1 |    |  R-ISP  |    |  IOU2 R2 |    | IOU4    |    |  PC2   |
| VPCS   +----+ SW LAN-A+----+  Peer A  +----+         +----+  Peer B  +----+ SW LAN-B+----+ VPCS   |
+--------+    +---------+    +----------+    +---------+    +----------+    +---------+    +--------+
10.10.1.100   e0/0  e0/1    e0/1   e0/0    e0/1   e0/0    e0/0   e0/1    e0/0  e0/1    10.10.2.100
```

### Configuración de Red

#### IOU1 — R1 (Peer A)
- WAN e0/0: `200.0.0.2/30` — hacia ISP
- LAN e0/1: `10.10.1.1/24` — hacia SW-A
- Peer remoto: `200.0.0.6`

#### R-ISP
- e0/1 hacia R1: `200.0.0.1/30`
- e0/0 hacia R2: `200.0.0.5/30`

#### IOU2 — R2 (Peer B)
- WAN e0/0: `200.0.0.6/30` — hacia ISP
- LAN e0/1: `10.10.2.1/24` — hacia SW-B
- Peer remoto: `200.0.0.2`

#### Dispositivos Finales

| Dispositivo | IP | Gateway |
|---|---|---|
| PC1 | 10.10.1.100/24 | 10.10.1.1 |
| PC2 | 10.10.2.100/24 | 10.10.2.1 |

---

##  Parámetros IPSec IKEv1

| Parámetro | Valor |
|---|---|
| Versión IKE | IKEv1 |
| Modo Fase 1 | Main Mode (6 mensajes) |
| Autenticación | Pre-Shared Key (PSK) |
| Clave PSK | `Cisco12345` |
| Cifrado Fase 1 | AES-256 |
| Hash Fase 1 | SHA-256 |
| Grupo Diffie-Hellman | Group 14 (2048-bit) |
| Lifetime Fase 1 | 86400 segundos (24h) |
| Cifrado Fase 2 | ESP-AES-256 |
| Hash Fase 2 | ESP-SHA256-HMAC |
| Modo IPSec | Tunnel |
| PFS | Group 14 |
| Selección de tráfico | Crypto ACL |

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

! ── Rutas estáticas hacia las LANs ──────────────────────────
ip route 10.10.1.0 255.255.255.0 200.0.0.2
ip route 10.10.2.0 255.255.255.0 200.0.0.6
```

### IOU1 — R1 Peer A (IKEv1 Policy-Based)

```cisco
hostname R1-IKEv1

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
!  RUTA DEFAULT HACIA ISP
! ════════════════════════════════════════════════════════════
ip route 0.0.0.0 0.0.0.0 200.0.0.1

! ════════════════════════════════════════════════════════════
!  FASE 1 — ISAKMP POLICY
!  Define parámetros para la negociación IKE Fase 1
! ════════════════════════════════════════════════════════════
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

! PSK para autenticar con el peer remoto (R2)
crypto isakmp key Cisco12345 address 200.0.0.6

! ════════════════════════════════════════════════════════════
!  FASE 2 — TRANSFORM SET
!  Define cifrado y hash para el tráfico IPSec
! ════════════════════════════════════════════════════════════
crypto ipsec transform-set TS-IKEv1 esp-aes 256 esp-sha256-hmac
 mode tunnel

! ════════════════════════════════════════════════════════════
!  CRYPTO ACL — TRÁFICO INTERESANTE
!  Solo el tráfico que coincide aquí será cifrado
! ════════════════════════════════════════════════════════════
ip access-list extended ACL-VPN-IKEv1
 permit ip 10.10.1.0 0.0.0.255 10.10.2.0 0.0.0.255

! ════════════════════════════════════════════════════════════
!  CRYPTO MAP — vincula peer, transform-set y ACL
! ════════════════════════════════════════════════════════════
crypto map CMAP-IKEv1 10 ipsec-isakmp
 set peer 200.0.0.6
 set transform-set TS-IKEv1
 set pfs group14
 match address ACL-VPN-IKEv1

! ════════════════════════════════════════════════════════════
!  APLICAR CRYPTO MAP A INTERFAZ WAN
! ════════════════════════════════════════════════════════════
interface e0/0
 crypto map CMAP-IKEv1
```

### IOU2 — R2 Peer B (IKEv1 Policy-Based)

```cisco
hostname R2-IKEv1

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
!  RUTA DEFAULT HACIA ISP
! ════════════════════════════════════════════════════════════
ip route 0.0.0.0 0.0.0.0 200.0.0.5

! ════════════════════════════════════════════════════════════
!  FASE 1 — ISAKMP POLICY
! ════════════════════════════════════════════════════════════
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

! PSK para autenticar con el peer remoto (R1)
crypto isakmp key Cisco12345 address 200.0.0.2

! ════════════════════════════════════════════════════════════
!  FASE 2 — TRANSFORM SET
! ════════════════════════════════════════════════════════════
crypto ipsec transform-set TS-IKEv1 esp-aes 256 esp-sha256-hmac
 mode tunnel

! ════════════════════════════════════════════════════════════
!  CRYPTO ACL — TRÁFICO INTERESANTE (inverso a R1)
! ════════════════════════════════════════════════════════════
ip access-list extended ACL-VPN-IKEv1
 permit ip 10.10.2.0 0.0.0.255 10.10.1.0 0.0.0.255

! ════════════════════════════════════════════════════════════
!  CRYPTO MAP
! ════════════════════════════════════════════════════════════
crypto map CMAP-IKEv1 10 ipsec-isakmp
 set peer 200.0.0.2
 set transform-set TS-IKEv1
 set pfs group14
 match address ACL-VPN-IKEv1

! ════════════════════════════════════════════════════════════
!  APLICAR CRYPTO MAP A INTERFAZ WAN
! ════════════════════════════════════════════════════════════
interface e0/0
 crypto map CMAP-IKEv1
```

---

##  Verificación

### Generar tráfico interesante desde PC1
```
PC1> ping 10.10.2.100
```

Salida esperada:
```
10.10.2.100 icmp_seq=1 timeout
10.10.2.100 icmp_seq=2 timeout
10.10.2.100 icmp_seq=3 timeout
84 bytes from 10.10.2.100 icmp_seq=4   ← túnel activo 
```

### Verificar Fase 1 — ISAKMP SA
```cisco
show crypto isakmp sa
```
Salida esperada:
```
dst             src             state     conn-id status
200.0.0.6       200.0.0.2       QM_IDLE      1001 ACTIVE
```

### Verificar Fase 2 — IPSec SA
```cisco
show crypto ipsec sa
```

### Verificar paquetes cifrados/descifrados
```cisco
show crypto ipsec sa | include encaps|decaps
```
Salida esperada:
```
#pkts encaps: 5, #pkts encrypt: 5, #pkts digest: 5
#pkts decaps: 5, #pkts decrypt: 5, #pkts verify: 5
```

### Verificar crypto map aplicado a la interfaz
```cisco
show crypto map
```

### Verificar política ISAKMP
```cisco
show crypto isakmp policy
```

### Verificar clave PSK
```cisco
show crypto isakmp key
```

### Verificar ACL de tráfico interesante
```cisco
show ip access-lists ACL-VPN-IKEv1
```

---

##  Diferencia Policy-Based vs Route-Based

| Aspecto | Policy-Based | Route-Based (VTI) |
|---|---|---|
| Selección de tráfico | Crypto ACL | Tabla de enrutamiento |
| Interfaz de túnel | No existe | Sí — Tunnel0 (VTI) |
| Enrutamiento dinámico | No nativo | Sí — OSPF, EIGRP, BGP |
| Crypto Map | En interfaz física | IPSec Profile en Tunnel0 |
| Crypto ACL | Obligatoria | No requerida |
| Escalabilidad | Baja | Alta |

---

##  Requisitos

- GNS3
- Cisco IOU L3 (imagen 15.4.1T o superior)
- Cisco IOU L2 (para switches)
- VPCS

---

*Autor: Roberto de Jesus — Matrícula: 2023-0348*
*Fecha: 2025*
