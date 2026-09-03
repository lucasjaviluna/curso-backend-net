# Backend .NET --- Módulo 2

# .NET + ASP.NET Core

## 1. Objetivos

Al terminar este módulo deberías poder:

-   Entender qué es ASP.NET Core.
-   Entender cómo se estructura una Web API.
-   Crear una Web API con `dotnet`.
-   Entender el papel de `Program.cs`.
-   Entender HTTP desde la perspectiva de ASP.NET Core.
-   Crear Controllers.
-   Definir endpoints y rutas.
-   Recibir route parameters, query parameters y request bodies.
-   Devolver respuestas HTTP y utilizar status codes.
-   Entender model binding.
-   Entender el flujo de una request.
-   Integrar el dominio del módulo 1 con una API HTTP.

------------------------------------------------------------------------

## 2. Evolución del proyecto

En el módulo 1 teníamos:

``` text
Console
   ↓
TaskManager
   ↓
List<TaskItem>
```

Ahora construiremos:

``` text
HTTP Request
      ↓
Controller
      ↓
TaskManager
      ↓
List<TaskItem>
```

Todavía no utilizaremos:

-   Entity Framework Core;
-   SQL;
-   Repository;
-   CQRS;
-   Mediator;
-   Authentication.

------------------------------------------------------------------------

## 3. ¿Qué es ASP.NET Core?

ASP.NET Core es el framework de .NET para construir aplicaciones web.

Podemos construir:

``` text
Web APIs
Web applications
Minimal APIs
MVC applications
Real-time applications
```

En este curso utilizaremos:

``` text
ASP.NET Core Web API
```

Nuestro objetivo es construir una API HTTP que pueda ser consumida por
Angular, React u otros clientes.

------------------------------------------------------------------------

## 4. HTTP

Una request HTTP contiene conceptualmente:

``` text
HTTP Method
URL
Headers
Body
```

Por ejemplo:

``` http
GET /api/tasks/123
Authorization: Bearer ...
```

Una response contiene:

``` text
Status Code
Headers
Body
```

Por ejemplo:

``` http
200 OK
Content-Type: application/json

{
  "id": "123",
  "title": "Learn C#"
}
```

------------------------------------------------------------------------

## 5. HTTP Methods

Los métodos que utilizaremos principalmente son:

  Método   Uso
  -------- -----------------------
  GET      Obtener
  POST     Crear
  PUT      Actualizar/reemplazar
  PATCH    Actualización parcial
  DELETE   Eliminar

Nuestro CRUD tendrá inicialmente:

``` text
GET    /api/tasks
GET    /api/tasks/{id}
POST   /api/tasks
PUT    /api/tasks/{id}
DELETE /api/tasks/{id}
```

------------------------------------------------------------------------

## 6. Status Codes

Los códigos más importantes:

``` text
200 OK
201 Created
204 No Content
400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
500 Internal Server Error
```

Modelo mental:

``` text
2xx → éxito
4xx → problema relacionado con la request
5xx → error del servidor
```

------------------------------------------------------------------------

## 7. Crear una Web API

Crear el proyecto:

``` bash
dotnet new webapi -n TaskManagement.Api
cd TaskManagement.Api
dotnet run
```

La aplicación iniciará un servidor HTTP en una URL similar a:

``` text
https://localhost:7000
```

El puerto puede variar según la configuración local.

------------------------------------------------------------------------

## 8. Estructura inicial

Una aplicación puede comenzar con:

``` text
TaskManagement.Api/
│
├── Program.cs
├── TaskManagement.Api.csproj
├── appsettings.json
└── Properties/
    └── launchSettings.json
```

Lo más importante por ahora es:

``` text
Program.cs
```

Es el punto de entrada y configuración principal de nuestra aplicación.

------------------------------------------------------------------------

## 9. Program.cs

Una aplicación moderna puede comenzar con:

``` csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

var app = builder.Build();

app.MapControllers();

app.Run();
```

Aunque parece poco código, configura varias piezas importantes.

