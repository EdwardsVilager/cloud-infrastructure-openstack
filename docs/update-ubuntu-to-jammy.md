# Actualización de Ubuntu 20.04 → 22.04 (Jammy) para Kolla-Ansible

Este documento describe cómo actualizar los nodos del laboratorio desde **Ubuntu 20.04 (Focal)** a **Ubuntu 22.04 (Jammy)**, paso necesario para que **Kolla-Ansible Bobcat** soporte el sistema operativo.

---

## ⚠️ Consideraciones previas

- Hacer **snapshot o backup de cada nodo**, especialmente los nodos compute con LVM/Cinder.
- Este procedimiento aplica tanto para:
  - Nodos de control (`ctrl01-03`)
  - Nodos compute (`cmp01-02`)
  - Nodo de deployment (`deploy01`)
- Se requiere **acceso root** o sudo en todos los nodos.
- Se usa **mirror interno**: `https://repository.devops.in.idemia.com/ubuntu/fr.archive.ubuntu.com/ubuntu/`

Modificar /etc/apt/sources.list:

```bash
deb https://repository.devops.in.idemia.com/ubuntu/fr.archive.ubuntu.com/ubuntu/ jammy main restricted universe multiverse
deb https://repository.devops.in.idemia.com/ubuntu/fr.archive.ubuntu.com/ubuntu/ jammy-updates main restricted universe multiverse
deb https://repository.devops.in.idemia.com/ubuntu/fr.archive.ubuntu.com/ubuntu/ jammy-backports main restricted universe multiverse
deb https://repository.devops.in.idemia.com/ubuntu/fr.archive.ubuntu.com/ubuntu/ jammy-security main restricted universe multiverse
```

---

## 1️⃣ Respaldos

Antes de actualizar:

```bash
# Verificar discos y volumenes
lsblk
vgs
lvs
```

- Confirmar que los Volume Groups de Cinder (cinder-volumes) estén respaldados si es necesario.
- Recomiendo snapshot de VM completa antes de upgrade.

## 2️⃣ Actualización de paquetes Focal

```bash
sudo apt update
sudo apt upgrade -y
sudo apt dist-upgrade -y
```

Verificar versión:

```bash
lsb_release -a
```

Resultado esperado:

```text
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 22.04.5 LTS
Release:        22.04
Codename:       jammy
```

Reiniciar el nodo:

```bash
sudo reboot
```
