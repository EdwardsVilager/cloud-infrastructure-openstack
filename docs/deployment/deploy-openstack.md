# PASO 12 — Deploy de OpenStack con Kolla-Ansible

Este documento describe el proceso completo de **despliegue de OpenStack** utilizando **Kolla-Ansible** en un entorno Ubuntu 20.04 con OpenStack Bobcat.

Se asume que los pasos previos (bootstrap, LVM, networking, prechecks) se han completado con éxito.

---

## 🎯 Objetivos del PASO 12

- Instalar y configurar todos los servicios de OpenStack
  - Keystone, Glance, Nova, Neutron, Cinder
- Configurar alta disponibilidad para control nodes
- Validar servicios corriendo y accesibles
- Preparar el acceso a Horizon y CLI
- Registrar VMs y Volúmenes iniciales
- Verificar conectividad de bases de datos con ProxySQL

---

## 🧭 ¿Dónde se ejecutan estos comandos?

Todos los comandos se ejecutan **desde el nodo de despliegue** (`deploy01`) o nodo compartido (`ctrl01` en laboratorio).

📌 Nunca ejecutar directamente desde los compute nodes.

---

## 1️⃣ Activar entorno virtual de Kolla-Ansible

```bash
source ~/kolla-venv19/bin/activate
which kolla-ansible
```

2️⃣ Ejecutar deploy de OpenStack

```bash
kolla-ansible deploy -i ansible/openstack/inventory/hosts.ini
kolla-ansible deploy -i ansible/openstack/inventory/hosts.ini -e @/etc/kolla/globals.yml
```

Qué hace este comando

- Despliega contenedores Docker para cada servicio
- Configura alta disponibilidad (HAProxy + Keepalived)
- Crea usuarios, proyectos y endpoints
- Inicializa bases de datos y colas
- Configura networking (OVN, provider network)
- Inicializa backend de Cinder (LVM en lab)

> ⏳ Dependiendo del hardware, puede tardar 15–30 minutos en un lab con 3 control + 2 compute.  
> Ruta de playbooks: `/home/mphadmin/.local/share/kolla-ansible/ansible/roles/`

### 🔹 Reconfigurar HAProxy

```bash
kolla-ansible deploy -i ansible/openstack/inventory/hosts.ini --tags haproxy
kolla-ansible deploy -i ansible/openstack/inventory/hosts.ini --skip-tags keystone
kolla-ansible reconfigure -i ansible/openstack/inventory/hosts.ini
```

---

### 🔹 Si falla MariaDB

**Problema común:** La tarea `Creating shard root mysql user` puede fallar si el contenedor `kolla_toolbox` **no está corriendo** o si ProxySQL no acepta la conexión todavía.

**Solución:**

### 1. Asegurarse de que la imagen toolbox esté disponible (Opcional ** Descarga todas las Imagenes)

```bash
kolla-ansible pull -i ansible/openstack/inventory/hosts.ini
```

### 2. Descargar  `kolla_toolbox` en nodo con VIP

```bash
docker pull kolla/kolla-toolbox:2024.2-ubuntu-noble
```

### 3. Verificar que `kolla_toolbox` se pueda levantar

```bash
docker run -d --name kolla_toolbox \
  --network host \
  -v /etc/kolla:/etc/kolla \
  -v /var/log/kolla:/var/log/kolla \
  kolla/kolla-toolbox:2024.2-ubuntu-noble tail -f /dev/null
```

> Esto solo es necesario si Kolla no logra levantarlo automáticamente.

### 3. Validar conectividad a MariaDB vía ProxySQL

```bash
docker exec -it mariadb mysql \
  -h 50.50.1.100 \
  -P 6032 \
  -u root_shard_0 \
  -p<password> \
  -e "SHOW DATABASES;"
```

> Reemplaza `<password>` por el valor en `/etc/kolla/passwords.yml` → `database_password`.

### 4. Reintentar deploy de MariaDB si falló

```bash
kolla-ansible deploy -i ansible/openstack/inventory/hosts.ini --tags mariadb
kolla-ansible mariadb-recovery -i ansible/openstack/inventory/hosts.ini
```

---

### 🔹 Si falla OVN Controller

**Problema común:** La tarea `Create br-int bridge on OpenvSwitch` puede fallar si el contenedor `kolla_toolbox` **no está corriendo** en cada nodo de compute.

**Solución:**

