# Módulo 6 --- Entity Framework Core + SQL

## Objetivo

En este módulo vamos a dar uno de los saltos más importantes del curso:

``` text
Memory
   ↓
EF Core
   ↓
SQL Database
```

Hasta ahora nuestro `Task Management API` pierde todos los datos cuando
reiniciamos la aplicación.

Al terminar este módulo tendremos:

-   una base de datos SQL;
-   Entity Framework Core;
-   `DbContext`;
-   `DbSet`;
-   entidades persistidas;
-   conexión mediante connection string;
-   migrations;
-   creación y actualización del esquema;
-   consultas LINQ contra SQL;
-   operaciones asíncronas;
-   tracking y `AsNoTracking`;
-   relaciones básicas;
-   una primera versión persistente de nuestro proyecto.

------------------------------------------------------------------------

## 1. ¿Qué problema resuelve EF Core?

Hasta ahora tenemos:

``` text
TaskService
    ↓
List<TaskItem>
```

La información vive en memoria.

Si detenemos la aplicación:

``` text
💥
```

las tareas desaparecen.

Necesitamos:

``` text
TaskService
    ↓
EF Core
    ↓
SQL Database
```

EF Core es el ORM de Microsoft para .NET.

ORM significa:

> Object-Relational Mapper.

Su objetivo es permitir trabajar con una base de datos relacional
utilizando objetos y código .NET.

------------------------------------------------------------------------

## 2. ¿Qué es un ORM?

Una base de datos relacional trabaja con:

``` text
Tables
Columns
Rows
Relationships
```

Nuestro código trabaja con:

``` text
Classes
Properties
Objects
Collections
```

El ORM crea un puente entre ambos mundos.

Por ejemplo:

``` text
C#                         SQL

TaskItem            →      Tasks
Id                  →      Id
Title               →      Title
Status              →      Status
CreatedAt           →      CreatedAt
```

Conceptualmente:

``` text
Object
  ↕
ORM
  ↕
Database
```

------------------------------------------------------------------------

## 3. EF Core no es la base de datos

Es importante separar estos conceptos.

EF Core:

``` text
ORM
```

SQL Server:

``` text
Database Engine
```

PostgreSQL:

``` text
Database Engine
```

SQLite:

``` text
Database Engine
```

Podemos utilizar EF Core con diferentes proveedores.

Por ejemplo:

``` text
EF Core
   ↓
SQL Server
```

o:

``` text
EF Core
   ↓
PostgreSQL
```

o:

``` text
EF Core
   ↓
SQLite
```

En este curso utilizaremos SQL Server como referencia principal.

------------------------------------------------------------------------

## 4. Instalar EF Core

Desde el proyecto API instalaremos los paquetes necesarios.

Para SQL Server:

``` bash
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
```

Para migrations:

``` bash
dotnet add package Microsoft.EntityFrameworkCore.Design
```

También necesitaremos la herramienta de EF Core:

``` bash
dotnet tool install --global dotnet-ef
```

Si ya está instalada:

``` bash
dotnet tool update --global dotnet-ef
```

Podemos comprobar:

``` bash
dotnet ef --version
```

------------------------------------------------------------------------

## 5. DbContext

El corazón de EF Core es:

``` text
DbContext
```

Podemos pensarlo como la unidad de trabajo entre nuestra aplicación y la
base de datos.

Por ejemplo:

``` csharp
public class TaskManagementDbContext : DbContext
{
    public TaskManagementDbContext(
        DbContextOptions<TaskManagementDbContext> options)
        : base(options)
    {
    }
}
```

Todavía no tenemos ninguna tabla.

Vamos a agregar un `DbSet`.

------------------------------------------------------------------------

## 6. DbSet

Un `DbSet<T>` representa una colección de entidades que EF Core puede
consultar y persistir.

Por ejemplo:

``` csharp
public DbSet<TaskItem> Tasks => Set<TaskItem>();
```

Entonces:

``` text
DbContext
    │
    └── DbSet<TaskItem>
              ↓
            Tasks
```

Conceptualmente:

