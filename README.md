# ☁️ Cloud Infrastructure – OpenStack

Repositorio de infraestructura como servicio **(IaaS)** para la instalación, configuración y operación de OpenStack como capa base de una plataforma cloud privada.

Este repositorio define el **datacenter cloud lógico** sobre el cual se despliegan plataformas superiores como Kubernetes, PaaS y workloads empresariales.

---

## 🎯 Objetivo

Proveer un **único punto de verdad (single source of truth)** para:

- Instalación y operación de OpenStack
- Definición de proyectos, redes, imágenes y flavors
- Automatización de la infraestructura IaaS
- Exposición controlada de capacidades de infraestructura a capas superiores
- Este repositorio cubre la fundación IaaS de la plataforma.

---

## 🧠 Alcance del repositorio

✅ **Incluye**:

- Instalación de OpenStack (Kolla-Ansible / TripleO)
- Preparación de nodos (OS hardening, prerequisitos)
- Configuración de servicios core:
  - Keystone
  - Nova
  - Neutron
  - Glance
  - Cinder
- Terraform para:
  - Proyectos
  - Redes
  - Flavors
  - Imágenes
  - Seguridad
- Documentación arquitectónica y ADRs
- Outputs reutilizables por capas superiores

❌ **NO incluye**:

- Kubernetes
- RKE2 / Kubespray
- GitOps
- Argo CD
- Aplicaciones o workloads
- Operación day-to-day de plataformas PaaS

> Kubernetes y la plataforma cloud viven en el repositorio:
> **`cloud-platform-foundation`**

---

## 🗂️ Estructura del repositorio

```text
cloud-infrastructure-openstack/
│
├── README.md
│
├── docs/
│   ├── architecture.md        # Arquitectura OpenStack
│   ├── services.md            # Nova, Neutron, Cinder, Glance, Keystone
│   ├── networking.md          # Provider / Tenant networks
│   ├── storage.md             # Cinder / backends
│   └── adr/
│       ├── 0001-kolla-ansible.md
│       └── 0002-cinder-backend.md
│
├── ansible/
│   ├── openstack/
│   │   ├── kolla-ansible/     # Framework de instalación
│   │   ├── inventory/        # Hosts y roles
│   │   └── globals.yml        # Configuración global
│   │
│   └── playbooks/
│       ├── prepare-nodes.yml
│       ├── deploy-openstack.yml
│       └── validate.yml
│
├── terraform/
│   ├── providers/
│   ├── modules/
│   │   ├── project/
│   │   ├── network/
│   │   ├── flavor/
│   │   ├── image/
│   │   └── security/
│   └── environments/
│       ├── dev/
│       └── prod/
│
├── outputs/
│   ├── openstack-cloud.yaml   # clouds.yaml para consumo externo
│   └── kube-targets.json      # VMs destinadas a Kubernetes
│
└── scripts/
    ├── bootstrap.sh
    └── validate-openstack.sh
```

## 🔁 Arquitectura

```text
                 ┌─────────────┐
                 │   Usuarios  │
                 └──────┬──────┘
                        │
                  Load Balancer (HAProxy)
                        │
        ┌───────────────┼───────────────┐
        │               │               │
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ Controller1 │ │ Controller2 │ │ Controller3 │
└─────────────┘ └─────────────┘ └─────────────┘
        │               │               │
        └─────────── DB + MQ Cluster ───┘
                        │
     ┌──────────────────┼──────────────────┐
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│ Compute 01  │   │ Compute 02  │   │ Compute 03  │
└─────────────┘   └─────────────┘   └─────────────┘
                        │
                 ┌─────────────┐
                 │   Ceph      │
                 └─────────────┘
```

## 🧩 Servicios OpenStack incluidos

| Servicio | Rol                       |
| -------- | ------------------------- |
| Keystone | Identidad y autenticación |
| Nova     | Cómputo (VMs)             |
| Neutron  | Networking                |
| Glance   | Imágenes                  |
| Cinder   | Almacenamiento en bloques |

---

## 🔁 Relación con otros repositorios

```text
cloud-infrastructure-openstack
│
├── Provee infraestructura IaaS
│   ├── VMs
│   ├── Redes
│   ├── Volúmenes
│
└── Es consumido por → cloud-platform-foundation
       ├── Kubernetes (RKE2 / Kubespray)
       ├── Harbor
       ├── Argo CD
       └── GitOps bootstrap
```

## 🧩 Componentes desplegados y roles (según globals.yml e inventory)

Esta sección describe los componentes reales habilitados en el laboratorio OpenStack, su rol y cómo interactúan entre sí.
La instalación se realiza usando Kolla-Ansible con servicios containerizados (Docker) y Neutron + OVN como backend de networking.

🖥️ Nodos de la plataforma

🔹 Deployment Node
Rol: Orquestación y control del despliegue

- Ejecuta `kolla-ansible`
- Contiene:
  - Inventario (`hosts.ini`)
  - `globals.yml`
  - Playbooks de despliegue y validación
- No ejecuta servicios OpenStack productivos

👉 Es el punto de control de toda la infraestructura.