### 1. Descargar  `kolla_toolbox` en nodo con VIP

```bash
docker pull kolla/kolla-toolbox:2024.2-ubuntu-noble
```

### 2. Verificar que `kolla_toolbox` se pueda levantar

```bash
docker rm -f kolla_toolbox
docker run -d --name kolla_toolbox \
  --network host \
  --privileged \
  --security-opt apparmor=unconfined \
  --pid=host \
  -u $(id -u):$(id -g) \
  -v /etc/kolla:/etc/kolla \
  -v /var/log/kolla:/var/log/kolla \
  -v /run/openvswitch:/run/openvswitch:shared \
  -v /var/run/openvswitch:/var/run/openvswitch:shared \
  kolla/kolla-toolbox:2024.2-ubuntu-noble \
  tail -f /dev/null
```

> Esto solo es necesario si Kolla no logra levantarlo automáticamente.

---

### 🔹 Eliminar Kolla-Ansible containers y configuración

```bash
kolla-ansible destroy -i ansible/openstack/inventory/hosts.ini --yes-i-really-really-mean-it
```

> Si se cambia la ruta de Docker (`/var/lib/docker`), crear enlace simbólico:

```bash
rm -Rf /var/lib/docker/
ln -s /docker /var/lib/docker/var/lib/docker
```

---

## 3️⃣ Post-deploy — Inicialización de servicios

```bash
kolla-ansible post-deploy -i ansible/openstack/inventory/hosts.ini
```

Esto realiza:

- Exporta variables de entorno para CLI (`/etc/kolla/admin-openrc.sh`)
- Configura usuarios y roles
- Carga imágenes base en Glance (ej. Cirros)
- Valida endpoints
- Verifica que MariaDB y ProxySQL respondan correctamente

---

## 4️⃣ Validación de servicios

### 🔹 Contenedores corriendo

```bash
docker ps
```

Se esperan contenedores como:

- `keystone_api`
- `glance_api`
- `nova_api` / `nova_compute`
- `neutron_server`
- `cinder_api` / `cinder_volume`
- `haproxy` / `keepalived` (si HA)
- `redis`, `rabbitmq`, `mariadb`, `proxysql`

### 🔹 Endpoints OpenStack

```bash
openstack service list
openstack endpoint list
```

### 🔹 Nodos reportando estado correcto

```bash
openstack compute service list
openstack volume service list
```

---

## 5️⃣ Acceso a Horizon

En el nodo de control, si tienes External VIP:

```text
http://50.50.1.101/horizon
```

Login con:

- Usuario: `admin`
- Password: el generado en `/etc/kolla/passwords.yml` → `keystone_admin_password`

---

## 6️⃣ Validación de funcionamiento básico

Crear proyecto de prueba:

```bash
openstack project create demo
```

Crear usuario y asignar rol:

```bash
openstack user create demo --project demo --password DEMO_PASS
openstack role add --project demo --user demo _member_
```

Crear red y subred (para lab):

```bash
openstack network create demo-net
openstack subnet create demo-subnet --network demo-net --subnet-range 192.168.200.0/24
```

Crear un flavor y lanzar una VM:

```bash
openstack flavor create m1.tiny --ram 512 --disk 1 --vcpus 1
openstack image list
openstack server create --flavor m1.tiny --image cirros-0.5.2-x86_64-disk --network demo-net test-vm
```

Verificar que la VM quede en estado `ACTIVE`.

---

## 7️⃣ Validaciones de Cinder / LVM

Listar volúmenes:

```bash
openstack volume list
```

Crear un volumen de prueba:

```bash
openstack volume create --size 1 test-vol
```

Adjuntar a VM:

```bash
openstack server add volume test-vm test-vol
```

---

## 8️⃣ Estado del PASO

Este paso se considera completo cuando:

- Todos los servicios de OpenStack están corriendo
- Contenedores reportan estado healthy
- Endpoints y usuarios funcionales
- Red, VMs y volúmenes funcionan correctamente
- MariaDB y ProxySQL responden correctamente al usuario `root_shard_0`

---

## 9️⃣ Próximo paso

👉 **PASO 13 — Configuración de GitOps (Argo CD)**

- Registrar aplicaciones de OpenStack o workloads en repositorio GitOps
- Despliegue automatizado desde Argo CD

> ☁️ Este paso garantiza que tu plataforma OpenStack base está operativa y lista para Day-2.