``` text
DbSet<TaskItem>
      ↓
   tabla Tasks
```

------------------------------------------------------------------------

## 7. Crear el DbContext

Podemos crear:

``` text
Data/
    TaskManagementDbContext.cs
```

Con:

``` csharp
using Microsoft.EntityFrameworkCore;
using TaskManagement.Api.Domain;

namespace TaskManagement.Api.Data;

public class TaskManagementDbContext : DbContext
{
    public TaskManagementDbContext(
        DbContextOptions<TaskManagementDbContext> options)
        : base(options)
    {
    }

    public DbSet<TaskItem> Tasks => Set<TaskItem>();
}
```

Ahora tenemos la conexión conceptual:

``` text
TaskItem
   ↓
DbSet<TaskItem>
   ↓
DbContext
```

------------------------------------------------------------------------

## 8. Connection String

En el módulo anterior vimos:

``` json
{
  "ConnectionStrings": {
    "Default": "..."
  }
}
```

Ahora finalmente vamos a utilizarla.

Por ejemplo, para desarrollo local:

``` json
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=TaskManagement;Trusted_Connection=True;TrustServerCertificate=True"
  }
}
```

La connection string contiene la información necesaria para que el
provider pueda conectarse a SQL Server.

En una aplicación real, los detalles dependen de dónde esté ejecutándose
SQL Server.

------------------------------------------------------------------------

## 9. Registrar DbContext

En `Program.cs`:

``` csharp
using Microsoft.EntityFrameworkCore;
using TaskManagement.Api.Data;

builder.Services.AddDbContext<TaskManagementDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("Default")));
```

Estamos haciendo dos cosas:

``` text
DbContext
   ↓
DI
```

y:

``` text
DbContext
   ↓
SQL Server provider
```

La configuración completa queda conceptualmente:

``` text
appsettings
     ↓
Connection String
     ↓
AddDbContext
     ↓
DbContext
     ↓
SQL Server
```

------------------------------------------------------------------------

## 10. ¿Por qué AddDbContext?

Una pregunta importante:

¿Por qué no hacemos?

``` csharp
new TaskManagementDbContext(...)
```

La respuesta es la misma idea que vimos en Dependency Injection.

Queremos que el framework gestione el ciclo de vida del `DbContext`.

Por defecto:

``` csharp
AddDbContext
```

registra el contexto con lifetime:

``` text
Scoped
```

Esto normalmente significa:

``` text
HTTP Request
     ↓
DbContext
     ↓
HTTP Request termina
     ↓
DbContext disposed
```

Es una característica importante en aplicaciones web.

------------------------------------------------------------------------

## 11. Crear una migration

Ahora EF Core necesita conocer el esquema inicial.

Ejecutamos:

``` bash
dotnet ef migrations add InitialCreate
```

Esto genera archivos de migration.

Podemos tener:

``` text
Migrations/
    2026..._InitialCreate.cs
    TaskManagementDbContextModelSnapshot.cs
```

La migration describe cómo pasar de:

``` text
Database vacía
```

a:

``` text
Database con el esquema de nuestra aplicación
```

------------------------------------------------------------------------

## 12. Aplicar la migration

Para aplicar la migration:

``` bash
dotnet ef database update
```

EF Core ejecutará los cambios necesarios en la base de datos.

Conceptualmente:

``` text
C# Model
   ↓
Migration
   ↓
SQL
   ↓
Database
```

------------------------------------------------------------------------

## 13. ¿Qué es una migration?

Una migration es una forma de versionar el esquema de la base de datos.

Imaginemos:

``` text
Migration 1
    ↓
Tasks

Migration 2
    ↓
Comments

Migration 3
    ↓
Users

Migration 4
    ↓
Assignments
```

Esto permite evolucionar el esquema junto con el código.

Es conceptualmente parecido a versionar código con Git, pero aplicado al
esquema de la base de datos.

------------------------------------------------------------------------

## 14. Verificar la tabla

Después de ejecutar:

``` bash
dotnet ef database update
```

