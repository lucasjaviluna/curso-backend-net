# Módulo 7 --- Services + Repository

## 1. Objetivos

Al terminar este módulo podrás:

-   Entender por qué aparece un Repository en una aplicación .NET.
-   Separar responsabilidades entre Controller, Service y acceso a
    datos.
-   Crear una abstracción `ITaskRepository`.
-   Implementar `TaskRepository` usando Entity Framework Core.
-   Hacer que `TaskService` dependa del Repository y no directamente de
    `DbContext`.
-   Registrar Repository y Service mediante Dependency Injection.
-   Entender cuándo un Repository aporta valor y cuándo puede ser
    innecesario.
-   Evitar abstraer EF Core de forma artificial.
-   Preparar la arquitectura para introducir CQRS/Mediator en el módulo
    siguiente.

La arquitectura evolucionará de:

``` text
Controller
    ↓
ITaskService
    ↓
TaskService
    ↓
DbContext
    ↓
SQL Server
```

a:

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
    ↓
SQL Server
```

------------------------------------------------------------------------

# 2. ¿Por qué necesitamos un Repository?

En el módulo anterior nuestro Service hacía dos cosas:

1.  Contenía reglas de negocio.
2.  Accedía directamente a Entity Framework Core.

Por ejemplo:

``` csharp
public class TaskService : ITaskService
{
    private readonly TaskManagementDbContext _db;

    public TaskService(TaskManagementDbContext db)
    {
        _db = db;
    }

    public async Task<TaskItem?> GetByIdAsync(Guid id)
    {
        return await _db.Tasks
            .FirstOrDefaultAsync(x => x.Id == id);
    }
}
```

Esto funciona perfectamente.

Pero con el crecimiento de la aplicación podemos terminar con Services
llenos de consultas:

``` text
TaskService
 ├── GetById
 ├── GetAll
 ├── Search
 ├── Filter
 ├── Pagination
 ├── Create
 ├── Update
 ├── Delete
 └── muchas consultas más
```

Entonces aparece una pregunta:

> ¿El Service debería saber cómo se consulta SQL?

Idealmente, el Service debería preocuparse principalmente por:

``` text
¿Qué debe hacer la aplicación?
```

Mientras que el Repository se preocupa por:

``` text
¿Cómo obtengo o persisto los datos?
```

Ahí aparece el patrón Repository.

------------------------------------------------------------------------

# 3. ¿Qué es Repository?

Un Repository es una abstracción sobre el acceso a datos.

Conceptualmente:

``` text
Service
   │
   │ "Dame la tarea 123"
   ↓
Repository
   │
   │ "Voy a consultar la base de datos"
   ↓
DbContext
   │
   ↓
SQL Server
```

El Service no necesita saber si el Repository utiliza:

-   EF Core
-   SQL directo
-   otra fuente de datos
-   una API externa
-   una implementación fake para tests

Solo conoce el contrato.

Por ejemplo:

``` csharp
public interface ITaskRepository
{
    Task<TaskItem?> GetByIdAsync(
        Guid id,
        CancellationToken cancellationToken = default);
}
```

El Service puede utilizarlo:

``` csharp
var task = await _repository.GetByIdAsync(id);
```

No necesita saber que internamente existe:

``` csharp
_db.Tasks.FirstOrDefaultAsync(...)
```

------------------------------------------------------------------------

# 4. Repository no significa "una clase por tabla" obligatoriamente

Una implementación muy común es:

``` text
ITaskRepository
TaskRepository
```

porque estamos trabajando con tareas.

Pero no hay una regla que diga:

> Cada tabla debe tener obligatoriamente un Repository.

El Repository debería representar una responsabilidad de acceso a datos
que tenga sentido para el dominio o aplicación.

Por ejemplo:

``` text
ITaskRepository
IProjectRepository
IUserRepository
ICommentRepository
```

puede tener sentido.

Pero crear interfaces y clases para absolutamente todo sin necesidad
puede generar mucho código.

La pregunta correcta es:

> ¿Esta abstracción está ayudando a separar responsabilidades o
> solamente estoy agregando archivos?

------------------------------------------------------------------------

# 5. Repository vs DbContext

Esto es importante.

EF Core ya implementa conceptos similares a:

-   Repository
-   Unit of Work

Por ejemplo:

``` csharp
_db.Tasks
```

se comporta de manera similar a una colección/repository.

Y:

``` csharp
_db.SaveChangesAsync()
```

representa la confirmación de los cambios.

Por eso muchas aplicaciones pequeñas o medianas utilizan directamente:

``` text
Service
   ↓
