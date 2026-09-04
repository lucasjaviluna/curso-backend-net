# Módulo 8 --- CQRS + Mediator

## 1. Objetivos

Al terminar este módulo podrás:

-   Entender qué problema intenta resolver CQRS.
-   Diferenciar Commands y Queries.
-   Entender qué es un Handler.
-   Entender qué hace un Mediator.
-   Implementar el flujo `Controller → Mediator → Handler`.
-   Diferenciar Handler, Service y Repository.
-   Entender Pipeline Behaviors.
-   Aplicar Behaviors para validación, logging, autorización y
    transacciones.
-   Integrar CQRS con EF Core y Repository.
-   Entender cuándo CQRS aporta valor y cuándo puede ser innecesario.
-   Comprender conceptos avanzados como read models, projections y
    event-driven architecture sin convertirlos todavía en parte
    obligatoria de la aplicación.

La arquitectura evolucionará de:

``` text
Controller
    ↓
ITaskService
    ↓
TaskService
    ↓
ITaskRepository
    ↓
TaskRepository
    ↓
DbContext
```

a:

``` text
Controller
    ↓
Mediator
    ↓
Command / Query
    ↓
Handler
    ↓
Repository
    ↓
DbContext
```

------------------------------------------------------------------------

# 2. El problema que intenta resolver CQRS

Hasta ahora teníamos:

``` text
Controller
    ↓
Service
```

Por ejemplo:

``` csharp
[HttpGet("{id:guid}")]
public async Task<ActionResult<TaskItem>> GetById(Guid id)
{
    var task = await _service.GetByIdAsync(id);

    if (task is null)
    {
        return NotFound();
    }

    return Ok(task);
}
```

Y para crear:

``` csharp
[HttpPost]
public async Task<ActionResult<TaskItem>> Create(
    CreateTaskRequest request)
{
    var task = await _service.CreateAsync(
        request.Title,
        request.Description);

    return CreatedAtAction(
        nameof(GetById),
        new { id = task.Id },
        task);
}
```

Funciona.

Pero a medida que crece la aplicación, `TaskService` puede convertirse
en:

``` text
TaskService
 ├── GetById
 ├── GetAll
 ├── Search
 ├── Filter
 ├── Create
 ├── Update
 ├── Delete
 ├── Complete
 ├── Assign
 ├── AddComment
 └── muchas operaciones más
```

Y empiezan a aparecer dos tipos de operaciones muy diferentes:

``` text
LECTURA
    ↓
Obtener información

ESCRITURA
    ↓
Cambiar el estado del sistema
```

CQRS parte precisamente de esta separación.

------------------------------------------------------------------------

# 3. ¿Qué significa CQRS?

CQRS significa:

> **Command Query Responsibility Segregation**

La idea básica es:

``` text
Command
    ↓
Cambia el estado

Query
    ↓
Lee información
```

Por ejemplo:

``` text
CreateTaskCommand
CompleteTaskCommand
DeleteTaskCommand
```

son Commands.

Mientras que:

``` text
GetTaskByIdQuery
GetTasksQuery
SearchTasksQuery
```

son Queries.

------------------------------------------------------------------------

# 4. Command

Un Command representa una intención de cambiar el estado de la
aplicación.

Ejemplo:

``` csharp
public record CreateTaskCommand(
    string Title,
    string? Description);
```

El Command contiene los datos necesarios para realizar la operación.

Conceptualmente:

``` text
CreateTaskCommand
        │
        ├── Title
        └── Description
```

Un Command expresa:

> "Quiero crear una tarea."

No debería ejecutar nada por sí mismo.

Es solamente el mensaje que representa la intención.

------------------------------------------------------------------------

# 5. Query

Una Query representa una intención de obtener información.

Por ejemplo:

``` csharp
public record GetTaskByIdQuery(Guid Id);
```

Expresa:

> "Quiero obtener esta tarea."

Otro ejemplo:

``` csharp
public record GetTasksQuery(
    int Page,
    int PageSize,
    TaskStatus? Status);
```

Expresa:

> "Quiero obtener las tareas que cumplen estos criterios."

Una Query no debería modificar el estado de la aplicación.

------------------------------------------------------------------------

# 6. La regla fundamental

Una forma sencilla de recordarlo:

``` text
COMMAND
   ↓
WRITE

QUERY
   ↓
READ
```