SQL Server debería tener una tabla relacionada con:

``` text
Tasks
```

Y EF Core también mantiene una tabla interna para registrar las
migrations aplicadas:

``` text
__EFMigrationsHistory
```

Esto permite saber qué migrations ya fueron ejecutadas.

------------------------------------------------------------------------

## 15. Entity vs Domain Model

Aquí aparece una decisión arquitectónica importante.

Podríamos utilizar directamente:

``` csharp
TaskItem
```

como entidad de EF Core.

Para nuestro proyecto inicial está bien.

No necesitamos crear inmediatamente:

``` text
Domain Task
Persistence TaskEntity
Mapper
Repository
```

Eso sería introducir demasiada arquitectura demasiado pronto.

Por ahora:

``` text
TaskItem
   ↓
EF Core
   ↓
Tasks
```

Más adelante veremos cuándo puede ser conveniente separar dominio y
persistencia.

------------------------------------------------------------------------

## 16. Configuración de la entidad

EF Core puede inferir muchas cosas por convención.

Por ejemplo:

``` csharp
public class TaskItem
{
    public Guid Id { get; private set; }

    public string Title { get; private set; }

    public string? Description { get; private set; }

    public TaskStatus Status { get; private set; }

    public DateTime CreatedAt { get; private set; }
}
```

EF Core puede inferir:

``` text
Id
   ↓
Primary Key

Title
   ↓
Column

Description
   ↓
Nullable Column

Status
   ↓
Column

CreatedAt
   ↓
Column
```

Pero no debemos depender siempre de convenciones.

Podemos configurar explícitamente el modelo.

------------------------------------------------------------------------

## 17. OnModelCreating

Dentro del `DbContext`:

``` csharp
protected override void OnModelCreating(
    ModelBuilder modelBuilder)
{
    modelBuilder.Entity<TaskItem>(entity =>
    {
        entity.HasKey(task => task.Id);

        entity.Property(task => task.Title)
            .IsRequired()
            .HasMaxLength(200);

        entity.Property(task => task.Description)
            .HasMaxLength(2000);
    });
}
```

Ahora estamos definiendo reglas del modelo.

Por ejemplo:

``` text
Title
required
max 200
```

y:

``` text
Description
optional
max 2000
```

------------------------------------------------------------------------

## 18. Configuración separada

Podemos mantener el `DbContext` simple usando clases separadas.

Por ejemplo:

``` text
Data/
├── TaskManagementDbContext.cs
└── Configurations/
    └── TaskItemConfiguration.cs
```

La clase:

``` csharp
public class TaskItemConfiguration
    : IEntityTypeConfiguration<TaskItem>
{
    public void Configure(
        EntityTypeBuilder<TaskItem> builder)
    {
        builder.HasKey(task => task.Id);

        builder.Property(task => task.Title)
            .IsRequired()
            .HasMaxLength(200);

        builder.Property(task => task.Description)
            .HasMaxLength(2000);
    }
}
```

Y en `OnModelCreating`:

``` csharp
protected override void OnModelCreating(
    ModelBuilder modelBuilder)
{
    modelBuilder.ApplyConfigurationsFromAssembly(
        typeof(TaskManagementDbContext).Assembly);
}
```

Para el proyecto puedes utilizar esta estructura desde ahora.

------------------------------------------------------------------------

## 19. Primera consulta

Antes teníamos:

``` csharp
_tasks.ToList()
```

Ahora podemos consultar:

``` csharp
await _context.Tasks.ToListAsync();
```

La diferencia fundamental es:

``` text
List<T>
   ↓
Memory

DbSet<T>
   ↓
Database
```

------------------------------------------------------------------------

## 20. LINQ sigue existiendo

Una de las ventajas de EF Core es que podemos seguir utilizando LINQ.

Por ejemplo:

``` csharp
var tasks = await _context.Tasks
    .Where(task => task.Status == TaskStatus.Pending)
    .ToListAsync();
```

Conceptualmente:

``` text
LINQ
 ↓
EF Core
 ↓
SQL
```

