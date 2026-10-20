# Diseño de Preimplementación — Proyecto 2 BanTech GT
## Redes de Computadoras 1 — Segundo Semestre 2026

> Objetivo: definir y validar la arquitectura lógica y física antes de construirla en Cisco Packet Tracer.  
> Se distinguen los **requisitos del enunciado** de las **decisiones propuestas por el grupo**.

---

# 1. Datos del grupo

| Parámetro | Valor |
|---|---|
| Grupo | **7** |
| Coordinador | **Carlos Hugo Rios Mancilla** |
| Carné coordinador | **9520488** |
| `XX` | **88** |
| `Y` | **7** |
| Dominio VTP Occidente | **bantech88** |
| Password VTP | **cisco** |

Reglas aplicadas:

```text
XX = últimos 2 dígitos del carné del coordinador = 88
Y  = último dígito del número de grupo = 7
```

---

# 2. Requisitos obligatorios y decisiones propuestas

## 2.1 Requisitos obligatorios del proyecto

- Backbone redundante con al menos **3 dispositivos de Capa 3**.
- OSPF, EIGRP, RIPv2 y rutas estáticas.
- Al menos **2 puntos de redistribución**.
- Al menos un enlace de **fibra**, uno **Ethernet** y uno **serial**.
- Inter-VLAN en todas las sedes.
- Occidente: Router-on-a-Stick.
- Norte: Rapid PVST+ y Root Bridge explícito.
- Oriente: HSRP o VRRP con MS1 y MS2.
- Central/Data Center: rutas estáticas y EtherChannel de mínimo 2 enlaces.
- VTP en Occidente si se usa más de un switch.
- Mínimo **7 dispositivos finales por área** según el enunciado.
- **Decisión del grupo:** representar físicamente una cantidad proporcional a los hosts requeridos, usando aproximadamente el 20% de cada área y respetando siempre el mínimo de 7 dispositivos.
- VLANs con nombre y todos los routers/switches con hostname.
- Pruebas de conectividad, tablas de enrutamiento y failover.

## 2.2 Protocolos propuestos por sede

| Sede | Protocolo | Justificación |
|---|---|---|
| Occidente | **RIPv2** | Sencillo, soporta VLSM y es adecuado para un dominio pequeño. |
| Norte | **EIGRP** | Buena convergencia; DUAL permite rápida recuperación. |
| Oriente | **OSPF** | Sede crítica; protocolo empresarial con rápida convergencia. |
| Central/Data Center | **Rutas estáticas** | Obligatorio por el enunciado y apropiado para un segmento controlado. |

> La asignación RIP/EIGRP/OSPF a Occidente/Norte/Oriente es una **propuesta**, porque el enunciado permite escoger libremente.

---

# 3. VLANs definitivas

| Sede | Área | Hosts | Fórmula | VLAN |
|---|---|---:|---:|---:|
| Occidente | Cajas | 45 | 1Y | **17** |
| Occidente | Asesores | 30 | 2Y | **27** |
| Occidente | Gerencia | 10 | 3Y | **37** |
| Occidente | Seguridad | 12 | 4Y | **47** |
| Norte | Análisis | 50 | 5Y | **57** |
| Norte | Auditoría | 25 | 6Y | **67** |
| Norte | Legal | 14 | 7Y | **77** |
| Oriente | Bóveda | 40 | 8Y | **87** |
| Oriente | Plataforma | 60 | 9Y | **97** |
| Central | Core_BD | 14 | 1Y+1 | **18** |
| Central | Web_Apps | 28 | 2Y+1 | **28** |
| Central | NOC | 10 | 3Y+1 | **38** |

---

# 4. Estrategia de direccionamiento

Se propone el bloque:

```text
10.88.0.0/16
```

Distribución por sede:

```text
10.88.0.0/22    → Occidente
10.88.4.0/22    → Norte
10.88.8.0/22    → Oriente
10.88.12.0/22   → Central/Data Center
10.88.252.0/24  → Backbone y enlaces punto a punto
```

Se utiliza VLSM de forma conservadora, considerando también las IP necesarias para gateways y HSRP.