Por ejemplo:

  Operación              Tipo
  ---------------------- ---------
  Crear tarea            Command
  Actualizar tarea       Command
  Eliminar tarea         Command
  Completar tarea        Command
  Asignar tarea          Command
  Obtener tarea          Query
  Buscar tareas          Query
  Listar tareas          Query
  Obtener estadísticas   Query

------------------------------------------------------------------------

# 7. CQRS NO significa dos bases de datos

Este punto es muy importante.

CQRS no obliga a tener:

``` text
Write DB
     +
Read DB
```

Podemos comenzar perfectamente con:

``` text
                 ┌─────────────┐
Command ────────→│             │
                 │ SQL Server  │
Query ──────────→│             │
                 └─────────────┘
```

La separación inicial es lógica:

``` text
Command
Query
```

No necesariamente física.

Más adelante veremos que CQRS puede evolucionar hacia:

``` text
Write Model
     ↓
Write Database

Read Model
     ↓
Read Database
```

pero eso es una evolución arquitectónica, no un requisito inicial.

------------------------------------------------------------------------

# 8. ¿Qué es un Handler?

Un Handler es el componente que sabe ejecutar un Command o Query
específico.

Por ejemplo:

``` text
CreateTaskCommand
        ↓
CreateTaskCommandHandler
```

Y:

``` text
GetTaskByIdQuery
        ↓
GetTaskByIdQueryHandler
```

Esto permite pasar de:

``` text
TaskService
 ├── Create
 ├── GetById
 ├── GetAll
 ├── Delete
 ├── Complete
 └── ...
```

a:

``` text
CreateTaskCommand
        ↓
CreateTaskCommandHandler

GetTaskByIdQuery
        ↓
GetTaskByIdQueryHandler

DeleteTaskCommand
        ↓
DeleteTaskCommandHandler

CompleteTaskCommand
        ↓
CompleteTaskCommandHandler
```

Cada operación tiene un lugar mucho más específico.

------------------------------------------------------------------------

# 9. Handler vs Service

Esta es probablemente la pregunta más importante del módulo.

## Service tradicional

Un Service puede agrupar múltiples operaciones:

``` text
TaskService
 ├── Create
 ├── GetById
 ├── Update
 ├── Delete
 └── Complete
```

## Handler

En CQRS cada caso de uso puede tener su propio Handler:

``` text
CreateTaskCommandHandler
GetTaskByIdQueryHandler
UpdateTaskCommandHandler
DeleteTaskCommandHandler
CompleteTaskCommandHandler
```

Por lo tanto:

> Un Handler normalmente representa la ejecución de un caso de uso
> específico.

Esto puede hacer que una aplicación grande sea más fácil de navegar.

------------------------------------------------------------------------

# 10. ¿Entonces desaparecen los Services?

No necesariamente.

Podemos tener:

``` text
Handler
   ↓
Domain Service
   ↓
Repository
```

o:

``` text
Handler
   ↓
Repository
```

dependiendo del caso.

Por ejemplo:

``` text
CompleteTaskCommandHandler
        ↓
Task domain
        ↓
ITaskRepository
```

No necesitamos crear:

``` text
CompleteTaskService
```

solo porque existe un Handler.

CQRS no significa:

> "Cambiar Services por otra capa con otro nombre."

CQRS busca separar las responsabilidades de lectura y escritura y
organizar los casos de uso.

------------------------------------------------------------------------

# 11. ¿Qué hace el Mediator?

Ahora tenemos:

``` text
Controller
    ↓
CreateTaskCommand
    ↓
CreateTaskCommandHandler
```

Pero el Controller necesita alguna forma de enviar el Command al
Handler.

Ahí aparece el Mediator.

Conceptualmente:

``` text
Controller
    │
    │ Send(command)
    ↓
 Mediator
    │
    │ encuentra Handler
    ↓
Handler
```

El Controller no necesita conocer directamente:

``` csharp
CreateTaskCommandHandler
```

Solo necesita conocer:

``` text
Mediator
```

------------------------------------------------------------------------

# 12. Analogía con frontend

Puedes imaginar el Mediator como un dispatcher especializado.

En frontend podemos tener:

``` text
Component
   ↓
dispatch(action)
   ↓
Reducer / Effect
```

En CQRS:

``` text
Controller
   ↓
Mediator.Send(command)
   ↓
Handler
```

