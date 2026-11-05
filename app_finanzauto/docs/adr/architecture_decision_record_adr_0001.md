# ADR 0001: Adopción del Patrón Backend for Frontend (BFF) y Cómputo Serverless con AWS Fargate

* **Estado:** Aceptado
* **Fecha:** 2024-10-24
* **Decisores:** Equipo de Arquitectura de Software, Líder Técnico Móvil, DevOps Lead
* **Proyecto:** App Mobile Finanzauto 2024

---

## 1. Contexto del Problema

La nueva aplicación móvil de Finanzauto necesita interactuar con múltiples sistemas centrales existentes:
1. **Sistemas Legados y Core:** El core financiero (*Faizer*) y el radicador corporativo (*Atenea*), diseñados históricamente para consumo web y operaciones backoffice.
2. **Proveedores Externos:** Pasarela transaccional (*PlacetoPay*), sistemas de autenticación y servicios de campañas (*DataPRO*).

### Desafíos identificados:
* **Sobrecarga de red móvil (Chatty I/O):** Si el cliente móvil realizara llamadas directas a cada sistema (por ejemplo, para consultar el crédito, calcular la semaforización de mora y verificar los radicados abiertos), se dispararían múltiples peticiones HTTP por pantalla, degradando la batería y el rendimiento en redes móviles 4G/5G.
* **Acoplamiento directo al cliente:** Exponer las APIs internas de Atenea o Faizer directamente a internet obligaría a la aplicación móvil a lidiar con esquemas de autenticación internos, formatos XML/SOAP antiguos y lógica de negocio sensible (como las reglas de asignación territorial y analistas).
* **Gestión de infraestructura:** El equipo requiere una solución de cómputo elástica, con aislamiento de red VPC, sin la carga operativa de administrar parches de sistemas operativos en servidores virtuales (instancias EC2).

---

## 2. Decisión Arquitectural

Se decide implementar:

1. **El patrón arquitectónico Backend for Frontend (BFF) para el canal móvil**, desarrollado como una capa intermedia ligera (Node.js/NestJS o Spring Boot).
2. **Despliegue sobre AWS ECS con AWS Fargate (Serverless Containers)** dentro de subredes privadas en una Virtual Private Cloud (VPC), con terminación TLS en un Application Load Balancer (ALB) y filtrado perimetral con AWS WAF.

---

## 3. Justificación Técnica

### Respecto al Patrón BFF:
* **Agregación y Adaptación de Payloads:** El BFF consolida llamadas. Por ejemplo, en una sola petición de la pantalla *Pago Ágil*, el BFF consulta *Faizer*, computa la semaforización de días de mora (Verde/Naranja/Rojo) y entrega un JSON optimizado para la vista móvil.
* **Aislamiento y Seguridad:** La app móvil únicamente se comunica con el API Gateway/BFF usando tokens JWT. Las credenciales de integración, VPNs corporativas y accesos directos a Atenea/Faizer nunca tocan el dispositivo del cliente.
* **Evolución desacoplada:** Si el contrato de Atenea cambia internamente, solo se modifica el BFF; no se requiere forzar una actualización de versión de la app en Google Play Store ni Apple App Store.

### Respecto a AWS ECS Fargate:
* **Cero gestión de servidores:** Elimina la necesidad de aprovisionar, actualizar parches de seguridad y gestionar clusters de EC2.
* **Escalabilidad horizontal automática:** Permite escalar contenedores de forma elástica ante picos de tráfico (por ejemplo, días de quincena o vencimiento de cuotas).
* **Seguridad de red Multi-AZ:** Los contenedores se ejecutan en subredes privadas sin IP pública; la salida a servicios externos se realiza mediante NAT Gateway y la conexión a las bases de datos locales ocurre a través de canales seguros (VPN / Direct Connect).

---

## 4. Opciones Descartadas

* **Opción A: Comunicación directa desde la App Móvil a Microservicios Core (Client-to-Microservice)**
  * *Motivo de descarte:* Alto consumo de ancho de banda móvil, exposición innecesaria de APIs corporativas al internet público y complejidad excesiva en el código cliente para orquestar llamadas transaccionales con PlacetoPay y Atenea.
* **Opción B: Despliegue en Máquinas Virtuales Tradicionales (Amazon EC2)**
  * *Motivo de descarte:* Exceso de sobrecarga operativa para el equipo (parches de SO, configuración manual de auto-scaling groups y mayor riesgo de deriva de configuración).
* **Opción C: Kubernetes Completo (Amazon EKS)**
  * *Motivo de descarte:* Complejidad operacional desproporcionada para el tamaño del equipo actual y el volumen inicial de microservicios de la solución móvil.

---

## 5. Consecuencias

### Positivas:
* **Optimización de experiencia de usuario:** Respuestas más rápidas en la app móvil gracias a payloads compactos y agregados.
* **Reducción de superficie de ataque:** El perímetro público se reduce al ALB/WAF y al BFF; los cores internos permanecen en la red privada.
* **Mantenibilidad:** Desacoplamiento entre el ciclo de vida del frontend móvil y los servicios backend legados.

### Negativas / Riesgos mitigados:
* **Salto de red adicional (Hop Latency):** Introduce un salto de red entre el dispositivo y los cores.  
  * *Mitigación:* Despliegue en la misma región de AWS (`us-east-1`) con conexiones internas de baja latencia hacia la VPN corporativa.
* **Riesgo de convertir el BFF en un "Monolito de Paso":**  
  * *Mitigación:* El BFF debe limitarse estrictamente a tareas de agregación, transformación de payloads y autenticación de canal; la lógica transaccional de negocio debe residir en los microservicios especializados (Pagos, Trámites).