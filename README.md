# 🔐 IPSec IKEv1 — VPN Site-to-Site Basada en Políticas

<div align="center">

**Instituto Tecnológico de las Américas — ITLA**  
Seguridad de Redes · Prof. Jonathan Esteban Rondón Corniel  
**Arlene Fernández Herrera · Matrícula: 2025-0730**

![IPSec](https://img.shields.io/badge/Protocolo-IPSec%20IKEv1-blue?style=for-the-badge&logo=cisco)
![Type](https://img.shields.io/badge/Tipo-Site--to--Site-brightgreen?style=for-the-badge)
![Mode](https://img.shields.io/badge/Modo-Policy--Based-orange?style=for-the-badge)
![Platform](https://img.shields.io/badge/Plataforma-PNetLab-blueviolet?style=for-the-badge)

</div>

---

## 📑 Tabla de Contenidos

1. [Objetivo](#1-objetivo)
2. [Topología](#2-topología)
3. [Direccionamiento IP](#3-direccionamiento-ip)
4. [Parámetros Configurados](#4-parámetros-configurados)
5. [Scripts de Configuración](#5-scripts-de-configuración)
6. [Verificación del Túnel](#6-verificación-del-túnel)
7. [Capturas de Pantalla](#7-capturas-de-pantalla)
8. [Video Demostrativo](#8-video-demostrativo)
9. [Referencias](#9-referencias)

---

## 1. Objetivo

Implementar y verificar una **VPN Site-to-Site basada en políticas** utilizando **IPSec con IKEv1** en routers Cisco IOS dentro de PNetLab. La práctica cubre:

- Negociación del canal seguro IKEv1 en **dos fases**: ISAKMP (Fase 1) e IPSec SA (Fase 2).
- Protección criptográfica del tráfico entre `202.50.73.128/25` (Site A) y `202.50.73.0/25` (Site B) a través de una red pública simulada (`192.168.19.0/24`).
- Uso de **ACL de tráfico interesante** + **Crypto Map** como mecanismo selector de flujos a cifrar (enfoque *policy-based*).
- Validación del túnel con comandos `show` nativos de IOS y pruebas de conectividad extremo a extremo.

### ¿Por qué Policy-Based?

En una VPN basada en políticas el router no crea una interfaz de túnel virtual; en su lugar consulta una ACL para decidir qué tráfico debe cifrarse antes de salir por la interfaz WAN física. Es el modelo más compatible entre distintos fabricantes y el más sencillo para implementaciones punto a punto sin necesidad de enrutamiento dinámico sobre el túnel.

```
PC1 → R1 → [¿Coincide con ACL?] → SÍ → [Cifra con IPSec] → ISP → R2 → [Descifra] → PC2
                                  ↓ NO
                          Tráfico sale sin cifrar
```

---

## 2. Topología

```
                              [ INTERNET / ISP ]
                               192.168.19.0/24
                              (Cloud/NAT: 192.168.19.2)
                                      │
                  ┌───────────────────┴───────────────────┐
                  │ e0/0: 192.168.19.5                     │ e0/0: 192.168.19.6
          ┌───────┴────────┐                      ┌────────┴───────┐
          │   R1 (Peer A)  │◄══════ TÚNEL ════════►  R2 (Peer B)  │
          │                │     IPSec IKEv1       │               │
          └───────┬────────┘    Policy-Based       └────────┬──────┘
                  │ e0/1: 202.50.73.129/25                  │ e0/1: 202.50.73.1/25
                  │                                         │
          ┌───────┴────────┐                      ┌─────────┴──────┐
          │     SW1        │                      │      SW2       │
          └───────┬────────┘                      └────────┬───────┘
                  │                                        │
          ┌───────┴────────┐                      ┌────────┴───────┐
          │      PC1       │                      │      PC2       │
          │ 202.50.73.130  │                      │  202.50.73.2   │
          └────────────────┘                      └────────────────┘
           ◄── SITE A ──►                          ◄── SITE B ──►
           202.50.73.128/25                        202.50.73.0/25
```

> El túnel IPSec se establece entre `192.168.19.5` (R1 e0/0) y `192.168.19.6` (R2 e0/0).  
> El tráfico que viaja sobre el segmento ISP va completamente **cifrado con ESP**.

**Flujo de establecimiento del túnel:**
1. PC1 intenta comunicarse con PC2 → R1 detecta tráfico interesante (ACL match).
2. R1 inicia **IKE Fase 1 (Main Mode)** con R2 → se establece la ISAKMP SA.
3. Usando ese canal seguro, **IKE Fase 2 (Quick Mode)** negocia las IPSec SAs.
4. El tráfico viaja cifrado por el segmento ISP → R2 desencapsula y entrega a PC2.

---

## 3. Direccionamiento IP

### Interfaces de Dispositivos

| Dispositivo | Interfaz | Dirección IP       | Máscara | Gateway         | Rol                    |
|-------------|----------|--------------------|---------|-----------------|------------------------|
| Cloud (NAT) | —        | 192.168.19.2       | /24     | —               | Gateway Cloud NAT (PNetLab)   |
| **R1**      | **e0/0** | **192.168.19.5**   | **/24** | 192.168.19.2    | WAN Peer A → Cloud     |
| **R1**      | **e0/1** | **202.50.73.129**  | **/25** | —               | Gateway LAN Site A     |
| **R2**      | **e0/0** | **192.168.19.6**   | **/24** | 192.168.19.2    | WAN Peer B → Cloud     |
| **R2**      | **e0/1** | **202.50.73.1**    | **/25** | —               | Gateway LAN Site B     |
| SW1         | —        | —                  | —       | —               | Capa 2 Site A          |
| SW2         | —        | —                  | —       | —               | Capa 2 Site B          |
| PC1         | eth0     | 202.50.73.130      | /25     | 202.50.73.129   | Host Site A            |
| PC2         | eth0     | 202.50.73.2        | /25     | 202.50.73.1     | Host Site B            |

### Tabla de Subredes

| Subred              | Rango Utilizable                  | Broadcast        | Uso              |
|---------------------|-----------------------------------|------------------|------------------|
| `192.168.19.0/24`   | 192.168.19.1 – 192.168.19.254     | 192.168.19.255   | Segmento WAN/ISP |
| `202.50.73.0/25`    | 202.50.73.1 – 202.50.73.126       | 202.50.73.127    | LAN Site B       |
| `202.50.73.128/25`  | 202.50.73.129 – 202.50.73.254     | 202.50.73.255    | LAN Site A       |

> Las subredes LAN se derivan de la matrícula **2025-0730** → `202.50.73.x`, dividida en dos mitades `/25`.

---

## 4. Parámetros Configurados

### IKE Fase 1 — ISAKMP Policy

| Parámetro           | Valor               | Descripción                                             |
|---------------------|---------------------|---------------------------------------------------------|
| Número de política  | `10`                | Prioridad de la política ISAKMP                         |
| Cifrado             | AES-256             | Cifrado simétrico del canal IKE                         |
| Hash / Integridad   | SHA-256             | Verificación de integridad de mensajes IKE              |
| Autenticación       | Pre-Shared Key      | Clave compartida entre ambos peers                      |
| Grupo DH            | Group 14 (2048-bit) | Intercambio Diffie-Hellman para derivar clave de sesión |
| Lifetime SA         | 86400 s (24 h)      | Duración del canal IKE antes de renegociar              |
| Pre-Shared Key      | `ITLA2025Arlene`    | Clave idéntica en R1 y R2                               |

### IKE Fase 2 — IPSec / Transform Set

| Parámetro       | Valor               | Descripción                                          |
|-----------------|---------------------|------------------------------------------------------|
| Nombre TS       | `TS_AES256_SHA256`  | Identificador del Transform Set                      |
| Cifrado ESP     | ESP-AES 256         | Cifrado + encapsulamiento del payload de datos       |
| Integridad ESP  | ESP-SHA256-HMAC     | Verificación de integridad del tráfico de datos      |
| Modo            | Tunnel              | Encapsula el paquete IP completo (gateway-to-gateway) |
| Lifetime SA     | 3600 s (1 h)        | Duración del túnel de datos antes de renegociar      |

### Tráfico Interesante (ACL Policy-Based)

| Router | ACL                       | Fuente                | Destino               |
|--------|---------------------------|-----------------------|-----------------------|
| R1     | `ACL_VPN_SITEA_TO_SITEB`  | `202.50.73.128/25`    | `202.50.73.0/25`      |
| R2     | `ACL_VPN_SITEB_TO_SITEA`  | `202.50.73.0/25`      | `202.50.73.128/25`    |

> Las ACLs deben ser **espejo exacto** entre sí — fuente y destino invertidos. Si no coinciden, la Fase 2 no se establece.

---

## 5. Scripts de Configuración

Los archivos completos están en [`scripts/`](./scripts/):

| Archivo | Dispositivo | Contenido |
|---|---|---|
| [`R1_config.txt`](./scripts/R1_config.txt) | R1 — Site A | Config completa del peer A |
| [`R2_config.txt`](./scripts/R2_config.txt) | R2 — Site B | Config completa del peer B |
| [`ISP_config.txt`](./scripts/ISP_config.txt) | ISP | Config básica del router ISP |

### R1 — Site A (fragmento comentado)

```cisco
! ══════════════════════════════════════════════════════════════
!  R1 — Site A | IPSec IKEv1 Policy-Based VPN
!  Arlene Fernández Herrera · 2025-0730 | Seguridad de Redes
! ══════════════════════════════════════════════════════════════

hostname R1

! ── Interfaces ──────────────────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-ISP
 ip address 192.168.19.5 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-SiteA
 ip address 202.50.73.129 255.255.255.128
 no shutdown

ip route 0.0.0.0 0.0.0.0 192.168.19.2

! ── Paso 1: ISAKMP Policy (IKE Fase 1) ─────────────────────
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key ITLA2025Arlene address 192.168.19.6

! ── Paso 2: Transform Set (IKE Fase 2) ──────────────────────
crypto ipsec transform-set TS_AES256_SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel

! ── Paso 3: ACL de tráfico interesante ──────────────────────
ip access-list extended ACL_VPN_SITEA_TO_SITEB
 permit ip 202.50.73.128 0.0.0.127 202.50.73.0 0.0.0.127

! ── Paso 4: Crypto Map ──────────────────────────────────────
crypto map CMAP_SITEA 10 ipsec-isakmp
 set peer 192.168.19.6
 set transform-set TS_AES256_SHA256
 match address ACL_VPN_SITEA_TO_SITEB
 set security-association lifetime seconds 3600

! ── Paso 5: Aplicar en interfaz WAN ─────────────────────────
interface Ethernet0/0
 crypto map CMAP_SITEA
```

### R2 — Site B (fragmento comentado)

```cisco
! ══════════════════════════════════════════════════════════════
!  R2 — Site B | IPSec IKEv1 Policy-Based VPN
!  Arlene Fernández Herrera · 2025-0730 | Seguridad de Redes
! ══════════════════════════════════════════════════════════════

hostname R2

! ── Interfaces ──────────────────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-ISP
 ip address 192.168.19.6 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-SiteB
 ip address 202.50.73.1 255.255.255.128
 no shutdown

ip route 0.0.0.0 0.0.0.0 192.168.19.2

! ── Paso 1: ISAKMP Policy (IKE Fase 1) ─────────────────────
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key ITLA2025Arlene address 192.168.19.5

! ── Paso 2: Transform Set (IKE Fase 2) ──────────────────────
crypto ipsec transform-set TS_AES256_SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel

! ── Paso 3: ACL de tráfico interesante (espejo de R1) ───────
ip access-list extended ACL_VPN_SITEB_TO_SITEA
 permit ip 202.50.73.0 0.0.0.127 202.50.73.128 0.0.0.127

! ── Paso 4: Crypto Map ──────────────────────────────────────
crypto map CMAP_SITEB 10 ipsec-isakmp
 set peer 192.168.19.5
 set transform-set TS_AES256_SHA256
 match address ACL_VPN_SITEB_TO_SITEA
 set security-association lifetime seconds 3600

! ── Paso 5: Aplicar en interfaz WAN ─────────────────────────
interface Ethernet0/0
 crypto map CMAP_SITEB
```

### Hosts (VPCs en PNetLab)

```bash
# PC1 — Site A
ip 202.50.73.130 255.255.255.128 202.50.73.129

# PC2 — Site B
ip 202.50.73.2 255.255.255.128 202.50.73.1
```

---

## 6. Verificación del Túnel

### Estado IKE Fase 1 — ISAKMP SA

```cisco
R1# show crypto isakmp sa
```

Salida esperada con túnel activo:

```
IPv4 Crypto ISAKMP SA
dst              src              state       conn-id  status
192.168.19.6     192.168.19.5     QM_IDLE     1001     ACTIVE
```

> `QM_IDLE` = Fase 1 completada exitosamente. Si aparece `MM_NO_STATE`, hay incompatibilidad en la política ISAKMP (verificar cifrado, hash y grupo DH en ambos routers).

---

### Estado IPSec Fase 2 — IPSec SA

```cisco
R1# show crypto ipsec sa
```

Salida esperada (fragmento):

```
interface: Ethernet0/0
    Crypto map tag: CMAP_SITEA, local addr 192.168.19.5

   local  ident: (202.50.73.128/255.255.255.128/0/0)
   remote ident: (202.50.73.0/255.255.255.128/0/0)
   current_peer: 192.168.19.6 port 500

    #pkts encaps: 10, #pkts encrypt: 10, #pkts digest: 10
    #pkts decaps: 10, #pkts decrypt: 10, #pkts verify: 10
```

> Los contadores `#pkts encaps` y `#pkts decaps` deben incrementarse con cada ping. Si permanecen en 0, el tráfico no está siendo capturado por la ACL.

---

### Conectividad extremo a extremo

```bash
# Desde PC1 hacia PC2 (con túnel activo)
PC1> ping 202.50.73.2

# Desde R1 — ping extendido
R1# ping 202.50.73.2 source 202.50.73.129 repeat 10
```

Resultado esperado:

```
!!!!!!!!!! 
Success rate is 100 percent (10/10)
```

> Un TTL de 62 en los pings desde PC1→PC2 confirma que el tráfico atraviesa los 2 routers correctamente.

---

### Tabla de Comandos de Verificación

| Comando | Qué verifica |
|---|---|
| `show crypto isakmp sa` | Estado de IKE Fase 1. Debe mostrar `QM_IDLE`. |
| `show crypto ipsec sa` | SAs de Fase 2 activas, SPIs y contadores de paquetes. |
| `show crypto isakmp policy` | Parámetros configurados en la política ISAKMP. |
| `show crypto map` | Crypto Maps aplicadas y sus parámetros. |
| `show crypto session` | Resumen rápido del estado de todas las sesiones IPSec. |
| `show ip access-lists ACL_VPN_SITEA_TO_SITEB` | Hits en la ACL de tráfico interesante. |

---

### Troubleshooting Rápido

| Síntoma | Causa probable | Solución |
|---|---|---|
| `MM_NO_STATE` en Fase 1 | Parámetros ISAKMP no coinciden | Verificar que R1 y R2 tengan exactamente la misma política (`show crypto isakmp policy`) |
| Fase 1 OK pero Fase 2 no sube | ACLs no son espejo exacto | La ACL de R2 debe tener src/dst invertidos respecto a R1 |
| SAs activas pero pings fallan | Ruta faltante o NAT interceptando | Confirmar `ip route` y excluir tráfico VPN del NAT si aplica |
| Contadores en 0 pese a pings | ACL no hace match | Revisar wildcards en la ACL con `show ip access-lists` |

---

## 7. Capturas de Pantalla

Las evidencias se almacenan en la carpeta [`screenshots/`](./screenshots/):

| # | Captura | Descripción |
|---|---|---|
| 1 | [Topología general](evidencias/01_topologia.png) | Topología en PNetLab con nombre y matrícula visibles, todos los nodos encendidos. |
| 2 | [Fecha y hora del sistema](evidencias/02_fecha_hora.png) | Reloj del sistema operativo visible mostrando fecha y hora actuales. |
| 3 | [Config R1 – ISAKMP](evidencias/03_config_r1_isakmp.png) | Consola R1: `crypto isakmp policy 10` y `crypto isakmp key` configurados. |
| 4 | [Config R1 – Fase 2](evidencias/04_config_r1_fase2.png) | Consola R1: transform set, ACL de tráfico interesante y Crypto Map. |
| 5 | [Config R2 – Completa](evidencias/05_config_r2_completa.png) | Consola R2: configuración equivalente con ACL espejo y peer apuntando a R1. |
| 6 | [ISAKMP SA – QM_IDLE](evidencias/06_isakmp_sa_qmidle.png) | Salida `show crypto isakmp sa` → estado `QM_IDLE` en R1. |
| 7 | [IPSec SA – Contadores](evidencias/07_ipsec_sa_contadores.png) | Salida `show crypto ipsec sa` → contadores `encaps/decaps` con tráfico activo. |
| 8 | [Ping exitoso](evidencias/08_ping_exitoso.png) | Ping exitoso de PC1 (`202.50.73.130`) a PC2 (`202.50.73.2`) con túnel activo. |

---

## 8. Video Demostrativo

🎥 **[Ver en YouTube — enlace pendiente](#)**

**Duración estimada:** < 8 minutos

Contenido del video:
- ✅ Topología en PNetLab con nombre completo **Arlene Fernández Herrera** y matrícula **2025-0730** visibles.
- ✅ Reloj del sistema visible con fecha y hora actuales.
- ✅ Cara y voz de la autora durante toda la demostración.
- ✅ Aplicación de los scripts de configuración en R1 y R2.
- ✅ `show crypto isakmp sa` mostrando `QM_IDLE`.
- ✅ `show crypto ipsec sa` con contadores de paquetes incrementando.
- ✅ Ping exitoso entre PC1 y PC2 a través del túnel cifrado.
- ✅ Demostración mediante GUI de PNetLab.

---

## 9. Referencias

- Kent, S. & Seo, K. (2005). *RFC 4301 — Security Architecture for the Internet Protocol*. IETF.
- Harkins, D. & Carrel, D. (1998). *RFC 2409 — The Internet Key Exchange (IKEv1)*. IETF.
- Cisco Systems. (2024). *Cisco IOS Security Configuration Guide — IPSec VPN*.
- Cisco Systems. (2024). *Cisco IOS Security Command Reference — crypto isakmp / crypto map*.

---

<div align="center">

**Arlene Fernández Herrera · 2025-0730 · ITLA**  
*Seguridad de Redes — Lab 01: IPSec IKEv1 Policy-Based VPN*

</div>