------------------------------------------------------------------------

## 10. WebApplication.CreateBuilder

``` csharp
var builder = WebApplication.CreateBuilder(args);
```

Prepara la aplicación y sus servicios fundamentales, incluyendo:

-   configuración;
-   logging;
-   dependency injection;
-   environment;
-   configuración web.

No necesitamos estudiar todavía cada una de estas piezas.

------------------------------------------------------------------------

## 11. Services

Tenemos:

``` csharp
builder.Services.AddControllers();
```

`builder.Services` representa la colección de servicios de Dependency
Injection.

Todavía no vamos a profundizar en DI.

Lo haremos en el módulo 4.

------------------------------------------------------------------------

## 12. Build

``` csharp
var app = builder.Build();
```

Construye la aplicación utilizando la configuración anterior.

------------------------------------------------------------------------

## 13. MapControllers

``` csharp
app.MapControllers();
```

Indica que ASP.NET Core debe utilizar los Controllers como endpoints
HTTP.

Conceptualmente:

``` text
HTTP Request
      ↓
Routing
      ↓
Controller
```

------------------------------------------------------------------------

## 14. Run

``` csharp
app.Run();
```

Inicia el servidor y deja la aplicación esperando requests.

``` text
HTTP
 ↓
ASP.NET Core
 ↓
Application
```

------------------------------------------------------------------------

# 15. Controllers

Un Controller es una clase que expone endpoints HTTP.

Ejemplo:

``` csharp
using Microsoft.AspNetCore.Mvc;

namespace TaskManagement.Api.Controllers;

[ApiController]
[Route("api/[controller]")]
public class TasksController : ControllerBase
{
}
```

La ruta será:

``` text
/api/tasks
```

------------------------------------------------------------------------

## 16. ControllerBase

Nuestro controller hereda de:

``` csharp
ControllerBase
```

Esto proporciona funcionalidades para construir APIs HTTP:

``` csharp
Ok()
NotFound()
BadRequest()
CreatedAtAction()
NoContent()
```

Para una Web API no necesitamos heredar de `Controller`, porque no
estamos trabajando con vistas HTML.

------------------------------------------------------------------------

## 17. \[ApiController\]

La anotación:

``` csharp
[ApiController]
```

activa comportamientos específicos de las APIs de ASP.NET Core,
incluyendo soporte para:

-   model binding;
-   inferencia de parámetros;
-   validación;
-   respuestas automáticas para ciertos errores.

Volveremos sobre esto en el módulo 9.

------------------------------------------------------------------------

## 18. Routing

Tenemos:

``` csharp
[Route("api/[controller]")]
```

Para:

``` text
TasksController
```

`[controller]` se resuelve conceptualmente como:

``` text
tasks
```

Por lo tanto:

``` text
/api/[controller]
```

se convierte en:

``` text
/api/tasks
```

------------------------------------------------------------------------

# 19. Primer endpoint GET

Podemos crear:

``` csharp
[HttpGet]
public IActionResult GetAll()
{
    return Ok();
}
```

Entonces:

``` http
GET /api/tasks
```

ejecuta:

``` csharp
GetAll()
```

Y:

``` csharp
return Ok();
```

produce:

``` http
200 OK
```

------------------------------------------------------------------------

## 20. Devolver datos

Podemos devolver una colección:

``` csharp
[HttpGet]
public IActionResult GetAll()
{
    var tasks = ...;

    return Ok(tasks);
}
```

ASP.NET Core serializará los objetos a JSON.

Por ejemplo:

``` json
{
  "id": "8a...",
  "title": "Learn C#",
  "description": "Study C# fundamentals",
  "status": "Pending",
  "createdAt": "2026-09-02T..."
}
```

------------------------------------------------------------------------

## 21. IActionResult

Podemos declarar:

``` csharp
public IActionResult GetAll()
```

Esto permite devolver diferentes respuestas HTTP:

``` csharp
return Ok(tasks);
```