No son equivalentes técnicamente, pero la idea de enviar una intención y
permitir que otro componente la procese ayuda a entenderlo.

------------------------------------------------------------------------

# 13. Flujo completo

Supongamos:

``` http
POST /api/tasks
```

El flujo será:

``` text
HTTP Request
     ↓
TasksController
     ↓
CreateTaskCommand
     ↓
Mediator
     ↓
CreateTaskCommandHandler
     ↓
ITaskRepository
     ↓
TaskRepository
     ↓
DbContext
     ↓
SQL Server
```

La respuesta vuelve:

``` text
SQL Server
    ↓
Repository
    ↓
Handler
    ↓
Mediator
    ↓
Controller
    ↓
HTTP Response
```

------------------------------------------------------------------------

# 14. Implementación conceptual sin librería

Antes de utilizar una librería, es útil entender el concepto.

Podríamos tener:

``` csharp
public interface ICommand<TResult>
{
}
```

Y:

``` csharp
public interface ICommandHandler<TCommand, TResult>
    where TCommand : ICommand<TResult>
{
    Task<TResult> Handle(
        TCommand command,
        CancellationToken cancellationToken);
}
```

Para Queries:

``` csharp
public interface IQuery<TResult>
{
}
```

Y:

``` csharp
public interface IQueryHandler<TQuery, TResult>
    where TQuery : IQuery<TResult>
{
    Task<TResult> Handle(
        TQuery query,
        CancellationToken cancellationToken);
}
```

No vamos a construir nuestro propio framework de Mediator.

El objetivo es entender qué hay detrás del concepto.

------------------------------------------------------------------------

# 15. Usando MediatR

En proyectos .NET es común utilizar una librería de Mediator como
MediatR.

Instalación:

``` bash
dotnet add package MediatR
```

La API concreta de MediatR puede variar según la versión instalada, por
lo que en este curso nos concentraremos en el modelo conceptual y en una
implementación compatible con la versión que adoptemos en el proyecto.

El concepto fundamental permanece:

``` text
Request
   ↓
Mediator
   ↓
Handler
```

------------------------------------------------------------------------

# 16. Nuestro primer Command

Creamos:

``` text
Features/
└── Tasks/
    └── Create/
        ├── CreateTaskCommand.cs
        └── CreateTaskCommandHandler.cs
```

Command:

``` csharp
public record CreateTaskCommand(
    string Title,
    string? Description);
```

Si utilizamos MediatR:

``` csharp
using MediatR;

public record CreateTaskCommand(
    string Title,
    string? Description) : IRequest<TaskItem>;
```

El Command ahora indica que el resultado será:

``` text
TaskItem
```

------------------------------------------------------------------------

# 17. CreateTaskCommandHandler

``` csharp
using MediatR;
using TaskManagement.Api.Domain;
using TaskManagement.Api.Repositories;

public class CreateTaskCommandHandler
    : IRequestHandler<CreateTaskCommand, TaskItem>
{
    private readonly ITaskRepository _repository;

    public CreateTaskCommandHandler(
        ITaskRepository repository)
    {
        _repository = repository;
    }

    public async Task<TaskItem> Handle(
        CreateTaskCommand command,
        CancellationToken cancellationToken)
    {
        var task = new TaskItem(
            command.Title,
            command.Description);

        await _repository.AddAsync(
            task,
            cancellationToken);

        await _repository.SaveChangesAsync(
            cancellationToken);

        return task;
    }
}
```

Observa algo importante:

El Handler no conoce:

``` text
HTTP
Controller
URL
StatusCode
```

Su responsabilidad es ejecutar el caso de uso:

``` text
Create Task
```

------------------------------------------------------------------------

# 18. Controller con Mediator

Ahora el Controller cambia.

Antes:

``` csharp
_service.CreateAsync(...)
```

Ahora:

``` csharp
private readonly IMediator _mediator;

public TasksController(IMediator mediator)
{
    _mediator = mediator;
}
```

Y:

``` csharp
[HttpPost]
public async Task<ActionResult<TaskItem>> Create(
    CreateTaskRequest request,
    CancellationToken cancellationToken)
{
    var command = new CreateTaskCommand(
        request.Title,
        request.Description);

    var task = await _mediator.Send(
        command,
        cancellationToken);

    return CreatedAtAction(
        nameof(GetById),
        new { id = task.Id },
        task);
}
```

El Controller ahora hace:

