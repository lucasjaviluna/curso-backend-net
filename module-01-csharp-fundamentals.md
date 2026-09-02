# Backend .NET — Módulo 1
# Fundamentos de C# para desarrolladores TypeScript

## 1. Objetivos

Al terminar este módulo deberías poder:

- Entender las diferencias fundamentales entre C# y TypeScript.
- Crear y ejecutar un proyecto .NET desde la CLI.
- Trabajar con variables y tipos.
- Crear clases, propiedades y métodos.
- Utilizar interfaces.
- Entender `record` y cuándo utilizarlo.
- Trabajar con colecciones.
- Utilizar LINQ para consultar colecciones.
- Entender `async` / `await` y `Task`.
- Utilizar `nullable reference types`.
- Entender excepciones básicas.
- Aplicar estos conceptos construyendo la primera pieza de nuestro proyecto backend.

No buscamos aprender absolutamente todo C#. Buscamos aprender **el C# que necesitamos para empezar a desarrollar backend con ASP.NET Core**.

---

# 2. El proyecto del curso

Durante todo el curso construiremos una aplicación backend llamada:

**Task Management API**

La aplicación permitirá administrar:

- usuarios;
- proyectos;
- tareas;
- comentarios;
- estados;
- asignaciones;
- autenticación y autorización.

La aplicación comenzará siendo extremadamente simple y evolucionará durante los 14 módulos.

La progresión será aproximadamente:

```text
Módulo 1
C# + dominio en memoria

        ↓

Módulo 2
ASP.NET Core

        ↓

Módulo 3
CRUD HTTP en memoria

        ↓

Módulo 4
Dependency Injection

        ↓

Módulo 5
Configuration

        ↓

Módulo 6
EF Core + SQL

        ↓

Módulo 7
Services + Repository

        ↓

Módulo 8
CQRS + Mediator

        ↓

...

Módulo 14
Aplicación backend profesional
```

En este módulo solamente construiremos la **base del dominio**.

---

# 3. ¿Qué es .NET?

.NET es una plataforma de desarrollo creada por Microsoft.

Dentro de .NET tenemos:

```text
.NET
│
├── Runtime
├── SDK
├── CLI
├── Base Class Libraries
└── Frameworks
      │
      └── ASP.NET Core
```

### Runtime

Es el entorno encargado de ejecutar aplicaciones .NET.

Podemos pensarlo de forma simplificada como:

```text
Código C#
    ↓
Compilación
    ↓
Código intermedio
    ↓
.NET Runtime
    ↓
CPU / Sistema operativo
```

No necesitamos profundizar todavía en CLR, JIT y otros detalles internos.

Los veremos cuando sean útiles.

---

# 4. .NET SDK

El SDK contiene las herramientas necesarias para desarrollar aplicaciones .NET.

Por ejemplo:

```bash
dotnet new
dotnet build
dotnet run
dotnet test
dotnet restore
```

Podés comprobar si tenés .NET instalado:

```bash
dotnet --version
```

También:

```bash
dotnet --info
```

---

# 5. Crear nuestro primer proyecto

Vamos a crear una aplicación de consola.

```bash
dotnet new console -n TaskManagement
```

Entramos al proyecto:

```bash
cd TaskManagement
```

Y ejecutamos:

```bash
dotnet run
```

Inicialmente tendremos algo similar a:

```csharp
Console.WriteLine("Hello, World!");
```

Esto es suficiente para empezar.

---

# 6. El archivo .csproj

El proyecto tendrá un archivo:

```text
TaskManagement.csproj
```

Por ejemplo:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