``` csharp
return NotFound();
```

``` csharp
return BadRequest();
```

Es útil cuando el endpoint puede tener distintos resultados.

------------------------------------------------------------------------

## 22. ActionResult`<T>`{=html}

También podemos utilizar:

``` csharp
public ActionResult<List<TaskItem>> GetAll()
```

Esto expresa que normalmente esperamos devolver:

``` text
List<TaskItem>
```

pero también podemos devolver:

``` text
404
400
etc.
```

No necesitamos obsesionarnos con la diferencia todavía. Utilizaremos
ambos conceptos cuando tengan sentido.

------------------------------------------------------------------------

# 23. Route Parameters

Para obtener una tarea:

``` http
GET /api/tasks/{id}
```

podemos utilizar:

``` csharp
[HttpGet("{id:guid}")]
public IActionResult GetById(Guid id)
{
    ...
}
```

Si llega:

``` http
GET /api/tasks/550e8400-e29b-41d4-a716-446655440000
```

ASP.NET Core coloca ese valor en:

``` csharp
Guid id
```

------------------------------------------------------------------------

## 24. Route Constraints

Tenemos:

``` csharp
"{id:guid}"
```

Esto indica que la ruta espera un GUID.

Otros constraints comunes:

``` text
int
long
bool
datetime
min
max
length
```

Por ejemplo:

``` csharp
[HttpGet("{id:int}")]
```

esperaría un entero.

------------------------------------------------------------------------

# 25. Query Parameters

Los query parameters aparecen después de `?`.

Ejemplo:

``` http
GET /api/tasks?status=Pending
```

Podemos recibirlo:

``` csharp
[HttpGet]
public IActionResult GetAll(string? status)
{
    ...
}
```

ASP.NET Core realiza el model binding.

Más adelante utilizaremos query parameters para:

``` text
pagination
filtering
sorting
search
```

------------------------------------------------------------------------

## 26. Route vs Query

Es importante distinguir:

``` http
GET /api/tasks/123
```

de:

``` http
GET /api/tasks?status=Pending
```

El primero identifica un recurso:

``` text
task id = 123
```

El segundo aplica un criterio sobre una colección:

``` text
tasks filtered by status
```

Modelo mental:

``` text
/tasks/{id}
       ↑
     recurso

/tasks?status=Pending
             ↑
          criterio
```

------------------------------------------------------------------------

# 27. Request Body

Para crear una tarea:

``` http
POST /api/tasks
Content-Type: application/json
```

Body:

``` json
{
  "title": "Learn ASP.NET Core",
  "description": "Build first API"
}
```

ASP.NET Core puede convertir ese JSON en un objeto C#.

------------------------------------------------------------------------

## 28. Request Model

Por ahora utilizaremos:

``` csharp
public class CreateTaskRequest
{
    public string Title { get; set; } = string.Empty;

    public string? Description { get; set; }
}
```

Y:

``` csharp
[HttpPost]
public IActionResult Create(CreateTaskRequest request)
{
    ...
}
```

------------------------------------------------------------------------

# 29. Model Binding

Model binding convierte los datos de una HTTP request en valores u
objetos C#.

Por ejemplo:

``` http
GET /api/tasks/123
```

puede producir:

``` csharp
Guid id
```

Mientras:

``` json
{
  "title": "Learn C#"
}
```

puede producir:

``` csharp
CreateTaskRequest request
```

Conceptualmente:

``` text
HTTP
 ↓
Model Binding
 ↓
C# values / objects
```

------------------------------------------------------------------------

# 30. POST

Podemos crear:

``` csharp
[HttpPost]
public IActionResult Create(CreateTaskRequest request)
{
    var task = new TaskItem(
        request.Title,
        request.Description);

    return Ok(task);
}
```

Todavía no utilizamos Service, Repository ni Database.

Eso es intencional.

Primero estamos aprendiendo el flujo HTTP.

------------------------------------------------------------------------

# 31. CreatedAtAction

Para creación de recursos normalmente queremos:

``` http
201 Created
```

Podemos utilizar:

``` csharp
return CreatedAtAction(
    nameof(GetById),
    new { id = task.Id },
    task);
```

Esto devuelve:

``` text
201 Created
```

y permite indicar el endpoint donde puede obtenerse el recurso creado.

------------------------------------------------------------------------

# 32. DELETE

Podemos crear:

``` csharp
[HttpDelete("{id:guid}")]
public IActionResult Delete(Guid id)
{
    ...
}
```

Si eliminamos correctamente:

``` csharp
return NoContent();
```

produce:

``` http
204 No Content
```

Si no existe:

``` csharp
return NotFound();
```

produce:

``` http
404 Not Found
```

------------------------------------------------------------------------

# 33. PUT

Para actualizar:

``` csharp
[HttpPut("{id:guid}")]
public IActionResult Update(
    Guid id,
    UpdateTaskRequest request)
{
    ...
}
```

Request:

``` http
PUT /api/tasks/123
```

Body:

``` json
{
  "title": "Learn ASP.NET Core",
  "description": "Updated description"
}
```

------------------------------------------------------------------------

# 34. PATCH

También existe:

``` http
PATCH
```

que suele utilizarse para actualizaciones parciales.

Ejemplo:

``` http
PATCH /api/tasks/123
```

``` json
{
  "title": "New title"
}
```

No lo utilizaremos inicialmente.

Primero consolidaremos:

``` text
GET
POST
PUT
DELETE
```

------------------------------------------------------------------------

# 35. El flujo completo

Ahora podemos visualizar:

``` text
Client
  │
  │ HTTP
  ↓
ASP.NET Core
  │
  ↓
Routing
  │
  ↓
Controller
  │
  ↓
TaskManager
  │
  ↓
Memory
```

Ejemplo:

``` text
GET /api/tasks
      ↓
TasksController.GetAll()
      ↓
TaskManager.GetAll()
      ↓
List<TaskItem>
      ↓
JSON
```

Este flujo será fundamental durante todo el curso.

------------------------------------------------------------------------

# 36. Integrar el módulo 1

La estructura comienza a ser:

``` text
TaskManagement.Api/
│
├── Domain/
│   ├── TaskItem.cs
│   └── TaskStatus.cs
│
├── Services/
│   └── TaskManager.cs
│
├── Controllers/
│   └── TasksController.cs
│
└── Program.cs
```

No estamos aplicando todavía una arquitectura sofisticada.

------------------------------------------------------------------------

# 37. TaskManager

Podemos reutilizar el manager del módulo anterior:

``` csharp
using TaskManagement.Api.Domain;

namespace TaskManagement.Api.Services;

public class TaskManager
{
    private readonly List<TaskItem> _tasks = [];

    public TaskItem Create(
        string title,
        string? description)
    {
        var task = new TaskItem(
            title,
            description);

        _tasks.Add(task);

        return task;
    }

    public List<TaskItem> GetAll()
    {
        return _tasks.ToList();
    }

    public TaskItem? GetById(Guid id)
    {
        return _tasks
            .FirstOrDefault(task => task.Id == id);
    }

    public bool Delete(Guid id)
    {
        var task = GetById(id);

        if (task is null)
        {
            return false;
        }

        return _tasks.Remove(task);
    }
}
```

------------------------------------------------------------------------

# 38. Un problema importante

Podríamos hacer esto:

``` csharp
public class TasksController : ControllerBase
{
    private readonly TaskManager _manager;

    public TasksController()
    {
        _manager = new TaskManager();
    }
}
```

Funciona.

Pero el controller queda acoplado a una implementación concreta.

También dificulta:

-   testing;
-   reemplazo de implementaciones;
-   administración del ciclo de vida;
-   composición de dependencias.

Esto nos lleva al módulo 4:

**Dependency Injection.**

------------------------------------------------------------------------

# 39. Solución temporal

Para que nuestro proyecto funcione podemos registrar:

``` csharp
builder.Services.AddSingleton<TaskManager>();
```

Esto utiliza Dependency Injection, aunque todavía no vamos a estudiar DI
en profundidad.

En el módulo 4 aprenderemos:

``` text
Singleton
Scoped
Transient
```

Por ahora queremos que el mismo `TaskManager` conserve las tareas
mientras la aplicación esté ejecutándose.

------------------------------------------------------------------------

# 40. Program.cs completo

``` csharp
using TaskManagement.Api.Services;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

builder.Services.AddSingleton<TaskManager>();

var app = builder.Build();

app.MapControllers();

app.Run();
```

------------------------------------------------------------------------

# 41. TasksController

``` csharp
using Microsoft.AspNetCore.Mvc;
using TaskManagement.Api.Domain;
using TaskManagement.Api.Services;

namespace TaskManagement.Api.Controllers;

[ApiController]
[Route("api/[controller]")]
public class TasksController : ControllerBase
{
    private readonly TaskManager _manager;

    public TasksController(TaskManager manager)
    {
        _manager = manager;
    }

    [HttpGet]
    public ActionResult<List<TaskItem>> GetAll()
    {
        return Ok(_manager.GetAll());
    }

    [HttpGet("{id:guid}")]
    public ActionResult<TaskItem> GetById(Guid id)
    {
        var task = _manager.GetById(id);

        if (task is null)
        {
            return NotFound();
        }

        return Ok(task);
    }

    [HttpPost]
    public ActionResult<TaskItem> Create(
        CreateTaskRequest request)
    {
        var task = _manager.Create(
            request.Title,
            request.Description);

        return CreatedAtAction(
            nameof(GetById),
            new { id = task.Id },
            task);
    }

    [HttpDelete("{id:guid}")]
    public IActionResult Delete(Guid id)
    {
        var deleted = _manager.Delete(id);

        if (!deleted)
        {
            return NotFound();
        }

        return NoContent();
    }
}
```

------------------------------------------------------------------------

# 42. CreateTaskRequest

Por ahora:

``` csharp
namespace TaskManagement.Api.Controllers;

public class CreateTaskRequest
{
    public string Title { get; set; } = string.Empty;

    public string? Description { get; set; }
}
```

En el módulo 9 vamos a mejorar esto mediante:

``` text
DTOs
Validation
Error handling
```

------------------------------------------------------------------------

# 43. Probar la API

Podemos utilizar:

-   Swagger/OpenAPI;
-   Postman;
-   Bruno;
-   curl;
-   un frontend;
-   cualquier cliente HTTP.

Si la plantilla/configuración incluye OpenAPI, podremos utilizar la
documentación generada por la aplicación.

------------------------------------------------------------------------

# 44. Crear una tarea

Request:

``` http
POST /api/tasks
Content-Type: application/json
```

Body:

``` json
{
  "title": "Learn ASP.NET Core",
  "description": "Build first API"
}
```

Respuesta esperada:

``` http
201 Created
```

------------------------------------------------------------------------

# 45. Obtener tareas

``` http
GET /api/tasks
```

Respuesta:

``` http
200 OK
```

``` json
[
  {
    "id": "....",
    "title": "Learn ASP.NET Core",
    "description": "Build first API",
    "status": "Pending",
    "createdAt": "..."
  }
]
```

------------------------------------------------------------------------

# 46. Obtener una tarea

``` http
GET /api/tasks/{id}
```

Si existe:

``` http
200 OK
```

Si no existe:

``` http
404 Not Found
```

Este patrón será constante en nuestras APIs.

------------------------------------------------------------------------

# 47. Resource

Una API REST normalmente expone recursos.

En nuestro caso:

``` text
/tasks
```

representa una colección.

``` text
/tasks/{id}
```

representa un recurso individual.

Por eso:

``` http
GET /api/tasks
```

significa:

> Obtener la colección de tareas.

Mientras:

``` http
GET /api/tasks/123
```

significa:

> Obtener la tarea 123.

