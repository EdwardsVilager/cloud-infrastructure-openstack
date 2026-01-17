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
# Crear cookie SIN newline
echo -n "hXhYB1jRKj27ea8hNkGVGmTPq7qDUMxcZSSeI4Oo" > /tmp/.erlang.cookie

# Permisos estrictos
chmod 400 /tmp/.erlang.cookie
chown 999:999 /tmp/.erlang.cookie

# Recrear toolbox
docker rm -f kolla_toolbox

docker run -d --name kolla_toolbox \
  --network host \
  -v /etc/kolla:/etc/kolla \
  -v /var/log/kolla:/var/log/kolla \
  -v /tmp/.erlang.cookie:/var/lib/rabbitmq/.erlang.cookie \
  --user root \
  harbor.infra.local/kolla/openstack.kolla/kolla-toolbox:2024.2-rocky-9 \
  tail -f /dev/null

docker exec kolla_toolbox chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie
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
  harbor.infra.local/kolla/openstack.kolla/kolla-toolbox:2024.2-rocky-9 \
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
kolla-ansible post-deploy -i ansible/openstack/inventory/hosts.ini --become
```

Esto realiza:

- Exporta variables de entorno para CLI (`/etc/kolla/admin-openrc.sh`)
- Configura usuarios y roles
- Carga imágenes base en Glance (ej. Cirros)
- Valida endpoints
- Verifica que MariaDB y ProxySQL respondan correctamente

---

## 4️⃣ Validación de servicios

### 🔹 Instalar openstackclient

```bash
pip3 install --user python-openstackclient
```

### 🔹 Cargar variables de acceso a OpenStack

```bash
sudo bash
source /etc/kolla/admin-openrc.sh

