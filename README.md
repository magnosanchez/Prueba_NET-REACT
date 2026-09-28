# 🧪 Prueba — Pedidos (.NET 8 + React)

Aplicación **end-to-end** de gestión de Pedidos con autenticación JWT, construida con
**.NET 8 (Clean Architecture)** en el backend y **React + TypeScript + Tailwind CSS** en el frontend.

\---

## Estructura del repositorio


Prueba\_NET\_React/
├── Pedidos.API/
│   |
│   ├── Pedidos.API/
│   │   ├── Pedidos.Domain/          # Entidades y reglas de negocio puras
│   │   ├── Pedidos.Application/     # Casos de uso, DTOs, interfaces
│   │   ├── Pedidos.Infrastructure/  # EF Core, JWT, BCrypt, repositorios
│   │   └── Pedidos.API/             # Controllers, middleware, Program.cs

|   |   |\_\_ Pedidos.Tests 	     # Pruebas unitarias (xUnit + Moq)	

|   |	|\_\_ Pedidos.sln


│   └── database/
│       └── schema.sql                      # Script SQL referencial
├── frontend/                               # SPA en React + Vite + TypeScript + Tailwind
└── postman\_collection.json                 # Colección de Postman lista para importar
```

\---

## Arquitectura y decisiones técnicas

### Backend — Clean Architecture

* **Domain**: entidades `Pedido` y `Usuario` con sus propias invariantes (el total no puede
ser ≤ 0, el estado debe ser uno de los permitidos, etc.), sin dependencias externas.
* **Application**: casos de uso (`PedidoService`, `AuthService`) e interfaces (patrón
Repository + Dependency Inversion). No conoce EF Core ni ASP.NET.
* **Infrastructure**: implementación concreta con **EF Core + SQL Server**, generación de
**JWT**, hashing de contraseñas con **BCrypt**.
* **API**: controllers delgados, middleware de manejo global de excepciones, Swagger,
autenticación JWT Bearer, CORS y rate limiting.

### Seguridad

* Autenticación vía `POST /auth/login` → JWT Bearer con expiración configurable (60 min por defecto).
* Endpoints de Pedidos protegidos con `\[Authorize]`.
* Contraseñas hasheadas con **BCrypt** (nunca en texto plano ni con hash reversible).
* Mensajes de error de login genéricos (no se revela si el email existe, previene *user enumeration*).
* **Rate limiting** por IP (`AspNetCoreRateLimit`) como patrón de resiliencia: máx. 10 intentos/min
en `/auth/login` y 10 req/seg en el resto de endpoints.
* CORS restringido a los orígenes del frontend.

### Reglas de negocio implementadas

* No se pueden crear/editar pedidos con `total <= 0` (validado en la entidad de dominio, no
solo en el DTO — así la regla se cumple sin importar quién invoque el constructor).
* El `numeroPedido` debe ser único (validado en el `PedidoService` antes de persistir).
* Eliminación **lógica** (soft delete): los pedidos eliminados no se borran físicamente,
se marcan con `Eliminado = true` y se excluyen automáticamente de las consultas
(vía `HasQueryFilter` de EF Core).
* Solo usuarios autenticados acceden al CRUD de Pedidos.

### Manejo de errores

Middleware global (`ExceptionHandlingMiddleware`) que traduce excepciones de dominio a
respuestas HTTP consistentes:

|Excepción|HTTP Status|
|-|-|
|`DomainException`|400 Bad Request|
|`NotFoundException`|404 Not Found|
|`ConflictException`|409 Conflict|
|`AuthenticationException`|401 Unauthorized|
|Cualquier otra|500 (sin exponer detalles internos en producción)|

### Frontend — React + TypeScript

* **Vite** + **React Router v6** + **Tailwind CSS**.
* `AuthContext` maneja el estado de sesión; el token JWT se guarda en `localStorage`.
* Cliente **Axios** centralizado con interceptores: adjunta el token automáticamente y
redirige a `/login` si el backend responde `401`.
* Rutas protegidas con `<PrivateRoute />`.
* Pantallas: **Login**, **Listado de Pedidos** (con eliminación con confirmación), **Crear
Pedido**, **Editar Pedido**, **Navbar** con navegación y logout.

\---

## Cómo ejecutar el proyecto

### Requisitos previos

* [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0)
* [Node.js 18+](https://nodejs.org/) y npm
* SQL Server (local, Docker o Azure SQL)

### 1\. Base de datos

**Opción A (recomendada):** dejar que la app cree la base de datos y las tablas automáticamente.
La aplicación aplica las migraciones de EF Core y siembra datos de prueba al arrancar
(ver `DbSeeder.cs`), por lo que **no necesitas ejecutar ningún script manualmente**.

Si prefieres levantar SQL Server rápido con Docker:

```bash
docker run -e "ACCEPT\_EULA=Y" -e "SA\_PASSWORD=YourStrong(!)Password" \\
  -p 1433:1433 --name sqlserver -d mcr.microsoft.com/mssql/server:2022-latest