------------------------------------------------------------------------

# 48. Controller vs Service

Es importante comenzar a separar responsabilidades.

El Controller se ocupa principalmente de:

``` text
HTTP
Routing
Request
Response
Status codes
```

El Service/Manager se ocupa de:

``` text
Application logic
Business operations
```

Por ahora:

``` text
Controller
   ↓
TaskManager
```

Más adelante tendremos:

``` text
Controller
   ↓
Mediator
   ↓
Handler
   ↓
Service / Domain
   ↓
Repository
   ↓
Database
```

Esto aparecerá progresivamente.

------------------------------------------------------------------------

# 49. No colocar toda la lógica en el Controller

Evitemos:

``` csharp
[HttpPost]
public IActionResult Create(CreateTaskRequest request)
{
    // Decenas de líneas de lógica
    // validaciones
    // cálculos
    // acceso a DB
    // llamadas externas
}
```

Preferimos:

``` text
Controller
     ↓
Application logic
```

El Controller actúa como frontera entre:

``` text
HTTP
```

y:

``` text
Application
```

------------------------------------------------------------------------

# 50. Async en Controllers

En aplicaciones reales veremos:

``` csharp
[HttpGet]
public async Task<ActionResult<List<TaskItem>>> GetAll()
{
    var tasks = await ...;

    return Ok(tasks);
}
```

Esto será especialmente importante con:

``` text
EF Core
External APIs
Redis
Message brokers
```

Nuestro `TaskManager` trabaja en memoria, por lo que no debemos forzar
async artificialmente.

Regla práctica:

> Utilizá async cuando exista una operación asíncrona que lo justifique.

------------------------------------------------------------------------

# 51. Swagger / OpenAPI

Una API profesional necesita documentación.

OpenAPI puede describir:

``` text
Endpoints
Parameters
Request bodies
Responses
Schemas
Authentication
```

Swagger UI puede utilizar esa especificación para ofrecer una interfaz
interactiva.

Más adelante documentaremos correctamente nuestra API.

------------------------------------------------------------------------

# 52. Ejercicio 1 --- Implementar PUT

Implementá:

``` http
PUT /api/tasks/{id}
```

Debe permitir modificar:

``` text
Title
Description
```

No debería modificar:

``` text
Id
CreatedAt
Status
```

Pensá primero cómo lo resolverías.

------------------------------------------------------------------------

# 53. Ejercicio 2 --- Complete

Agregá:

``` http
POST /api/tasks/{id}/complete
```

Debe:

1.  buscar la tarea;
2.  devolver `404` si no existe;
3.  completar la tarea;
4.  devolver la tarea actualizada.

------------------------------------------------------------------------

# 54. Ejercicio 3 --- Query Parameter

Implementá:

``` http
GET /api/tasks?status=Pending
```

Debe permitir:

``` text
Pending
InProgress
Completed
```

Si no se proporciona `status`, debe devolver todas.

Utilizá LINQ dentro de `TaskManager`.

------------------------------------------------------------------------

# 55. Ejercicio 4 --- Search

Implementá:

``` http
GET /api/tasks?search=aspnet
```

Debe buscar por título.

Por ejemplo:

``` text
Learn ASP.NET Core
ASP.NET Core Authentication
Build ASP.NET API
```

deberían coincidir con:

``` text
search=aspnet
```

------------------------------------------------------------------------

# 56. Ejercicio 5 --- Status codes

Revisá todos los endpoints y asegurate de utilizar correctamente:

``` text
200
201
204
404
```

Preguntate:

> ¿Qué representa cada código desde el punto de vista del consumidor de
> la API?

------------------------------------------------------------------------

# 57. Challenge --- Task Summary

Creá:

``` http
GET /api/tasks/{id}/summary
```

que devuelva:

``` json
{
  "id": "....",
  "title": "Learn ASP.NET Core",
  "status": "Pending"
}
```

Podés utilizar:

``` csharp
public record TaskSummary(
    Guid Id,
    string Title,
    TaskStatus Status
);
```

