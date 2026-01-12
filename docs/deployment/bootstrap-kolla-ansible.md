# Bootstrap de Kolla-Ansible

Este documento describe el proceso completo de **bootstrap del entorno de despliegue de OpenStack** utilizando **Kolla-Ansible** sobre **Ubuntu 20.04** y **OpenStack Bobcat**.

Este paso prepara el nodo de despliegue con todas las dependencias necesarias antes de ejecutar el deploy.

---

## 🧭 Escenarios de despliegue soportados

Este bootstrap contempla dos escenarios válidos para el nodo de despliegue:

### Opción A — Nodo de despliegue dedicado (recomendado)

- Un nodo exclusivo para Ansible / Kolla-Ansible
- No ejecuta servicios de OpenStack
- Mayor limpieza y control del entorno
- Ejemplo:
  - deploy01 → Ansible / Kolla-Ansible
  - ctrl01-03 → Control Plane
  - cmp01-02 → Compute

### Opción B — Nodo de despliegue compartido

- El nodo de despliegue es también un control node
- Escenario común en laboratorios
- Menor consumo de recursos
- Ejemplo:
  - ctrl01 → Deployment + Control Plane

> 📌 Ambos escenarios están soportados. Los pasos siguientes aplican a cualquiera de los dos; solo cambia en qué nodo se ejecutan.

---

## Alcance

- Nodo de despliegue: `ctrl01`
- OpenStack release: **Bobcat**
- Base OS: **Ubuntu 22.04**
- Método de despliegue: **Kolla-Ansible**

📌 Este procedimiento **solo se ejecuta en el nodo de deployment**.

---

## 1. Preparación del sistema

### Actualización del sistema

```bash
sudo apt update && sudo apt upgrade -y
```

### Instalación de dependencias base

```bash
sudo apt install -y \
  python3-pip \
  python3-dev \
  libffi-dev \
  gcc \
  libssl-dev \
  python3-venv \
  git \
  rsync \
  sshpass
```

---

## 2. Entorno Python para Kolla-Ansible

> Instalar con usuarios con privilegios `root`

### Crear virtualenv

```bash
python3 -m venv ~/kolla-venv
```

### Activar el entorno

```bash
source ~/kolla-venv/bin/activate
```

📌 Todos los comandos de `kolla-ansible` se ejecutan dentro de este virtualenv.

---

## 3. Instalación de Kolla-Ansible (Bobcat)

### Actualizar pip

```bash
pip install --upgrade pip
```

### Instalar Ansible compatible

```bash
pip install "ansible-core>=2.14,<2.16"
```

### Instalar Kolla-Ansible

```bash
pip install kolla-ansible==17.*
```

### Verificación

```bash
kolla-ansible --version
```

Resultado esperado:

```text
kolla-ansible 17.x.x
```

---

## 4. Preparación del directorio de configuración

### Crear estructura de configuración

```bash
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla
```

### Copiar archivos base

```bash
cp -r ~/kolla-venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla/
```

Verificación:

```bash
ls /etc/kolla
```

Archivos esperados:

- `globals.yml`
- `passwords.yml`

---

## 5. Inventario Ansible

El inventario ya se encuentra definido en:

```text
ansible/openstack/inventory/hosts.ini
```

Ejemplo:

```ini
# ------------------------------------------------
# CONTROL NODES (HA)
# ------------------------------------------------
[control]
ctrl01
ctrl02
ctrl03

[network]
ctrl01
ctrl02
ctrl03

[monitoring]
ctrl01
ctrl02
ctrl03

# ------------------------------------------------
# COMPUTE NODES
# ------------------------------------------------
[compute]
cmp01
cmp02

# ------------------------------------------------
# STORAGE (LVM)
# ------------------------------------------------
[storage]
cmp01
cmp02

# ------------------------------------------------
# REQUIRED KOLLA GROUPS
# ------------------------------------------------
[deployment]
ctrl01

[baremetal:children]
control
compute
```

---

## 6. Generación de passwords

Generación automática de credenciales:

```bash
kolla-genpwd
```

Archivo generado:

```text
/etc/kolla/passwords.yml
```

> 📌 No se recomienda editar este archivo manualmente.
> 📌 Este comando utiliza automáticamente `/etc/kolla/passwords.yml`

---

## 7. Validación de configuración básica

Verificación de parámetros críticos de Kolla-Ansible.

> 📌 Nota importante
> Kolla-Ansible solo utiliza el archivo ubicado en: `/etc/kolla/globals.yml`.
> Este archivo es el que el operador mantiene, edita y versiona.
> Cualquier otro `globals.yml` fuera de esta ruta NO es utilizado durante prechecks ni `deploy`.

Reemplazar por archivo globals.yml personalizado.

```bash
sudo cp /etc/kolla/globals.yml /etc/kolla/globals.yml.init
sudo cp ansible/openstack/globals.yml /etc/kolla/globals.yml
```

Verificar:

```bash
grep kolla_base_distro /etc/kolla/globals.yml
grep openstack_release /etc/kolla/globals.yml
```

Resultado esperado:

```text
kolla_base_distro: ubuntu
openstack_release: bobcat
```

---

## 8. Verificación de conectividad SSH

> Todos los nodos deben tener las llaves par y pueda conectarse mediante `ssh -i ~/.ssh/id_rsa <user>@<IP>`

Desde el nodo de deployment:

```bash
ansible -i ansible/openstack/inventory/hosts.ini all -m ping
```

Yes a todo:

```bash
ANSIBLE_HOST_KEY_CHECKING=False ansible -i ansible/openstack/inventory/hosts.ini all -m ping
```

Resultado esperado:

```text
SUCCESS => ping
```

---

## 9. Prechecks de Kolla-Ansible

Este paso valida:

- Docker
- Networking
- Kernel params
- LVM para Cinder
- Dependencias del sistema

Ejecución:

```bash
kolla-ansible -i ansible/openstack/inventory/hosts.ini prechecks
```

Resultado esperado:

```text
PLAY RECAP
all nodes: ok
```

⚠️ No continuar si existen errores en este paso.

---

## Estado del Paso

- ✔ Entorno Python preparado
- ✔ Kolla-Ansible instalado
- ✔ Configuración base lista
- ✔ Passwords generados
- ✔ Prechecks exitosos

---

## Pasos anteriores

👉 **PASO 9 → LVM**

## Próximo Paso

👉 **PASO 10 → Bootstrap**
👉 **PASO 11 → Prechecks**
👉 **PASO 12 — Deploy de OpenStack con Kolla-Ansible**

- Incluye:
  - Ejecución de `kolla-ansible deploy`
  - Validación de servicios
  - Acceso a Horizon
  - Pruebas funcionales iniciales.
