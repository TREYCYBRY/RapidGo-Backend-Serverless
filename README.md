# Backend Serverless para Aplicación Móvil - RapidGo

**Curso:** Computación en la Nube | Semestre 2026-1  
**Profesor:** Julian David Florez Sanchez  
**Integrantes del grupo:**  
* Treycy Bridney Andres Sebastian  
* Adler Clin Omonte Sanchez  

---

## Matriz de Control de Cambios

| ID | Responsable | Observación | Fecha |
|:---:|---|---|---|
| 01 |Adler Omonte Sanchez y Treycy Andres Sebastian |Inicializacion del Caso de Uso | 07/05/2026 |
| 02 | | | |
| 03 | | | |

---

## Índice

1. [Caso de Uso](#caso-de-uso)
   - [3.1 Descripción de la empresa](#31-descripción-de-la-empresa)
   - [3.2 Situación tecnológica actual y problemas identificados](#32-situación-tecnológica-actual-y-problemas-identificados)
   - [3.3 Requerimientos para la nueva arquitectura](#33-requerimientos-para-la-nueva-arquitectura)
   - [3.4 Restricciones del proyecto](#34-restricciones-del-proyecto)
2. [Modelo C4](#modelo-c4)
   - [C1 - Contexto](#c1---contexto)
   - [C2 - Contenedores](#c2---contenedores)
   - [C3 - Componentes](#c3---componentes)
3. [Decisiones Arquitectónicas (ADRs)](#decisiones-arquitectónicas-adrs)
4. [Evidencias de Implementación](#evidencias-de-implementación)
5. [Conclusiones](#conclusiones)

---

## Caso de Uso

### 3.1 Descripción de la empresa
RapidGo es una startup colombiana de servicios de domicilios fundada en 2022 que opera actualmente en Medellín, Manizales y Pereira. La plataforma conecta a clientes con restaurantes y tiendas locales a través de una aplicación móvil disponible en Android e iOS, desarrollada en React Native, y cuenta con una red de 340 repartidores activos.

En sus primeros dos años de operación, RapidGo procesó en promedio 1.200 pedidos diarios con picos de hasta 4.500 pedidos en días festivos y fines de semana. Su modelo de negocio cobra una comisión del 18% por pedido completado, lo que hace que la disponibilidad del sistema sea directamente proporcional a sus ingresos: cada minuto de caída representa pérdidas estimadas de $180.000 COP en horas pico.

### 3.2 Situación tecnológica actual y problemas identificados
El backend actual es una aplicación monolítica en Node.js desplegada en un servidor dedicado en un datacenter de Medellín. El equipo de tecnología ha documentado los siguientes problemas críticos que bloquean el crecimiento de la empresa:

* **Escalabilidad manual:** En horas pico (12m-2pm y 6pm-9pm) el servidor se satura y el tiempo de respuesta de la API supera los 8 segundos, generando cancelaciones espontáneas de pedidos estimadas en un 12% del tráfico.
* **Costo fijo ineficiente:** El servidor dedicado cuesta $4.200.000 COP mensuales independientemente del tráfico. En horas de baja demanda (2am-8am) el uso de CPU no supera el 4%, lo que representa un desperdicio significativo de recursos.
* **Despliegues con tiempo de inactividad:** Cualquier actualización del backend requiere 20-30 minutos de inactividad programada, impactando ventas nocturnas y generando mala experiencia de usuario.
* **Notificaciones no confiables:** El sistema actual de push notifications tiene una tasa de entrega del 67% debido a la falta de integración directa con FCM y APNs, generando confusión en clientes sobre el estado de sus pedidos.
* **Sin tolerancia a fallos:** No existe redundancia ni plan de recuperación. Un fallo de hardware implica caída total del servicio con tiempos históricos de restauración de 2 a 6 horas.
* **Deuda técnica en autenticación:** El manejo de tokens JWT está implementado de forma artesanal en el monolito, sin un API Gateway centralizado, lo que dificulta agregar nuevos clientes (app web, API pública) en el futuro.

### 3.3 Requerimientos para la nueva arquitectura
El equipo directivo de RapidGo ha definido los siguientes requerimientos no funcionales que la nueva arquitectura debe cumplir. El grupo debe verificar en los ADRs que las decisiones tomadas satisfacen estos requerimientos:
<img width="500" height="175" alt="image" src="https://github.com/user-attachments/assets/3cff8f3d-04ff-4bb7-927b-9a6e3accca47" />


### 3.4 Restricciones del proyecto
El grupo debe considerar estas restricciones al tomar las decisiones documentadas en los ADRs. Ignorar una restricción sin justificarlo explícitamente en el ADR correspondiente se considera un error de diseño:

* **Stack tecnológico:** El equipo de desarrollo de RapidGo tiene experiencia en Node.js y Python, pero no en Java ni .NET. Las Functions deben implementarse en uno de estos lenguajes.
* **Presupuesto inicial limitado:** Se deben priorizar servicios con capa gratuita. El gasto mensual en Azure no debe superar los $50 USD durante la fase piloto.
* **Migración de Base de Datos:** La base de datos actual es MySQL relacional con 3 años de datos históricos. Si se propone un cambio de paradigma (relacional a NoSQL), debe estar explícitamente justificado en el ADR-02.
* **Cumplimiento normativo y latencia:** Los datos de usuarios colombianos deben almacenarse en la región *Brazil South* o *East US* por latencia y consideraciones de soberanía de datos.
* **Compatibilidad de cliente:** La app móvil en React Native no se rediseñará. La nueva API debe mantener compatibilidad con los contratos de endpoints actuales (mismas rutas y estructura de respuesta JSON).
* **Operación de Infraestructura:** El equipo de infraestructura es de una sola persona. La solución debe minimizar la carga operativa y evitar servicios que requieran administración manual de servidores o clústeres.

---

## Modelo C4

### C1 - Contexto

<img width="1364" height="664" alt="image" src="https://github.com/user-attachments/assets/ae6381d3-5962-4e5f-973f-fc1800b75681" />


### Descripción del Modelo C1 (Contexto del Sistema)

El presente diagrama de contexto modela a RapidGo como un sistema de caja negra, identificando claramente a los actores principales como cliente, repartidor, administrador y sus interacciones con los sistemas externos como app móvil, pasarela de pagos, FCM, APNS.

**Componentes:**

* **Sistema Central RapidGo:** Es el núcleo orquestador que centraliza la lógica de negocio, procesa los pedidos y almacena la información, sirviendo como único punto de entrada para las plataformas cliente.

**Actores:**

* **Administrador,Clientes y Repartidores:** Interactúan de forma indirecta con el sistema central utilizando la App Móvil como interfaz para configurar la plataforma, solicitar pedidos y actualizar estados segun el rol.

**Sistemas Externos:**

* **App Móvil:** Se modela como un sistema externo dado que está desarrollada en React Native y no será rediseñada y actúa como el cliente principal que consume la API del backend mediante JSON/HTTPS.
* **Pasarela de Pagos:** Plataforma segura externa encargada de procesar las transacciones financieras y cobrar la comisión del 18% correspondiente al modelo de negocio.
* **Servicios de Notificaciones (FCM y APNS):** Proveedores de infraestructura para el ecosistema Android e iOS. El backend se comunica con ellos para despachar alertas en tiempo real, solucionando los problemas de entrega del sistema anterior.


### C2 - Contenedores

<img width="1454" height="954" alt="image" src="https://github.com/user-attachments/assets/b8667089-2f1a-4499-b1fb-e413a1da00bd" />

### Descripción del Modelo C2 (Contenedores)

El presente diagrama de contenedores desglosa el interior del sistema "Plataforma RapidGo-Backend Serverless", mostrando las piezas arquitectónicas encargadas de ejecutar la lógica de negocio y los protocolos de comunicación establecidos entre ellas.

**Punto de Entrada y Actores:**

* **Usuarios (Administrador, Repartidores, Clientes):** Interactúan a través de la Interfaz de Usuario (UI) de la App Móvil unificada.
* **App Móvil:** Funciona como el cliente principal desarrollado en React Native. Centraliza las interacciones de todos los roles y realiza las solicitudes hacia la plataforma mediante contratos JSON/HTTPS.

**Contenedores Lógicos Internos:**

* **API Gateway:** Actúa como el punto de entrada único al backend. Su responsabilidad es recibir las peticiones externas, gestionar la seguridad en la autenticación y autorización y enrutar el tráfico mediante HTTPS hacia el nivel de procesamiento.
* **Motor de Reglas de Negocio:** Contenedor de cómputo Serverless que ejecuta la lógica de dominio central bajo demanda
* **Base de Datos (NoSQL):** Capa de persistencia transaccional altamente escalable. El motor de reglas de negocio lee y escribe en ella utilizando un SDK para gestionar perfiles, catálogos y el ciclo de vida de las órdenes.
* **Almacén de Archivos (Object Storage):** Repositorio inmutable que se comunica vía HTTPS, encargado de la persistencia de recursos binarios como comprobantes fotográficos de entrega y recursos estáticos.
* **Motor de Notificaciones (Push Orchestration):** Servicio de publicación y suscripción que recibe comandos asíncronos vía AMQP desde el motor de negocio. Centraliza y orquesta los eventos que deben enviarse a los usuarios.

**Sistemas Externos de Interacción:**

* **Pasarela de Pagos:** Recibe de forma segura los datos de cobro vía HTTPS para procesar la comisión correspondiente al modelo de negocio de RapidGo.
* **Servicios Push (FCM y APNS):** Reciben las cargas de trabajo despachadas por el Motor de Notificaciones mediante APIs nativas para entregar alertas en tiempo real a los ecosistemas Android e iOS.

---
> *Nota: Este documento se irá actualizando conforme avance el desarrollo del proyecto.*