DbContext
```

y funcionan perfectamente.

Entonces:

> Repository no es obligatorio solo porque estamos usando Clean
> Architecture o una arquitectura empresarial.

En nuestro proyecto lo introduciremos porque queremos aprender la
separación de responsabilidades y porque será útil como frontera de
acceso a datos.

------------------------------------------------------------------------

# 6. Separación de responsabilidades

Vamos a definir claramente qué hace cada capa.

## Controller

Responsabilidad:

``` text
HTTP
```

Por ejemplo:

``` text
GET /api/tasks/123
```

El Controller recibe la petición y delega.

No debería contener consultas SQL.

------------------------------------------------------------------------

## Service

Responsabilidad:

``` text
Reglas de negocio / casos de uso
```

Por ejemplo:

``` text
Completar una tarea
Validar una transición
Asignar una tarea
Crear una tarea
```

El Service coordina operaciones.

------------------------------------------------------------------------

## Repository

Responsabilidad:

``` text
Persistencia / acceso a datos
```

Por ejemplo:

``` text
Buscar tarea
Guardar tarea
Eliminar tarea
Consultar tareas
```

------------------------------------------------------------------------

## DbContext

Responsabilidad:

``` text
Comunicación con EF Core y la base de datos
```

------------------------------------------------------------------------

# 7. Nuestro primer Repository

Vamos a crear:

``` text
Repositories/
├── ITaskRepository.cs
└── TaskRepository.cs
```

## ITaskRepository

``` csharp
using TaskManagement.Api.Domain;

namespace TaskManagement.Api.Repositories;

public interface ITaskRepository
{
    Task<TaskItem?> GetByIdAsync(
        Guid id,
        CancellationToken cancellationToken = default);

    Task<List<TaskItem>> GetAllAsync(
        CancellationToken cancellationToken = default);

    Task AddAsync(
        TaskItem task,
        CancellationToken cancellationToken = default);

    void Update(TaskItem task);

    void Delete(TaskItem task);

    Task<bool> ExistsAsync(
        Guid id,
        CancellationToken cancellationToken = default);

    Task SaveChangesAsync(
        CancellationToken cancellationToken = default);
}
```

Observa que el Service ahora puede trabajar con operaciones de
persistencia sin conocer EF Core.

------------------------------------------------------------------------

# 8. Implementación del Repository

Creamos:

``` text
Repositories/TaskRepository.cs
```

``` csharp
using Microsoft.EntityFrameworkCore;
using TaskManagement.Api.Data;
using TaskManagement.Api.Domain;

namespace TaskManagement.Api.Repositories;

public class TaskRepository : ITaskRepository
{
    private readonly TaskManagementDbContext _db;

    public TaskRepository(TaskManagementDbContext db)
    {
        _db = db;
    }

    public async Task<TaskItem?> GetByIdAsync(
        Guid id,
        CancellationToken cancellationToken = default)
    {
        return await _db.Tasks
            .FirstOrDefaultAsync(
                x => x.Id == id,
                cancellationToken);
    }

    public async Task<List<TaskItem>> GetAllAsync(
        CancellationToken cancellationToken = default)
    {
        return await _db.Tasks
            .ToListAsync(cancellationToken);
    }

    public async Task AddAsync(
        TaskItem task,
        CancellationToken cancellationToken = default)
    {
        await _db.Tasks.AddAsync(
            task,
            cancellationToken);
    }

    public void Update(TaskItem task)
    {
        _db.Tasks.Update(task);
    }

    public void Delete(TaskItem task)
    {
        _db.Tasks.Remove(task);
    }

    public async Task<bool> ExistsAsync(
        Guid id,
        CancellationToken cancellationToken = default)
    {
        return await _db.Tasks
            .AnyAsync(
                x => x.Id == id,
                cancellationToken);
    }

