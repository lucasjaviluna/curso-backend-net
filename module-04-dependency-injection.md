# Módulo 4 --- Dependency Injection en .NET

## Objetivo del módulo

En este módulo vamos a incorporar **Dependency Injection (DI)** a
nuestro Task Management API.

Hasta ahora tenemos una arquitectura sencilla:

``` text
HTTP
 ↓
TasksController
 ↓
TaskManager
 ↓
List<TaskItem>
```

Al finalizar el módulo tendremos:

``` text
HTTP
 ↓
TasksController
 ↓
ITaskService
 ↓
TaskService
 ↓
memoria
```

La idea principal no es aprender a memorizar `AddScoped`, `AddSingleton`
o `AddTransient`, sino entender **por qué existe DI, qué problema
resuelve y cómo usarla correctamente**.

------------------------------------------------------------------------

# 1. ¿Qué problema resuelve Dependency Injection?

Supongamos que nuestro controller hace esto:

``` csharp
public class TasksController : ControllerBase
{
    private readonly TaskManager _taskManager;

    public TasksController()
    {
        _taskManager = new TaskManager();
    }
}
```

Funciona.

Pero el controller está tomando una decisión que no debería tomar:

> "Yo sé exactamente qué implementación necesito y yo mismo voy a
> crearla."

Esto genera acoplamiento.

Si mañana `TaskManager` cambia por `TaskService`, el controller tiene
que cambiar.

Y si `TaskService` necesita otras dependencias:

``` csharp
public TaskService()
{
    _repository = new TaskRepository();
    _logger = new Logger();
}
```

el problema crece.

La clase termina siendo responsable de construir todo el árbol de
objetos que necesita.

DI cambia esta responsabilidad.

En lugar de:

``` text
Controller
   │
   ├── new TaskService()
   │       ├── new Repository()
   │       └── new Logger()
   │
   └── ...
```

tenemos:

``` text
.NET DI Container
       │
       ├── TaskService
       ├── Repository
       └── Logger
             │
             ↓
        Controller
```

El framework construye los objetos y se los entrega al controller.

------------------------------------------------------------------------

# 2. Dependency Injection en una frase

Podemos resumir DI así:

> Una clase recibe desde afuera los objetos que necesita en lugar de
> crearlos por sí misma.

Por ejemplo:

``` csharp
public class TasksController
{
    private readonly ITaskService _taskService;

    public TasksController(ITaskService taskService)
    {
        _taskService = taskService;
    }
}
```

El controller no hace:

``` csharp
new TaskService()
```

El controller solamente declara:

> "Necesito algo que implemente `ITaskService`."

.NET se encarga de proporcionárselo.

------------------------------------------------------------------------

# 3. Inversion of Control

Dependency Injection está relacionada con un concepto más general:

**Inversion of Control (IoC).**

Sin IoC:

``` text
Controller
   │
   └── crea → Service
```

Con IoC:

``` text
DI Container
   │
   └── crea → Service
                │
                ↓
             Controller
```

El control sobre la creación de dependencias se mueve desde nuestras
clases hacia el contenedor de DI.

Por eso hablamos de **inversión de control**.

DI es una de las formas más comunes de implementar IoC.

------------------------------------------------------------------------

# 4. El contenedor de Dependency Injection de .NET

ASP.NET Core incluye un contenedor de DI.

Lo configuramos principalmente mediante:

``` csharp
builder.Services
```

Por ejemplo:

``` csharp
builder.Services.AddScoped<ITaskService, TaskService>();
```

Estamos diciendo:

``` text
Cuando alguien solicite ITaskService
        ↓
entregar una instancia de TaskService
```

El registro ocurre normalmente en `Program.cs`.

------------------------------------------------------------------------

# 5. Interface vs implementación

Esta distinción es fundamental.

Tenemos:

``` csharp
public interface ITaskService
{
    TaskItem Create(string title, string? description);

    List<TaskItem> GetAll();

    TaskItem? GetById(Guid id);

    bool Delete(Guid id);
}
```

Y una implementación:

``` csharp
public class TaskService : ITaskService
{
    private readonly List<TaskItem> _tasks = [];

    public TaskItem Create(string title, string? description)
    {
        var task = new TaskItem(title, description);
        _tasks.Add(task);

        return task;
    }

    public List<TaskItem> GetAll()
    {
        return _tasks.ToList();
    }

    public TaskItem? GetById(Guid id)
    {
        return _tasks.FirstOrDefault(task => task.Id == id);
    }

    public bool Delete(Guid id)
    {
        var task = GetById(id);

        if (task is null)
            return false;

        return _tasks.Remove(task);
    }
}
```

