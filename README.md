# PCBuilderShop — Tienda Web de Componentes de PC

> Aplicación web full-stack para la venta de componentes de PC (CPU, GPU, RAM, placas base, SSD…), desarrollada con **Spring Boot, MySQL y Thymeleaf**, con **API REST documentada**, arquitectura de **microservicios** y despliegue completo con **Docker**.

<p align="left">
  <img src="https://img.shields.io/badge/Java-21-orange.svg" alt="Java">
  <img src="https://img.shields.io/badge/Spring%20Boot-3.x-green.svg" alt="Spring Boot">
  <img src="https://img.shields.io/badge/MySQL-8-blue.svg" alt="MySQL">
  <img src="https://img.shields.io/badge/Thymeleaf-005F0F.svg" alt="Thymeleaf">
  <img src="https://img.shields.io/badge/Spring%20Security-JWT-brightgreen.svg" alt="Security">
  <img src="https://img.shields.io/badge/Docker-compose-2496ED.svg" alt="Docker">
  <img src="https://img.shields.io/badge/OpenAPI-SpringDoc-85EA2D.svg" alt="OpenAPI">
  <img src="https://img.shields.io/badge/status-academic%20project-lightgrey.svg" alt="Status">
</p>

-----

## Descripción general

PCBuilderShop es una tienda online especializada en componentes de hardware. Ofrece un catálogo navegable con búsqueda y filtros, carrito de compra, proceso de pago con generación de factura, sistema de reseñas con valoraciones, recomendación de productos y un panel de administración completo con estadísticas.

La aplicación expone **simultáneamente una interfaz web (HTML generado en servidor con Thymeleaf) y una API REST**, de modo que la misma funcionalidad es accesible tanto desde el navegador como desde clientes externos. El sistema está dividido en **dos microservicios** (aplicación principal y servicio de utilidades) que se comunican por REST, y se despliega de forma reproducible mediante Docker Compose.

> **Contexto:** proyecto académico desarrollado **en equipo de 4 personas** para la asignatura Sistemas Distribuidos (URJC), en tres entregas incrementales: maquetación → web dinámica en servidor → API REST, microservicios y despliegue.

-----

## Características principales

- **Catálogo de productos** con búsqueda por términos, filtros (marca, rango de precio) y ordenación.
- **Tres roles de usuario** (anónimo, registrado, administrador) con permisos diferenciados y control de propiedad sobre los datos.
- **Carrito de compra y proceso de pago**, con resumen del pedido y confirmación.
- **Sistema de reseñas** con valoración por estrellas, pros/contras e imágenes.
- **Panel de administración** con CRUD completo de productos, usuarios, pedidos y reseñas, y dashboard con estadísticas y gráficas.
- **Generación de facturas en PDF** (iText) y **envío automático de correos** (JavaMailSender).
- **Algoritmo de recomendación** basado en el historial de compras (filtrado colaborativo).
- **API REST documentada con OpenAPI** (SpringDoc), con DTOs, paginación y seguridad por roles.
- **Autenticación con Spring Security y JWT**; aplicación servida por HTTPS.
- **Arquitectura de microservicios** y **despliegue con Docker Compose** (app, servicio de utilidades y base de datos).

-----

## Arquitectura del sistema

La aplicación se compone de dos microservicios y una base de datos, orquestados con Docker Compose:

```
                   ┌──────────────────────────────────────────────┐
   Navegador  ────▶│  app-service  (HTTPS :8443)                   │
   / Cliente       │                                              │
   API REST        │  Capa web (Thymeleaf) + API REST (/api/v1)   │
                   │  @Controller / @RestController                │
                   │         │                                     │
                   │         ▼                                     │
                   │  @Service (lógica de negocio compartida)      │──── JPA ───▶ ┌─────────────┐
                   │         │                                     │              │  MySQL 8    │
                   │         ▼                                     │              │  (db)       │
                   │  @Repository (Spring Data JPA)                │◀─────────────└─────────────┘
                   └───────────────┬──────────────────────────────┘
                                   │  REST (correo / PDF)
                                   ▼
                   ┌──────────────────────────────────────────────┐
                   │  utility-service  (HTTP :8080)                │
                   │  Envío de correos + generación de PDF         │
                   │  Sin acceso a base de datos                   │
                   └──────────────────────────────────────────────┘
```