    public async Task SaveChangesAsync(
        CancellationToken cancellationToken = default)
    {
        await _db.SaveChangesAsync(cancellationToken);
    }
}
```

Ahora toda esta lógica:

``` csharp
_db.Tasks
```

está concentrada en el Repository.

------------------------------------------------------------------------

# 9. Refactorizando TaskService

Antes:

``` text
TaskService
   ↓
DbContext
```

Ahora:

``` text
TaskService
   ↓
ITaskRepository
```

El Service pasa a ser:

``` csharp
using TaskManagement.Api.Domain;
using TaskManagement.Api.Repositories;

namespace TaskManagement.Api.Services;

public class TaskService : ITaskService
{
    private readonly ITaskRepository _repository;

    public TaskService(ITaskRepository repository)
    {
        _repository = repository;
    }

    public async Task<TaskItem?> GetByIdAsync(
        Guid id,
        CancellationToken cancellationToken = default)
    {
        return await _repository.GetByIdAsync(
            id,
            cancellationToken);
    }

    public async Task<List<TaskItem>> GetAllAsync(
        CancellationToken cancellationToken = default)
    {
        return await _repository.GetAllAsync(
            cancellationToken);
    }

    public async Task<TaskItem> CreateAsync(
        string title,
        string? description,
        CancellationToken cancellationToken = default)
    {
        var task = new TaskItem(
            title,
            description);

        await _repository.AddAsync(
            task,
            cancellationToken);

        await _repository.SaveChangesAsync(
            cancellationToken);

        return task;
    }

    public async Task<bool> DeleteAsync(
        Guid id,
        CancellationToken cancellationToken = default)
    {
        var task = await _repository.GetByIdAsync(
            id,
            cancellationToken);

        if (task is null)
        {
            return false;
        }

        _repository.Delete(task);

        await _repository.SaveChangesAsync(
            cancellationToken);

        return true;
    }
}
```

Ahora existe una separación clara:

``` text
TaskService
    │
    │ reglas / casos de uso
    ↓
ITaskRepository
    │
    │ persistencia
    ↓
TaskRepository
    │
    ↓
DbContext
```

------------------------------------------------------------------------

# 10. Dependency Injection

Tenemos que registrar el Repository.

En `Program.cs`:

``` csharp
builder.Services.AddScoped<ITaskRepository, TaskRepository>();

builder.Services.AddScoped<ITaskService, TaskService>();
```

La aplicación podrá resolver:

``` text
ITaskService
    ↓
TaskService
    ↓
ITaskRepository
    ↓
TaskRepository
    ↓
TaskManagementDbContext
```

ASP.NET Core construirá automáticamente toda la cadena.

Esto es justamente una de las ventajas que vimos en el módulo 4.

------------------------------------------------------------------------

# 11. ¿Por qué Scoped?

Recordemos:

``` csharp
AddSingleton
AddScoped
AddTransient
```

Nuestro:

``` csharp
DbContext
```

normalmente es Scoped.

Por lo tanto tiene sentido que Repository y Service también sean Scoped:

``` csharp
builder.Services.AddScoped<ITaskRepository, TaskRepository>();
builder.Services.AddScoped<ITaskService, TaskService>();
```

La idea es:

``` text
HTTP Request
     │
     ├── TaskService
     │
     ├── TaskRepository
     │
     └── DbContext

     ↓

HTTP Response
```

Durante una request se comparte el mismo contexto de trabajo.

------------------------------------------------------------------------

# 12. Repository y SaveChanges

Hay una decisión de diseño interesante.

Podemos hacer:

``` csharp
_repository.AddAsync(task);
_repository.SaveChangesAsync();
```

o hacer que el Repository solamente prepare los cambios y que otro
componente confirme la transacción.

Por ahora utilizaremos:

``` csharp
AddAsync()
Update()
Delete()
SaveChangesAsync()
```

porque es sencillo y nos permite entender el flujo.

Más adelante, cuando estudiemos CQRS y arquitectura avanzada, veremos
situaciones donde puede tener sentido separar:

``` text
operaciones
    ↓
transacción
    ↓