</Project>
```

No necesitamos memorizar XML.

Lo importante es entender que:

```text
.csproj
```

describe el proyecto .NET.

Tiene cierta similitud conceptual con:

```text
package.json
```

en Node.js.

---

# 7. C# y TypeScript

Como ya conocés TypeScript, vamos a utilizarlo como referencia.

TypeScript:

```typescript
const name: string = "Lucas";
```

C#:

```csharp
string name = "Lucas";
```

También podemos utilizar:

```csharp
var name = "Lucas";
```

`var` no significa que la variable no tenga tipo.

El compilador determina el tipo:

```csharp
var name = "Lucas";
```

equivale conceptualmente a:

```csharp
string name = "Lucas";
```

El tipo sigue existiendo.

---

# 8. Tipos básicos

Los tipos que utilizaremos constantemente son:

```csharp
string
int
long
decimal
double
bool
DateTime
Guid
```

Ejemplo:

```csharp
string title = "Learn .NET";
int priority = 1;
bool completed = false;
decimal price = 150.50m;
Guid id = Guid.NewGuid();
DateTime createdAt = DateTime.UtcNow;
```

---

# 9. Diferencia importante: decimal

Para valores monetarios utilizaremos normalmente:

```csharp
decimal
```

Por ejemplo:

```csharp
decimal price = 100.50m;
```

El sufijo `m` indica que estamos utilizando un literal decimal.

Regla práctica:

> Dinero → `decimal`

---

# 10. Guid

En aplicaciones backend veremos mucho:

```csharp
Guid
```

Un GUID es un identificador único.

Por ejemplo:

```csharp
Guid id = Guid.NewGuid();
```

Podría producir:

```text
550e8400-e29b-41d4-a716-446655440000
```

En sistemas reales es frecuente encontrar entidades identificadas mediante `Guid`.

---

# 11. Strings

Podemos utilizar:

```csharp
string name = "Task 1";
```

Interpolación:

```csharp
string message = $"Task: {name}";
```

Es conceptualmente similar a:

```typescript
const message = `Task: ${name}`;
```

También tenemos:

```csharp
string.IsNullOrEmpty(name)
```

y:

```csharp
string.IsNullOrWhiteSpace(name)
```

La segunda considera también strings que contienen únicamente espacios.

---

# 12. Nullable Reference Types

Este concepto es muy importante.

Podemos declarar:

```csharp
string name = "Lucas";
```

El compilador entiende que `name` no debería ser `null`.

Si queremos permitir `null`:

```csharp
string? name = null;
```

El `?` indica que el valor puede ser null.

Esto se parece conceptualmente a:

```typescript
let description: string | null = null;
```

aunque el sistema de tipos funciona de forma diferente.

---

# 13. Clases

Una clase representa una estructura con datos y comportamiento.

TypeScript:

```typescript
class Task {
  id: number;
  title: string;
  completed: boolean;
}
```

C#:

```csharp
public class TaskItem
{
    public int Id { get; set; }

    public string Title { get; set; } = string.Empty;

    public bool Completed { get; set; }
}
```

---

# 14. Properties

Esta sintaxis:

```csharp
public string Title { get; set; }
```

define una propiedad.

Podemos restringir la modificación:

```csharp
public int Id { get; private set; }
```

Ahora otras clases pueden leer:

```csharp
task.Id
```

pero no modificar directamente:

```csharp
task.Id = 10;
```

Esto es importante para encapsulación.

---

# 15. Constructors

Podemos crear objetos mediante constructores.

```csharp
public class TaskItem
{
    public string Title { get; set; }

    public TaskItem(string title)
    {
        Title = title;
    }
}
```

Utilización:

```csharp
var task = new TaskItem("Learn .NET");
```

---

# 16. Primary Constructors

Las versiones modernas de C# ofrecen una sintaxis más compacta:

```csharp
public class TaskItem(string title)
{
    public string Title { get; set; } = title;
}
```

No necesitamos utilizar esta sintaxis siempre.

La sintaxis tradicional sigue siendo perfectamente válida.

La aprenderemos porque probablemente encuentres ambas formas en proyectos existentes.

---

# 17. Métodos

Una clase puede tener comportamiento.

```csharp
public class TaskItem
{
    public string Title { get; set; } = string.Empty;

    public bool Completed { get; private set; }