La interface representa el contrato:

``` text
ITaskService
```

La clase representa la implementación:

``` text
TaskService
```

Esto permite que el controller dependa del contrato.

------------------------------------------------------------------------

# 6. Constructor Injection

La forma más habitual de DI en ASP.NET Core es **constructor
injection**.

Ejemplo:

``` csharp
public class TasksController : ControllerBase
{
    private readonly ITaskService _taskService;

    public TasksController(ITaskService taskService)
    {
        _taskService = taskService;
    }
}
```

Observa algo importante:

El controller no sabe cómo construir `TaskService`.

Solamente sabe que necesita:

``` csharp
ITaskService
```

Esto reduce el acoplamiento.

------------------------------------------------------------------------

# 7. Registrar una dependencia

En `Program.cs`:

``` csharp
builder.Services.AddScoped<ITaskService, TaskService>();
```

La relación queda:

``` text
ITaskService
     ↓
TaskService
```

Cuando ASP.NET Core cree un `TasksController`, detectará:

``` csharp
TasksController(ITaskService taskService)
```

y buscará en el contenedor:

``` text
¿Quién implementa ITaskService?
```

Encontrará:

``` text
TaskService
```

y lo inyectará.

------------------------------------------------------------------------

# 8. Refactorizando nuestro proyecto

Hasta ahora teníamos algo similar a:

``` csharp
public class TaskManager
{
    private readonly List<TaskItem> _tasks = [];

    // ...
}
```

Y el controller utilizaba directamente `TaskManager`.

Ahora vamos a convertir esa responsabilidad en un servicio.

Una estructura sencilla puede ser:

``` text
TaskManagement.Api/
├── Controllers/
│   └── TasksController.cs
├── Models/
│   └── TaskItem.cs
├── Services/
│   ├── ITaskService.cs
│   └── TaskService.cs
└── Program.cs
```

Todavía no necesitamos Repository.

Eso llegará en el módulo 7.

------------------------------------------------------------------------

# 9. Crear ITaskService

Archivo:

``` text
Services/ITaskService.cs
```

Contenido:

``` csharp
using TaskManagement.Api.Models;

namespace TaskManagement.Api.Services;

public interface ITaskService
{
    TaskItem Create(string title, string? description);

    List<TaskItem> GetAll();

    TaskItem? GetById(Guid id);

    bool Delete(Guid id);
}
```

La interface define lo que el servicio puede hacer.

No define cómo lo hace.

------------------------------------------------------------------------

# 10. Crear TaskService

Archivo:

``` text
Services/TaskService.cs
```

Contenido:

``` csharp
using TaskManagement.Api.Models;

namespace TaskManagement.Api.Services;

public class TaskService : ITaskService
{
    private readonly List<TaskItem> _tasks = [];

    public TaskItem Create(string title, string? description)
    {
        var task = new TaskItem(title, description);

        _tasks.Add(task);

        return task;
    }

    public List<TaskItem> GetAll()
    {
        return _tasks.ToList();
    }

    public TaskItem? GetById(Guid id)
    {
        return _tasks.FirstOrDefault(task => task.Id == id);
    }

    public bool Delete(Guid id)
    {
        var task = GetById(id);

        if (task is null)
            return false;

        return _tasks.Remove(task);
    }
}
```

La implementación sigue utilizando memoria.

Eso está bien.

El objetivo de este módulo es DI, no persistencia.

------------------------------------------------------------------------

# 11. Registrar el servicio

En `Program.cs`:

``` csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

builder.Services.AddScoped<ITaskService, TaskService>();

var app = builder.Build();

app.MapControllers();

app.Run();
```

La línea importante es:

``` csharp
builder.Services.AddScoped<ITaskService, TaskService>();
```

------------------------------------------------------------------------

# 12. Cambiar el controller

Antes:

``` csharp
public class TasksController : ControllerBase
{
    private readonly TaskManager _taskManager;

    public TasksController(TaskManager taskManager)
    {
        _taskManager = taskManager;
    }
}
```

Ahora:

``` csharp
public class TasksController : ControllerBase
{
    private readonly ITaskService _taskService;

    public TasksController(ITaskService taskService)
    {
        _taskService = taskService;
    }
}
```

El controller ya no conoce `TaskService`.

Conoce solamente:

``` csharp
ITaskService
```