---

# 5. Subnetting VLSM

## 5.1 Occidente

| VLAN | Área | Red | Prefijo | Máscara | Gateway | Broadcast |
|---:|---|---|---:|---|---|---|
| 17 | Cajas | `10.88.0.0` | /26 | 255.255.255.192 | `10.88.0.1` | `10.88.0.63` |
| 27 | Asesores | `10.88.0.64` | /26 | 255.255.255.192 | `10.88.0.65` | `10.88.0.127` |
| 37 | Gerencia | `10.88.0.128` | /28 | 255.255.255.240 | `10.88.0.129` | `10.88.0.143` |
| 47 | Seguridad | `10.88.0.144` | /28 | 255.255.255.240 | `10.88.0.145` | `10.88.0.159` |

> VLAN 27 usa /26 porque 30 hosts finales + 1 gateway exceden los 30 hosts utilizables de /27.

## 5.2 Norte

| VLAN | Área | Red | Prefijo | Máscara | Gateway | Broadcast |
|---:|---|---|---:|---|---|---|
| 57 | Análisis | `10.88.4.0` | /26 | 255.255.255.192 | `10.88.4.1` | `10.88.4.63` |
| 67 | Auditoría | `10.88.4.64` | /27 | 255.255.255.224 | `10.88.4.65` | `10.88.4.95` |
| 77 | Legal | `10.88.4.96` | /27 | 255.255.255.224 | `10.88.4.97` | `10.88.4.127` |

> VLAN 77 usa /27 porque 14 hosts finales + 1 gateway no caben en /28.

## 5.3 Oriente — HSRP

### VLAN 97 — Plataforma

```text
Red:            10.88.8.0/25
Máscara:        255.255.255.128
HSRP VIP:       10.88.8.1
MS1:            10.88.8.2
MS2:            10.88.8.3
Broadcast:      10.88.8.127
```

60 hosts + VIP + IP de MS1 + IP de MS2 = 63 direcciones; por eso se utiliza /25.

### VLAN 87 — Bóveda

```text
Red:            10.88.8.128/26
Máscara:        255.255.255.192
HSRP VIP:       10.88.8.129
MS1:            10.88.8.130
MS2:            10.88.8.131
Broadcast:      10.88.8.191
```

## 5.4 Central/Data Center

| VLAN | Área | Red | Prefijo | Máscara | Gateway SVI | Broadcast |
|---:|---|---|---:|---|---|---|
| 28 | Web_Apps | `10.88.12.0` | /27 | 255.255.255.224 | `10.88.12.1` | `10.88.12.31` |
| 18 | Core_BD | `10.88.12.32` | /27 | 255.255.255.224 | `10.88.12.33` | `10.88.12.63` |
| 38 | NOC | `10.88.12.64` | /28 | 255.255.255.240 | `10.88.12.65` | `10.88.12.79` |

---

# 6. Backbone Core propuesto

## 6.1 Dispositivos

```text
CORE-MLS1 → Cisco 3560
CORE-MLS2 → Cisco 3560
CORE-R1   → Cisco 2911
CORE-R2   → Cisco 2911
```

Se utilizan 4 dispositivos de Capa 3, superando el mínimo de 3.

## 6.2 Esquema lógico

```text
                 CORE-MLS1 ═══════ CORE-MLS2
                    ||    Po1     ||
                    ||   LACP     ||
                    ||   FIBRA    ||
                     |             |
                  Ethernet      Ethernet
                     |             |
                  CORE-R1 ------ CORE-R2
                       SERIAL

       RIP              OSPF CORE              EIGRP
        |                                          |
   R-OCCIDENTE                                 R-NORTE

       OSPF                                      OSPF
        |                                         |
       MS1                                       MS2
          \                                     /
                SEDE ORIENTE

                     OSPF
                      |
                 R-CENTRAL1
                      |
                   STATIC
                      |
                 R-CENTRAL2
                      |
                   STATIC
                      |
                   DC-DIST
```

---

# 7. Medios físicos