    public void Complete()
    {
        Completed = true;
    }
}
```

Podemos hacer:

```csharp
var task = new TaskItem();

task.Complete();
```

---

# 18. Encapsulación

Esto:

```csharp
public bool Completed { get; private set; }
```

es mejor que:

```csharp
public bool Completed { get; set; }
```

si queremos impedir que cualquier parte del sistema cambie el estado directamente.

Por ejemplo:

```csharp
task.Completed = true;
```

no estaría permitido desde fuera.

En cambio:

```csharp
task.Complete();
```

expresa una intención.

La idea importante es:

> Una tarea no simplemente cambia una propiedad; una tarea puede ser completada.

No vamos a profundizar todavía en DDD.

---

# 19. Interfaces

Una interfaz define un contrato.

TypeScript:

```typescript
interface TaskService {
  create(task: Task): Task;
}
```

C#:

```csharp
public interface ITaskService
{
    TaskItem Create(TaskItem task);
}
```

Una implementación:

```csharp
public class TaskService : ITaskService
{
    public TaskItem Create(TaskItem task)
    {
        return task;
    }
}
```

---

# 20. ¿Por qué usamos interfaces?

Principalmente para separar:

```text
qué hace algo
```

de:

```text
cómo lo hace
```

Por ejemplo:

```text
ITaskRepository
```

define el contrato.

Mientras:

```text
TaskRepository
```

define la implementación.

Más adelante tendremos:

```text
ITaskRepository
       ↑
       │
TaskRepository
```

Y posiblemente:

```text
InMemoryTaskRepository
```

durante testing.

---

# 21. Collections

En backend trabajaremos constantemente con colecciones.

La más común inicialmente:

```csharp
List<TaskItem>
```

Ejemplo:

```csharp
var tasks = new List<TaskItem>();

tasks.Add(new TaskItem("Learn C#"));
tasks.Add(new TaskItem("Learn ASP.NET"));
```

Podemos obtener:

```csharp
var first = tasks[0];
```

Cantidad:

```csharp
var count = tasks.Count;
```

Eliminar:

```csharp
tasks.Remove(task);
```

---

# 22. Dictionary

También tenemos:

```csharp
Dictionary<int, TaskItem>
```

Por ejemplo:

```csharp
var tasks = new Dictionary<int, TaskItem>();

tasks[1] = new TaskItem("Learn C#");
tasks[2] = new TaskItem("Learn ASP.NET");
```

Obtener:

```csharp
var task = tasks[1];
```

Los diccionarios son muy útiles cuando queremos acceder rápidamente por una clave.

---

# 23. Records

Los `record` son muy importantes en aplicaciones modernas de .NET.

Por ejemplo:

```csharp
public record TaskSummary(
    int Id,
    string Title,
    bool Completed
);
```

Podemos crear:

```csharp
var task = new TaskSummary(
    1,
    "Learn C#",
    false
);
```

Los records están especialmente orientados a representar datos.

Más adelante serán muy útiles para:

```text
DTOs
Commands
Queries
Responses
```

Y especialmente en:

```text
CQRS + Mediator
```

---

# 24. Class vs Record

Regla práctica inicial:

### Class

Usaremos una clase cuando representamos una entidad o algo con identidad y comportamiento.

```text
TaskItem
User
Project
```

### Record

Lo utilizaremos frecuentemente para transportar datos.

```text
CreateTaskCommand
GetTaskQuery
TaskResponse
```

No es una regla absoluta, pero es un buen modelo mental inicial.

---

# 25. Enums

Un enum representa un conjunto limitado de valores.

```csharp
public enum TaskStatus
{
    Pending,
    InProgress,
    Completed
}
```

Podemos utilizar:

```csharp
var status = TaskStatus.Pending;
```

Esto evita utilizar strings arbitrarios como:

```text
"pending"
"in progress"
"done"
```

---

# 26. Nuestro modelo de Task

Vamos a crear una primera versión:

```csharp
public enum TaskStatus
{
    Pending,
    InProgress,
    Completed
}