commit
```

No vamos a introducir todavía un Unit of Work adicional solo por agregar
otra abstracción.

------------------------------------------------------------------------

# 13. Consultas más complejas

Nuestro Repository puede encapsular consultas.

Por ejemplo:

``` csharp
public async Task<List<TaskItem>> GetByStatusAsync(
    TaskStatus status,
    CancellationToken cancellationToken = default)
{
    return await _db.Tasks
        .Where(x => x.Status == status)
        .OrderByDescending(x => x.CreatedAt)
        .ToListAsync(cancellationToken);
}
```

La interfaz:

``` csharp
Task<List<TaskItem>> GetByStatusAsync(
    TaskStatus status,
    CancellationToken cancellationToken = default);
```

El Service solamente necesita:

``` csharp
var tasks = await _repository.GetByStatusAsync(
    TaskStatus.InProgress,
    cancellationToken);
```

La consulta SQL sigue siendo responsabilidad del Repository.

------------------------------------------------------------------------

# 14. ¿Y la paginación?

Podemos mover también las consultas de paginación.

Por ejemplo:

``` csharp
public async Task<List<TaskItem>> GetPageAsync(
    int page,
    int pageSize,
    CancellationToken cancellationToken = default)
{
    return await _db.Tasks
        .OrderByDescending(x => x.CreatedAt)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .ToListAsync(cancellationToken);
}
```

Esto mantiene la consulta cerca de la infraestructura de persistencia.

Sin embargo, hay que tener cuidado.

Si empezamos a crear métodos para absolutamente todas las combinaciones:

``` text
GetByStatus
GetByStatusAndUser
GetByStatusAndDate
GetByStatusAndDateAndUser
GetByStatusAndDateAndUserAndProject
...
```

el Repository puede convertirse en un monstruo.

Esto nos lleva a una decisión arquitectónica importante.

------------------------------------------------------------------------

# 15. ¿Debe el Repository devolver IQueryable?

Una opción sería:

``` csharp
IQueryable<TaskItem> Query();
```

y después:

``` csharp
_repository.Query()
    .Where(...)
    .OrderBy(...)
    .Skip(...)
    .Take(...);
```

Esto parece flexible.

Pero tiene un problema.

Ahora el Service conoce:

``` text
IQueryable
EF Core
LINQ
forma de construir consultas
```

Y el Repository deja de ser una verdadera frontera.

Terminamos con:

``` text
Service
   ↓
IRepository
   ↓
IQueryable
   ↓
EF Core
```

Es decir, EF Core se está filtrando hacia arriba.

Por eso:

> No debemos devolver `IQueryable` automáticamente solo porque sea
> cómodo.

Puede ser válido en determinados diseños, pero debe ser una decisión
consciente.

Para nuestro proyecto mantendremos inicialmente métodos de consulta
explícitos.

------------------------------------------------------------------------

# 16. ¿Repository genérico?

Probablemente hayas visto algo como:

``` csharp
public interface IRepository<T>
{
    Task<T?> GetByIdAsync(Guid id);
    Task<List<T>> GetAllAsync();
    Task AddAsync(T entity);
    void Update(T entity);
    void Delete(T entity);
}
```

Y luego:

``` csharp
Repository<TaskItem>
Repository<Project>
Repository<User>
```

Parece elegante.

Pero tenemos que preguntarnos:

> ¿Qué comportamiento común estamos abstrayendo realmente?

Las entidades pueden tener consultas completamente diferentes.

Por ejemplo:

``` text
Task
 ├── FilterByStatus
 ├── AssignedTo
 └── DueDate

Project
 ├── ActiveProjects
 └── Owner

User
 ├── Email
 └── Roles