EF Core traduce la expresión a una consulta SQL equivalente.

No necesitamos escribir SQL manualmente para las operaciones habituales.

------------------------------------------------------------------------

## 21. Insertar una entidad

Para crear una tarea:

``` csharp
var task = new TaskItem(
    title,
    description);

_context.Tasks.Add(task);

await _context.SaveChangesAsync();
```

Hay dos operaciones conceptuales:

``` text
Add
 ↓
EF Core empieza a trackear el objeto
```

y luego:

``` text
SaveChangesAsync
 ↓
ejecuta INSERT
```

Una idea importante:

> `Add()` no significa necesariamente que el INSERT ya se ejecutó en la
> base de datos.

El cambio se persiste cuando ejecutamos:

``` csharp
SaveChangesAsync()
```

------------------------------------------------------------------------

## 22. Obtener por Id

Podemos utilizar:

``` csharp
var task = await _context.Tasks
    .FirstOrDefaultAsync(task => task.Id == id);
```

También existe:

``` csharp
var task = await _context.Tasks.FindAsync(id);
```

`FindAsync` tiene un comportamiento especial: puede buscar primero en el
contexto antes de consultar la base de datos.

Para una búsqueda simple por primary key suele ser una buena opción.

------------------------------------------------------------------------

## 23. Actualizar

Podemos obtener la entidad:

``` csharp
var task = await _context.Tasks
    .FindAsync(id);
```

Cambiarla:

``` csharp
task.Start();
```

y luego:

``` csharp
await _context.SaveChangesAsync();
```

EF Core detectará los cambios y generará un `UPDATE`.

Conceptualmente:

``` text
Database
   ↓
Entity
   ↓
Change
   ↓
SaveChangesAsync
   ↓
UPDATE
```

------------------------------------------------------------------------

## 24. El concepto de Tracking

Por defecto EF Core realiza tracking de las entidades que consulta.

Por ejemplo:

``` csharp
var task = await _context.Tasks
    .FirstOrDefaultAsync(...);
```

El contexto mantiene información sobre la entidad.

Después:

``` csharp
task.Start();
```

EF Core detecta que cambió.

Cuando hacemos:

``` csharp
SaveChangesAsync();
```

puede generar el `UPDATE`.

Esto se llama:

``` text
Change Tracking
```

------------------------------------------------------------------------

## 25. AsNoTracking

Si solo queremos leer información y no vamos a modificar las entidades:

``` csharp
var tasks = await _context.Tasks
    .AsNoTracking()
    .ToListAsync();
```

Esto evita el tracking.

Es especialmente útil para consultas de lectura.

Conceptualmente:

``` text
Read-only query
     ↓
AsNoTracking
     ↓
menos overhead
```

No significa que debamos colocar `AsNoTracking()` indiscriminadamente.

Primero entendemos qué necesitamos.

------------------------------------------------------------------------

## 26. Eliminar

Podemos hacer:

``` csharp
var task = await _context.Tasks
    .FindAsync(id);

if (task is null)
{
    return false;
}

_context.Tasks.Remove(task);

await _context.SaveChangesAsync();

return true;
```

EF Core generará un `DELETE`.

------------------------------------------------------------------------

## 27. Servicio persistente

Nuestro `TaskService` debe dejar de utilizar:

``` csharp
private readonly List<TaskItem> _tasks = [];
```

y recibir:

``` csharp
private readonly TaskManagementDbContext _context;
```

Por ejemplo:

``` csharp
public class TaskService : ITaskService
{
    private readonly TaskManagementDbContext _context;
    private readonly TaskManagementOptions _options;

    public TaskService(
        TaskManagementDbContext context,
        IOptions<TaskManagementOptions> options)
    {
        _context = context;
        _options = options.Value;
    }
}
```

Ahora el servicio tiene:

``` text
Configuration
       +
Database
```

proporcionadas mediante DI.

------------------------------------------------------------------------

## 28. GetAll con paginación

Podemos implementar:

``` csharp
public async Task<List<TaskItem>> GetAllAsync(
    int? page,
    int? pageSize)
{
    var effectivePage =
        page ?? 1;

    var effectivePageSize =
        pageSize ?? _options.DefaultPageSize;

    effectivePageSize = Math.Min(
        effectivePageSize,
        _options.MaxPageSize);

    return await _context.Tasks
        .AsNoTracking()
        .OrderByDescending(task => task.CreatedAt)
        .Skip((effectivePage - 1) * effectivePageSize)
        .Take(effectivePageSize)
        .ToListAsync();
}
```

La idea es prácticamente la misma que en el módulo 3.

La diferencia es dónde se ejecuta:

Antes:

``` text
List
 ↓
LINQ
 ↓
Memory
```

Ahora:

``` text
DbSet
 ↓
LINQ
 ↓
EF Core
 ↓
SQL
 ↓
Database
```

------------------------------------------------------------------------

## 29. Importante: IQueryable

Cuando escribimos:

``` csharp
_context.Tasks
    .Where(...)
    .OrderBy(...)
    .Skip(...)
    .Take(...)
```

EF Core normalmente está construyendo una consulta.

No necesariamente ejecuta la base de datos en cada llamada.

La ejecución ocurre cuando utilizamos métodos como:

``` csharp
ToListAsync()
FirstOrDefaultAsync()
AnyAsync()
CountAsync()
```

Por eso se habla de ejecución diferida.

Conceptualmente:

``` text
Construir query
      ↓
IQueryable
      ↓
ToListAsync()
      ↓
SQL
      ↓
Database
```

------------------------------------------------------------------------

## 30. Evitar traer toda la tabla

Esto:

``` csharp
var tasks = await _context.Tasks
    .ToListAsync();
```

trae todos los registros.

Si tenemos:

``` text
10 registros
```

no pasa nada.

Si tenemos:

``` text
10.000.000 registros
```

tenemos un problema.

Por eso la paginación que introdujimos en el módulo 3 se vuelve todavía
más importante cuando trabajamos con una base de datos.

------------------------------------------------------------------------

## 31. Projection con Select

No siempre necesitamos traer todas las columnas.

Podemos proyectar:

``` csharp
var summaries = await _context.Tasks
    .AsNoTracking()
    .Select(task => new TaskSummary
    {
        Id = task.Id,
        Title = task.Title,
        Status = task.Status
    })
    .ToListAsync();
```

Conceptualmente:

``` text
Entity
  ↓
Select
  ↓
solo datos necesarios
  ↓
SQL
```

Esto será muy importante cuando trabajemos con DTOs en el módulo 9.

------------------------------------------------------------------------

## 32. Enum y base de datos

Nuestro dominio tiene:

``` csharp
public enum TaskStatus
{
    Pending,
    InProgress,
    Completed
}
```

EF Core puede persistir enums.

Por defecto, dependiendo de la configuración del provider, puede
representarlos como valores numéricos.

También podemos configurarlos como string:

``` csharp
builder.Property(task => task.Status)
    .HasConversion<string>();
```

Esto produciría valores como:

``` text
Pending
InProgress
Completed
```

La decisión depende de los requisitos del proyecto.

Para nuestro curso podemos mantener el comportamiento por defecto
inicialmente.

------------------------------------------------------------------------

## 33. Relaciones

Hasta ahora `TaskItem` es independiente.

Pero el proyecto final tendrá:

``` text
User
Project
Task
Comment
Assignment
```

Por ejemplo:

``` text
Project
   │
   └── Tasks
          │
          └── Comments
```

Esto significa que tendremos relaciones:

``` text
Project 1 ─── N Task
Task    1 ─── N Comment
User    1 ─── N Task
```

No vamos a implementar todas todavía.

Primero queremos dominar:

``` text
Entity
DbContext
DbSet
Migration
CRUD
```

Las relaciones las iremos agregando progresivamente.

------------------------------------------------------------------------

## 34. Async en EF Core

Las operaciones de base de datos deben realizarse de forma asíncrona.

Por ejemplo:

``` csharp
await _context.Tasks.ToListAsync();
```

y:

``` csharp
await _context.SaveChangesAsync();
```

En lugar de bloquear el thread mientras esperamos la base de datos,
ASP.NET Core puede utilizar mejor sus recursos.

La analogía con TypeScript es:

``` typescript
await httpClient.get(...)
```

y en .NET:

``` csharp
await _context.Tasks.ToListAsync();
```

No son técnicamente iguales, pero el modelo mental de programación
asíncrona es similar.

------------------------------------------------------------------------

## 35. CancellationToken

Las operaciones de EF Core pueden recibir un:

``` csharp
CancellationToken
```

Por ejemplo:

``` csharp
await _context.Tasks
    .ToListAsync(cancellationToken);
```

Esto permite cancelar una operación cuando la request ya no necesita
continuar.

Más adelante lo incorporaremos de forma sistemática en nuestros
servicios.

Por ahora debemos conocer su existencia y entender por qué es útil.

------------------------------------------------------------------------

## 36. Actualización del proyecto

La arquitectura pasa de:

``` text
HTTP
 ↓
Controller
 ↓
ITaskService
 ↓
TaskService
 ↓
List<TaskItem>
```

a:

``` text
HTTP
 ↓
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

Y tenemos:

``` text
Configuration
      ↓
Options
      ↓
TaskService
```

y:

``` text
DI
 ↓
TaskService
 ↓
DbContext
```

------------------------------------------------------------------------

## 37. Estructura propuesta

Podemos llegar a:

``` text
TaskManagement.Api/
│
├── Controllers/
│   └── TasksController.cs
│
├── Data/
│   ├── TaskManagementDbContext.cs
│   └── Configurations/
│       └── TaskItemConfiguration.cs
│
├── Domain/
│   └── TaskItem.cs
│
├── Options/
│   └── TaskManagementOptions.cs
│
├── Services/
│   ├── ITaskService.cs
│   └── TaskService.cs
│
├── Migrations/
│   └── ...
│
├── appsettings.json
├── appsettings.Development.json
└── Program.cs
```

Todavía no estamos creando una Clean Architecture completa.

El objetivo es entender primero las piezas.

------------------------------------------------------------------------

## 38. Ejercicio práctico principal

El objetivo del módulo es migrar completamente nuestro CRUD desde
memoria a SQL.

### Paso 1

Instala:

``` bash
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design
```

### Paso 2

Instala o actualiza:

``` bash
dotnet tool install --global dotnet-ef
```

### Paso 3

Crea:

``` text
TaskManagementDbContext
```

### Paso 4

Agrega:

``` csharp
DbSet<TaskItem>
```

### Paso 5

Configura:

``` text
ConnectionStrings:Default
```

### Paso 6

Registra el contexto:

``` csharp
builder.Services.AddDbContext<TaskManagementDbContext>(...);
```

### Paso 7

Crea:

``` bash
dotnet ef migrations add InitialCreate
```

### Paso 8

Aplica:

``` bash
dotnet ef database update
```

### Paso 9

Modifica `TaskService`.

Elimina:

``` csharp
private readonly List<TaskItem> _tasks = [];
```

y reemplázalo por:

``` csharp
private readonly TaskManagementDbContext _context;
```

### Paso 10

Implementa:

``` text
Create
GetAll
GetById
Update
Delete
```

utilizando EF Core.

------------------------------------------------------------------------

## 39. Ejercicio adicional --- SQL generado

Activa logging de EF Core y observa las consultas SQL que genera.

El objetivo no es memorizar SQL.

Queremos empezar a desarrollar esta capacidad:

``` text
C# LINQ
   ↓
SQL generado
```

Debemos poder mirar una consulta LINQ y tener una idea aproximada de qué
consulta SQL terminará ejecutándose.

Esta habilidad será muy importante para Performance en el módulo 13.

------------------------------------------------------------------------

## 40. Ejercicio adicional --- AsNoTracking

Compara:

``` csharp
_context.Tasks
    .ToListAsync();
```

con:

``` csharp
_context.Tasks
    .AsNoTracking()
    .ToListAsync();