El servicio `utility-service` está deliberadamente **desacoplado y sin conexión a la base de datos**: recibe en cada petición todos los datos que necesita para enviar un correo o generar un PDF. Esto separa responsabilidades y permite escalar y mantener cada servicio de forma independiente.

Para una descripción detallada de las capas, entidades y flujo de despliegue, consulta [`ARCHITECTURE.md`](ARCHITECTURE.md).

-----

## Tecnologías utilizadas

|Categoría    |Tecnología                                          |
|-------------|----------------------------------------------------|
|Lenguaje     |Java 21                                             |
|Framework    |Spring Boot (MVC, Data JPA, Security)               |
|Vistas       |Thymeleaf, HTML, CSS, JavaScript                    |
|Base de datos|MySQL 8 (JPA / Hibernate)                           |
|Seguridad    |Spring Security · JWT · HTTPS                       |
|API REST     |Controladores REST con DTOs · OpenAPI / SpringDoc   |
|Utilidades   |iText (PDF) · JavaMailSender (correo)               |
|Construcción |Maven                                               |
|Contenedores |Docker · Docker Compose · artefactos OCI (DockerHub)|

-----

## Entidades y modelo de datos

La aplicación gestiona cuatro entidades principales, todas relacionadas entre sí:

- **Usuario** — Pedido: un usuario registrado puede generar varios pedidos (1:N).
- **Pedido** — Producto: un pedido contiene varios productos y un producto aparece en varios pedidos (N:M, mediante tabla intermedia).
- **Usuario** — Reseña: un usuario puede escribir varias reseñas (1:N).
- **Producto** — Reseña: un producto puede tener varias reseñas (1:N).

![Diagrama entidad-relación de la base de datos](assets/images/DiagramaEntidades_PcBuilderShop.png)

*Modelo relacional con las entidades User, Product, Order y Review, las direcciones de usuario y la tabla intermedia que resuelve la relación N:M entre pedidos y productos.*

-----

## Roles y permisos

|Rol              |Permisos                                                                                                            |
|-----------------|--------------------------------------------------------------------------------------------------------------------|
|**Anónimo**      |Explorar el catálogo, buscar productos, registrarse e iniciar sesión. No es dueño de ninguna entidad.               |
|**Registrado**   |Gestionar su perfil, sus pedidos y sus reseñas. Es dueño de sus propios datos (solo él puede editarlos o borrarlos).|
|**Administrador**|Control total: CRUD de productos, gestión de usuarios, pedidos y reseñas, y acceso al dashboard con estadísticas.   |

El control de propiedad está implementado a nivel de seguridad: un usuario solo puede modificar o borrar los elementos de los que es dueño, tanto en la web como en la API REST.

-----

## Capturas de la aplicación

### Tienda (vista de usuario)

![Página principal](assets/images/images-update/index-update.png)

*Home con productos recomendados, novegades de hardware y acceso a categorías.*

![Página de búsqueda con filtros](assets/images/images-update/paginabusqueda-update.png)

*Resultados en cuadrícula con filtros por marca y precio y ordenación.*

![Página de producto](assets/images/images-update/productpage-update.png)

*Ficha de producto con carrusel de imágenes, especificaciones técnicas y reseñas.*

![Carrito de compra](assets/images/images-update/carrito-update.png)

*Carrito con cantidades, precios y productos recomendados.*

### Panel de administración

![Dashboard de administración](assets/images/readme-images/admin-dashboard.png)

*Resumen del sistema con estadísticas, gráficas de ventas e inventario y pedidos recientes.*

![Gestión de productos](assets/images/readme-images/admin-item-list.png)

*Listado de productos con búsqueda, filtros y acciones de creación, edición y borrado.*

-----

## API REST

La API REST de `app-service` está documentada con OpenAPI generado por SpringDoc y sigue buenas prácticas: rutas bajo `/api/v1/`, recursos en plural, uso correcto de verbos HTTP y códigos de estado, header `Location` en las creaciones, paginación por parámetros y DTOs para no exponer las entidades.