public class TaskItem
{
    public Guid Id { get; private set; }

    public string Title { get; private set; }

    public string? Description { get; private set; }

    public TaskStatus Status { get; private set; }

    public DateTime CreatedAt { get; private set; }

    public TaskItem(
        string title,
        string? description)
    {
        Id = Guid.NewGuid();
        Title = title;
        Description = description;
        Status = TaskStatus.Pending;
        CreatedAt = DateTime.UtcNow;
    }

    public void Complete()
    {
        Status = TaskStatus.Completed;
    }
}
```

Este será nuestro primer modelo de dominio.

---

# 27. LINQ

LINQ significa:

**Language Integrated Query**

Es una de las herramientas más importantes de C#.

Permite consultar colecciones de manera declarativa.

Tenemos:

```csharp
var tasks = new List<TaskItem>();
```

Podemos buscar:

```csharp
var completedTasks = tasks
    .Where(t => t.Status == TaskStatus.Completed)
    .ToList();
```

Conceptualmente:

```text
tasks
  ↓
Where
  ↓
solo Completed
  ↓
ToList
```

---

# 28. Where

Filtrar:

```csharp
var pendingTasks = tasks
    .Where(t => t.Status == TaskStatus.Pending)
    .ToList();
```

Equivalentemente, en JavaScript:

```typescript
const pendingTasks = tasks
  .filter(t => t.status === "Pending");
```

---

# 29. Select

Transformar:

```csharp
var titles = tasks
    .Select(t => t.Title)
    .ToList();
```

Conceptualmente:

```typescript
const titles = tasks.map(t => t.title);
```

---

# 30. FirstOrDefault

Buscar un elemento:

```csharp
var task = tasks
    .FirstOrDefault(t => t.Id == id);
```

Si no encuentra nada devuelve `null`.

---

# 31. Any

Preguntar si existe algo:

```csharp
var hasCompletedTasks = tasks
    .Any(t => t.Status == TaskStatus.Completed);
```

Equivalentemente:

```typescript
tasks.some(...)
```

---

# 32. OrderBy

Ordenar:

```csharp
var orderedTasks = tasks
    .OrderBy(t => t.CreatedAt)
    .ToList();
```

Descendente:

```csharp
var orderedTasks = tasks
    .OrderByDescending(t => t.CreatedAt)
    .ToList();
```

---

# 33. LINQ encadenado

Podemos combinar operaciones:

```csharp
var result = tasks
    .Where(t => t.Status != TaskStatus.Completed)
    .OrderByDescending(t => t.CreatedAt)
    .Select(t => t.Title)
    .ToList();
```

Leé esto como:

```text
tasks
 ↓
filtrar
 ↓
ordenar
 ↓
transformar
 ↓
convertir a lista
```

---

# 34. LINQ y SQL

Una de las razones por las que LINQ es tan importante es que EF Core puede traducir muchas expresiones LINQ a SQL.

Por ejemplo:

```csharp
var tasks = await db.Tasks
    .Where(t => t.Status == TaskStatus.Pending)
    .ToListAsync();
```

Conceptualmente:

```text
LINQ
 ↓
EF Core
 ↓
SQL
 ↓
Database
```

Esto será central en el módulo 6.

---

# 35. Funciones y lambdas

Ya vimos:

```csharp
t => t.Status == TaskStatus.Completed
```

Esto es una lambda.

Podemos pensar:

```typescript
t => t.status === "Completed"
```

Es prácticamente el mismo concepto.

---

# 36. Métodos de extensión

LINQ utiliza mucho los métodos de extensión.

Por eso podemos hacer:

```csharp
tasks.Where(...)
```

aunque `Where` no sea realmente un método definido dentro de `List<T>`.

No necesitamos dominar cómo funcionan internamente todavía.

Solo reconocer:

```csharp
.Where()
.Select()
.Any()
.OrderBy()
```

como métodos de extensión proporcionados principalmente por LINQ.

---

# 37. Async / Await

En JavaScript ya conocés:

```typescript
const response = await fetch(url);
```

En C# encontramos:

```csharp
var response = await httpClient.GetAsync(url);
```

La idea general es la misma:

> Esperar una operación que se está realizando de forma asíncrona sin bloquear innecesariamente el thread que atiende la operación.

---

# 38. Task

En C# usamos:

```csharp
Task
```

o:

```csharp
Task<T>
```

Ejemplo:

```csharp
public async Task Save()
{
    await SaveToDatabase();
}
```

Si devuelve información:

```csharp
public async Task<TaskItem> GetTask()
{
    ...
}
```

Podemos pensar:

```text
Task
    ↓