Esto es una mejora arquitectónica importante.

------------------------------------------------------------------------

# 13. Controller completo

Una versión inicial podría ser:

``` csharp
using Microsoft.AspNetCore.Mvc;
using TaskManagement.Api.Models;
using TaskManagement.Api.Services;

namespace TaskManagement.Api.Controllers;

[ApiController]
[Route("api/[controller]")]
public class TasksController : ControllerBase
{
    private readonly ITaskService _taskService;

    public TasksController(ITaskService taskService)
    {
        _taskService = taskService;
    }

    [HttpGet]
    public ActionResult<List<TaskItem>> GetAll()
    {
        return Ok(_taskService.GetAll());
    }

    [HttpGet("{id:guid}")]
    public ActionResult<TaskItem> GetById(Guid id)
    {
        var task = _taskService.GetById(id);

        if (task is null)
            return NotFound();

        return Ok(task);
    }

    [HttpPost]
    public ActionResult<TaskItem> Create(CreateTaskRequest request)
    {
        var task = _taskService.Create(
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
        var deleted = _taskService.Delete(id);

        if (!deleted)
            return NotFound();

        return NoContent();
    }
}
```

La arquitectura queda:

``` text
HTTP
 ↓
TasksController
 ↓
ITaskService
 ↓
TaskService
 ↓
List<TaskItem>
```

------------------------------------------------------------------------

# 14. ¿Qué significa AddScoped?

.NET tiene tres lifetimes principales:

``` text
Singleton
Scoped
Transient
```

Son reglas que determinan cuánto tiempo vive una instancia.

------------------------------------------------------------------------

# 15. Singleton

``` csharp
builder.Services.AddSingleton<ITaskService, TaskService>();
```

Significa:

> Crear una instancia y reutilizarla durante toda la vida de la
> aplicación.

Conceptualmente:

``` text
Application
    │
    └── TaskService #1
          ↑
          ├── Request 1
          ├── Request 2
          ├── Request 3
          └── Request 4
```

Hay una única instancia.

Esto puede ser útil para objetos que realmente deben ser globales y que
son seguros para compartir.

Pero hay que tener mucho cuidado con estado mutable.

Por ejemplo, nuestro servicio contiene:

``` csharp
private readonly List<TaskItem> _tasks = [];
```

Con `Singleton`, esa lista sería compartida por todas las requests.

Eso cambia el comportamiento de nuestra aplicación.

------------------------------------------------------------------------

# 16. Scoped

``` csharp
builder.Services.AddScoped<ITaskService, TaskService>();
```

Significa:

> Una instancia por scope.

En una aplicación ASP.NET Core, normalmente el scope corresponde a una
request HTTP.

Conceptualmente:

``` text
Request 1
   ↓
TaskService #1

Request 2
   ↓
TaskService #2

Request 3
   ↓
TaskService #3
```

Por eso `Scoped` es muy habitual para servicios de aplicación y
especialmente para dependencias relacionadas con una request.

Más adelante veremos que `DbContext` de Entity Framework Core
normalmente se utiliza con lifetime scoped.

------------------------------------------------------------------------

# 17. Transient

``` csharp
builder.Services.AddTransient<ITaskService, TaskService>();
```

Significa:

> Crear una nueva instancia cada vez que se solicita la dependencia.

Conceptualmente:

``` text
Request
  │
  ├── resolve → TaskService #1
  ├── resolve → TaskService #2
  └── resolve → TaskService #3
```

Se utiliza cuando queremos objetos pequeños y sin estado que no
necesiten ser compartidos.

------------------------------------------------------------------------

# 18. Comparación

  -----------------------------------------------------------------------
  Lifetime                Duración aproximada     Uso típico
  ----------------------- ----------------------- -----------------------
  Singleton               Toda la aplicación      objetos
                                                  globales/stateless
                                                  cuidadosamente
                                                  diseñados

  Scoped                  Una request             servicios de
                                                  aplicación, DbContext

  Transient               Cada resolución         servicios
                                                  livianos/stateless
  -----------------------------------------------------------------------

Una regla práctica para empezar:

``` text
¿Necesito compartir una única instancia global?
    → Singleton

¿La dependencia pertenece al ciclo de una request?
    → Scoped

¿Es una dependencia liviana y no necesita compartir estado?
    → Transient
```

No conviertas esta tabla en una regla absoluta. La elección depende de
la naturaleza de la dependencia.

------------------------------------------------------------------------

# 19. Un error importante: lifetime incorrecto

Imaginemos:

``` csharp
builder.Services.AddScoped<IRepository, Repository>();
builder.Services.AddSingleton<ITaskService, TaskService>();
```

Y:

``` csharp
public class TaskService : ITaskService
{
    private readonly IRepository _repository;

    public TaskService(IRepository repository)
    {
        _repository = repository;
    }
}
```

Tenemos:

``` text
Singleton
   ↓
Scoped
```

Esto es problemático.

Una dependencia singleton vive más tiempo que una dependencia scoped.

En general, no debemos hacer que una dependencia de mayor duración
capture una dependencia de menor duración.

Una forma simple de recordarlo:

``` text
Singleton
   ↓
Singleton

Scoped
   ↓
Scoped / Singleton

Transient
   ↓
Transient / Scoped / Singleton
```

El detalle exacto depende de cómo se resuelven las dependencias, pero
como regla práctica evita que un singleton capture dependencias scoped.

------------------------------------------------------------------------

# 20. DI no significa usar interfaces para absolutamente todo

Es importante no caer en:

``` text
Class
 ↓
Interface
 ↓
Factory
 ↓
Decorator
 ↓
Adapter
 ↓
...
```

DI no requiere que cada clase tenga una interface.

La pregunta correcta es:

> ¿Existe una razón para desacoplar el consumidor de esta
> implementación?

En nuestro caso:

``` text
TasksController
       ↓
ITaskService
       ↓
TaskService
```

tiene sentido porque el controller representa una capa HTTP y no debería
estar acoplado a una implementación concreta.

------------------------------------------------------------------------

# 21. Evitar `new` para dependencias

Un principio práctico:

Evita esto dentro de tus clases:

``` csharp
public class TasksController
{
    private readonly TaskService _service;

    public TasksController()
    {
        _service = new TaskService();
    }
}
```

Preferimos:

``` csharp
public class TasksController
{
    private readonly ITaskService _service;

    public TasksController(ITaskService service)
    {
        _service = service;
    }
}
```

La creación de dependencias queda centralizada en la composición de la
aplicación.

En nuestro proyecto:

``` text
Program.cs
   │
   └── composition root
           │
           ├── ITaskService → TaskService
           └── ...
```

El término **composition root** describe el lugar donde configuramos
cómo se construye nuestra aplicación.

En ASP.NET Core normalmente está principalmente en `Program.cs`.

------------------------------------------------------------------------

# 22. Service composition

Las dependencias pueden formar una cadena.

Por ejemplo, más adelante tendremos:

``` text
Controller
    ↓
Service
    ↓
Repository
    ↓
DbContext
    ↓
Database
```

El controller solamente necesita:

``` csharp
ITaskService
```

El service puede necesitar:

``` csharp
ITaskRepository
```

El repository puede necesitar:

``` csharp
AppDbContext
```

DI permite construir automáticamente ese grafo de dependencias.

Por ejemplo:

``` csharp
public class TaskService : ITaskService
{
    private readonly ITaskRepository _repository;

    public TaskService(ITaskRepository repository)
    {
        _repository = repository;
    }
}
```

Y:

``` csharp
builder.Services.AddScoped<ITaskService, TaskService>();
builder.Services.AddScoped<ITaskRepository, TaskRepository>();
```

Todavía **no vamos a implementar Repository**.

Lo veremos en el módulo 7.

------------------------------------------------------------------------

# 23. ¿Por qué DI mejora los tests?

Supongamos que tenemos:

``` csharp
public class TaskService : ITaskService
{
    // ...
}
```

Y nuestro controller depende de:

``` csharp
ITaskService
```

Durante un test podemos crear una implementación falsa:

``` csharp
public class FakeTaskService : ITaskService
{
    public TaskItem Create(string title, string? description)
    {
        return new TaskItem(title, description);
    }

    public List<TaskItem> GetAll()
    {
        return [];
    }

    public TaskItem? GetById(Guid id)
    {
        return null;
    }

    public bool Delete(Guid id)
    {
        return true;
    }
}
```

Entonces podemos crear el controller utilizando:

``` csharp
var service = new FakeTaskService();

var controller = new TasksController(service);
```

El controller no sabe ni necesita saber que está usando un fake.

Ve solamente:

``` text
ITaskService
```

Esto hace que las pruebas sean mucho más sencillas.

Testing será tratado formalmente en el módulo 12.

------------------------------------------------------------------------

# 24. Analogía con Angular

Como desarrollador Angular, este concepto debería resultarte familiar.

