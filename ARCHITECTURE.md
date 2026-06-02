# Arquitectura del Sistema

Este documento describe la arquitectura de PCBuilderShop: una aplicación web full-stack construida con Spring Boot que ofrece a la vez interfaz web y API REST, dividida en dos microservicios y desplegada con Docker. El proyecto evolucionó en tres entregas incrementales.

-----

## 1. Visión general

El sistema se compone de tres contenedores orquestados con Docker Compose:

```
   Navegador / Cliente API
            │  HTTPS
            ▼
   ┌────────────────────────────┐      REST (correo / PDF)     ┌────────────────────────────┐
   │  app-service  :8443 (https)│ ───────────────────────────▶ │ utility-service :8080 (http)│
   │  Web (Thymeleaf) + API REST│                              │ Correo (JavaMailSender)     │
   │                            │                              │ PDF (iText)                 │
   └──────────────┬─────────────┘                              │ Sin base de datos           │
                  │ JPA                                         └────────────────────────────┘
                  ▼
        ┌──────────────────┐
        │   MySQL 8  (db)  │
        └──────────────────┘
```

- **app-service** — contiene la aplicación web y toda la lógica de negocio. Sirve tanto las páginas HTML (Thymeleaf) como la API REST. Se publica por HTTPS en el puerto 8443.
- **utility-service** — servicio auxiliar para envío de correos y generación de PDF. No tiene acceso a la base de datos y trabaja solo con los datos recibidos en cada petición. Se publica por HTTP en el puerto 8080.
- **db** — base de datos MySQL 8.

-----

## 2. Arquitectura en capas (app-service)

```
@Controller (web)   @RestController (API REST, /api/v1)
        │                     │
        └──────────┬──────────┘
                   ▼
              @Service          ← lógica de negocio compartida
                   │
                   ▼
            @Repository         ← Spring Data JPA
                   │
                   ▼
               MySQL 8
```

Principios aplicados:

- **Lógica de negocio en los `@Service`**, reutilizada por el controlador web y el controlador REST. Así una misma operación (por ejemplo, crear un pedido) tiene una única implementación.
- **Los controladores nunca acceden directamente a los repositorios**: siempre pasan por un servicio.
- **DTOs en la capa REST**: los `@RestController` no exponen las entidades del dominio; usan objetos de transferencia para controlar la entrada y la salida.
- **Consultas en la base de datos**: el filtrado y la búsqueda se resuelven con consultas de repositorio y paginación, no filtrando listas en memoria.

-----

## 3. Modelo de datos

Cuatro entidades principales, todas relacionadas:

|Relación         |Tipo|Descripción                      |
|-----------------|----|---------------------------------|
|Usuario → Pedido |1:N |Un usuario tiene varios pedidos  |
|Pedido ↔ Producto|N:M |Tabla intermedia `order_products`|
|Usuario → Reseña |1:N |Un usuario escribe varias reseñas|
|Producto → Reseña|1:N |Un producto recibe varias reseñas|

Además, el usuario tiene direcciones asociadas (1:N) y los productos una galería de imágenes. Las **imágenes se almacenan en la base de datos** (no en el sistema de ficheros) para facilitar el despliegue en entornos restringidos.

-----

## 4. Seguridad

- **Spring Security** gestiona la autenticación y la autorización.
- **Web**: autenticación basada en sesión; las rutas se protegen según el rol (anónimo, registrado, administrador).
- **API REST**: autenticación mediante **JWT**, replicando los mismos roles que la web.
- **Control de propiedad**: un usuario solo puede editar o borrar los recursos de los que es dueño; el intento de acceder a recursos ajenos devuelve un error de acceso.
- La protección CSRF se mantiene en la web y se desactiva en la API REST (que usa tokens).
- La aplicación se sirve por **HTTPS**.

-----

## 5. API REST

Diseñada siguiendo buenas prácticas REST:

- Todas las rutas bajo `/api/v1/`.
- Recursos identificados en inglés y en plural.
- Uso adecuado de `GET`, `POST`, `PUT`, `DELETE` y de los códigos de estado.
- Las creaciones devuelven la cabecera `Location` con la URL del recurso creado.
- Filtros y búsquedas como parámetros de query; sin verbos en las URLs.
- Listados paginados (`?page=`).
- **Documentación OpenAPI** generada con SpringDoc, disponible como YAML y HTML, y una colección **Postman** para probar todos los endpoints.

-----

## 6. Microservicios y comunicación

`app-service` delega en `utility-service` cuando necesita enviar un correo o generar un PDF (por ejemplo, la factura tras una compra). La comunicación es por **API REST**.

El diseño clave es que **utility-service es sin estado y sin base de datos**: cada petición incluye toda la información necesaria (destinatario, asunto y cuerpo del correo; o los datos completos del PDF). Esto desacopla por completo ambos servicios y permite escalarlos y mantenerlos por separado.

-----

## 7. Despliegue

- **Imágenes Docker** independientes para `app-service` y `utility-service`, construidas sin asumir herramientas instaladas en la máquina (Maven/JDK van dentro del proceso de build).
- **Docker Compose** levanta los tres contenedores; `app-service` y `utility-service` esperan mediante *healthcheck* a que MySQL esté disponible.
- La configuración de la base de datos se pasa por **variables de entorno** (`SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD`).
- El `docker-compose.yml` se publica como **artefacto OCI** en DockerHub, lo que permite desplegar el sistema con un único comando.
- Despliegue verificado en **máquina virtual remota** por SSH.

-----

## 8. Evolución del proyecto

|Entrega|Contenido                                                                                                                                                                                                   |
|-------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|**1**  |Maquetación de las páginas con HTML y CSS; esquema de navegación.                                                                                                                                           |
|**2**  |Aplicación dinámica: Spring Boot + MySQL + Thymeleaf, Spring Security, modelos/repositorios/controladores, datos de ejemplo, funcionalidades completas (carrito, pago, reseñas, recomendación, PDF, correo).|
|**3**  |API REST con DTOs y OpenAPI, separación en microservicios (app-service + utility-service), JWT, contenedores Docker y despliegue (Compose, OCI, máquina virtual).                                           |