| Medio | Aplicación |
|---|---|
| Fibra | CORE-MLS1 ↔ CORE-MLS2, dos enlaces agrupados |
| Ethernet | Core ↔ routers y routers ↔ sedes |
| Serial | CORE-R1 ↔ CORE-R2 |

## 7.1 EtherChannel del Core

```text
CORE-MLS1 Gi0/1 ───── Gi0/1 CORE-MLS2
CORE-MLS1 Gi0/2 ───── Gi0/2 CORE-MLS2
             \         /
                Po1
               LACP
```

Se propone VLAN de tránsito 900 sobre el Port-Channel.

---

# 8. Direccionamiento del Backbone

Bloque: `10.88.252.0/24`

| Enlace | Red | IP A | IP B |
|---|---|---|---|
| MLS1 ↔ MLS2 Po1/VLAN900 | `10.88.252.0/30` | `.1` | `.2` |
| MLS1 ↔ CORE-R1 | `10.88.252.4/30` | `.5` | `.6` |
| MLS2 ↔ CORE-R2 | `10.88.252.8/30` | `.9` | `.10` |
| CORE-R1 ↔ CORE-R2 Serial | `10.88.252.12/30` | `.13` | `.14` |
| CORE-R1 ↔ R-Occidente | `10.88.252.16/30` | `.17` | `.18` |
| CORE-R2 ↔ R-Norte | `10.88.252.20/30` | `.21` | `.22` |
| MLS1 ↔ MS1 | `10.88.252.24/30` | `.25` | `.26` |
| MLS2 ↔ MS2 | `10.88.252.28/30` | `.29` | `.30` |
| MLS1 ↔ R-Central1 | `10.88.252.32/30` | `.33` | `.34` |
| MLS2 ↔ R-Central1 | `10.88.252.36/30` | `.37` | `.38` |
| R-Central1 ↔ R-Central2 | `10.88.252.40/30` | `.41` | `.42` |
| R-Central2 ↔ DC-DIST | `10.88.252.44/30` | `.45` | `.46` |

---

# 9. Routing y redistribución

## 9.1 OSPF del Core

Se propone:

```text
OSPF process 1
Área 0
```

Participantes principales:

```text
CORE-MLS1
CORE-MLS2
CORE-R1
CORE-R2
MS1
MS2
R-CENTRAL1
```

## 9.2 Puntos de redistribución

### Punto 1 — CORE-R1

```text
RIPv2 ↔ OSPF
```

### Punto 2 — CORE-R2

```text
EIGRP ↔ OSPF
```

### Punto 3 — R-CENTRAL1

```text
Static → OSPF
```

Esta arquitectura supera el mínimo de dos puntos de redistribución.

---

# 10. Diseño por sede

## 10.1 Occidente

### Requisitos
- R-Occidente.
- Router-on-a-Stick obligatorio.
- VLAN 17, 27, 37 y 47.
- Si se usa más de un switch, VTP obligatorio.
- Dominio VTP: `bantech88`.
- Password: `cisco`.

### Dispositivos propuestos

```text
R-OCCIDENTE → Cisco 2911
OCC-MAIN    → Cisco 2960
OCC-ACC     → Cisco 2960
```

### Topología propuesta

```text
CORE-R1
   |
R-OCCIDENTE
   |
TRUNK 802.1Q
   |
OCC-MAIN
   |
TRUNK
   |
OCC-ACC
```

### Router-on-a-Stick

```text
G0/0.17 → VLAN 17 → 10.88.0.1/26
G0/0.27 → VLAN 27 → 10.88.0.65/26
G0/0.37 → VLAN 37 → 10.88.0.129/28
G0/0.47 → VLAN 47 → 10.88.0.145/28
```

Cada subinterfaz utilizará:

```text
encapsulation dot1Q <VLAN>
```

### VTP

```text
OCC-MAIN → Server
OCC-ACC  → Client
Domain   → bantech88
Password → cisco
```

### Routing externo

```text
R-OCCIDENTE ↔ CORE-R1
RIPv2
```

---

## 10.2 Norte

### Requisitos
- R-Norte.
- VLAN 57, 67 y 77.
- Redundancia de Capa 2.
- Bucles físicos intencionales.
- Rapid PVST+.
- Root Bridge forzado.