```

Un Repository genérico puede terminar agregando métodos que no
representan necesidades reales.

Por ahora utilizaremos:

``` text
ITaskRepository
TaskRepository
```

porque expresa claramente lo que necesita nuestro dominio.

------------------------------------------------------------------------

# 17. Repository y lógica de negocio

Este límite es fundamental.

Supongamos que tenemos:

> Una tarea completada no puede volver a Pending.

¿Dónde debería estar?

No en:

``` text
TaskRepository
```

porque eso es una regla de negocio.

Debe estar en:

``` text
TaskService
```

o eventualmente en el propio dominio.

El Repository debería preocuparse por:

``` text
¿Cómo guardo la tarea?
¿Cómo la recupero?
¿Cómo la elimino?
```

No por:

``` text
¿Está permitido completar esta tarea?
¿Puede cambiar de estado?
¿Puede este usuario hacerlo?
```

Estas últimas son responsabilidades de negocio/autorización.

------------------------------------------------------------------------

# 18. Ejemplo: completar una tarea

Supongamos que agregamos:

``` csharp
public async Task<bool> CompleteAsync(
    Guid id,
    CancellationToken cancellationToken = default)
{
    var task = await _repository.GetByIdAsync(
        id,
        cancellationToken);

    if (task is null)
    {
        return false;
    }

    if (task.Status == TaskStatus.Completed)
    {
        return true;
    }

    task.Complete();

    await _repository.SaveChangesAsync(
        cancellationToken);

    return true;
}
```

Observa la separación.

El Service decide:

``` text
¿qué significa completar?
```

El dominio ejecuta:

``` csharp
task.Complete();
```

El Repository ejecuta:

``` text
persistir cambios
```

------------------------------------------------------------------------

# 19. Controller después del refactor

El Controller prácticamente no cambia.

``` csharp
[ApiController]
[Route("api/[controller]")]
public class TasksController : ControllerBase
{
    private readonly ITaskService _service;

    public TasksController(ITaskService service)
    {
        _service = service;
    }

    [HttpGet("{id:guid}")]
    public async Task<ActionResult<TaskItem>> GetById(
        Guid id,
        CancellationToken cancellationToken)
    {
        var task = await _service.GetByIdAsync(
            id,
            cancellationToken);

        if (task is null)
        {
            return NotFound();
        }

        return Ok(task);
    }
}
```

Esto es importante.

El Controller no sabe:

``` text
EF Core
DbContext
SQL
Repository
```

Solo conoce:

``` text
ITaskService
```

------------------------------------------------------------------------

# 20. La arquitectura completa

Después del módulo 7 tenemos:

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
                  ITaskRepository
                          │
                          ↓
                   TaskRepository
                          │
                          ↓
                     DbContext
                          │
                          ↓
                     SQL Server
```

Cada componente tiene una responsabilidad.

  Componente   Responsabilidad
  ------------ ------------------------
  Controller   HTTP
  Service      Casos de uso / negocio
  Repository   Persistencia
  DbContext    EF Core
  SQL Server   Persistencia física

------------------------------------------------------------------------

# 21. Comparación con frontend

Como desarrollador Angular/React, puedes verlo así.

En frontend:

``` text
Component
   ↓
Service
   ↓
HttpClient
   ↓
Backend
```

En nuestro backend:

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

La idea es similar:

> Cada capa conoce solamente lo que necesita conocer de la siguiente
> capa.

Pero recuerda que no son equivalencias exactas.

------------------------------------------------------------------------

# 22. Repository y testing

Una ventaja potencial de esta separación es poder sustituir la
implementación.

Por ejemplo:

``` csharp
public class FakeTaskRepository : ITaskRepository
{
    private readonly List<TaskItem> _tasks = [];

    // implementación para tests
}
```

Entonces:

``` text
Test
  ↓
TaskService
  ↓
FakeTaskRepository
```

en lugar de:

``` text
Test
  ↓
TaskService
  ↓
SQL Server
```

Esto puede hacer algunos tests más simples y rápidos.

Sin embargo:

> No debemos crear un FakeRepository únicamente para justificar el
> Repository.

Los tests y la facilidad de sustitución son beneficios; la razón
principal es separar responsabilidades.

------------------------------------------------------------------------

# 23. Un error común: Repository demasiado inteligente

Un Repository no debería convertirse en una segunda capa de negocio.

Mala idea:

``` csharp
public async Task<bool> CompleteTaskAsync(Guid id)
{
    var task = await ...;

    if (task.Status == TaskStatus.Pending)
    {
        ...
    }

    ...
}
```

¿Por qué?

Porque:

``` text
CompleteTask
```

representa una operación de negocio.

Sería mejor:

``` text
TaskService
   ↓
Get task
   ↓
Task.Complete()
   ↓
Repository.SaveChanges()
```

El Repository persiste.

El Service coordina.

El dominio protege sus reglas.

------------------------------------------------------------------------

# 24. Repository demasiado genérico

Otro error:

``` csharp
public interface IRepository<T>
{
    Task<T?> GetAsync(...);
    Task<T?> FindAsync(...);
    Task<T?> QueryAsync(...);
    Task<List<T>> SearchAsync(...);
    Task<List<T>> FilterAsync(...);
    ...
}
```

Puede parecer reutilizable.

Pero muchas veces termina creando una abstracción que no representa el
dominio.

La reutilización no siempre significa:

> menos código.

También puede significar:

> una abstracción que representa correctamente una responsabilidad
> común.

------------------------------------------------------------------------

# 25. ¿Cuándo usar Repository?

Repository puede ser útil cuando:

### 1. Queremos una frontera clara de persistencia

``` text
Service
   ↓
Repository
   ↓
Persistence
```

### 2. Tenemos consultas complejas

El Repository puede encapsularlas.

### 3. Tenemos múltiples fuentes de datos

Por ejemplo:

``` text
ITaskRepository
   ├── SqlTaskRepository
   └── ExternalTaskRepository
```

### 4. Queremos aislar determinados detalles de infraestructura

El Service no necesita conocer esos detalles.

### 5. La arquitectura del proyecto requiere explícitamente esa separación

En empresas grandes puede formar parte de los estándares
arquitectónicos.

------------------------------------------------------------------------

# 26. ¿Cuándo puede no ser necesario?

Para una API sencilla:

``` text
Controller
   ↓
Service
   ↓
DbContext
```

puede ser suficiente.

Especialmente cuando:

-   hay poca lógica;
-   las consultas son simples;
-   el proyecto es pequeño;
-   EF Core ya cubre perfectamente la necesidad;
-   el Repository solamente repite métodos de `DbSet`.

Por ejemplo:

``` csharp
_repository.GetByIdAsync(id);
```

que internamente solo hace:

``` csharp
_db.Tasks.FirstOrDefaultAsync(x => x.Id == id);
```

no siempre aporta mucho valor.

La arquitectura debe justificarse por el problema que resuelve.

------------------------------------------------------------------------

# 27. Regla práctica

Cuando evalúes un Repository, pregúntate:

``` text
¿Qué problema concreto me está resolviendo?
```

Si la respuesta es:

> "Porque así lo hace Clean Architecture."

No es suficiente.

Una mejor respuesta sería:

> "Centraliza nuestras consultas complejas y evita que los Services
> conozcan detalles de persistencia."

Eso sí es una justificación arquitectónica.

------------------------------------------------------------------------

# 28. Ejercicio práctico 1 --- Crear el Repository

Refactoriza completamente `TaskService`.

Debe quedar:

``` text
TaskService
    ↓
ITaskRepository
    ↓
TaskRepository
    ↓
TaskManagementDbContext
```

Implementa:

``` csharp
GetByIdAsync
GetAllAsync
AddAsync
Update
Delete
ExistsAsync
SaveChangesAsync
```

------------------------------------------------------------------------

# 29. Ejercicio práctico 2 --- Filtrar por estado

Agrega:

``` csharp
GetByStatusAsync(...)
```

Debe permitir:

``` http
GET /api/tasks?status=InProgress
```

El Controller delega al Service.

El Service delega al Repository.

El Repository construye la consulta.

------------------------------------------------------------------------

# 30. Ejercicio práctico 3 --- Paginación

Implementa:

``` text
page
pageSize
```

El Repository debe realizar la consulta utilizando:

``` csharp
Skip(...)
Take(...)
```

No cargues todos los registros para después paginar en memoria.

Incorrecto:

``` csharp
var tasks = await _repository.GetAllAsync();

var result = tasks
    .Skip(...)
    .Take(...);
```

Preferible:

``` text
HTTP
 ↓
Service
 ↓
Repository
 ↓
SQL Server
      ↓
   Skip/Take
```

La base de datos debe hacer el trabajo.

------------------------------------------------------------------------

# 31. Ejercicio práctico 4 --- Ordenamiento

Permite ordenar por:

``` text
createdAt
title
status
```

Piensa dónde debería vivir cada responsabilidad.

El Controller interpreta HTTP.

El Service aplica reglas del caso de uso.

El Repository construye la consulta.

------------------------------------------------------------------------

# 32. Ejercicio práctico 5 --- Completar una tarea