```

Pregúntate:

-   ¿la consulta devuelve los mismos datos?
-   ¿cuándo necesito tracking?
-   ¿cuándo no lo necesito?

------------------------------------------------------------------------

## 41. Ejercicio adicional --- Projection

Crea una consulta que solamente devuelva:

``` text
Id
Title
Status
```

Utilizando:

``` csharp
Select(...)
```

No devuelvas la entidad completa.

Esto prepara el terreno para el módulo 9.

------------------------------------------------------------------------

## 42. Preguntas de comprensión

Antes de continuar intenta responder:

1.  ¿Qué problema resuelve EF Core?

2.  ¿Qué diferencia existe entre EF Core y SQL Server?

3.  ¿Qué es un `DbContext`?

4.  ¿Qué representa un `DbSet<T>`?

5.  ¿Qué hace una migration?

6.  ¿Cuál es la diferencia entre `Add()` y `SaveChangesAsync()`?

7.  ¿Qué es Change Tracking?

8.  ¿Cuándo utilizarías `AsNoTracking()`?

9.  ¿Cuándo se ejecuta realmente una consulta LINQ de EF Core?

10. ¿Por qué es importante utilizar `ToListAsync()` en lugar de bloquear
    esperando la base de datos?

------------------------------------------------------------------------

## 43. Error conceptual que debemos evitar

No debemos pensar:

``` text
EF Core = SQL
```

La relación correcta es:

``` text
Application
     ↓
EF Core
     ↓
Database Provider
     ↓
SQL Server
```

EF Core es una capa de abstracción.

Pero esa abstracción no significa que podamos ignorar SQL.

Un backend profesional debe entender ambos mundos:

``` text
C#
+
LINQ
+
EF Core
+
SQL
```

------------------------------------------------------------------------

## 44. Estado del proyecto

Después del módulo 6:

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
                  │     │
                  │     └── TaskManagementOptions
                  │
                  ↓
              DbContext
                  │
                  ↓
              SQL Server
```

Ya no tenemos:

``` text
List<TaskItem>
```

como almacenamiento principal.

Ahora tenemos:

``` text
SQL Server
```

y las tareas sobreviven al reinicio de la aplicación.

------------------------------------------------------------------------

## 45. Qué aprendimos

En este módulo aprendimos:

-   qué es un ORM;
-   qué es EF Core;
-   qué es `DbContext`;
-   qué es `DbSet`;
-   cómo configurar SQL Server;
-   cómo usar connection strings;
-   cómo registrar `DbContext` mediante DI;
-   migrations;
-   `database update`;
-   CRUD con EF Core;
-   LINQ sobre `DbSet`;
-   `SaveChangesAsync`;
-   Change Tracking;
-   `AsNoTracking`;
-   `IQueryable`;
-   ejecución diferida;
-   projection;
-   operaciones asíncronas;
-   `CancellationToken`;
-   primeras nociones de relaciones.

La idea fundamental es:

``` text
Nuestro código ya no administra directamente la memoria.

Nuestro servicio trabaja con EF Core.

EF Core se encarga de traducir nuestras operaciones
a interacciones con la base de datos.
```

------------------------------------------------------------------------

## 46. Próximo módulo

Hasta ahora tenemos:

``` text
Controller
    ↓
Service
    ↓
DbContext
    ↓
SQL Server
```

Pero hay una pregunta arquitectónica importante:

> ¿Debería el Service conocer directamente EF Core?

En el próximo módulo introduciremos:

``` text
Service
   ↓
Repository
   ↓
DbContext
   ↓
Database
```

Vamos a estudiar:

-   qué problema intenta resolver Repository;
-   `IRepository`;
-   implementación concreta;
-   separación de responsabilidades;
-   cuándo Repository aporta valor;
-   cuándo puede ser una abstracción innecesaria;
-   cómo combinar Repository con EF Core sin ocultar conceptos
    importantes.

Y comenzaremos a acercarnos a una arquitectura más parecida a la que
encontramos en aplicaciones backend empresariales reales.