### Dispositivos propuestos

```text
R-NORTE
NORTE-MAIN
NORTE-ACC1
NORTE-ACC2
```

### Topología

```text
               R-NORTE
                  |
               TRUNK
                  |
              NORTE-MAIN
              /        \
             /          \
     NORTE-ACC1 ------ NORTE-ACC2
```

Los tres enlaces entre switches serán trunks.

### Rapid PVST+

Se propone:

```text
NORTE-MAIN = Root Bridge
```

para:

```text
VLAN 57
VLAN 67
VLAN 77
```

Prioridad propuesta:

```text
4096
```

### Inter-VLAN

Se propone Router-on-a-Stick:

```text
G0/0.57 → 10.88.4.1/26
G0/0.67 → 10.88.4.65/27
G0/0.77 → 10.88.4.97/27
```

### Routing externo

```text
R-NORTE ↔ CORE-R2
EIGRP AS 88
```

> El AS 88 es una decisión propuesta; el enunciado no fija un AS específico.

---

## 10.3 Oriente

### Requisitos
- MS1 y MS2.
- Acceso conectado simultáneamente a ambos.
- VLAN 87 y 97.
- HSRP o VRRP.
- Si MS1 falla, MS2 debe mantener el gateway.

### Decisión propuesta
Utilizar **HSRP**.

### Dispositivos

```text
MS1      → Cisco 3560
MS2      → Cisco 3560
ORI-ACC1 → Cisco 2960
ORI-ACC2 → Cisco 2960
```

### Topología

```text
          CORE-MLS1             CORE-MLS2
              |                     |
             MS1-------------------MS2
             |\                     /|
             | \                   / |
             |  \                 /  |
           ACC1                  ACC2
             |                    |
           Hosts                Hosts
```

### HSRP VLAN 97

```text
VIP:           10.88.8.1
MS1:           10.88.8.2
MS2:           10.88.8.3
Grupo HSRP:    97
MS1 priority:  110
MS2 priority:  100
preempt:       habilitado
```

### HSRP VLAN 87

```text
VIP:           10.88.8.129
MS1:           10.88.8.130
MS2:           10.88.8.131
Grupo HSRP:    87
MS1 priority:  110
MS2 priority:  100
preempt:       habilitado
```

### Inter-VLAN

```text
interface vlan 87
interface vlan 97
ip routing
```

### Routing externo

```text
MS1 ↔ CORE-MLS1 → OSPF
MS2 ↔ CORE-MLS2 → OSPF
```

---

## 10.4 Central / Data Center

### Requisitos
- R-Central1.
- R-Central2.
- Segmento de rutas estáticas.
- Switches de distribución y acceso.
- EtherChannel de mínimo 2 enlaces.
- VLAN 18, 28 y 38.
- R-Central1 debe propagar estas rutas al backbone.

### Dispositivos propuestos

```text
R-CENTRAL1
R-CENTRAL2
DC-DIST  → Cisco 3560
DC-ACC1  → Cisco 2960
DC-ACC2  → Cisco 2960
```

### Topología

```text
          BACKBONE OSPF
            /       \
      CORE-MLS1   CORE-MLS2
            \       /
             R-CENTRAL1
                 |
               STATIC
                 |
             R-CENTRAL2
                 |
               STATIC
                 |
              DC-DIST
             /       \
        Po10 LACP   Po20 LACP
          ||          ||
       DC-ACC1      DC-ACC2
          |            |
      SERVIDORES   SERVIDORES
```

### EtherChannel

```text
DC-DIST ↔ DC-ACC1 → Po10 → 2 enlaces
DC-DIST ↔ DC-ACC2 → Po20 → 2 enlaces
```

Se propone LACP.

### Inter-VLAN

`DC-DIST` creará SVIs para:

```text
VLAN 18 → Core_BD
VLAN 28 → Web_Apps
VLAN 38 → NOC
```

y utilizará:

```text
ip routing
```

---

# 11. Rutas estáticas de Central

## 11.1 DC-DIST

