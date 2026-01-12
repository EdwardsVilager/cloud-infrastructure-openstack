# Arquitectura de Storage OpenStack

## Overview

Este documento describe la arquitectura de almacenamiento utilizada en el despliegue de OpenStack basado en Kolla-Ansible, detallando el backend
actual, las decisiones tomadas y el proceso de preparación del storage.

El enfoque inicial es simple, funcional y alineado a laboratorio, sin comprometer la evolución futura hacia soluciones distribuidas como Ceph.

## Backend de Almacenamiento

## Decisiones de Diseño

- Se utiliza LVM local como backend inicial por:
  - Simplicidad operativa
  - Bajo consumo de recursos
  - Facilidad de troubleshooting

- El backend LVM es exclusivo para laboratorio
- La arquitectura permite migración futura a Ceph sin rediseñar el plano de control

> Estas decisiones están alineadas con un enfoque Day-0 / Day-1 de la plataforma.

## Topología de Storage

```text
+-------------------+
|   cmp01           |
|  /dev/vdb (20GB)  |
|   VG: cinder-     |
|   volumes         |
+-------------------+

+-------------------+
|   cmp02           |
|  /dev/vdb (20GB)  |
|   VG: cinder-     |
|   volumes         |
+-------------------+
```

> Cada nodo de cómputo provee almacenamiento local independiente.

## Preparación de Storage

### 1. Verificación de discos disponibles

En cada compute node:

```bash
lsblk
```

Resultado esperado:

```text
vda   60G  (Sistema Operativo)
vdb   20G  (Disco libre para Cinder)
```

> ⚠️ El disco destinado a Cinder no debe contener particiones ni datos.

### 2. Creación del Physical Volume (PV)

```bash
sudo pvcreate /dev/vdb
```

Verificación:

```bash
sudo pvs
```

### 3. Creación del Volume Group

```bash
sudo vgcreate cinder-volumes /dev/vdb
```

Verificación:

```bash
sudo vgs
```

Resultado esperado:

```text
VG             #PV  #LV  VSize  VFree
cinder-volumes   1    0  20.00g 20.00g
```

### 4. Validación final del Volume Group

```bash
sudo vgdisplay cinder-volumes
```

## Backend LVM (Estado actual)

- Backend: Cinder LVM
- Volume Group: cinder-volumes
- Tamaño por nodo: 20GB
- Ubicación: compute nodes (cmp01, cmp02)

> Este backend se utiliza exclusivamente para laboratorio y validación funcional de Cinder.

## Consideraciones y Limitaciones

- El almacenamiento no es compartido
- No existe replicación entre nodos
- La pérdida de un compute node implica la pérdida de sus volúmenes locales
Adecuado para:
- LAB
- POCs
- Pruebas funcionales.

## Evolución Futura

- La arquitectura está preparada para evolucionar hacia:
- Ceph RBD como backend de Cinder
- Storage distribuido
- Alta disponibilidad de volúmenes
- Mejor tolerancia a fallos

> Este cambio se documentará mediante un ADR específico.
