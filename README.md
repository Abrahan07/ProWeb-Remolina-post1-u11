# Refactorización con SOLID, DAO/DTO y @ControllerAdvice

Aplicación Spring Boot de catálogo de productos refactorizada aplicando principios SOLID (SRP y DIP), patrones DAO y DTO con Factory para conversión de objetos, y manejo centralizado de excepciones mediante `@RestControllerAdvice`.

---

## Prerrequisitos

- Java 17 o superior
- Maven 3.9.x
- IDE con soporte Java (IntelliJ IDEA o VS Code con Extension Pack for Java)

---

## Cómo ejecutar el proyecto

### Compilar el proyecto

```bash
mvn compile
```

### Iniciar la aplicación

```bash
mvn spring-boot:run
```

La aplicación inicia en `http://localhost:8080`. No requiere base de datos externa — usa H2 en memoria automáticamente.

---

## Arquitectura en capas

```
com.empresa.catalogo/
│
├── controller/
│   └── ProductoController          ──► Recibe peticiones HTTP, delega en ProductoService
│       │                               Depende de la INTERFAZ (DIP)
│       ▼
├── service/
│   ├── ProductoService             ──► Interfaz — abstracción del servicio (DIP)
│   └── ProductoServiceImpl         ──► Implementación — lógica de negocio (SRP)
│       │                               Delega persistencia en ProductoRepository
│       ▼
├── repository/
│   └── ProductoRepository          ──► Interfaz DAO — extiende JpaRepository
│       │                               Spring Data genera la implementación automáticamente
│       ▼
├── entity/
│   └── Producto                    ──► Entidad JPA mapeada a la tabla producto
│
├── dto/
│   ├── ProductoRequestDTO          ──► Datos de entrada del cliente hacia la API
│   │                                   Contiene validaciones @NotBlank y @Positive
│   └── ProductoResponseDTO         ──► Datos de salida de la API hacia el cliente
│                                       No expone campos sensibles ni internos
├── factory/
│   └── ProductoFactory             ──► Centraliza la conversión entre entidad y DTOs
│                                       toEntity(RequestDTO) y toResponseDTO(Producto)
└── exception/
    ├── ApiError                    ──► Estructura estándar de respuesta de error
    ├── RecursoNoEncontradoException──► Excepción de negocio para recursos no encontrados
    └── GlobalExceptionHandler      ──► @RestControllerAdvice — captura y estandariza errores

Relaciones:
ProductoController  ──usa──►  ProductoService (interfaz)
ProductoServiceImpl ──usa──►  ProductoRepository
ProductoServiceImpl ──usa──►  ProductoFactory
ProductoFactory     ──convierte──►  Producto ◄──► ProductoRequestDTO / ProductoResponseDTO
GlobalExceptionHandler ──intercepta──►  RecursoNoEncontradoException / MethodArgumentNotValidException / Exception
```

---

## Principios SOLID aplicados

### SRP — Single Responsibility Principle
Cada clase tiene una única responsabilidad:
- `ProductoController` solo maneja peticiones HTTP
- `ProductoServiceImpl` solo contiene lógica de negocio
- `ProductoRepository` solo gestiona el acceso a datos
- `ProductoFactory` solo realiza conversiones entre objetos
- `GlobalExceptionHandler` solo maneja excepciones

### DIP — Dependency Inversion Principle
`ProductoController` depende de la interfaz `ProductoService`, no de la implementación concreta `ProductoServiceImpl`. Spring inyecta la implementación en tiempo de ejecución mediante inversión de control.

---

## Endpoints disponibles

| Método | URL | Descripción | Status |
|--------|-----|-------------|--------|
| GET | `/api/productos` | Lista todos los productos activos | 200 |
| GET | `/api/productos/{id}` | Busca un producto por id | 200 / 404 |
| POST | `/api/productos` | Crea un nuevo producto | 201 / 400 |
| DELETE | `/api/productos/{id}` | Elimina un producto por id | 204 / 404 |

---

## Manejo de errores — GlobalExceptionHandler

Todos los errores retornan un JSON estandarizado con la estructura `ApiError`:

```json
{
  "status": 404,
  "error": "Not Found",
  "mensaje": "Producto con id 999 no encontrado.",
  "timestamp": "2026-05-12 00:40:22",
  "path": "/api/productos/999"
}
```

| Excepción | Status | Descripción |
|-----------|--------|-------------|
| `RecursoNoEncontradoException` | 404 | Producto no existe en la base de datos |
| `MethodArgumentNotValidException` | 400 | Campos inválidos en el request body |
| `Exception` | 500 | Error inesperado del servidor |

---

## Descripción de las clases principales

### `ProductoRequestDTO`
DTO de entrada que recibe los datos del cliente. Aplica validaciones con Bean Validation:
- `@NotBlank` en `nombre` — el nombre no puede estar vacío
- `@Positive` en `precio` — el precio debe ser mayor a cero

### `ProductoResponseDTO`
DTO de salida que expone solo los campos necesarios al cliente: `id`, `nombre`, `precio` y `categoria`. No expone el campo `activo` ni metadatos internos.

### `ProductoFactory`
Componente Spring (`@Component`) que centraliza la conversión:
- `toEntity(ProductoRequestDTO)` — convierte el DTO de entrada en entidad JPA
- `toResponseDTO(Producto)` — convierte la entidad en DTO de salida

### `ProductoRepository`
Interfaz DAO que extiende `JpaRepository<Producto, Long>`. Agrega el método `findByActivoTrue()` para listar solo productos activos.

### `GlobalExceptionHandler`
Anotado con `@RestControllerAdvice`, intercepta excepciones de toda la aplicación y retorna respuestas JSON estandarizadas con `ApiError`.

---

## Evidencias

### Checkpoint 1 — Compilación exitosa
![mvn compile](capturas/mvn-compile.png)

### Checkpoint 2 — POST exitoso retornando DTO de respuesta
![POST exitoso](capturas/post-exitoso.png)

### Checkpoint 3 — Manejo de errores 404 y 400
![Error 404](capturas/error-404.png)
![Error 400](capturas/error-400.png)