Ruta por defecto hacia R-Central2:

```text
ip route 0.0.0.0 0.0.0.0 10.88.252.45
```

## 11.2 R-Central2

Rutas hacia las VLAN del Data Center:

```text
ip route 10.88.12.0 255.255.255.224 10.88.252.46
ip route 10.88.12.32 255.255.255.224 10.88.252.46
ip route 10.88.12.64 255.255.255.240 10.88.252.46
```

Ruta por defecto hacia R-Central1:

```text
ip route 0.0.0.0 0.0.0.0 10.88.252.41
```

## 11.3 R-Central1

```text
ip route 10.88.12.0 255.255.255.224 10.88.252.42
ip route 10.88.12.32 255.255.255.224 10.88.252.42
ip route 10.88.12.64 255.255.255.240 10.88.252.42
```

Luego estas rutas se redistribuirán hacia OSPF.

---

# 12. Representación proporcional de dispositivos finales

El enunciado exige un mínimo de **7 dispositivos finales por área**. Para que la simulación represente mejor la magnitud relativa de cada departamento, el grupo utilizará una **representación proporcional aproximada del 20% de los hosts requeridos**, respetando siempre el mínimo de 7 dispositivos.

> **Importante:** el 20% es una **decisión de diseño del grupo**, no un porcentaje impuesto por el enunciado. El requisito oficial es mantener al menos 7 dispositivos finales por área.

## 12.1 Criterio de representación

Se aplicará la siguiente regla práctica:

```text
Equipos representados ≈ 20% de los hosts requeridos
Si el resultado es menor que 7 → colocar 7
```

Los dispositivos colocados físicamente sirven como representación visual y para las pruebas.  
El **subnetting continúa dimensionado para la totalidad de hosts exigida** en cada área.

## 12.2 Distribución por área

| Sede | Área | Hosts requeridos | Equipos físicos propuestos |
|---|---|---:|---:|
| Occidente | Cajas | 45 | **9** |
| Occidente | Asesores | 30 | **7** |
| Occidente | Gerencia | 10 | **7** |
| Occidente | Seguridad | 12 | **7** |
| Norte | Análisis | 50 | **10** |
| Norte | Auditoría | 25 | **7** |
| Norte | Legal | 14 | **7** |
| Oriente | Bóveda | 40 | **8** |
| Oriente | Plataforma | 60 | **12** |
| Central | Core_BD | 14 | **7** |
| Central | Web_Apps | 28 | **7** |
| Central | NOC | 10 | **7** |

## 12.3 Totales por sede

```text
Occidente → 30 dispositivos
Norte     → 24 dispositivos
Oriente   → 20 dispositivos
Central   → 21 dispositivos
--------------------------------
Total     → 95 dispositivos finales
```

## 12.4 Justificación para la defensa

La cantidad física de dispositivos no sustituye el dimensionamiento real de la red. Por ejemplo:

```text
VLAN 17 - Cajas
Hosts requeridos por el proyecto: 45
Equipos representados:            9
Subred utilizada:                 /26
Capacidad utilizable:             62 hosts
```

Una respuesta breve para la defensa puede ser:

> La simulación utiliza una cantidad proporcional de dispositivos finales para representar visualmente el tamaño relativo de cada área. Sin embargo, todas las subredes fueron calculadas para soportar la cantidad completa de hosts solicitada por el enunciado. Además, ninguna área tiene menos de siete dispositivos finales.

## 12.5 Consecuencia para el diseño físico

Las áreas con más usuarios también tendrán una representación física mayor:

```text
Plataforma → 60 hosts → 12 equipos
Análisis   → 50 hosts → 10 equipos
Cajas      → 45 hosts → 9 equipos
Bóveda     → 40 hosts → 8 equipos
```

Las áreas menores conservarán el mínimo de 7 equipos.

Esto permitirá que la topología en Packet Tracer refleje visualmente cuáles áreas tienen mayor demanda, sin necesidad de colocar los 338 hosts finales indicados por capacidad en las tablas del proyecto.

---

# 13. Plan de comandos de verificación

## VLANs

```text
show vlan brief
```