operación asíncrona

Task<T>
    ↓
operación asíncrona que devolverá T
```

---

# 39. ¿Por qué backend utiliza tanto async?

Porque muchas operaciones requieren esperar recursos externos:

```text
Database
External API
File system
Redis
Message broker
```

Por ejemplo:

```text
API
 ↓
Database
 ↓
esperar
 ↓
resultado
```

Mientras esperamos I/O, queremos que el servidor pueda atender otras solicitudes.

---

# 40. CancellationToken

También empezaremos a ver:

```csharp
CancellationToken
```

Por ejemplo:

```csharp
public async Task<TaskItem?> GetTask(
    Guid id,
    CancellationToken cancellationToken)
{
    ...
}
```

La idea es:

> Permitir cancelar una operación que ya no tiene sentido continuar.

Ejemplo:

```text
Browser
   ↓
Request
   ↓
API
   ↓
Database

Usuario cancela la navegación
   ↓
Request cancelado
   ↓
CancellationToken
   ↓
Database operation puede cancelarse
```

Lo veremos mucho más en profundidad cuando lleguemos a ASP.NET Core.

---

# 41. Excepciones

En C# podemos manejar errores con:

```csharp
try
{
    ...
}
catch
{
    ...
}
```

Por ejemplo:

```csharp
try
{
    var task = GetTask(id);
}
catch (Exception ex)
{
    Console.WriteLine(ex.Message);
}
```

Pero una advertencia importante:

**No vamos a llenar cada método de `try/catch`.**

Más adelante aprenderemos a manejar errores de manera centralizada en ASP.NET Core.

---

# 42. Access modifiers

Los más importantes:

```text
public
private
protected
internal
```

Inicialmente concentrémonos en:

### public

Accesible desde fuera.

```csharp
public class TaskItem
```

### private

Solo accesible dentro de la clase.

```csharp
private void Validate()
{
}
```

### protected

Disponible para la clase y clases derivadas.

No necesitaremos utilizarlo demasiado en este curso.

---

# 43. Namespaces

Podemos organizar el código:

```csharp
namespace TaskManagement.Domain;
```

Y después:

```csharp
using TaskManagement.Domain;
```

Esto permite organizar tipos y evitar conflictos de nombres.

Una estructura inicial:

```text
TaskManagement
│
├── Domain
│   ├── TaskItem.cs
│   └── TaskStatus.cs
│
└── Program.cs
```

---

# 44. Proyecto práctico del módulo

Nuestro objetivo será construir un pequeño gestor de tareas **completamente en memoria**.

No habrá todavía:

- HTTP;
- Controllers;
- Database;
- EF Core;
- Dependency Injection.

Eso llegará después.

Por ahora:

```text
Console Application
       ↓
TaskManager
       ↓
List<TaskItem>
```

---

# 45. Estructura

Crear:

```text
TaskManagement/
│
├── Domain/
│   ├── TaskItem.cs
│   └── TaskStatus.cs
│
├── Services/
│   └── TaskManager.cs
│
└── Program.cs
```

---

# 46. TaskStatus.cs

```csharp
namespace TaskManagement.Domain;

public enum TaskStatus
{
    Pending,
    InProgress,
    Completed
}
```

---

# 47. TaskItem.cs

```csharp
namespace TaskManagement.Domain;