En Angular puedes tener:

``` typescript
@Injectable()
export class TaskService {}
```

y consumirlo:

``` typescript
constructor(private taskService: TaskService) {}
```

Angular crea y entrega la dependencia.

En ASP.NET Core:

``` csharp
builder.Services.AddScoped<ITaskService, TaskService>();
```

y:

``` csharp
public TasksController(ITaskService taskService)
{
    _taskService = taskService;
}
```

Conceptualmente:

``` text
Angular DI
     ↕
ASP.NET Core DI
```

No son idénticos internamente, pero el concepto fundamental es muy
parecido.

------------------------------------------------------------------------

# 25. Qué ocurre cuando llega una request

Supongamos:

``` http
GET /api/tasks
```

ASP.NET Core necesita ejecutar:

``` csharp
TasksController.GetAll()
```

Para crear el controller necesita:

``` csharp
ITaskService
```

El contenedor busca el registro:

``` csharp
AddScoped<ITaskService, TaskService>()
```

Entonces crea:

``` text
TaskService
```

y se lo entrega al controller:

``` text
HTTP Request
     ↓
ASP.NET Core
     ↓
DI Container
     ↓
TaskService
     ↓
TasksController
     ↓
GetAll()
```

Ese proceso ocurre automáticamente.

------------------------------------------------------------------------

# 26. Qué pasa si olvidamos registrar la dependencia

Si tenemos:

``` csharp
public TasksController(ITaskService taskService)
{
    _taskService = taskService;
}
```

pero olvidamos:

``` csharp
builder.Services.AddScoped<ITaskService, TaskService>();
```

ASP.NET Core no sabe qué implementación utilizar.

Obtendremos un error de resolución de dependencia.

Conceptualmente:

``` text
Unable to resolve service for type ITaskService
```

Cuando aparezca un error de este tipo, una de las primeras cosas que
debemos revisar es:

``` text
¿La dependencia está registrada?
```

------------------------------------------------------------------------

# 27. DI y configuración centralizada

Una ventaja importante es que las decisiones de composición quedan
concentradas.

Por ejemplo:

``` csharp
builder.Services.AddScoped<ITaskService, TaskService>();
```

Si mañana queremos utilizar:

``` csharp
AdvancedTaskService
```

podemos cambiar:

``` csharp
builder.Services.AddScoped<ITaskService, AdvancedTaskService>();
```

El controller puede permanecer igual:

``` csharp
public TasksController(ITaskService taskService)
{
    _taskService = taskService;
}
```

Esto es uno de los beneficios más importantes:

> Cambiar la implementación sin cambiar el consumidor.

------------------------------------------------------------------------

# 28. Qué NO vamos a hacer todavía

En este módulo no vamos a introducir:

-   Repository Pattern en profundidad
-   MediatR
-   CQRS
-   decorators
-   factories complejas
-   Scrutor
-   containers externos
-   módulos de DI personalizados
-   arquitecturas excesivamente abstractas

Nuestro objetivo es dominar:

``` text
Interface
   ↓
Implementation
   ↓
Registration
   ↓
Constructor Injection
   ↓
Lifetime
```

Eso es suficiente por ahora.

------------------------------------------------------------------------

# 29. Práctica principal

## Ejercicio 1 --- Completar la migración

Asegúrate de que el proyecto ya no utiliza:

``` csharp
TaskManager
```

y que utiliza:

``` csharp
ITaskService
TaskService
```

El controller debe depender de:

``` csharp
ITaskService
```

------------------------------------------------------------------------

## Ejercicio 2 --- Agregar Update

Extiende:

``` csharp
ITaskService
```

con:

``` csharp
TaskItem? Update(
    Guid id,
    string title,
    string? description);
```

Implementa el método en:

``` text
TaskService
```

y agrega:

``` http
PUT /api/tasks/{id}
```

al controller.

------------------------------------------------------------------------

## Ejercicio 3 --- Agregar Complete

Agrega:

``` csharp
bool Complete(Guid id);
```

a `ITaskService`.

Después implementa:

``` http
POST /api/tasks/{id}/complete
```

------------------------------------------------------------------------

## Ejercicio 4 --- Crear un Fake

Crea:

``` text
FakeTaskService
```

que implemente:

``` csharp
ITaskService
```

No necesitas hacer un test todavía.

El objetivo es comprobar que entiendes que el controller puede trabajar
con distintas implementaciones.

------------------------------------------------------------------------

## Ejercicio 5 --- Experimentar con lifetimes

Prueba:

