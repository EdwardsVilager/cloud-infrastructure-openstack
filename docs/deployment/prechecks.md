# PASO 11 — Prechecks y Validación del entorno (Kolla-Ansible)

Este documento describe **el proceso completo de validación previa** antes de desplegar OpenStack con Kolla-Ansible.

El objetivo de este paso es **detectar errores temprano**, asegurar consistencia entre nodos y validar que la plataforma está lista para el despliegue.

---

## 🎯 Objetivos del PASO 11

- Validar conectividad SSH entre nodos
- Preparar servidores para ejecución de contenedores
- Verificar dependencias de Kolla-Ansible
- Validar networking, VIPs y puertos
- Confirmar disponibilidad de storage (LVM)

Este paso **NO despliega OpenStack**, solo prepara y valida.

---

## 🧭 ¿Dónde se ejecutan estos comandos?

Todos los comandos de este documento se ejecutan **desde el nodo de despliegue**:

- Nodo dedicado (`deploy01`) **o**
- Nodo compartido (`ctrl01` en entorno lab)

📌 **Nunca desde compute nodes directamente**.

---

## 1️⃣ Preparación del entorno de Kolla-Ansible

### 1.1 Activar entorno virtual (si aplica)

```bash
source ~/kolla-venv/bin/activate
```

Verificar:

```bash
which kolla-ansible
```

---

### 1.2 Verificar versión

```bash
kolla-ansible --version
```

Debe corresponder a **OpenStack Bobcat**.

---

## 2️⃣ Instalación de dependencias

```bash
kolla-ansible install-deps
```

Esto instala:

- Ansible collections
- Roles requeridos
- Dependencias del sistema

📌 **Debe ejecutarse solo una vez por nodo de despliegue**.

---

## 3️⃣ Bootstrap de servidores

Este paso prepara **TODOS los nodos** (control y compute):

```bash
kolla-ansible bootstrap-servers -i ansible/openstack/inventory/hosts.ini
```

### ¿Qué hace este paso?

- Configura usuarios y permisos
- Ajusta sysctl
- Instala Docker
- Configura acceso SSH
- Sincroniza relojes

---

## 4️⃣ Validación de conectividad SSH

Kolla-Ansible requiere **SSH passwordless**.

Validar manualmente:

```bash
ansible -i ansible/openstack/inventory/hosts.ini all -m ping
```

Resultado esperado:

```text
SUCCESS => ping
```

❌ Errores comunes:

- Claves SSH no distribuidas
- Usuario incorrecto
- Firewall activo

---

## 5️⃣ Prechecks oficiales de Kolla-Ansible

Este es el paso **más importante** antes del deploy:

```bash
kolla-ansible prechecks -i ansible/openstack/inventory/hosts.ini
```

---

## 6️⃣ Qué valida `prechecks`

### 🔹 Sistema operativo

- Versión Ubuntu soportada
- Kernel compatible

### 🔹 Networking

- Interfaces declaradas existen
- Resolución DNS
- Reachability entre nodos
- VIPs libres

### 🔹 Puertos

- 5000 (Keystone)
- 3306 (MariaDB)
- 5672 (RabbitMQ)
- 9696 (Neutron)

### 🔹 Contenedores

- Docker funcionando
- Espacio disponible

### 🔹 Storage

- VG `cinder-volumes` presente en compute nodes


---

## 7️⃣ Validación de configuración básica

Verificación de parámetros críticos de Kolla-Ansible.

> 📌 **Nota importante**  
> Kolla-Ansible **solo utiliza** el archivo ubicado en: `/etc/kolla/globals.yml`
> Este archivo es el que el operador **mantiene, edita y versiona**.  
> Cualquier otro `globals.yml` fuera de esta ruta **NO es utilizado** durante `prechecks` ni `deploy`.

---

### 📄 Parámetros mínimos requeridos

En `/etc/kolla/globals.yml` deben existir al menos las siguientes líneas:

```yaml
kolla_base_distro: "ubuntu"
openstack_release: "bobcat"
```

---

### 🔍 Comandos de validación

```bash
grep kolla_base_distro /etc/kolla/globals.yml
grep openstack_release /etc/kolla/globals.yml
```

### ✅ Resultado esperado

```text
kolla_base_distro: ubuntu
openstack_release: bobcat
```

---

### 🛠️ ¿Dónde reemplazar o editar `globals.yml`?

Si ya existe un archivo previo, **debe editarse o reemplazarse directamente** en:

```bash
/etc/kolla/globals.yml
```

Ejemplo para reemplazar completamente:

```bash
sudo cp globals.yml /etc/kolla/globals.yml
```

Ejemplo para editarlo:

```bash
sudo vi /etc/kolla/globals.yml
```

📌 Después de cualquier cambio en este archivo, **siempre se deben ejecutar nuevamente los `prechecks`**.

---

## 7️⃣ Validación específica de LVM (Cinder)

En cada compute node:

```bash
vgs
```

Resultado esperado:

```text
cinder-volumes   1   0   0   20g
```

📌 Si este paso falla, **NO continuar** con el deploy.

---

## 8️⃣ Validación de VIPs

Verificar que las IPs:

- `kolla_internal_vip_address`
- `kolla_external_vip_address`

NO estén en uso:

```bash
ping <VIP>
```

Resultado esperado:

```text
Destination Host Unreachable
```

---

## 9️⃣ Errores comunes y resolución

### ❌ Error: interface not found

- Verificar `network_interface`
- Verificar `neutron_external_interface`

### ❌ Error: LVM not found

- Confirmar creación de VG
- Revisar `globals.yml`

### ❌ Error: Docker not running

- Validar servicio Docker
- Revisar logs en `/var/log/kolla/`

---

## ✅ Criterio de salida (Exit Criteria)

Este paso se considera **completo** cuando:

- `bootstrap-servers` finaliza sin errores
- `prechecks` finaliza sin errores
- Todos los nodos responden a Ansible
- LVM validado en compute nodes

📌 Solo después de esto se puede avanzar a:

👉 **PASO 12 — Deploy de OpenStack con Kolla-Ansible**

---

☁️ *Este paso garantiza una base sólida antes del despliegue de OpenStack.*
