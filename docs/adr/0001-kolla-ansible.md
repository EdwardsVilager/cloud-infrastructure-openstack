# ADR 0001: Elección de OpenStack como IaaS y Kolla-Ansible como método de instalación

## Estado

Aceptado

## Fecha

2026-01-11

---

## Contexto

Se requiere una **plataforma de infraestructura como servicio (IaaS)** que sirva como base para el despliegue de plataformas superiores como Kubernetes, CI/CD y workloads empresariales.

Los requerimientos principales son:

- Control total sobre la infraestructura
- Independencia de proveedores cloud públicos
- Soporte para múltiples proyectos y tenants
- Integración con automatización (Terraform, Ansible)
- Arquitectura escalable y enterprise-ready

---

## Decisión

Se decide:

1. Utilizar **OpenStack** como plataforma IaaS.
2. Utilizar **Kolla-Ansible** como método de instalación y operación de OpenStack.

---

## Justificación de OpenStack

OpenStack fue elegido por las siguientes razones:

### ✔️ Arquitectura modular

Permite desplegar y operar solo los servicios necesarios (Nova, Neutron, Cinder, etc.), manteniendo una arquitectura flexible.

### ✔️ Estándar de facto en clouds privados

OpenStack es ampliamente utilizado en entornos enterprise y telco, con un ecosistema maduro y bien documentado.

### ✔️ Integración con Kubernetes

OpenStack se integra de forma nativa con Kubernetes mediante:

- Provisión de VMs
- Volúmenes persistentes (Cinder CSI)
- Redes avanzadas

### ✔️ Multi-tenancy

Soporta separación clara por proyectos, redes y cuotas, lo cual es crítico para entornos compartidos.

---

## Justificación de Kolla-Ansible

Kolla-Ansible fue seleccionado como método de instalación debido a:

### ✔️ Contenerización de servicios

Los servicios de OpenStack se ejecutan como contenedores, simplificando upgrades y operaciones.

### ✔️ Automatización y reproducibilidad

Permite instalaciones declarativas, repetibles y controladas mediante Ansible.

### ✔️ Alineación con prácticas modernas

El uso de contenedores y automatización se alinea con prácticas DevOps y Platform Engineering.

### ✔️ Menor complejidad operativa

Reduce la complejidad frente a instalaciones manuales o altamente personalizadas.

---

## Alternativas consideradas

### ❌ OpenStack + TripleO

- Mayor complejidad
- Curva de aprendizaje más alta
- Más orientado a grandes despliegues telco

### ❌ VMware vSphere

- Dependencia de licenciamiento
- Menor flexibilidad para automatización abierta

### ❌ Cloud público (AWS / GCP / Azure)

- Dependencia de proveedor
- Costos operativos más altos
- Menor control de infraestructura

---

## Consecuencias

### Positivas

- Plataforma IaaS robusta y escalable
- Base sólida para Kubernetes y GitOps
- Arquitectura desacoplada por capas

### Negativas

- Mayor complejidad inicial
- Requiere conocimiento especializado en OpenStack

---

## Notas

Esta decisión es fundamental y **impacta todas las capas superiores**.
Cualquier cambio futuro deberá evaluarse mediante un nuevo ADR.