``` text
HTTP
 ↓
Command
 ↓
Mediator
```

y deja de conocer la implementación del caso de uso.

------------------------------------------------------------------------

# 19. Registrando MediatR

En `Program.cs` debemos registrar los handlers.

La configuración depende de la versión de MediatR utilizada.
Conceptualmente debemos indicar:

``` text
Busca los handlers del assembly
```

Por ejemplo, en versiones modernas de MediatR se utiliza una
configuración basada en el assembly donde están los handlers.

La idea es:

``` csharp
builder.Services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(
        typeof(CreateTaskCommandHandler).Assembly);
});
```

Así el contenedor puede resolver:

``` text
IMediator
    ↓
CreateTaskCommandHandler
```

------------------------------------------------------------------------

# 20. Primera Query

Ahora implementemos:

``` text
GET /api/tasks/{id}
```

Creamos:

``` text
Features/
└── Tasks/
    └── GetById/
        ├── GetTaskByIdQuery.cs
        └── GetTaskByIdQueryHandler.cs
```

Query:

``` csharp
public record GetTaskByIdQuery(
    Guid Id) : IRequest<TaskItem?>;
```

Handler:

``` csharp
using MediatR;
using TaskManagement.Api.Domain;
using TaskManagement.Api.Repositories;

public class GetTaskByIdQueryHandler
    : IRequestHandler<GetTaskByIdQuery, TaskItem?>
{
    private readonly ITaskRepository _repository;

    public GetTaskByIdQueryHandler(
        ITaskRepository repository)
    {
        _repository = repository;
    }

    public async Task<TaskItem?> Handle(
        GetTaskByIdQuery query,
        CancellationToken cancellationToken)
    {
        return await _repository.GetByIdAsync(
            query.Id,
            cancellationToken);
    }
}
```

------------------------------------------------------------------------

# 21. Controller para la Query

``` csharp
[HttpGet("{id:guid}")]
public async Task<ActionResult<TaskItem>> GetById(
    Guid id,
    CancellationToken cancellationToken)
{
    var query = new GetTaskByIdQuery(id);

    var task = await _mediator.Send(
        query,
        cancellationToken);

    if (task is null)
    {
        return NotFound();
    }

    return Ok(task);
}
```

El flujo ahora es:

``` text
GET
 ↓
Controller
 ↓
GetTaskByIdQuery
 ↓
Mediator
 ↓
GetTaskByIdQueryHandler
 ↓
Repository
 ↓
DbContext
```

------------------------------------------------------------------------

# 22. Command y Query lado a lado

Tenemos:

``` text
CREATE

Controller
   ↓
CreateTaskCommand
   ↓
Mediator
   ↓
CreateTaskCommandHandler
   ↓
Repository
```

Y:

``` text
READ

Controller
   ↓
GetTaskByIdQuery
   ↓
Mediator
   ↓
GetTaskByIdQueryHandler
   ↓
Repository
```

Esta separación es la esencia de CQRS.

------------------------------------------------------------------------

# 23. Handler y Repository tienen responsabilidades diferentes

Esto es fundamental.

El Repository sabe:

``` text
¿Cómo acceder a los datos?
```

El Handler sabe:

``` text
¿Qué debe hacer este caso de uso?
```

Por ejemplo:

``` text
CompleteTaskCommandHandler
```

puede:

1.  Obtener la tarea.
2.  Verificar que existe.
3.  Ejecutar la transición de dominio.
4.  Guardar los cambios.
5.  Devolver el resultado.

Mientras que:

``` text
TaskRepository
```

sabe:

``` text
Cómo obtener TaskItem
Cómo persistir TaskItem
```

------------------------------------------------------------------------

# 24. Un Handler puede coordinar varios Repositories

Por ejemplo, asignar una tarea:

``` text
AssignTaskCommandHandler
       │
       ├── ITaskRepository
       │
       └── IUserRepository
```

El Handler podría:

``` text
1. Buscar tarea
2. Buscar usuario
3. Validar reglas
4. Asignar usuario
5. Guardar cambios
```

Eso es coordinación de un caso de uso.

No necesitamos meter toda esa lógica dentro de un Repository.

------------------------------------------------------------------------

# 25. ¿Y los Services?

En nuestro proyecto vamos a evitar una migración artificial.

Podemos tener tres escenarios.

### Caso 1 --- Handler suficiente

``` text
Handler
   ↓
Repository
```

