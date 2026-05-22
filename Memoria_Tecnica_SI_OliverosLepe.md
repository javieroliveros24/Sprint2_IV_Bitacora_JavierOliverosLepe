# Memoria Técnica — UD07 Sprint 2
## Infraestructura Cloud, Transferencia de Ficheros y El Investigador

| Campo            | Detalle                                  |
|------------------|------------------------------------------|
| **Alumno**       | Javier Oliveros Lepe                     |
| **Módulo**       | Sistemas Informáticos (SI)               |
| **Sprint**       | 2 — UD07                                 |
| **Fecha**        | 22 de mayo de 2026                       |
| **Modalidad**    | Individual                               |
| **Repositorio**  | `Memoria_Tecnica_SI_OliverosLepe.md`     |

---

## Índice

1. [Análisis de Necesidades *(referencia Sprint 1)*](#1-análisis-de-necesidades-referencia-sprint-1)
2. [Estimación de Costes de Infraestructura](#2-estimación-de-costes-de-infraestructura)
3. [Estrategia de Despliegue y Comunicación](#3-estrategia-de-despliegue-y-comunicación)
4. [Justificación Científica](#4-justificación-científica)
5. [Referencias](#5-referencias)

---

## 1. Análisis de Necesidades *(referencia Sprint 1)*

El proyecto consiste en el desarrollo de una **aplicación web full-stack** con arquitectura cliente-servidor. El frontend está construido con **React**, el backend con **Node.js / Express** y la base de datos relacional utilizada es **PostgreSQL**. Toda la aplicación se conteneriza mediante **Docker** para garantizar la portabilidad y reproducibilidad del entorno entre desarrollo y producción.

El marco legal y las licencias de las dependencias principales fueron documentadas en el Sprint 1. La presente memoria amplía esa base con la infraestructura cloud, los protocolos de transferencia seguros y la justificación científica de las decisiones tecnológicas adoptadas.

---

## 2. Estimación de Costes de Infraestructura

### 2.1 Metodología

Para estimar el **Coste Total de Propiedad (TCO) mensual** se ha utilizado la herramienta **Google Sheets** (archivo `Presupuesto_Cloud_Proyecto`). La hoja de cálculo emplea fórmulas dinámicas (`=C*D` para subtotales por línea, `=SUM(...)` para el subtotal global y multiplicación por `0.21` para el IVA), de modo que cualquier cambio en precio o unidades recalcula automáticamente el total.

### 2.2 Proveedor elegido: AWS (Amazon Web Services)

Se selecciona AWS como proveedor principal por su **madurez**, su extensa documentación, su plan gratuito inicial (*Free Tier*) y su integración nativa con Docker a través de **Amazon ECS** y **ECR**.

### 2.3 Tabla de costes mensuales estimados

<img width="1130" height="337" alt="PresupuestoCloud jpg" src="https://github.com/user-attachments/assets/0df1a818-e0fa-4098-b89c-87b10b4f7459" />


> **Nota sobre escalabilidad:** En caso de crecimiento del tráfico se puede migrar a EC2 `t3.large` (+30 €/mes) y RDS `db.t3.small` (+15 €/mes), activando además AWS Auto Scaling Group sin coste adicional de configuración. El diseño actual permite absorber hasta ~500 usuarios concurrentes.

### 2.4 Exportación del presupuesto

La hoja de cálculo completa con fórmulas dinámicas se adjunta como archivo independiente: **`Presupuesto_Cloud_Proyecto.xlsx`**

---

## 3. Estrategia de Despliegue y Comunicación

### 3.1 Protocolo seguro de transferencia de ficheros

Para el despliegue de la aplicación en el servidor de producción se utilizará una combinación de **SFTP** (*SSH File Transfer Protocol*) y un pipeline de **integración continua (CI/CD) con GitHub Actions**, descartando explícitamente el FTP tradicional por transmitir las credenciales en texto plano sin ningún tipo de cifrado, lo que lo hace inviable en cualquier entorno profesional.

El flujo concreto es el siguiente: el código se sube a la rama `main` del repositorio de GitHub, lo que activa automáticamente un *workflow* de GitHub Actions que construye la imagen Docker, la publica en **Amazon ECR** mediante el SDK oficial de AWS (conexión autenticada con claves IAM y cifrada con TLS 1.3), y finalmente despliega el contenedor actualizado en la instancia EC2 a través de **SSH/SFTP** (puerto 22, clave RSA de 4096 bits almacenada como *GitHub Secret*, nunca en el repositorio). Todo el tráfico queda cifrado extremo a extremo: tanto la transferencia de la imagen al registry como la conexión SSH al servidor. Como capa adicional, los *Security Groups* de AWS restringen el acceso SSH únicamente a la IP del runner de GitHub Actions. Este enfoque cumple el criterio **CE 7.e** de uso seguro de servicios de transferencia de ficheros.

### 3.2 Mensajería y alertas de equipo

Para la comunicación de incidencias técnicas y la recepción de alertas automáticas se utilizará **Slack** (plan Pro, 2 usuarios). Se configurará un canal dedicado `#alertas-producción` al que se conectará el *webhook* de **AWS CloudWatch**: cuando la CPU de la instancia EC2 supere el 80 %, la memoria disponible baje de 200 MB o el servidor deje de responder (alarma de *health check*), CloudWatch enviará automáticamente una notificación al canal de Slack con el tipo de alarma, la hora y el recurso afectado. Además, el propio pipeline de GitHub Actions notificará al canal el resultado de cada despliegue (éxito o fallo), con un enlace directo al log de ejecución. Esto cubre el criterio **CE 7.d** de uso de sistemas de mensajería electrónica.

---

## 4. Justificación Científica

### 4.1 Tecnología a justificar: Seguridad en contenedores Docker

La decisión de contenerizar toda la aplicación con Docker no solo obedece a razones de portabilidad, sino también de seguridad. Para fundamentar esta elección con rigor académico se ha localizado el siguiente artículo en **MDPI** (base de datos de acceso abierto indexada en Scopus):

**Artículo:** *"Container Security in Cloud Environments: A Comprehensive Analysis and Future Directions for DevSecOps"*, presentado en la *International Conference on Recent Advances in Science and Engineering* (Dubai, EAU, 2023) y publicado en *Engineering Proceedings*, vol. 59, n.º 1, MDPI, 2023.

### 4.2 Resumen y relación con el proyecto

El estudio analiza el nivel de seguridad real de imágenes Docker ampliamente utilizadas (Nginx, httpd, MySQL, Debian, MariaDB) mediante los escáneres de vulnerabilidades **Trivy** y **Grype** en un entorno de laboratorio controlado. Los autores detectaron vulnerabilidades críticas en varias imágenes base de Docker Hub —entre ellas, dos CVEs críticos en la imagen `httpd` y tres en la imagen `Nginx`— y concluyen que la integración de escáneres automáticos de vulnerabilidades dentro del pipeline CI/CD (práctica conocida como **DevSecOps**) reduce significativamente el riesgo de desplegar imágenes comprometidas en producción.

Esta conclusión apoya directamente la arquitectura del presente proyecto: en el pipeline de GitHub Actions se incluirá una etapa de análisis de la imagen Docker con **Trivy** antes de cada despliegue, bloqueando el pipeline si se detecta alguna vulnerabilidad crítica. De esta forma, la seguridad queda integrada en el flujo de trabajo sin depender de revisiones manuales, siguiendo las recomendaciones del artículo citado.

---

## 5. Referencias

[1] B. Gajbhiye, O. Goel, y P. K. G. Pandian, "Container Security in Cloud Environments: A Comprehensive Analysis and Future Directions for DevSecOps," *Eng. Proc.*, vol. 59, no. 1, p. 57, 2023. doi: 10.3390/engproc2023059057.

[2] H. Zhu, C. Gehrmann, y P. Roth, "Access Security Policy Generation for Containers as a Cloud Service," *SN Comput. Sci.*, vol. 4, no. 748, 2023. doi: 10.1007/s42979-023-02186-1.

[3] Amazon Web Services, "AWS Pricing Calculator — EC2, RDS, S3," aws.amazon.com/pricing, mayo 2026. [En línea]. Disponible en: https://aws.amazon.com/pricing

---

*Documento generado el 22 de mayo de 2026. Versión 1.0.*
*Control de versiones: repositorio Git — commit inicial Sprint 2.*