🔹 Controller Nodes (ctrl01, ctrl02, ctrl03)
Rol: Control plane y alta disponibilidad

Servicios principales:

- Keystone – Identidad y autenticación
- Nova API / Scheduler / Conductor – Control de cómputo
- Neutron Server – Control de redes
- Glance API – Catálogo de imágenes
- Placement API – Gestión de recursos
- Horizon – Dashboard (opcional)

Servicios de soporte (HA):

- HAProxy – Balanceo de carga
- Keepalived – VIP
- MariaDB / Galera – Base de datos
- RabbitMQ – Mensajería
- Memcached – Cache
- ProxySQL – Abstracción de acceso a DB

Servicios OVN:

- ovn-northd
- OVN Northbound DB
- OVN Southbound DB

👉 Los controllers no ejecutan VMs, solo controlan el estado del cloud.

🔹 Compute Nodes (cmp01, cmp02, …)
Rol: Ejecución de cargas de trabajo (VMs)

Servicios:

- nova-compute
- OVN Controller
- Open vSwitch (OVS)

Características del laboratorio:

- VMs sobre un hypervisor sin virtualización anidada
- Se utiliza:
  - `veth` + `Open vSwitch`
  - `ovs-dpdk = false`

👉 Esto es válido para laboratorio y pruebas funcionales, no para performance real.

---

🌐 Networking – Neutron con OVN

Modelo de red:

- Neutron + OVN (Open Virtual Network)
- No se usan:
  - Linux Bridge
  - Open vSwitch agent clásico

Componentes clave:

- Neutron Server
- OVN Northbound DB
  - Define redes lógicas (switches, routers)
- OVN Southbound DB
  - Estado real aplicado en los nodos
- OVN Controller (en computes)
  - Programa flujos en OVS
- neutron-ovn-metadata-agent
  - Provee metadata (cloud-init) a las VMs

Flujo simplificado:

```text
Neutron API
   ↓
OVN Northbound DB
   ↓
ovn-northd
   ↓
OVN Southbound DB
   ↓
OVN Controller (Compute)
   ↓
Open vSwitch → VM
```

🧩 Rol del neutron-ovn-metadata-agent

- Permite que las VMs accedan a:
  - `http://169.254.169.254`

- Provee:
  - hostname
  - user-data
  - SSH keys
  - cloud-init

👉 Sin este agente:

- Las VMs arrancan
- Pero cloud-init falla
- No se inyectan llaves ni configuración inicial

🔁 Interacción entre componentes (visión lógica)

```text
Usuario / API / Horizon
          │
       HAProxy (VIP)
          │
 ┌────────┴────────┐
 │   Controllers   │
 │  (API + Control)│
 └────────┬────────┘
          │
   DB (Galera) + MQ
          │
     Neutron + OVN
          │
 ┌────────┴────────┐
 │    Computes     │
 │ nova-compute    │
 │ OVN Controller  │
 └────────┬────────┘
          │
        Open vSwitch
          │
         VMs
```

---

## 📌 Regla clave

Este repositorio no depende de Kubernetes.
Kubernetes sí depende de este repositorio.

---

## 🚀 Flujo de bootstrap de la plataforma

1️⃣ **Ansible** – prepare-nodes → Prepara nodos (SO, Docker/Podman, prerequisitos)
2️⃣ **Ansible** – deploy-openstack → Instala OpenStack (Kolla-Ansible / TripleO)
3️⃣ **Terraform** → Crea proyectos, redes, flavors, imágenes
4️⃣ **Outputs** → Genera clouds.yaml y targets para Kubernetes
5️⃣ **cloud-platform-foundation** → Consume la infraestructura creada

---

## 📆 Day-0 / Day-1 / Day-2

### 🟢 Day-0 (Foundation IaaS)

- Instalación OpenStack
- Networking base
- Storage backend
- Accesos administrativos

### 🟡 Day-1 (Enablement)

- Proyectos
- Flavors
- Imágenes
- Redes tenant

### 🔵 Day-2 (Operations)

- Monitoreo
- Backups
- Escalamiento
- Hardening

---

## 🔐 Seguridad

- Separación por proyectos
- Principio de menor privilegio
- Control de acceso vía Keystone
- Redes aisladas (tenant / provider)
- Hardening de nodos OpenStack

## 📌 Principios de diseño

- Infraestructura como código
- Desacoplamiento de capas
- Reproducibilidad
- Escalabilidad
- Preparado para entornos **enterprise**

## 🧭 Decisiones arquitectónicas (ADR)

Las decisiones clave se documentan en:

```text
docs/adr/
```

Ejemplos:

- Elección de Kolla-Ansible
- Backend de almacenamiento Cinder
- Modelo de networking

---

## 📌 Notas finales

Este repositorio representa la **base física y lógica del cloud**.

Cualquier cambio aquí impacta todas las plataformas superiores, por lo tanto:

- Usar Pull Requests
- Documentar decisiones
- Versionar cuidadosamente
- Validar antes de aplicar cambios

---

☁️ **Cloud Infrastructure – OpenStack**

Fundación IaaS para una plataforma cloud privada moderna, escalable y desacoplada.

---