Cuando el caso de uso es sencillo.

### Caso 2 --- Handler + dominio

``` text
Handler
   ↓
TaskItem
   ↓
Repository
```

Cuando la entidad contiene reglas.

### Caso 3 --- Handler + Domain/Application Service

``` text
Handler
   ↓
Service
   ↓
Repository
```

Cuando una lógica compleja necesita ser reutilizada o coordinada.

Por lo tanto:

> CQRS no prohíbe Services.

------------------------------------------------------------------------

# 26. Pipeline Behaviors

Ahora aparece una de las ventajas más interesantes del Mediator.

Supongamos que queremos ejecutar antes de todos los Handlers:

``` text
Validación
Logging
Autorización
```

Podríamos repetir:

``` text
Validate
Log
Authorize
Execute
```

en cada Handler.

Eso sería horrible.

En cambio podemos construir un pipeline:

``` text
Request
   ↓
Validation Behavior
   ↓
Logging Behavior
   ↓
Authorization Behavior
   ↓
Transaction Behavior
   ↓
Handler
```

Esto se parece conceptualmente a middleware.

------------------------------------------------------------------------

# 27. Pipeline vs Middleware

Middleware:

``` text
HTTP
 ↓
Middleware
 ↓
Controller
```

Pipeline Behavior:

``` text
Mediator Request
 ↓
Behavior
 ↓
Handler
```

Middleware trabaja alrededor del pipeline HTTP.

Behavior trabaja alrededor del pipeline de Commands/Queries.

Por eso pueden coexistir.

------------------------------------------------------------------------

# 28. Validation Behavior

Imaginemos:

``` text
CreateTaskCommand
```

con:

``` text
Title = ""
```

Podríamos tener:

``` text
Controller
   ↓
Mediator
   ↓
ValidationBehavior
   ↓
❌ Handler no ejecutado
```

El Handler nunca recibe una petición inválida.

Esto centraliza la validación transversal.

En el próximo módulo estudiaremos DTOs y validación en profundidad.

------------------------------------------------------------------------

# 29. Logging Behavior

Podemos registrar:

``` text
Request:
CreateTaskCommand

Started:
10:30:01

Finished:
10:30:02

Duration:
120ms
```

sin modificar cada Handler.

Conceptualmente:

``` text
LoggingBehavior
      ↓
next()
      ↓
Handler
```

------------------------------------------------------------------------

# 30. Authorization Behavior

Podríamos tener:

``` text
CompleteTaskCommand
```

y verificar antes del Handler:

``` text
¿El usuario tiene permiso?
```

Flujo:

``` text
Command
   ↓
AuthorizationBehavior
   ↓
¿Permitido?
   │
   ├── No → error
   │
   └── Sí
        ↓
      Handler
```

La autorización específica del caso de uso puede complementarse con
ASP.NET Core Authorization.

------------------------------------------------------------------------

# 31. Transaction Behavior

Para determinados Commands podemos querer:

``` text
BEGIN TRANSACTION
       ↓
Handler
       ↓
COMMIT
```

Si algo falla:

``` text
ROLLBACK
```

Un Behavior puede centralizar esta lógica.

No significa que todos los Commands deban necesariamente abrir una
transacción explícita.

Es una herramienta para los casos donde la operación lo necesita.

------------------------------------------------------------------------

# 32. Orden de Behaviors

Podríamos terminar con:

``` text
Request
   ↓
Logging
   ↓
Validation
   ↓
Authorization
   ↓
Transaction
   ↓
Handler
```

El orden puede ser importante.

Por ejemplo, no tiene mucho sentido abrir una transacción de base de
datos si la request ni siquiera pasa la validación.

La configuración exacta depende de la implementación utilizada.

------------------------------------------------------------------------

# 33. CQRS + EF Core

CQRS no reemplaza EF Core.

Seguimos teniendo:

``` text
Handler
   ↓
Repository
   ↓
DbContext
   ↓
SQL Server
```

La diferencia está en cómo organizamos los casos de uso.

Antes:

``` text
TaskService
```

Ahora:

``` text
CreateTaskHandler
GetTaskByIdHandler
GetTasksHandler
CompleteTaskHandler
DeleteTaskHandler
```

------------------------------------------------------------------------

# 34. Queries y optimización

Una ventaja importante de separar Queries es que podemos optimizar las
lecturas independientemente.