## Trunks

```text
show interfaces trunk
```

## VTP Occidente

```text
show vtp status
```

## Rapid PVST+ Norte

```text
show spanning-tree
show spanning-tree vlan 57
show spanning-tree vlan 67
show spanning-tree vlan 77
```

## EtherChannel

```text
show etherchannel summary
```

## HSRP

```text
show standby
show standby brief
```

## Interfaces L3

```text
show ip interface brief
```

## Tabla de routing

```text
show ip route
```

Códigos esperados:

```text
C → Connected
S → Static
R → RIP
O → OSPF
D → EIGRP
```

## OSPF

```text
show ip ospf neighbor
show ip protocols
```

## EIGRP

```text
show ip eigrp neighbors
show ip protocols
```

## RIP

```text
show ip protocols
show ip route rip
```

## Conectividad

```text
ping <IP>
traceroute <IP>
```

---

# 14. Pruebas de alta disponibilidad

## 14.1 Norte — Rapid PVST+

1. Verificar qué enlace está bloqueado.
2. Apagar un enlace activo.
3. Esperar convergencia.
4. Revisar STP.
5. Hacer ping.
6. Restaurar enlace.

```text
show spanning-tree vlan 57
```

Para simular falla:

```text
shutdown
```

Restaurar:

```text
no shutdown
```

## 14.2 Oriente — HSRP

Estado inicial:

```text
show standby brief
```

Esperado:

```text
MS1 → Active
MS2 → Standby
```

Prueba:

1. Mantener un ping hacia otra sede.
2. Apagar la interfaz crítica de MS1 o simular pérdida del equipo.
3. Verificar que MS2 pase a Active.
4. Confirmar que la IP virtual siga disponible.
5. Restaurar MS1.
6. Verificar `preempt`.

## 14.3 Data Center — EtherChannel

```text
show etherchannel summary
```

Prueba:

1. Confirmar 2 miembros activos.
2. Apagar un enlace físico.
3. Confirmar que el Port-Channel siga UP.
4. Hacer ping.
5. Restaurar el enlace.

## 14.4 Backbone

Apagar un enlace del núcleo y comprobar que OSPF converja por otra ruta:

```text
show ip route
show ip ospf neighbor
ping
traceroute
```

---

# 15. Nomenclatura propuesta

## Backbone

```text
CORE-MLS1
CORE-MLS2
CORE-R1
CORE-R2
```

## Occidente

```text
R-OCCIDENTE
OCC-MAIN
OCC-ACC
```

## Norte

```text
R-NORTE
NORTE-MAIN
NORTE-ACC1
NORTE-ACC2
```

## Oriente

```text
MS1
MS2
ORI-ACC1
ORI-ACC2
```

## Central

```text
R-CENTRAL1
R-CENTRAL2
DC-DIST
DC-ACC1
DC-ACC2
```

---

# 16. Checklist antes de implementar

- [ ] Confirmar modelo de 3560 y funciones disponibles.
- [ ] Confirmar funcionamiento de EtherChannel en los puertos elegidos.
- [ ] Confirmar compatibilidad de enlaces de fibra.
- [ ] Instalar módulos seriales en routers 2911.
- [ ] Confirmar HSRP sobre SVIs en MS1/MS2.
- [ ] Verificar OSPF, EIGRP y RIPv2 en los modelos elegidos.
- [ ] Evitar bucles de redistribución.
- [ ] Nombrar explícitamente todas las VLANs.
- [ ] Configurar hostname en todos los dispositivos.
- [ ] Colocar la cantidad proporcional definida por área (mínimo 7 por área), para un total propuesto de 95 dispositivos finales.
- [ ] Identificar trunks y puertos de acceso.
- [ ] Validar Inter-VLAN en todas las sedes.
- [ ] Validar conectividad entre todas las sedes.
- [ ] Validar `show ip route`.
- [ ] Documentar pruebas de failover con capturas.

---

# 17. Orden recomendado de construcción en Packet Tracer

## Fase 1 — Backbone físico

```text
CORE-MLS1
CORE-MLS2
CORE-R1
CORE-R2
Fibra / EtherChannel
Ethernet
Serial
```

