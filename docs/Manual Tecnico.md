# Proyecto 2 - Infraestructura Financiera BanTech GT
## Redes de Computadoras 1 — Segundo Semestre 2026

**Coordinador:** Carlos Hugo Rios Mancilla, 9520488
**Integrante 1:** Yury Urbano Santos Chagil, 202113318
**Integrante 2:** Javier Estuardo Oliva Santos, 202200082
**Grupo:** 7  
**Parámetros:** XX = 88, Y = 7 (Dominio VTP: `bantech88`)  
**Repositorio:** [Enlace a tu GitHub]

---

## Índice
1. [Objetivos y Preimplementación](#1-objetivos-y-preimplementación)
2. [Direccionamiento IP y Subnetting (VLSM)](#2-direccionamiento-ip-y-subnetting-vlsm)
3. [Implementación y Resultados (Por Sede)](#3-implementación-y-resultados-por-sede)
4. [Configuración de Alta Disponibilidad](#4-configuración-de-alta-disponibilidad)
5. [Conclusiones y Pruebas de Validación](#5-conclusiones-y-pruebas-de-validación)
6. [Tablas de Referencia y Presupuesto](#6-tablas-de-referencia-y-presupuesto)

## 1. Objetivos y Preimplementación
La red de BanTech GT se diseñó bajo una arquitectura jerárquica y redundante, dividida en un **Backbone Core** y 4 sedes de borde (Occidente, Norte, Oriente y Data Center/Central). El objetivo principal fue garantizar conectividad, segmentación y alta disponibilidad.

**Decisiones de Diseño:**
*   **Backbone:** Núcleo redundante con 3 dispositivos de Capa 3 (CORE-1, CORE-2, CORE-3). Interconectados por Fibra, Cobre y Serial.
*   **Enrutamiento:** OSPF en el núcleo. Redistribución hacia RIPv2 (Occidente), EIGRP (Norte) y Rutas Estáticas (Data Center).
*   **Alta Disponibilidad:** Rapid PVST+ en Norte (triángulo de switches), HSRP en Oriente (MS1/MS2) y EtherChannel LACP en el Data Center.
*   **Segmentación:** Uso del último dígito del grupo (Y = 7) para las VLANs, asegurando el aislamiento del tráfico y la seguridad.

## 2. Direccionamiento IP y Subnetting (VLSM)
Para el direccionamiento se utilizó la red base `10.0.0.0/8` para los enlaces del Backbone, y subredes específicas para cada sede, optimizando el espacio con VLSM (Variable Length Subnet Mask).

**Enlaces del Backbone Core:**
| Enlace | Red IP | Máscara | IPs Asignadas |
| :--- | :--- | :--- | :--- |
| Fibra (CORE-1 ↔ CORE-2) | 10.0.0.0/30 | 255.255.255.252 | 10.0.0.1 / 10.0.0.2 |
| Serial (CORE-1 ↔ CORE-3) | 10.0.0.4/30 | 255.255.255.252 | 10.0.0.5 / 10.0.0.6 |
| Cobre (CORE-2 ↔ CORE-3) | 10.0.0.8/30 | 255.255.255.252 | 10.0.0.9 / 10.0.0.10 |
| Enlace Occidente | 10.0.0.12/30 | 255.255.255.252 | 10.0.0.13 / 10.0.0.14 |
| Enlace Norte | 10.0.0.16/30 | 255.255.255.252 | 10.0.0.17 / 10.0.0.18 |
| Enlace Oriente | 10.0.0.20/30 | 255.255.255.252 | 10.0.0.21 / 10.0.0.22 |
| Enlace Data Center | 10.0.0.24/30 | 255.255.255.252 | 10.0.0.25 / 10.0.0.26 |

**Subredes de las Sedes y VLANs (Y=7):**
*   **Occidente:** Red `10.7.0.0/24`. VLANs: Cajas (457), Asesores (307), Gerencia (107), Seguridad (127).
*   **Norte:** Red `10.8.0.0/24`. VLANs: Análisis (507), Auditoría (257), Legal (147).
*   **Oriente:** Red `10.9.0.0/24`. VLANs: Bóveda (407), Plataforma (607).
*   **Data Center/Central:** Red `10.10.0.0/24`. VLANs: Web_Apps (288), Core_BD (148), NOC (108), Oficinas (7).

---

## 3. Implementación y Resultados (Por Sede)

> *A continuación se detallan las configuraciones realizadas y validadas en Cisco Packet Tracer.*

### 3.1 Backbone Core
Se configuró OSPF Área 0 para interconectar los 3 routers del núcleo a través de fibra, serial y cobre.
*   **Resultado:** El enrutamiento dinámico converge correctamente y el CORE-1 aprende todas las rutas del país.

![show ip route en el CORE-1](./capturas/show%20ip%20route%20en%20el%20CORE-1.png)

### 3.2 Sede Occidente (RIPv2)
Se implementó el enrutamiento **RIPv2** para esta sede, utilizando la técnica de **Router-on-a-Stick** en el router R-Occidente para el enrutamiento inter-VLAN. 
Para garantizar que el resto de la red nacional aprendiera las rutas de Occidente, se configuró una ruta estática en el CORE-1 hacia la red de Occidente, la cual fue **redistribuida hacia OSPF** (primer punto de redistribución).

*   **Configuración RIPv2 en R-Occidente:**
    ```cisco
    router rip
     version 2
     no auto-summary
     network 10.7.0.0
     network 10.0.0.12
     default-information originate

    ```
*    **Redistribución en CORE-1:**
    ```cisco
    router ospf 1
     redistribute rip subnets
    ```
![show ip route en el R-Occidente](./capturas/show%20run%20|%20section%20router%20rip.png)

### 3.3 Sede Norte (EIGRP)

Se implementó EIGRP (AS 7) para esta sede. Al igual que en Occidente, se utilizó **Router-on-a-Stick** en el **R-Norte**. La integración con el núcleo se realizó mediante redistribución mutua en el **CORE-2** (segundo punto de redistribución).

*   **Configuración EIGRP en R-Norte:**
    ```cisco
    router eigrp 7
     no auto-summary
     network 10.8.0.0 0.0.0.255
     network 10.0.0.16 0.0.0.3
     ```
*   **Redistribución en CORE-2:**
    ```cisco
    router ospf 1
     redistribute eigrp subnets
    ```
![show ip route en el R-Norte](./capturas/show%20ip%20protocols%20en%20el%20CORE-2.png)

### 3.4 Sede Oriente (HSRP y OSPF)

La Sede Oriente se diseñó con alta disponibilidad de Capa 3. Se utilizaron dos switches multicapa (MS1 y MS2) unidos por EtherChannel, con el protocolo HSRP para dar redundancia al Gateway de las VLANs 407 (Bóveda) y 607 (Plataforma). El enrutamiento externo hacia el Core se realizó por OSPF.

*   **Configuración HSRP (MS1-Activo):**
    ```cisco
    interface vlan 407
     ip address 10.9.0.2 255.255.255.192
     standby version 2
     standby 407 ip 10.9.0.1
     standby 407 priority 150
     standby 407 preempt
    ```

*   **OSPF en MS1:**
    ```cisco
    network 10.9.0.0 0.0.0.255 area 0
    ```
![show standby brief en MS1](./capturas/show%20standby%20brief%20en%20MS1.png)

### 3.5 Data Center y Sede Central (Rutas Estáticas y EtherChannel)

El Data Center se aisló mediante rutas estáticas. El **R-Central-1** actúa como punto de redistribución hacia **OSPF** (tercer punto de redistribución) y el **R-Central-2** como router de acceso. Para evitar cuellos de botella en los servidores, se implementó un **EtherChannel (LACP)** entre los switches de acceso y distribución.

*   **Rutas Estáticas (R-Central-1):**
    ```cisco
    ip route 10.10.0.0 255.255.255.192 10.0.0.30
    ip route 10.10.0.64 255.255.255.224 10.0.0.30
    ip route 10.10.0.96 255.255.255.240 10.0.0.30
    ```
*   **Configuración EtherChannel (SW-Central-1):**
    ```cisco
    interface range GigabitEthernet 1/0/10 - 11
     channel-group 1 mode active
     switchport trunk encapsulation dot1q
     switchport mode trunk
    ```
*   **Adición de la Sede Central (Oficinas):** Se agregó la VLAN 7 para las oficinas administrativas, demostrando la escalabilidad de la red.

![show ip route en el R-Central-1](./capturas/show%20etherchannel%20summary%20en%20SW-Central-1.png)

![show ip route en el R-Central-2](./capturas/Las%20Rutas%20del%20Data%20Center%20(O%20E2).png)

## 4. Configuración de Alta Disponibilidad

Para garantizar la continuidad del negocio financiero, se implementaron mecanismos de tolerancia a fallos en las tres capas del modelo OSI:

### 4.1 Capa 2: Rapid PVST+ (Sede Norte)
Se configuró **Rapid PVST+** en el triángulo de switches de la Sede Norte para prevenir bucles de Capa 2. Se forzó al **SW-NORTE-1** como Root Bridge principal para todas las VLANs de la sede (prioridad 4096), asegurando que el tráfico tome la ruta más eficiente.

![show rapid pvst+ en el SW-NORTE-1](./capturas/show%20spanning-tree%20en%20SW-NORTE-1.png)

### 4.2 Capa 2: EtherChannel (Data Center)
Para evitar cuellos de botella en las granjas de servidores, se implementó **EtherChannel con LACP** entre los switches del Data Center (SW-Central-1 y SW-Central-2), agrupando dos enlaces físicos en un solo enlace lógico de mayor capacidad.

![show etherchannel summary en SW-Central-1](./capturas/show%20etherchannel%20summary%20en%20SW-Central-1.png)

### 4.3 Capa 3: HSRP (Sede Oriente)
Se configuró **HSRP versión 2** en los switches multicapa MS1 y MS2 para las VLANs 407 y 607. El MS1 actúa como Gateway Activo y el MS2 como Standby, compartiendo una IP Virtual.

![show standby brief en MS1](./capturas/show%20standby%20brief%20en%20MS1.png)

![show standby brief en MS2](./capturas/show%20standby%20brief%20en%20MS2.png)

### 4.4 Capa 3: Redundancia en el Backbone (OSPF)
El núcleo de la red está protegido por OSPF. Si un enlace del CORE falla, OSPF reconverge automáticamente y envía el tráfico por el camino alterno (ej. de Fibra a Serial).

![show ip route en el R-Central-1](./capturas/show%20ip%20ospf%20neighbor%20en%20CORE-1%204.4.png)

---

## 5. Conclusiones y Pruebas de Validación

El proyecto cumplió exitosamente con el 100% de los requerimientos del enunciado. La red de BanTech GT está completamente operativa, convergida y es tolerante a fallos.

### 5.1 Prueba de Convergencia Total (Ping Extremo a Extremo)
Se demostró que todas las sedes pueden comunicarse entre sí, validando la correcta redistribución de rutas entre los distintos protocolos (OSPF, EIGRP, RIP y Estáticas). Se realizaron pings desde la Sede Central (Data Center) hacia las 3 sedes regionales.

![show ip route en el R-Central-1](./capturas/ping%20desde%20la%20PC%20de%20Oficinas%20(10.10.0.130)%20hacia%20Occidente%20(10.7.0.10).png)

![show ip route en el R-Central-1](./capturas/Ping%20desde%20Web_Apps%20a%20Norte.png)

### 5.2 Prueba de Failover HSRP (Sede Oriente)
Se realizó un ping continuo desde una PC de la VLAN 407 (Bóveda) y se procedió a apagar la interfaz del Gateway en el MS1.
*   **Resultado:** El MS2 asumió el rol de Activo en milisegundos, y el ping continuó respondiendo sin que el usuario final perdiera la conexión a la red nacional.

![show ip route en el MS1](./capturas/Ping%20%20de%20Boveda%20a%20Server.png)

### 5.3 Prueba de Failover de EtherChannel (Data Center)
Se desconectó uno de los dos cables físicos del EtherChannel en el Data Center.
*   **Resultado:** El tráfico siguió fluyendo por el cable restante sin interrupciones, demostrando la resiliencia de la capa de acceso.

![show ip route en el R-Central-1](./capturas/Ping%20de%20PC2%20a%20Server.png)
---

## 6. Tablas de Referencia y Presupuesto

### 6.1 Tabla de VLANs
| Sede | VLAN ID | Nombre | Función |
| :--- | :--- | :--- | :--- |
| Occidente | 457 | Cajas | Transacciones críticas |
| Occidente | 307 | Asesores | Servicio al cliente |
| Occidente | 107 | Gerencia | Administración local |
| Occidente | 127 | Seguridad | Cámaras y biométricos |
| Norte | 507 | Análisis | Analistas de riesgo |
| Norte | 257 | Auditoría | Revisión de cuentas |
| Norte | 147 | Legal | Contratos |
| Oriente | 407 | Bóveda | Operaciones de valores |
| Oriente | 607 | Plataforma | Atención VIP |
| Data Center | 148 | Core_BD | Servidores de BD |
| Data Center | 288 | Web_Apps | Banca virtual |
| Data Center | 108 | NOC | Monitoreo central |
| Sede Central | 7 | Oficinas | Administración |

### 6.2 Presupuesto Estimado
| Equipo | Cantidad | Precio Unitario (USD) | Total (USD) |
| :--- | :--- | :--- | :--- |
| Router ISR 4331 | 2 | $1,500.00 | $3,000.00 |
| Router 2911 | 4 | $800.00 | $3,200.00 |
| Switch 3650-24PS | 2 | $2,000.00 | $4,000.00 |
| Switch 3560-24PS | 3 | $1,200.00 | $3,600.00 |
| Switch 2960 | 4 | $400.00 | $1,600.00 |
| Cableado y Módulos | - | $2,500.00 | $2,500.00 |
| **TOTAL ESTIMADO** | | | **$17,900.00** |

---
**Fin del documento. Proyecto validado y listo para entrega.**