Esto anticipa los DTOs que veremos en el módulo 9.

------------------------------------------------------------------------

# 58. Preguntas de comprensión

Antes de pasar al siguiente módulo deberías poder responder:

### 1

¿Qué diferencia hay entre:

``` text
.NET
```

y:

``` text
ASP.NET Core
```

### 2

¿Qué hace:

``` csharp
builder.Services.AddControllers();
```

### 3

¿Qué hace:

``` csharp
app.MapControllers();
```

### 4

¿Qué es un Controller?

### 5

¿Qué diferencia hay entre:

``` text
Route Parameter
```

y:

``` text
Query Parameter
```

### 6

¿Para qué sirve:

``` csharp
[ApiController]
```

### 7

¿Qué diferencia conceptual existe entre:

``` text
IActionResult
```

y:

``` text
ActionResult<T>
```

### 8

¿Cuándo usarías:

``` text
200
201
204
404
```

### 9

¿Qué hace Model Binding?

### 10

¿Por qué no queremos poner toda la lógica de negocio dentro del
Controller?

------------------------------------------------------------------------

# 59. Relación con Angular / React

Como frontend developer, podés visualizarlo así:

  Frontend              ASP.NET Core
  --------------------- ---------------------
  HttpClient            HTTP client
  Service               Application Service
  Interceptor           Middleware
  Guard                 Authorization
  Environment           Configuration
  Jest                  xUnit
  Promise`<T>`{=html}   Task`<T>`{=html}
  array.filter()        Where()
  array.map()           Select()
  array.some()          Any()
  find()                FirstOrDefault()

No son equivalencias exactas.

Son solamente referencias para construir un mapa mental más rápido.

------------------------------------------------------------------------

# 60. Algo importante: Singleton temporal

En este módulo registramos:

``` csharp
builder.Services.AddSingleton<TaskManager>();
```

Esto significa que ASP.NET Core mantiene una instancia de `TaskManager`
durante la vida de la aplicación.

Por eso nuestras tareas permanecen en memoria:

``` text
Request 1
   ↓
TaskManager
   ↓
Create task

Request 2
   ↓
same TaskManager
   ↓
Get tasks
```

Esta no será nuestra solución definitiva.

En el módulo 4 estudiaremos:

``` text
Singleton
Scoped
Transient
```

Y en el módulo 6 reemplazaremos la memoria por una base de datos.

------------------------------------------------------------------------

# 61. Qué deberías llevarte

El modelo mental principal es:

``` text
ASP.NET Core
      │
      ↓
    HTTP
      │
      ↓
   Routing
      │
      ↓
 Controller
      │
      ↓
 Application logic
```

Nuestro proyecto ahora tiene:

``` text
Client
  ↓
HTTP
  ↓
TasksController
  ↓
TaskManager
  ↓
List<TaskItem>
```

Ya construimos un backend HTTP funcional.

------------------------------------------------------------------------

# 62. Estado del proyecto

``` text
Task Management API
│
├── Domain
│   ├── TaskItem
│   └── TaskStatus
│
├── Services
│   └── TaskManager
│
├── Controllers
│   ├── TasksController
│   └── CreateTaskRequest
│
└── Program.cs
```

Endpoints principales:

``` text
GET    /api/tasks
GET    /api/tasks/{id}
POST   /api/tasks
PUT    /api/tasks/{id}
DELETE /api/tasks/{id}
```

------------------------------------------------------------------------

# 63. Próximo módulo

En el módulo 3 nos concentraremos en:

# CRUD en memoria

Vamos a completar nuestra API para tener un CRUD HTTP completo y
profundizar en:

-   diseño de endpoints;
-   request/response;
-   route parameters;
-   query parameters;
-   model binding;
-   status codes;
-   actualización;
-   eliminación;
-   filtrado;
-   ordenamiento;
-   paginación básica.

El resultado será un CRUD HTTP completo antes de introducir persistencia
y arquitectura adicional.