```

### 🔹 Proyectos OpenStack

```bash
/root/.local/bin/openstack  project list
```

Resultado esperado

```text
+----------------------------------+---------+
| ID                               | Name    |
+----------------------------------+---------+
| 054fdeee7e104c52afd4d7d4afd16956 | service |
| a7ab7ee0db3d4242b9eacd23fac6d4be | admin   |
+----------------------------------+---------+
```

### 🔹 Lista de servicios OpenStack

```bash
/root/.local/bin/openstack service list
```

Resutado esperado:

```text
+----------------------------------+-----------+-----------+
| ID                               | Name      | Type      |
+----------------------------------+-----------+-----------+
| 26ba4129b6dc49a6b99b52eb68b67a1b | keystone  | identity  |
| 8aae7eefffa74ce691f65ba028cd108e | neutron   | network   |
| bf01480f5de04599b0091fee99001bca | nova      | compute   |
| d6e1997b54bd4f46ab304ec7cd00510f | placement | placement |
| f84a21d3bc554bddab30cbe29eb2a5b4 | glance    | image     |
+----------------------------------+-----------+-----------+
```

### 🔹 Nodos reportando estado correcto

```bash
/root/.local/bin/openstack compute service list
```

Resutado esperado:

```text
+--------------------------------------+----------------+--------+----------+---------+-------+----------------------------+
| ID                                   | Binary         | Host   | Zone     | Status  | State | Updated At                 |
+--------------------------------------+----------------+--------+----------+---------+-------+----------------------------+
| 967edd52-448c-4a1e-8036-868e4e53d658 | nova-scheduler | ctrl-1 | internal | enabled | up    | 2026-01-15T01:29:24.000000 |
| ed1ffef2-964d-428f-945c-69d5021ac12b | nova-scheduler | ctrl-2 | internal | enabled | up    | 2026-01-15T01:29:23.000000 |
| c51b558b-f90d-40c9-b8cf-876b5984eb0a | nova-scheduler | ctrl-3 | internal | enabled | up    | 2026-01-15T01:29:24.000000 |
| f4186dc7-1862-4b7e-b497-607fffa43d82 | nova-conductor | ctrl-2 | internal | enabled | up    | 2026-01-15T01:29:22.000000 |
| 2b2bfa49-232a-4004-8329-b8f98afcdc13 | nova-conductor | ctrl-1 | internal | enabled | up    | 2026-01-15T01:29:23.000000 |
| 3fbd2b0b-626c-4b5a-943b-cbc857bfbb9f | nova-conductor | ctrl-3 | internal | enabled | up    | 2026-01-15T01:29:24.000000 |
| a5c82057-8685-4e14-bc5e-122784854a26 | nova-compute   | cmp-1  | nova     | enabled | up    | 2026-01-15T01:29:24.000000 |
| e825db5e-7395-4e85-95a6-bcaf73ae16e7 | nova-compute   | cmp-2  | nova     | enabled | up    | 2026-01-15T01:29:24.000000 |
+--------------------------------------+----------------+--------+----------+---------+-------+----------------------------+
```

### 🔹 Endpoints OpenStack

```bash
/root/.local/bin/openstack endpoint list
```

Resutado esperado:

```text
+----------------------------------+-----------+--------------+--------------+---------+-----------+----------------------------------+
| ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                              |
+----------------------------------+-----------+--------------+--------------+---------+-----------+----------------------------------+
| 0ea23712538c4379ad7fec8711a8e175 | RegionOne | keystone     | identity     | True    | public    | http://openstack.local:5000      |
| 1fc7e370baa149488c05b8b5a5395631 | RegionOne | placement    | placement    | True    | public    | http://openstack.local:8780      |
| 3e852e9c907642ce9f4cca7fc7bf70e1 | RegionOne | placement    | placement    | True    | internal  | http://50.50.1.100:8780          |
| 3fdb6b827a0347b9a7489632df59b8b4 | RegionOne | glance       | image        | True    | public    | http://openstack.local:9292      |
| 6634671a5ada487db305761a913f6245 | RegionOne | neutron      | network      | True    | internal  | http://50.50.1.100:9696          |
| 857e6575468446e2a018512aac04c7bd | RegionOne | glance       | image        | True    | internal  | http://50.50.1.100:9292          |
| a0e48315f0eb412985e28ad1190cf506 | RegionOne | neutron      | network      | True    | public    | http://openstack.local:9696      |
| e3b6fd643ea64de2ad90dc6d86aaa758 | RegionOne | keystone     | identity     | True    | internal  | http://50.50.1.100:5000          |
| f116df335f3f43d98a76f08000bb36ad | RegionOne | nova         | compute      | True    | internal  | http://50.50.1.100:8774/v2.1     |
| fa52fb7292ed45ce98dbc0f3de648610 | RegionOne | nova         | compute      | True    | public    | http://openstack.local:8774/v2.1 |
+----------------------------------+-----------+--------------+--------------+---------+-----------+----------------------------------+
```

### 🔹 Lista los servicios de Cinder

```bash
/root/.local/bin/openstack volume service list
```

Resutado esperado:

```text
internalURL endpoint for block-storage service in RegionOne region not found
```

> No esta activado Cinder

### 🔹 Lista los agentes de red

```bash
/root/.local/bin/openstack network agent list
```

Resutado esperado:

```text
+--------------------------------------+------------------------------+-------+-------------------+-------+-------+----------------+
| ID                                   | Agent Type                   | Host  | Availability Zone | Alive | State | Binary         |
+--------------------------------------+------------------------------+-------+-------------------+-------+-------+----------------+
| afa28615-b0ce-43c4-aaee-feef327e9987 | OVN Controller Gateway agent | cmp-1 |                   | :-)   | UP    | ovn-controller |
| ed167c8a-3992-446b-85c7-b037fec4d954 | OVN Controller Gateway agent | cmp-2 |                   | :-)   | UP    | ovn-controller |
+--------------------------------------+------------------------------+-------+-------------------+-------+-------+----------------+
```

> Falta mas servicios de red (parcial funcionando)

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

---

## 5️⃣ Acceso a Horizon

En el nodo de control, si tienes External VIP:

```text
http://<kolla_external_fqdn>
http://50.50.1.101/horizon
```

Login con:

- Usuario: `admin`
- Password: el generado en `/etc/kolla/passwords.yml` → `keystone_admin_password`

> Puedes validar las claves de accesso en archivo `/etc/kolla/public-openrc.sh`

---

## 6️⃣ Validación de funcionamiento básico

Crear proyecto de prueba:

```bash
/root/.local/bin/openstack project create demo
```

Resultado esperado:

```text
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description |                                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 8821a428ee484eb594a0d2dac6cc8f10 |
| is_domain   | False                            |
| name        | demo                             |
| options     | {}                               |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+
```

Crear usuario y asignar rol:

```bash
/root/.local/bin/openstack user create demo --project demo --password demopass
/root/.local/bin/openstack role add --project demo --user demo _member_
```

Resultado esperado:

```text
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| default_project_id  | 8821a428ee484eb594a0d2dac6cc8f10 |
| domain_id           | default                          |
| email               | None                             |
| enabled             | True                             |
| id                  | 9d3ea43273214ed29e789730e94beb2a |
| name                | demo                             |
| description         | None                             |
| password_expires_at | None                             |
| options             | {}                               |
+---------------------+----------------------------------+
```

Crear red y subred (para lab):

```bash
/root/.local/bin/openstack network create demo-net
/root/.local/bin/openstack subnet create demo-subnet --network demo-net --subnet-range 192.168.200.0/24
```

Resultado esperado:

```text
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2026-01-15T02:06:13Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | 65e2c535-c224-4e7f-abe7-3850e38dcbb1 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | False                                |
| is_vlan_qinq              | None                                 |
| is_vlan_transparent       | None                                 |
| mtu                       | 1442                                 |
| name                      | demo-net                             |
| port_security_enabled     | True                                 |
| project_id                | a7ab7ee0db3d4242b9eacd23fac6d4be     |
| provider:network_type     | geneve                               |
| provider:physical_network | None                                 |
| provider:segmentation_id  | 1702                                 |
| qos_policy_id             | None                                 |
| revision_number           | 1                                    |
| router:external           | Internal                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| updated_at                | 2026-01-15T02:06:13Z                 |
+---------------------------+--------------------------------------+