Por ejemplo, en lugar de obtener:

``` csharp
TaskItem
```

completo, una Query podría proyectar directamente:

``` csharp
public record TaskListItem(
    Guid Id,
    string Title,
    TaskStatus Status);
```

Y consultar:

``` csharp
var result = await _db.Tasks
    .AsNoTracking()
    .Select(x => new TaskListItem(
        x.Id,
        x.Title,
        x.Status))
    .ToListAsync(cancellationToken);
```

Esto reduce los datos que necesitamos cargar.

------------------------------------------------------------------------

# 35. Read Models

En aplicaciones más grandes puede aparecer:

``` text
Write Model
     ↓
Domain entities

Read Model
     ↓
Optimized projections
```

Por ejemplo:

``` text
Task
 ├── Id
 ├── Title
 ├── Description
 ├── Status
 ├── CreatedAt
 └── ...
```

Pero una pantalla quizá solamente necesita:

``` text
TaskListItem
 ├── Id
 ├── Title
 ├── Status
 └── AssignedUserName
```

La Query puede tener un modelo específico para lectura.

Esto evita cargar información innecesaria.

------------------------------------------------------------------------

# 36. CQRS avanzado: bases separadas

En sistemas de mayor escala podemos llegar a:

``` text
                  Commands
                     ↓
                Write Model
                     ↓
                Write DB
                     │
                     │ events
                     ↓
                Read Model
                     ↓
                  Read DB
```

Por ejemplo:

``` text
SQL Server
   ↓
Events
   ↓
Projection
   ↓
Read Database
```

Esto permite optimizar cada lado de forma independiente.

Pero tiene un coste importante:

-   mayor complejidad;
-   consistencia eventual;
-   sincronización;
-   observabilidad;
-   recuperación ante errores;
-   infraestructura adicional.

No vamos a implementarlo en nuestro proyecto todavía.

------------------------------------------------------------------------

# 37. CQRS y Event-Driven Architecture

CQRS tampoco significa automáticamente Event Sourcing.

Podemos tener:

``` text
Command
   ↓
Handler
   ↓
Database
```

sin eventos.

También podemos evolucionar hacia:

``` text
Command
   ↓
Handler
   ↓
Domain Event
   ↓
Event Bus
   ↓
Consumers
```

Y eso nos lleva a arquitecturas event-driven.

Son conceptos relacionados, pero diferentes.

------------------------------------------------------------------------

# 38. Estructura del proyecto

Nuestro proyecto puede evolucionar hacia:

``` text
TaskManagement.Api/
│
├── Controllers/
│   └── TasksController.cs
│
├── Domain/
│   └── TaskItem.cs
│
├── Data/
│   ├── TaskManagementDbContext.cs
│   └── Configurations/
│       └── TaskItemConfiguration.cs
│
├── Repositories/
│   ├── ITaskRepository.cs
│   └── TaskRepository.cs
│
├── Features/
│   └── Tasks/
│       ├── Create/
│       │   ├── CreateTaskCommand.cs
│       │   └── CreateTaskCommandHandler.cs
│       │
│       ├── GetById/
│       │   ├── GetTaskByIdQuery.cs
│       │   └── GetTaskByIdQueryHandler.cs
│       │
│       ├── Delete/
│       │   ├── DeleteTaskCommand.cs
│       │   └── DeleteTaskCommandHandler.cs
│       │
│       └── Complete/
│           ├── CompleteTaskCommand.cs
│           └── CompleteTaskCommandHandler.cs
│
├── Options/
│   └── TaskManagementOptions.cs
│
└── Program.cs
```

Este estilo agrupa por Feature/Caso de Uso.

------------------------------------------------------------------------

# 39. ¿Por qué Feature folders?

Con una arquitectura puramente por tipo podríamos tener:

``` text
Controllers/
Services/
Repositories/
Commands/
Queries/
Handlers/
DTOs/
```

En una aplicación grande esto puede producir carpetas enormes.

Feature folders permiten:

``` text
Tasks/
   Create/
   GetById/
   Delete/
   Complete/
```

Todo lo relacionado con una operación está cerca.

No significa que exista una única estructura correcta.

Es una decisión arquitectónica.

------------------------------------------------------------------------

# 40. Refactor de nuestro proyecto

Nuestro objetivo no será eliminar todo de golpe.

Vamos a migrar progresivamente.

Primero:

