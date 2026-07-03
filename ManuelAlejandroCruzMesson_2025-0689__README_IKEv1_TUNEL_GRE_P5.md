# VPN Site-to-Site GRE sobre IPsec (IKEv1)

**Asignatura:** Seguridad de Redes
**Estudiante:** Manuel Cruz
**Docente:** Jonathan Rondón
**Fecha:** 2 de julio de 2026

> **Nota:** el direccionamiento no se realizó en base a la matrícula del estudiante; este requerimiento se recordó demasiado tarde y no fue posible rehacer las topologías y videos.

---

## Tabla de Contenidos

- [1. Resumen y Objetivos](#1-resumen-y-objetivos)
- [2. Topología de Red y Direccionamiento](#2-topología-de-red-y-direccionamiento)
- [3. Especificación de Políticas y Parámetros de Seguridad (IKEv1 / ISAKMP)](#3-especificación-de-políticas-y-parámetros-de-seguridad-ikev1--isakmp)
- [4. Configuración de la Interfaz de Túnel GRE Protegido](#4-configuración-de-la-interfaz-de-túnel-gre-protegido)
- [5. Puntos Críticos Analizados](#5-puntos-críticos-analizados)
- [6. Protocolo de Verificación y Diagnóstico Técnico](#6-protocolo-de-verificación-y-diagnóstico-técnico)

---

## 1. Resumen y Objetivos

Este documento describe la evolución tecnológica y la reestructuración del laboratorio original de VPN Site-to-Site. Se migra la arquitectura de túnel IPsec puro (VTI nativo) hacia un esquema de encapsulamiento **GRE (Generic Routing Encapsulation)** protegido mediante IPsec en **modo transporte**, utilizando el protocolo heredado **IKEv1 (ISAKMP)** para el establecimiento de la Fase 1. Se mantiene intacto el direccionamiento IP de la infraestructura física del proyecto original.

### Objetivos del Proyecto

- **Encapsulamiento Flexible mediante GRE:** implementar un túnel GRE sobre la interfaz `Tunnel0` para permitir el transporte de tráfico de forma flexible (incluyendo soporte potencial a protocolos de enrutamiento dinámico y tráfico multicast), delegando la protección criptográfica a un perfil IPsec en modo transporte.
- **Confidencialidad e Integridad Avanzada:** garantizar la protección estricta del flujo de datos entre la LAN de PEER A (`172.16.1.0/24`) y la LAN de PEER B (`172.16.2.0/24`) mediante el protocolo ISAKMP (IKEv1) para el intercambio de llaves y algoritmos de cifrado simétrico robustos (AES-256 / SHA-256) para el cifrado del payload GRE.
- **Coexistencia de Servicios:** asegurar el correcto funcionamiento del enrutamiento estático hacia la interfaz `Tunnel0` y el aislamiento criptográfico frente a la traducción de direcciones de red (NAT/PAT) mediante el ajuste de la lista de acceso `ACL_NAT_INTERNET`.

---

## 2. Topología de Red y Direccionamiento

La topología lógica mantiene el diseño de dos sedes principales interconectadas a través de un gateway WAN que emula un proveedor de servicios de Internet (R-ISP). La interfaz lógica `Tunnel0` en cada extremo actúa ahora como cabecera de encapsulamiento GRE, protegida en su totalidad por el perfil IPsec en modo transporte.

| Dispositivo / Sede | Interfaz | Dirección IP | Máscara de Subred | Propósito / Rol |
|---|---|---|---|---|
| PEER A | Gi0/0 | 10.0.0.60 | 255.255.255.0 | Enlace WAN (IP Pública) / NAT Outside |
| PEER A | Gi0/1 | 172.16.1.1 | 255.255.255.0 | Gateway LAN A / NAT Inside |
| PEER A | Tunnel0 | 192.168.100.1 | 255.255.255.252 | Interfaz GRE protegida por IPsec |
| PEER B | Gi0/0 | 10.0.0.70 | 255.255.255.0 | Enlace WAN (IP Pública) / NAT Outside |
| PEER B | Gi0/1 | 172.16.2.1 | 255.255.255.0 | Gateway LAN B / NAT Inside |
| PEER B | Tunnel0 | 192.168.100.2 | 255.255.255.252 | Interfaz GRE protegida por IPsec |
| R-ISP (Gateway) | N/A | 10.0.0.1 | 255.255.255.0 | Puerta de enlace predeterminada WAN |

---

## 3. Especificación de Políticas y Parámetros de Seguridad (IKEv1 / ISAKMP)

A diferencia de la suite basada en IKEv2, la presente implementación retorna al modelo heredado ISAKMP para la Fase 1, y delega el transporte del tráfico interesante a una encapsulación GRE previa, siendo IPsec el encargado exclusivo de cifrar el payload GRE en modo transporte.

### Ajuste de la Lista de Exclusión de NAT — PEER A

> En PEER B la entrada equivalente utiliza las redes `172.16.2.0/24` y `172.16.1.0/24` respectivamente, garantizando la exclusión bidireccional del tráfico inter-LAN del proceso de NAT hacia Internet.

### Fase 1: ISAKMP Policy y Pre-Shared Key — PEER A

| Parámetro | Valor |
|---|---|
| Algoritmo de Cifrado | AES de 256 bits (encr aes 256) |
| Integridad y Hash | SHA-256 (hash sha256) para los procesos de autenticación e intercambio seguro |
| Grupo Diffie-Hellman | Grupo 14 (Exponenciación modular de 2048 bits) para la generación segura de claves efímeras |
| Autenticación | Llaves precompartidas (pre-share), declaradas mediante el comando `crypto isakmp key` asociado directamente a la IP del peer remoto (sin uso de keyring modular) |
| Lifetime | 86,400 segundos (24 horas), tiempo de vigencia de la asociación de seguridad de Fase 1 |
| Pre-Shared Key | `CLAVE_SECRETA_VPN` |

### Fase 2: IPsec Transform Set en Modo Transporte

| Parámetro | Valor |
|---|---|
| Nombre del Set | `TS_GRE` |
| Encapsulación Criptográfica | ESP con AES de 256 bits (esp-aes 256) |
| Autenticación/Hashing de Datos | ESP bajo código de autenticación de mensajes cifrados (esp-sha256-hmac) |
| Modo de Operación | Modo Transporte (mode transport). A diferencia del modo túnel, no se encapsula una nueva cabecera IP completa; IPsec cifra únicamente el payload, dado que la cabecera de enrutamiento ya es provista por el encapsulamiento GRE externo |
| Perfil IPsec | `PROFILE_GRE`, el cual vincula el transform-set y se aplica directamente sobre la interfaz `Tunnel0` mediante `tunnel protection` |

---

## 4. Configuración de la Interfaz de Túnel GRE Protegido

El elemento central de esta migración es la interfaz `Tunnel0` configurada como túnel GRE estándar (sin declarar `tunnel mode ipsec ipv4`), sobre la cual se aplica la protección IPsec en modo transporte mediante el comando `tunnel protection ipsec profile`. Esto asegura que todo el tráfico encapsulado en GRE sea cifrado antes de salir por la interfaz física WAN.

- **PEER A** — Interfaz Tunnel0 y Enrutamiento
- **PEER B** — Interfaz Tunnel0 y Enrutamiento

---

## 5. Puntos Críticos Analizados

### A. Diferencia entre VTI Nativo (IPsec Tunnel Mode) y GRE Protegido (IPsec Transport Mode)

En la arquitectura anterior (VTI con IKEv2), la propia interfaz `Tunnel0` operaba en `tunnel mode ipsec ipv4`, por lo que IPsec era responsable tanto del cifrado como del encapsulamiento IP completo (modo túnel). En el presente esquema, GRE asume el rol de encapsulamiento (agregando su propia cabecera GRE + IP externa), mientras que IPsec, operando en modo transporte, se limita a cifrar el payload ya encapsulado. Esto reduce el overhead de doble encabezado IP, pero requiere que ambos protocolos (GRE e IPsec) trabajen de forma coordinada mediante `tunnel protection`.

### B. Ajuste de la Lista de Exclusión de NAT (NAT Bypass)

Al denegar el tráfico inter-LAN del proceso de traducción de direcciones (`ACL_NAT_INTERNET`), los paquetes retienen sus cabeceras e IPs de origen originales intactas antes de ser encapsulados por GRE. Esto permite que el tráfico sea correctamente enrutado hacia la interfaz `Tunnel0` sin ser traducido por la política NAT/PAT hacia Internet.

### C. Retorno al Modelo ISAKMP (IKEv1) para Fase 1

A diferencia de la suite modular basada en `crypto ikev2 proposal / policy / keyring / profile`, el modelo ISAKMP agrupa los parámetros de Fase 1 en una única directiva numerada (`crypto isakmp policy 10`) y asocia la llave precompartida directamente a la IP del peer mediante `crypto isakmp key`, sin la posibilidad de perfiles de identidad múltiples propios de IKEv2.

---

## 6. Protocolo de Verificación y Diagnóstico Técnico

### Paso 1: Generación de Tráfico Interesante para Levantamiento de Túnel

Por definición, los mecanismos IPsec levantan las asociaciones de seguridad bajo demanda en presencia del primer paquete de datos legítimo que atraviese la interfaz `Tunnel0`. Para activar la infraestructura, se emite una solicitud de eco ICMP desde un host terminal interno con destino a la LAN remota.

### Paso 2: Validación del Canal de Control en Fase 1 (ISAKMP SA)

Para examinar el estado del intercambio de llaves y la correcta concordancia criptográfica de los peers remotos bajo IKEv1, se emplea el comando de diagnóstico:

```
Router# show crypto isakmp sa
```

**Criterio de Aceptación:** el resultado en consola debe declarar de forma explícita el estado **QM_IDLE** en la columna de estado (State). Esto certifica que la negociación en Modo Rápido (Quick Mode) ha culminado con éxito y la SA de Fase 1 se encuentra activa en espera de tráfico.

### Paso 3: Validación del Canal de Datos en Fase 2 (IPsec SA sobre GRE)

Una vez establecido el enlace de control, es indispensable auditar que los flujos de datos GRE sean procesados por los algoritmos de encriptación simétrica ESP en modo transporte:

```
Router# show crypto ipsec sa | include pkts
```

Los registros y contadores del sistema para `#pkts encaps` (paquetes cifrados salientes) y `#pkts decaps` (paquetes descifrados entrantes) deben mostrar valores enteros mayores a cero e incrementar en tiempo real conforme persista el envío de ráfagas de datos a través del túnel GRE.