+----------------------+--------------------------------------+
| Field                | Value                                |
+----------------------+--------------------------------------+
| allocation_pools     | 192.168.200.2-192.168.200.254        |
| cidr                 | 192.168.200.0/24                     |
| created_at           | 2026-01-15T02:07:24Z                 |
| description          |                                      |
| dns_nameservers      |                                      |
| dns_publish_fixed_ip | None                                 |
| enable_dhcp          | True                                 |
| gateway_ip           | 192.168.200.1                        |
| host_routes          |                                      |
| id                   | 0c0f77a5-0abb-487d-8942-272c00fe838f |
| ip_version           | 4                                    |
| ipv6_address_mode    | None                                 |
| ipv6_ra_mode         | None                                 |
| name                 | demo-subnet                          |
| network_id           | 65e2c535-c224-4e7f-abe7-3850e38dcbb1 |
| project_id           | a7ab7ee0db3d4242b9eacd23fac6d4be     |
| revision_number      | 0                                    |
| router:external      | False                                |
| segment_id           | None                                 |
| service_types        |                                      |
| subnetpool_id        | None                                 |
| tags                 |                                      |
| updated_at           | 2026-01-15T02:07:24Z                 |
+----------------------+--------------------------------------+
```

Cargar imagen cirros-0.5.2-x86_64 a Glace

```bash
/root/.local/bin/openstack image create "cirros-0.5.2-x86_64-disk" \
    --file images/cirros-0.5.2-x86_64-disk.img \
    --disk-format qcow2 --container-format bare \
    --public
```

Crear un flavor y lanzar una VM:

```bash
/root/.local/bin/openstack flavor create m1.tiny --ram 512 --disk 1 --vcpus 1
/root/.local/bin/openstack image list
/root/.local/bin/openstack server create --flavor m1.tiny --image cirros-0.5.2-x86_64-disk --network demo-net test-vm
```

Resultado esperado:

```text
+----------------------------+--------------------------------------+
| Field                      | Value                                |
+----------------------------+--------------------------------------+
| OS-FLV-DISABLED:disabled   | False                                |
| OS-FLV-EXT-DATA:ephemeral  | 0                                    |
| description                | None                                 |
| disk                       | 1                                    |
| id                         | e0b0c3ad-a306-4469-92f9-3788caf53dfa |
| name                       | m1.tiny                              |
| os-flavor-access:is_public | True                                 |
| properties                 |                                      |
| ram                        | 512                                  |
| rxtx_factor                | 1.0                                  |
| swap                       | 0                                    |
| vcpus                      | 1                                    |
+----------------------------+--------------------------------------+

+--------------------------------------+--------------------------+--------+
| ID                                   | Name                     | Status |
+--------------------------------------+--------------------------+--------+
| 59650534-f0de-4581-a4fb-98923ed4decb | cirros-0.5.2-x86_64-disk | active |
+--------------------------------------+--------------------------+--------+
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