``` text
CreateTask
```

Luego:

``` text
GetTaskById
```

Después:

``` text
GetTasks
DeleteTask
CompleteTask
UpdateTask
```

Así podemos comparar:

``` text
Service-based
```

contra:

``` text
CQRS-based
```

y entender qué ganamos y qué complejidad agregamos.

------------------------------------------------------------------------

# 41. Ejercicio práctico 1 --- Create Command

Implementa:

``` text
CreateTaskCommand
CreateTaskCommandHandler
```

El Handler debe utilizar:

``` text
ITaskRepository
```

Debe:

1.  Crear `TaskItem`.
2.  Persistirlo.
3.  Devolver la tarea creada.

------------------------------------------------------------------------

# 42. Ejercicio práctico 2 --- GetById Query

Implementa:

``` text
GetTaskByIdQuery
GetTaskByIdQueryHandler
```

Debe utilizar:

``` text
ITaskRepository
```

y devolver:

``` text
TaskItem?
```

El Controller debe traducir:

``` text
null → 404
resultado → 200
```

------------------------------------------------------------------------

# 43. Ejercicio práctico 3 --- Delete Command

Implementa:

``` text
DeleteTaskCommand
DeleteTaskCommandHandler
```

Flujo:

``` text
Controller
    ↓
Mediator
    ↓
DeleteTaskCommand
    ↓
Handler
    ↓
Repository
```

El Handler debe determinar si la tarea existe.

------------------------------------------------------------------------

# 44. Ejercicio práctico 4 --- Complete Command

Implementa:

``` text
CompleteTaskCommand
CompleteTaskCommandHandler
```

El flujo:

``` text
Get task
   ↓
Check existence
   ↓
Task.Complete()
   ↓
SaveChanges
```

La regla de transición debe permanecer en el dominio:

``` csharp
task.Complete();
```

No en el Controller.

No en el Repository.

------------------------------------------------------------------------

# 45. Ejercicio práctico 5 --- GetTasks Query

Implementa:

``` text
GetTasksQuery
GetTasksQueryHandler
```

Debe soportar:

``` text
page
pageSize
status
```

El Repository debe realizar la consulta en SQL utilizando:

``` text
Where
OrderBy
Skip
Take
```

No debes cargar toda la tabla en memoria.

------------------------------------------------------------------------

# 46. Ejercicio práctico 6 --- Primer Behavior

Implementa conceptualmente un:

``` text
LoggingBehavior
```

El objetivo es registrar:

``` text
Request type
Start time
End time
Duration
```

No necesitas construir todavía una solución de logging avanzada.

La finalidad es entender:

``` text
Behavior
   ↓
next()
   ↓
Handler
```

------------------------------------------------------------------------

# 47. Ejercicio práctico 7 --- Comparación arquitectónica

Toma:

``` text
CreateTask
```

y documenta las dos implementaciones.

### Antes

``` text
Controller
    ↓
TaskService
    ↓
Repository
```

### Después

``` text
Controller
    ↓
Mediator
    ↓
CreateTaskCommand
    ↓
CreateTaskCommandHandler
    ↓
Repository
```

Escribe qué ventajas y desventajas encuentras.

Esta comparación es más importante que memorizar CQRS.

------------------------------------------------------------------------

# 48. Preguntas de comprensión

Antes de continuar, deberías poder responder:

1.  ¿Qué significa CQRS?
2.  ¿Qué diferencia existe entre Command y Query?
3.  ¿Un Command siempre necesita una base de datos separada?
4.  ¿Qué es un Handler?
5.  ¿Qué responsabilidad tiene el Mediator?
6.  ¿Por qué el Controller no debería conocer directamente el Handler?
7.  ¿Qué diferencia existe entre Handler y Repository?
8.  ¿Un proyecto CQRS puede seguir utilizando Services?
9.  ¿Qué es un Pipeline Behavior?
10. ¿Qué diferencia existe entre Middleware y Behavior?
11. ¿Qué es un Read Model?
12. ¿CQRS implica Event Sourcing?
13. ¿CQRS implica Event-Driven Architecture?
14. ¿Cuándo puede CQRS ser demasiado complejo para un proyecto?
15. ¿Por qué una Query puede utilizar `AsNoTracking()` y una proyección?
16. ¿Por qué no debemos crear un Handler que simplemente delegue en un
    Service sin aportar ninguna separación útil?