- [Especificación OpenAPI (YAML)](api-docs/apidocs.yaml)
- [Documentación HTML de la API](https://raw.githack.com/joel-domene/ProyectoSSDD/main/api-docs/api-docs.html)
- Colección de **Postman** incluida en [`postman/`](postman/) para probar todos los endpoints.

Durante la ejecución local también está disponible en:

```
https://localhost:8443/swagger-ui/index.html
```

La API implementa los mismos roles que la web (mediante JWT), de modo que una SPA o una app móvil podrían realizar cualquier operación disponible en la interfaz web.

-----

## Instalación y ejecución

### Requisitos

- Java 21+, Maven 3.8+, MySQL 8+ (para ejecución local), o bien Docker 20.10+ y Docker Compose 2.0+ (para ejecución en contenedores).

### Opción A — Local (solo app-service)

```bash
git clone https://github.com/joel-domene/PCBuilderShop-web.git
cd PCBuilderShop-web

# Crear la base de datos en MySQL
# CREATE DATABASE pc_builder_db;
# y configurar credenciales en app-service/src/main/resources/application.properties

cd app-service
mvn clean install
mvn spring-boot:run
```

La aplicación quedará disponible en `https://localhost:8443`.

### Opción B — Docker Compose (sistema completo)

```bash
git clone https://github.com/joel-domene/PCBuilderShop-web.git
cd PCBuilderShop-web

docker compose up -d --build
docker ps   # deberían aparecer: db, app-service, utility-service
```

- Web: `https://localhost:8443`
- utility-service: `http://localhost:8080`

`app-service` y `utility-service` esperan, mediante *healthcheck*, a que MySQL esté disponible antes de arrancar.

### Opción C — Despliegue desde artefacto OCI

La configuración de despliegue se publica en DockerHub como artefacto OCI, de modo que la aplicación se levanta sin necesidad de tener el `docker-compose.yml` en local:

```bash
docker compose -f oci://docker.io/<usuario-dockerhub>/pcbuildershop-compose up
```

-----

## Despliegue

- **Imágenes Docker** independientes para `app-service` y `utility-service`, publicadas en DockerHub.
- **Docker Compose** multicontenedor (base de datos + ambos servicios) con *healthchecks* y variables de entorno para la configuración de la base de datos.
- **Artefacto OCI** del `docker-compose.yml` publicado en DockerHub para un despliegue de un solo comando.
- Despliegue probado en **máquina virtual remota por SSH**.

-----

## Decisiones técnicas relevantes

- **Lógica de negocio en `@Service` compartido** entre el `@Controller` web y el `@RestController`, evitando duplicación y manteniendo los controladores finos. Los controladores nunca acceden directamente a los repositorios.
- **DTOs en la API REST** para no exponer las entidades del dominio y controlar exactamente qué datos entran y salen.
- **Microservicio de utilidades sin estado** (`utility-service`): al no tener base de datos y recibir todos los datos por petición, queda completamente desacoplado del servicio principal.
- **Imágenes almacenadas en la base de datos** en lugar del sistema de ficheros, para facilitar el despliegue en entornos restringidos.
- **Seguridad por roles homogénea** en web (sesión) y API (JWT), reutilizando la misma configuración de autorización.

-----

## Funcionalidades destacadas

- **Búsqueda y filtrado** resueltos en la capa de repositorio (consultas a base de datos), no filtrando en memoria.
- **Algoritmo de recomendación** por filtrado colaborativo a partir del historial de compras.
- **Facturas en PDF** generadas bajo demanda y enviadas por correo automáticamente tras la compra, delegando en el `utility-service`.
- **Dashboard de administración** con gráficas de ventas, stock y distribución de pedidos.

-----

## Estructura del repositorio

```
.
├── app-service/          # Microservicio principal: web (Thymeleaf) + API REST
│   └── src/main/...       # Modelos, repositorios, servicios, controladores, plantillas
├── utility-service/      # Microservicio de utilidades: correo y PDF (sin BD)
├── api-docs/             # Especificación OpenAPI (YAML) y documentación HTML
├── postman/              # Colección Postman de la API REST
├── assets/               # Imágenes: capturas y diagramas
├── docker-compose.yml    # Orquestación multicontenedor
├── README.md
├── ARCHITECTURE.md
└── LICENSE              
```

-----

## Equipo

Proyecto desarrollado en equipo de 4 personas.

-----

## Licencia

Distribuido bajo licencia **Apache 2.0**. Consulta [`LICENSE`](LICENSE) para más información.