Implementa:

``` http
POST /api/tasks/{id}/complete
```

Flujo:

``` text
Controller
    ↓
TaskService.CompleteAsync()
    ↓
ITaskRepository.GetByIdAsync()
    ↓
Task.Complete()
    ↓
ITaskRepository.SaveChangesAsync()
```

No pongas la lógica de negocio en el Controller.

No pongas la transición de estado en el Repository.

------------------------------------------------------------------------

# 33. Ejercicio práctico 6 --- Fake Repository

Crea:

``` csharp
FakeTaskRepository
```

que implemente:

``` csharp
ITaskRepository
```

No necesitas todavía crear un proyecto de tests.

El objetivo es comprobar que puedes reemplazar:

``` text
TaskRepository
```

por:

``` text
FakeTaskRepository
```

sin modificar:

``` text
TaskService
```

Eso demuestra que la dependencia está correctamente abstraída.

------------------------------------------------------------------------

# 34. Preguntas de comprensión

Antes de avanzar, deberías poder responder:

1.  ¿Qué problema intenta resolver Repository?
2.  ¿Por qué un Service no debería conocer detalles de SQL?
3.  ¿Qué responsabilidad tiene `DbContext`?
4.  ¿Qué responsabilidad tiene `TaskRepository`?
5.  ¿Qué responsabilidad tiene `TaskService`?
6.  ¿Por qué `TaskRepository` debería ser Scoped?
7.  ¿Por qué no debemos devolver `IQueryable` automáticamente?
8.  ¿Qué problemas puede crear un Repository genérico?
9.  ¿EF Core ya implementa conceptos similares a Repository?
10. ¿Cuándo podría ser mejor no utilizar Repository?
11. ¿Dónde debería vivir una regla como "una tarea completada no puede
    volver a Pending"?
12. ¿Por qué el Controller debería permanecer prácticamente ignorante de
    EF Core?

------------------------------------------------------------------------

# 35. Estado del proyecto

Hasta ahora construimos:

### Módulo 1

``` text
C#
Domain
Collections
LINQ
Async
```

### Módulo 2

``` text
ASP.NET Core
Controllers
HTTP
```

### Módulo 3

``` text
CRUD
Pagination
Filtering
Sorting
```

### Módulo 4

``` text
Dependency Injection
```

### Módulo 5

``` text
Configuration
Options
Environments
Secrets
```

### Módulo 6

``` text
EF Core
DbContext
Migrations
SQL Server
```

### Módulo 7

Ahora tenemos:

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
                  ITaskRepository
                          │
                          ↓
                   TaskRepository
                          │
                          ↓
                     DbContext
                          │
                          ↓
                     SQL Server
```

La aplicación ya tiene una separación razonable entre:

``` text
HTTP
Business / Application
Persistence
Database
```

------------------------------------------------------------------------

# 36. Algo importante antes de CQRS

Hasta este punto estamos usando:

``` text
Controller
   ↓
Service
   ↓
Repository
```

En el siguiente módulo vamos a introducir una arquitectura diferente:

``` text
Controller
   ↓
Mediator
   ↓
Command / Query
   ↓
Handler
```

No significa que todo lo aprendido desaparezca.

Vamos a analizar qué papel tiene cada pieza:

``` text
Controller
Service
Repository
Command
Query
Handler
Mediator
```

y, especialmente:

> ¿Por qué en algunos proyectos el Controller envía un Command o Query a
> un Mediator en lugar de llamar directamente a un Service?

Ese será el objetivo central del **Módulo 8 --- CQRS + Mediator**.

------------------------------------------------------------------------

# 37. Resumen del módulo

La idea fundamental de este módulo es:

> **El Service decide qué debe hacer la aplicación; el Repository se
> ocupa de cómo acceder a los datos.**

Arquitectura:

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

Pero también aprendimos algo más importante:

> **Repository no es una obligación arquitectónica.**

EF Core ya proporciona muchas capacidades que se parecen a Repository y
Unit of Work.

Por eso debemos utilizar Repository cuando aporta una frontera o una
abstracción útil, no simplemente porque sea un patrón conocido.

La arquitectura correcta no es la que tiene más capas.

Es la que tiene las capas necesarias para mantener responsabilidades
claras.