------------------------------------------------------------------------

# 49. Un criterio importante: CQRS no es magia

Es fácil caer en:

``` text
Command
Query
Handler
Mediator
Behavior
Repository
Service
UnitOfWork
Factory
...
```

y terminar con una arquitectura enorme para una operación sencilla.

Por ejemplo:

``` text
GET /health
```

no necesita:

``` text
HealthQuery
HealthQueryHandler
HealthRepository
HealthService
HealthBehavior
HealthMapper
```

La arquitectura debe responder al problema.

CQRS es especialmente interesante cuando:

-   hay muchos casos de uso;
-   lectura y escritura tienen necesidades diferentes;
-   la aplicación tiene lógica de negocio significativa;
-   necesitamos pipelines transversales;
-   queremos organizar operaciones por feature;
-   distintos casos de uso evolucionan independientemente.

------------------------------------------------------------------------

# 50. El criterio que vamos a utilizar en nuestro proyecto

Nuestro Task Management API está creciendo.

Tenemos:

``` text
Users
Projects
Tasks
Comments
Assignments
Statuses
Authentication
Authorization
```

Por eso CQRS empieza a tener sentido.

Pero vamos a mantenerlo simple:

``` text
Command
   ↓
Handler
   ↓
Repository
```

y:

``` text
Query
   ↓
Handler
   ↓
Repository
```

Sin introducir todavía:

``` text
Read DB separada
Event Bus
Event Sourcing
Kafka
Service Bus
Microservices
Distributed Transactions
```

Primero debemos dominar el patrón básico.

------------------------------------------------------------------------

# 51. Estado del proyecto

Después del módulo 8, conceptualmente tenemos:

``` text
                         HTTP
                          │
                          ↓
                  TasksController
                          │
                          ↓
                       Mediator
                     ↙         ↘
                    ↓           ↓
               Commands       Queries
                    ↓           ↓
                 Handler      Handler
                    │           │
                    └─────┬─────┘
                          ↓
                   ITaskRepository
                          ↓
                    TaskRepository
                          ↓
                       DbContext
                          ↓
                      SQL Server
```

Y alrededor del Mediator:

``` text
              ┌─────────────────────┐
              │ Pipeline Behaviors   │
              │                     │
              │ Validation          │
              │ Logging             │
              │ Authorization       │
              │ Transactions        │
              └─────────────────────┘
```

------------------------------------------------------------------------

# 52. Conexión con lo que probablemente ves en tu trabajo

El flujo que estamos construyendo explica la arquitectura:

``` text
Controller
    ↓
Mediator
    ↓
Query / Command
    ↓
Handler
    ↓
Service / Domain
    ↓
Repository
    ↓
Database
```

En algunos proyectos encontrarás:

``` text
Controller
    ↓
Mediator
    ↓
Handler
    ↓
Repository
```

En otros:

``` text
Controller
    ↓
Mediator
    ↓
Handler
    ↓
Service
    ↓
Repository
```

Ambos pueden ser correctos.

La pregunta importante no es:

> "¿Dónde debe existir obligatoriamente un Service?"

La pregunta es:

> "¿Dónde está implementada la responsabilidad que este caso de uso
> necesita?"

------------------------------------------------------------------------

# 53. Resumen del módulo

CQRS separa conceptualmente:

``` text
READ
 ↓
Query

WRITE
 ↓
Command
```

Mediator permite desacoplar:

``` text
Controller
    ↓
Request
    ↓
Handler
```

Handler representa la ejecución de un caso de uso.

Repository representa el acceso a datos.

Pipeline Behaviors permiten agregar comportamientos transversales:

``` text
Validation
Logging
Authorization
Transactions
```

La arquitectura resultante es:

``` text
Controller
    ↓
Mediator
    ↓
Command / Query
    ↓
Handler
    ↓
Repository
    ↓
DbContext
    ↓
Database
```

Y la idea más importante del módulo es:

> **CQRS no consiste en agregar Commands, Queries y Handlers porque sí.
> Consiste en separar y organizar claramente las operaciones de lectura
> y escritura para que los casos de uso puedan evolucionar de forma
> independiente.**

En el siguiente módulo abordaremos **DTOs + Validation + Errors**, donde
veremos cómo evitar exponer directamente nuestras entidades de dominio a
través de la API, cómo validar Commands/Requests y cómo construir
respuestas de error consistentes.