Primero comprobar solamente conectividad entre vecinos.

## Fase 2 — OSPF del Backbone

Configurar OSPF Área 0 entre los dispositivos del core.

Validar:

```text
show ip ospf neighbor
show ip route
ping
```

## Fase 3 — Occidente

```text
VLANs
VTP
Trunks
Router-on-a-Stick
RIPv2
```

Primero probar Inter-VLAN local.  
Después conectar RIP al backbone.  
Finalmente configurar redistribución en CORE-R1.

## Fase 4 — Norte

```text
VLANs
Topología triangular
Rapid PVST+
Root Bridge
Router-on-a-Stick
EIGRP
```

Primero validar STP e Inter-VLAN.  
Después conectar EIGRP al backbone.  
Finalmente configurar redistribución en CORE-R2.

## Fase 5 — Oriente

```text
VLANs
Trunks
SVIs
HSRP
OSPF
```

Primero comprobar HSRP local.  
Después integrar MS1/MS2 al OSPF.

## Fase 6 — Central/Data Center

```text
VLANs
SVIs
EtherChannel
Rutas estáticas
```

Orden:

```text
DC-DIST
↓
R-CENTRAL2
↓
R-CENTRAL1
↓
OSPF BACKBONE
```

Finalmente redistribuir las rutas estáticas desde R-Central1 hacia OSPF.

## Fase 7 — Convergencia total

Probar:

```text
Occidente → Norte
Occidente → Oriente
Occidente → Data Center
Norte → Oriente
Norte → Data Center
Oriente → Data Center
```

Revisar:

```text
show ip route
traceroute
```

## Fase 8 — Alta disponibilidad

Probar:

```text
Rapid PVST+ Norte
HSRP Oriente
EtherChannel Central
Fallo de enlace del Backbone
```

## Fase 9 — Documentación

Capturar y documentar:

- Topología general.
- Subnetting.
- VLANs.
- Trunks.
- VTP.
- Router-on-a-Stick.
- SVIs.
- RIP.
- EIGRP.
- OSPF.
- Rutas estáticas.
- Redistribución.
- Rapid PVST+.
- HSRP.
- EtherChannel.
- `show ip route`.
- Pings extremo a extremo.
- Failover.

---

# 18. Resumen final

```text
GRUPO 7
XX = 88
Y  = 7

BACKBONE
├── CORE-MLS1
├── CORE-MLS2
├── CORE-R1
├── CORE-R2
├── OSPF Área 0
├── Fibra + LACP
├── Ethernet
└── Serial

OCCIDENTE
├── VLAN 17 Cajas
├── VLAN 27 Asesores
├── VLAN 37 Gerencia
├── VLAN 47 Seguridad
├── Router-on-a-Stick
├── VTP bantech88
└── RIPv2

NORTE
├── VLAN 57 Análisis
├── VLAN 67 Auditoría
├── VLAN 77 Legal
├── Topología triangular
├── Rapid PVST+
├── Root Bridge forzado
├── Router-on-a-Stick
└── EIGRP AS 88

ORIENTE
├── VLAN 87 Bóveda
├── VLAN 97 Plataforma
├── MS1 / MS2
├── SVIs
├── HSRP
└── OSPF

CENTRAL / DATA CENTER
├── VLAN 18 Core_BD
├── VLAN 28 Web_Apps
├── VLAN 38 NOC
├── DC-DIST
├── EtherChannel LACP
├── R-CENTRAL2
├── R-CENTRAL1
├── Rutas estáticas
└── Redistribución Static → OSPF
```

---

# 19. Estado del diseño

**Estado:** PROPUESTA DE PREIMPLEMENTACIÓN.

Antes de construir toda la red, se debe validar físicamente en Packet Tracer:

1. Modelos exactos de routers y switches.
2. Módulos seriales.
3. Compatibilidad de fibra.
4. EtherChannel del backbone.
5. HSRP en MS1/MS2.

Una vez validado lo anterior, la implementación debe realizarse por fases, evitando construir y configurar toda la red al mismo tiempo.
