# Arquitectura de Networking OpenStack

## Overview

Este documento describe el modelo de red utilizado en el despliegue de OpenStack usando Kolla-Ansible, Neutron ML2 + OVN y alta disponibilidad.

El diseño prioriza una arquitectura lógica clara y escalable, aun cuando el entorno inicial se despliega como laboratorio sobre máquinas virtuales.

## 📊 Resumen visual rápido

```text
                   +--------------------+
                   |   External Access  |
                   | (Floating / API)   |
                   +----------+---------+
                              |
                              v
[ VM ] -- VXLAN/OVN --> [ cmp01 / cmp02 ]
                          |
                          v
                    [ ctrl01 / ctrl02 / ctrl03 ]
                          |
                          v
                 [ Management & Control Plane ]
```

## Redes Definidas (Modelo Lógico)

> ⚠️ Nota importante
> En el laboratorio actual, todas las redes comparten la misma interfaz física.
> Las redes descritas a continuación representan separación lógica, no física.

### Management Network

- Propósito: comunicación interna de OpenStack
- Usada por:
  - APIs (Keystone, Nova, Neutron, Glance)
  - RabbitMQ
  - MariaDB
  - etcd
  - HAProxy / Keepalived
- Tipo: red lógica
- Implementación: interfaz única (eth0)

### Tunnel Network

- Propósito: tráfico overlay entre VMs
- Tecnología: VXLAN (OVN)
- Usada por:
  - Comunicación VM ↔ VM
  - Redes tenant
- Tipo: red lógica overlay
- Implementación: interfaz única (eth0)

### External / Provider Network

- Propósito:
  - Floating IPs
  - Acceso externo a instancias
  - Exposición de APIs
- Tipo: Provider Network (flat)
- Implementación: interfaz única (eth0)
- Futuro: NIC dedicada o VLAN trunk

## Tipo de Networking

- Neutron ML2
- Backend: OVN
- Overlay: VXLAN
- Provider Network: flat
- Alta disponibilidad:
  - OVN Central en controllers
  - VIPs internos y externos

## Consideraciones

- El entorno se despliega sobre máquinas virtuales con una sola interfaz de red
- Todas las redes (management, tunnel y external) comparten la misma NIC
- Esta decisión:
  - Reduce complejidad inicial
  - Permite validar arquitectura y servicios
  - No limita una migración futura a múltiples NICs o VLANs

### Consideraciones de Producción (Futuro)

En un entorno productivo se espera:

- Separación física o lógica mediante:
  - Múltiples NICs
  - VLANs
- Aislamiento de:
  - Management
  - Dataplane
  - External traffic
- Mejor rendimiento y seguridad.

## Responsabilidades por Nodo

- Control Nodes
  - APIs de OpenStack
  - OVN Central
  - Base de datos
  - Message queue
  - Load balancing (HAProxy / Keepalived)

- Compute Nodes
  - Hypervisor (KVM)
  - OVN dataplane
  - Ejecución de instancias
  - Tráfico VXLAN

## Principios Clave

- El tráfico de gestión no debe exponerse directamente
- La separación lógica es obligatoria, incluso en LAB
- El diseño debe permitir crecer sin rediseño.