``` csharp
AddSingleton
AddScoped
AddTransient
```

Temporalmente.

Agrega un identificador a `TaskService`:

``` csharp
public Guid InstanceId { get; } = Guid.NewGuid();
```

Agrega un endpoint temporal:

``` csharp
[HttpGet("instance")]
public IActionResult GetInstance()
{
    if (_taskService is TaskService service)
        return Ok(service.InstanceId);

    return Ok();
}
```

Haz varias requests y observa el comportamiento.

La idea es experimentar:

``` text
Singleton
Scoped
Transient
```

y comprobar que el lifetime no es solamente teoría.

------------------------------------------------------------------------

# 30. Preguntas de comprensión

Antes de continuar, deberías poder responder:

### 1. ¿Qué problema resuelve Dependency Injection?

### 2. ¿Cuál es la diferencia entre una interface y una implementación?

### 3. ¿Por qué nuestro controller depende de `ITaskService` y no de `TaskService`?

### 4. ¿Qué hace esta línea?

``` csharp
builder.Services.AddScoped<ITaskService, TaskService>();
```

### 5. ¿Qué diferencia existe entre Singleton, Scoped y Transient?

### 6. ¿Por qué un Singleton que captura una dependencia Scoped puede ser problemático?

### 7. ¿Qué ventaja tiene DI para testing?

### 8. ¿Dónde estamos configurando las dependencias de nuestra aplicación?

------------------------------------------------------------------------

# 31. Estado del proyecto

Después del módulo 4, nuestro proyecto queda:

``` text
                    HTTP
                     │
                     ↓
             TasksController
                     │
                     ↓
               ITaskService
                     │
                     ↓
                TaskService
                     │
                     ↓
               List<TaskItem>
```

Y la configuración:

``` text
Program.cs
    │
    └── DI Container
           │
           └── ITaskService → TaskService
```

La arquitectura evolucionó desde:

``` text
Controller
    ↓
TaskManager
```

a:

``` text
Controller
    ↓
Interface
    ↓
Implementation
```

Esto parece un cambio pequeño, pero prepara el proyecto para la
siguiente evolución.

------------------------------------------------------------------------

# 32. Conexión con los próximos módulos

Hasta ahora:

``` text
Módulo 1
C#
 ↓
Domain

Módulo 2
ASP.NET Core
 ↓
HTTP/API

Módulo 3
CRUD
 ↓
In-memory API

Módulo 4
DI
 ↓
Controller → Service
```

El siguiente paso será:

``` text
Módulo 5
Configuration
```

Vamos a sacar configuraciones del código y aprender cómo ASP.NET Core
maneja:

-   `appsettings.json`
-   configuraciones por ambiente
-   variables de entorno
-   Options
-   secretos
-   configuración tipada

Más adelante, cuando lleguemos a EF Core, la arquitectura evolucionará
hacia:

``` text
Controller
    ↓
Service
    ↓
Repository
    ↓
DbContext
    ↓
SQL
```

Y posteriormente:

``` text
Controller
    ↓
Mediator
    ↓
Command / Query
    ↓
Handler
    ↓
Service / Repository
    ↓
Database
```

Ese será el camino hacia la arquitectura que utilizaremos en el proyecto
final.

------------------------------------------------------------------------

# Resumen del módulo

Los conceptos que debes llevarte son:

``` text
Dependency Injection
        │
        ├── IoC
        │
        ├── Interface
        │
        ├── Implementation
        │
        ├── Constructor Injection
        │
        ├── IServiceCollection
        │
        └── Lifetimes
              ├── Singleton
              ├── Scoped
              └── Transient
```

Y el principio más importante:

> **Una clase debería recibir sus dependencias en lugar de crearlas
> directamente.**

En nuestro proyecto:

``` csharp
public TasksController(ITaskService taskService)
{
    _taskService = taskService;
}
```

es mucho más importante que memorizar:

``` csharp
AddScoped
```

Porque primero debes entender **qué dependencia necesita tu clase y por
qué**, y después decidir cómo debe vivir esa dependencia.

## Resultado final

Al terminar este módulo deberías poder mirar un proyecto ASP.NET Core y
reconocer inmediatamente:

``` text
¿Dónde se registran las dependencias?
¿Quién implementa esta interface?
¿Qué lifetime tiene?
¿Cómo llega esta dependencia al controller?
¿Podría reemplazar esta implementación?
```

Si puedes responder esas preguntas, ya estás entendiendo Dependency
Injection en .NET.