public class TaskItem
{
    public Guid Id { get; private set; }

    public string Title { get; private set; }

    public string? Description { get; private set; }

    public TaskStatus Status { get; private set; }

    public DateTime CreatedAt { get; private set; }

    public TaskItem(
        string title,
        string? description)
    {
        Id = Guid.NewGuid();
        Title = title;
        Description = description;
        Status = TaskStatus.Pending;
        CreatedAt = DateTime.UtcNow;
    }

    public void Start()
    {
        Status = TaskStatus.InProgress;
    }

    public void Complete()
    {
        Status = TaskStatus.Completed;
    }
}
```

---

# 48. TaskManager

Ahora crearemos una clase que gestione las tareas.

```csharp
using TaskManagement.Domain;

namespace TaskManagement.Services;

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

---

# 49. ¿Qué estamos haciendo?

Tenemos:

```text
TaskManager
     │
     └── List<TaskItem>
```

`Create`:

```text
Create()
 ↓
new TaskItem
 ↓
Add
 ↓
return
```

`GetAll`:

```text
List
 ↓
ToList
 ↓
return copy
```

`GetById`:

```text
List
 ↓
FirstOrDefault
 ↓
TaskItem?
```

`Delete`:

```text
GetById
 ↓
null?
 ↓
Remove
```

---

# 50. Program.cs

Ahora podemos probarlo:

```csharp
using TaskManagement.Services;

var manager = new TaskManager();

var task1 = manager.Create(
    "Learn C#",
    "Study C# fundamentals");

var task2 = manager.Create(
    "Learn ASP.NET Core",
    "Build first API");

task1.Start();

foreach (var task in manager.GetAll())
{
    Console.WriteLine(
        $"{task.Id} - {task.Title} - {task.Status}");
}
```

Al ejecutar:

```bash
dotnet run
```

deberíamos obtener algo parecido a:

```text
a3e... - Learn C# - InProgress
b42... - Learn ASP.NET Core - Pending
```

Los GUID serán diferentes.

---

# 51. Primera consulta LINQ

Podemos obtener las tareas pendientes:

```csharp
var pendingTasks = manager
    .GetAll()
    .Where(task => task.Status == TaskStatus.Pending)
    .ToList();
```

Y mostrarlas:

```csharp
foreach (var task in pendingTasks)
{
    Console.WriteLine(task.Title);
}
```

---

# 52. Segunda consulta LINQ

Tareas completadas:

```csharp
var completedTasks = manager
    .GetAll()
    .Where(task => task.Status == TaskStatus.Completed)
    .ToList();
```

---

# 53. Tercera consulta LINQ

Obtener solamente los títulos:

```csharp
var titles = manager
    .GetAll()
    .Select(task => task.Title)
    .ToList();
```

---

# 54. Nuestros primeros casos de uso

Aunque todavía no tenemos CQRS, podemos empezar a pensar en términos de casos de uso.

Tenemos:

```text
Create Task
Get Task
Get Tasks
Delete Task
Start Task
Complete Task
```

Más adelante estos conceptos se convertirán en:

```text
CreateTaskCommand
GetTaskQuery
GetTasksQuery
DeleteTaskCommand
StartTaskCommand
CompleteTaskCommand
```

Esto será muy importante en el módulo 8.

---

# 55. Ejercicio 1 — Update

Agregar al `TaskManager`:

```csharp
Update(...)
```

Debe permitir modificar:

```text
Title
Description
```

No debería permitir modificar directamente:

```text
Id
CreatedAt
Status
```

Pensá primero cómo lo implementarías antes de mirar una solución.

---

# 56. Ejercicio 2 — Complete

Agregar:

```csharp
public bool Complete(Guid id)
```

Debe:

1. buscar la tarea;
2. devolver `false` si no existe;
3. completar la tarea;
4. devolver `true`.

Conceptualmente:

```text
Complete(id)
     ↓
GetById
     ↓
¿Existe?
 ┌───┴───┐
No      Sí
 ↓       ↓
false   Complete()
          ↓
        true
```