```

**Opción B:** ejecutar manualmente `backend/database/schema.sql` contra tu instancia de SQL Server
(script referencial, útil para revisar el modelo sin correr la app).

### 2\. Backend

```bash
cd backend

# Ajusta la cadena de conexión si es necesario en:
# src/RetoFullstack.API/appsettings.json  ->  ConnectionStrings:DefaultConnection

# Restaurar dependencias y compilar
dotnet restore
dotnet build

# (Opcional) generar la migración inicial si aún no existe:
cd src/RetoFullstack.API
dotnet tool install --global dotnet-ef   # si no lo tienes instalado
dotnet ef migrations add InitialCreate --project ../RetoFullstack.Infrastructure --startup-project .

# Ejecutar la API (aplica migraciones y siembra datos automáticamente)
dotnet run
```

La API queda disponible en `http://localhost:7064` (Swagger en `http://localhost:7064/swagger`).

**Usuarios de prueba sembrados automáticamente:**

|Email|Password|Rol|
|-|-|-|
|admin@pedido.com|Password123!|Admin|
|user@pedido.com|Password123!|User|

### 3\. Frontend

```bash
cd frontend
cp .env.example .env    # ajusta VITE\_API\_URL si tu API corre en otro puerto
npm install
npm run dev
```

La SPA queda disponible en `http://localhost:5173`.

### 4\. Ejecutar pruebas unitarias del backend

```bash
cd backend
dotnet test
```

### 5\. Probar la API con Postman

Importa `postman\_collection.json` en Postman. Ejecuta primero `Auth > Login (Admin)`
(guarda el token automáticamente en una variable de colección) y luego cualquier
endpoint de `Pedidos`.

\---

## Endpoints principales

|Método|Ruta|Descripción|Auth|
|-|-|-|-|
|POST|`/auth/login`|Login, devuelve JWT|No|
|GET|`/api/pedidos`|Lista todos los pedidos|Sí|
|GET|`/api/pedidos/{id}`|Obtiene un pedido por Id|Sí|
|POST|`/api/pedidos`|Crea un pedido|Sí|
|PUT|`/api/pedidos/{id}`|Actualiza un pedido|Sí|
|DELETE|`/api/pedidos/{id}`|Elimina (lógicamente) un pedido|Sí|

\---

## Extras implementados más allá de lo mínimo requerido

* Eliminación lógica (soft delete) en vez de borrado físico.
* Roles (`Admin` / `User`) incluidos como claim en el JWT.
* Rate limiting como patrón de resiliencia adicional.
* Manejo global de excepciones con respuestas HTTP consistentes.
* Logging estructurado con `ILogger` en servicios de aplicación.
* Proyecto de pruebas unitarias (xUnit + Moq + FluentAssertions) cubriendo reglas de negocio.
* Swagger con soporte de autenticación Bearer integrado.
* UX: confirmación antes de eliminar, badges de estado, resumen de totales, mensajes de error
claros, estado de carga en formularios.

## Posibles mejoras futuras (fuera de alcance del reto)

* Paginación y filtros en el listado de pedidos.
* Refresh tokens para renovar la sesión sin re-login.
* Tests de integración de la API con `WebApplicationFactory`.
* Dockerfile / docker-compose para levantar todo el stack con un solo comando.

