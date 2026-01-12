# OpenStack Architecture

## 1. Propósito

Este documento describe la **arquitectura de referencia** para la implementación de OpenStack, utilizada como la **base IaaS** de la plataforma en la nube.

El objetivo es proporcionar una **arquitectura clara, modular y escalable**
que pueda ser utilizada por plataformas de nivel superior como Kubernetes
sin un acoplamiento estrecho.

---

## 2. Arquitectura de alto nivel

A alto nivel, la plataforma se compone de tres capas principales:

```text
+-------------------------------+
|   Cloud Platforms (Consumers) |
|   - Kubernetes (RKE2)         |
|   - CI/CD                     |
+-------------------------------+
               ↓
+-------------------------------+
|        OpenStack (IaaS)       |
|  Nova | Neutron | Cinder     |
|  Glance | Keystone            |
+-------------------------------+
               ↓
+-------------------------------+
|   Physical / Virtual Infra    |
|   Compute | Network | Storage |
+-------------------------------+
```

OpenStack actúa como una capa de abstracción entre infraestructura física y las cargas de trabajo de la plataforma.

## 3. Capas de infraestructura (Infrastructure Layers)

La infraestructura se divide lógicamente en:

### 3.1 Capa física / Virtual

- Compute nodes
- Network connectivity
- Storage backend
- Operating system (Linux)

Esta capa está fuera del alcance de OpenStack, pero es necesaria para su funcionamiento.

### 3.2 OpenStack Control Plane

Responsable de:

- API exposure
- Scheduling
- Authentication
- State management

Servicios incluidos:

- Keystone
- Nova API
- Neutron API
- Glance API
- Cinder API

### 3.3 OpenStack Data Plane

Responsable de:

- VM execution
- Networking traffic
- Volume attachment

Componentes:

- Nova compute
- Neutron agents
- Storage backends

## 4. OpenStack Logical Architecture

La implementación de OpenStack sigue un **control plane** centralizado con nodos de cómputo distribuidos.

Logical view:

```text
[ Control Plane ]
   - Keystone
   - Nova API
   - Neutron API
   - Glance
   - Cinder

[ Compute Nodes ]
   - Nova Compute
   - Neutron Agents

[ Storage ]
   - Cinder Backend
```

Este modelo permite el escalamiento horizontal del cómputo y el almacenamiento sin afectar el plano de control.

## 5. Networking Overview

El entorno se despliega inicialmente en un laboratorio basado en máquinas virtuales.
A pesar de ello, se mantiene separación lógica de redes para facilitar una futura migración a infraestructura física sin cambios de arquitectura.

## 6. OpenStack Release

La plataforma se despliega usando OpenStack Bobcat (2023.2), priorizando estabilidad y compatibilidad con Kolla-Ansible.