---

# 57. Ejercicio 3 — Buscar tareas

Agregar:

```csharp
Search(string text)
```

Debe buscar tareas cuyo título contenga el texto.

Pista:

```csharp
.Contains(...)
```

Por ejemplo:

```text
Search("api")
```

debería encontrar:

```text
Learn ASP.NET Core API
Create API
Understand REST API
```

---

# 58. Ejercicio 4 — Estadísticas

Crear un método:

```csharp
GetStatistics()
```

que permita conocer:

```text
Total
Pending
InProgress
Completed
```

Podés resolverlo utilizando:

```text
Count
Where
```

Por ejemplo:

```csharp
tasks.Count
```

y:

```csharp
tasks.Count(t => t.Status == TaskStatus.Completed)
```

---

# 59. Challenge del módulo

Implementá:

```csharp
GetRecentTasks(int count)
```

Debe devolver las tareas más recientes.

Necesitarás combinar:

```text
OrderByDescending
Take
```

Conceptualmente:

```text
Tasks
 ↓
ordenar por CreatedAt descendente
 ↓
Take(count)
 ↓
return
```

---

# 60. Preguntas de comprensión

Antes de pasar al siguiente módulo deberías poder responder:

### 1

¿Qué diferencia existe entre:

```csharp
string
```

y:

```csharp
string?
```

### 2

¿Qué diferencia conceptual existe entre:

```csharp
class
```

y:

```csharp
record
```

### 3

¿Qué hace:

```csharp
FirstOrDefault()
```

### 4

¿Qué hace:

```csharp
Where()
```

### 5

¿Qué hace:

```csharp
Select()
```

### 6

¿Qué representa:

```csharp
Task<T>
```

### 7

¿Por qué `TaskManager` utiliza:

```csharp
List<TaskItem>
```

### 8

¿Por qué `Id` tiene:

```csharp
private set
```

---

# 61. Qué deberías llevarte de este módulo

No es necesario memorizar toda la sintaxis.

Lo realmente importante es este modelo mental:

```text
C#
│
├── Types
├── Classes
├── Interfaces
├── Records
├── Collections
├── LINQ
├── Async / Await
├── Task
└── Nullable
```

Y especialmente:

```text
Collection
     ↓
LINQ
     ↓
Filter / Transform / Sort
```

Esta forma de trabajar aparecerá nuevamente con:

```text
EF Core
     ↓
LINQ
     ↓
SQL
```

---

# 62. Relación con TypeScript

| TypeScript | C# |
|---|---|
| `string` | `string` |
| `number` | `int`, `long`, `decimal`, `double` |
| `boolean` | `bool` |
| `interface` | `interface` |
| `class` | `class` |
| `type` | `record` / otros tipos |
| `Array<T>` | `List<T>` |
| `array.filter()` | `Where()` |
| `array.map()` | `Select()` |
| `array.some()` | `Any()` |
| `find()` | `FirstOrDefault()` |
| `async/await` | `async/await` |
| `Promise<T>` | `Task<T>` |
| `null` | `null` |
| `string \| null` | `string?` |

No son equivalentes perfectos, pero esta tabla permite construir rápidamente un mapa mental.

---

# 63. Estado del proyecto

Al finalizar este módulo tenemos:

```text
Task Management
│
├── Domain
│   ├── TaskItem
│   └── TaskStatus
│
├── Services
│   └── TaskManager
│
└── Program
```

Y:

```text
Program
   ↓
TaskManager
   ↓
List<TaskItem>
```

Todavía **no tenemos backend HTTP**.

Eso es deliberado.

En el próximo módulo tomaremos este conocimiento y lo convertiremos en una aplicación **ASP.NET Core Web API**.

El flujo evolucionará de:

```text
Console
   ↓
TaskManager
   ↓
Memory
```

a:

```text
HTTP
 ↓
Controller
 ↓
TaskManager
 ↓
Memory
```

Ese será nuestro primer backend real.
