# Backend .NET --- Módulo 3

# CRUD en memoria

## 1. Objetivos

Al terminar este módulo deberías poder:

-   Entender qué significa CRUD.
-   Diseñar endpoints HTTP para operaciones CRUD.
-   Separar correctamente colección y recurso individual.
-   Implementar Create, Read, Update y Delete.
-   Trabajar con route parameters y query parameters.
-   Implementar filtros y búsquedas.
-   Implementar ordenamiento.
-   Implementar paginación básica.
-   Manejar correctamente recursos inexistentes.
-   Diferenciar request models de entidades de dominio.
-   Entender algunas decisiones básicas de diseño de una API.
-   Completar el primer CRUD funcional de nuestro proyecto.

En este módulo todavía trabajaremos **en memoria**.

La persistencia con SQL y Entity Framework Core llegará en el módulo 6.

------------------------------------------------------------------------

# 2. ¿Qué es CRUD?

CRUD representa las cuatro operaciones fundamentales sobre recursos:

``` text
C → Create
R → Read
U → Update
D → Delete
```

Para nuestras tareas:

``` text
Create → Crear tarea
Read   → Obtener tareas
Update → Actualizar tarea
Delete → Eliminar tarea
```

Nuestro objetivo será exponerlas mediante HTTP:

``` text
POST   /api/tasks
GET    /api/tasks
GET    /api/tasks/{id}
PUT    /api/tasks/{id}
DELETE /api/tasks/{id}
```

------------------------------------------------------------------------

# 3. Nuestro proyecto

Hasta ahora tenemos:

``` text
HTTP
 ↓
TasksController
 ↓
TaskManager
 ↓
List<TaskItem>
```

Después de este módulo tendremos un CRUD más completo:

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

Y soportaremos:

``` text
Create
Read
Update
Delete
Search
Filter
Sort
Pagination
```

------------------------------------------------------------------------

# 4. ¿Por qué empezar en memoria?

Podríamos empezar directamente con:

``` text
SQL
 ↓
EF Core
```

pero eso mezclaría demasiados conceptos.

Primero queremos entender:

``` text
HTTP
 ↓
CRUD
 ↓
Application logic
```

Después agregaremos:

``` text
Persistence
```

Esta separación nos permite entender qué problema resuelve cada
tecnología.

------------------------------------------------------------------------

# 5. El recurso Task

Nuestro dominio contiene:

``` csharp
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

Esta clase representa nuestro recurso de dominio.

------------------------------------------------------------------------

# 6. Create

La operación Create utiliza:

``` http
POST /api/tasks
```

Body:

``` json
{
  "title": "Learn ASP.NET Core",
  "description": "Build a Web API"
}
```

El servidor crea:

``` text
TaskItem
```

y asigna:

``` text
Id
Status
CreatedAt
```

No deberían venir desde el cliente.

------------------------------------------------------------------------

# 7. ¿Qué datos controla el cliente?

En una creación:

``` text
Client
 ↓
Title
Description
```

Mientras el servidor controla:

``` text
Id
Status
CreatedAt
```

Esto es importante.

No queremos que un cliente pueda enviar:

``` json
{
  "id": "otro-id",
  "status": "Completed",
  "createdAt": "2020-01-01"
}
```

y modificar arbitrariamente el estado interno de la entidad.

------------------------------------------------------------------------

# 8. CreateTaskRequest

Nuestro request model:

``` csharp
public class CreateTaskRequest
{
    public string Title { get; set; } = string.Empty;

    public string? Description { get; set; }
}
```

Por ahora no agregaremos validaciones complejas.

Eso será parte del módulo 9.

------------------------------------------------------------------------

# 9. Implementar Create

En `TaskManager`:

``` csharp
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
```

En el controller:

``` csharp
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
```

------------------------------------------------------------------------

# 10. ¿Por qué 201?

Para una creación exitosa utilizamos:

``` http
201 Created
```

en lugar de:

``` http
200 OK
```

porque estamos comunicando algo diferente:

``` text
200 → la operación fue exitosa
201 → se creó un recurso
```

Cuando diseñamos APIs, los status codes forman parte del contrato.

------------------------------------------------------------------------

# 11. Read --- colección

Para obtener todas las tareas:

``` http
GET /api/tasks
```

Controller:

``` csharp
[HttpGet]
public ActionResult<List<TaskItem>> GetAll()
{
    return Ok(_manager.GetAll());
}
```

Response:

``` http
200 OK
```

``` json
[
  {
    "id": "...",
    "title": "Learn C#",
    "status": "Completed"
  },
  {
    "id": "...",
    "title": "Learn ASP.NET Core",
    "status": "Pending"
  }
]
```

------------------------------------------------------------------------

# 12. Read --- recurso individual

Para obtener una tarea:

``` http
GET /api/tasks/{id}
```

Controller:

``` csharp
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
```

------------------------------------------------------------------------

# 13. ¿Qué ocurre si no existe?

Supongamos:

``` http
GET /api/tasks/123
```

pero la tarea no existe.

No deberíamos devolver:

``` http
200 OK
```

con:

``` json
null
```

En este contexto es más apropiado:

``` http
404 Not Found
```

El consumidor puede interpretar claramente:

``` text
El recurso solicitado no existe.
```

------------------------------------------------------------------------

# 14. Update

Ahora implementaremos:

``` http
PUT /api/tasks/{id}
```

Body:

``` json
{
  "title": "Learn ASP.NET Core",
  "description": "Build a complete API"
}
```

Necesitamos un request model:

``` csharp
public class UpdateTaskRequest
{
    public string Title { get; set; } = string.Empty;

    public string? Description { get; set; }
}
```

------------------------------------------------------------------------

# 15. Actualizar la entidad

Tenemos una decisión de diseño.

Nuestra entidad utiliza:

``` csharp
private set
```

Por lo tanto no podemos hacer desde fuera:

``` csharp
task.Title = request.Title;
```

Eso es intencional.

Podemos agregar un método:

``` csharp
public void Update(
    string title,
    string? description)
{
    Title = title;
    Description = description;
}
```

La entidad controla cómo se modifica.

------------------------------------------------------------------------

# 16. TaskItem actualizado

Quedaría:

``` csharp
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

    public void Update(
        string title,
        string? description)
    {
        Title = title;
        Description = description;
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

------------------------------------------------------------------------

# 17. TaskManager.Update

Podemos implementar:

``` csharp
public TaskItem? Update(
    Guid id,
    string title,
    string? description)
{
    var task = GetById(id);

    if (task is null)
    {
        return null;
    }

    task.Update(
        title,
        description);

    return task;
}
```

------------------------------------------------------------------------

# 18. Controller.Update

``` csharp
[HttpPut("{id:guid}")]
public ActionResult<TaskItem> Update(
    Guid id,
    UpdateTaskRequest request)
{
    var task = _manager.Update(
        id,
        request.Title,
        request.Description);

    if (task is null)
    {
        return NotFound();
    }

    return Ok(task);
}
```

El flujo:

``` text
PUT /api/tasks/{id}
        ↓
Controller
        ↓
TaskManager.Update
        ↓
GetById
        ↓
TaskItem.Update
        ↓
200 OK
```

------------------------------------------------------------------------

# 19. Delete

Nuestro endpoint:

``` http
DELETE /api/tasks/{id}
```

Manager:

``` csharp
public bool Delete(Guid id)
{
    var task = GetById(id);

    if (task is null)
    {
        return false;
    }

    return _tasks.Remove(task);
}
```

Controller:

``` csharp
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
```

------------------------------------------------------------------------

# 20. ¿Por qué 204?

Después de un DELETE exitoso normalmente no necesitamos devolver el
recurso.

Por eso:

``` http
204 No Content
```

es una respuesta apropiada.

Conceptualmente:

``` text
DELETE
 ↓
resource deleted
 ↓
nothing else to return
 ↓
204
```

------------------------------------------------------------------------

# 21. CRUD completo

Ahora tenemos:

``` text
POST   /api/tasks
GET    /api/tasks
GET    /api/tasks/{id}
PUT    /api/tasks/{id}
DELETE /api/tasks/{id}
```

Y:

``` text
Create
Read
Read
Update
Delete
```

Nuestro primer CRUD está completo.

------------------------------------------------------------------------

# 22. Search

Ahora queremos:

``` http
GET /api/tasks?search=aspnet
```

Podemos modificar:

``` csharp
GetAll()
```

para recibir un criterio:

``` csharp
public List<TaskItem> Search(string? search)
{
    var query = _tasks.AsEnumerable();

    if (!string.IsNullOrWhiteSpace(search))
    {
        query = query.Where(task =>
            task.Title.Contains(
                search,
                StringComparison.OrdinalIgnoreCase));
    }

    return query.ToList();
}
```

Esto permite mantener la búsqueda en el manager.

------------------------------------------------------------------------

# 23. Query parameter opcional

Controller:

``` csharp
[HttpGet]
public ActionResult<List<TaskItem>> GetAll(
    string? search)
{
    return Ok(_manager.Search(search));
}
```

Entonces:

``` http
GET /api/tasks
```

devuelve todo.

Mientras:

``` http
GET /api/tasks?search=aspnet
```

filtra.

------------------------------------------------------------------------

# 24. Filter por status

Queremos:

``` http
GET /api/tasks?status=Pending
```

Podemos recibir:

``` csharp
string? status
```

pero tenemos un problema.

El cliente podría enviar:

``` text
status=whatever
```

Por ahora podemos convertirlo mediante:

``` csharp
Enum.TryParse<TaskStatus>(
    status,
    true,
    out var taskStatus)
```

Por ejemplo:

``` csharp
if (!string.IsNullOrWhiteSpace(status))
{
    if (Enum.TryParse<TaskStatus>(
        status,
        true,
        out var parsedStatus))
    {
        query = query.Where(
            task => task.Status == parsedStatus);
    }
}
```

Más adelante mejoraremos esta estrategia con validación.

------------------------------------------------------------------------

# 25. Mejor diseño para el filtro

En lugar de llenar el Controller de lógica:

``` csharp
if (...)
{
    ...
}

if (...)
{
    ...
}

if (...)
{
    ...
}
```

queremos que el Controller delegue:

``` text
Controller
   ↓
TaskManager
   ↓
query
```

Esto es una idea que se volverá cada vez más importante.

------------------------------------------------------------------------

# 26. Un objeto para consultar

Podemos crear:

``` csharp
public class TaskQuery
{
    public string? Search { get; set; }

    public TaskStatus? Status { get; set; }
}
```

Y:

``` csharp
public List<TaskItem> GetAll(TaskQuery query)
{
    var tasks = _tasks.AsEnumerable();

    if (!string.IsNullOrWhiteSpace(query.Search))
    {
        tasks = tasks.Where(task =>
            task.Title.Contains(
                query.Search,
                StringComparison.OrdinalIgnoreCase));
    }

    if (query.Status.HasValue)
    {
        tasks = tasks.Where(task =>
            task.Status == query.Status.Value);
    }

    return tasks.ToList();
}
```

Este enfoque será especialmente útil cuando agreguemos:

``` text
sorting
pagination
```

------------------------------------------------------------------------

# 27. Sorting

Queremos poder hacer:

``` http
GET /api/tasks?sortBy=createdAt
```

o:

``` http
GET /api/tasks?sortBy=title
```

Por ahora podemos soportar pocos campos.

Ejemplo:

``` csharp
switch (query.SortBy?.ToLowerInvariant())
{
    case "title":
        tasks = tasks.OrderBy(task => task.Title);
        break;

    case "createdat":
        tasks = tasks.OrderBy(task => task.CreatedAt);
        break;

    default:
        tasks = tasks.OrderByDescending(
            task => task.CreatedAt);
        break;
}
```

No necesitamos construir todavía un sistema genérico de sorting.

------------------------------------------------------------------------

# 28. Ascending vs descending

Podemos agregar:

``` text
sortDirection=asc
sortDirection=desc
```

Ejemplo:

``` http
GET /api/tasks?sortBy=title&sortDirection=asc
```

El objetivo es entender el concepto.

Más adelante veremos cómo evitar que este tipo de lógica crezca
demasiado.

------------------------------------------------------------------------

# 29. Pagination

Una API que devuelve miles de recursos no debería devolverlos todos de
una vez.

Por ejemplo:

``` text
10.000 tasks
```

No queremos:

``` http
GET /api/tasks
```

→ devolver 10.000 objetos.

Queremos:

``` text
page
pageSize
```

Ejemplo:

``` http
GET /api/tasks?page=1&pageSize=20
```

------------------------------------------------------------------------

# 30. Skip y Take

LINQ proporciona:

``` csharp
Skip()
Take()
```

Por ejemplo:

``` csharp
var tasks = _tasks
    .Skip(20)
    .Take(20)
    .ToList();
```

Esto obtiene:

``` text
elementos 21 → 40
```

------------------------------------------------------------------------

# 31. Fórmula de paginación

Si:

``` text
page = 1
pageSize = 20
```

tenemos:

``` text
skip = (page - 1) * pageSize
```

Entonces:

``` text
skip = 0
take = 20
```

Para:

``` text
page = 2
pageSize = 20
```

tenemos:

``` text
skip = 20
take = 20
```

Para:

``` text
page = 3
pageSize = 20
```

tenemos:

``` text
skip = 40
take = 20
```

------------------------------------------------------------------------

# 32. Validar page y pageSize

Nunca deberíamos confiar completamente en el cliente.

Podríamos recibir:

``` text
page=-100
pageSize=1000000
```

Por ahora podemos limitar:

``` csharp
page = Math.Max(page, 1);
pageSize = Math.Clamp(pageSize, 1, 100);
```

Esto significa:

``` text
page >= 1
pageSize entre 1 y 100
```

La validación formal llegará más adelante.

------------------------------------------------------------------------

# 33. Resultado paginado

Podríamos devolver:

``` json
{
  "items": [
    {
      "id": "...",
      "title": "Learn C#"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "totalCount": 57,
  "totalPages": 3
}
```

Esto es más útil que devolver solamente:

``` json
[
  ...
]
```

porque el frontend necesita conocer información sobre la paginación.

------------------------------------------------------------------------

# 34. PagedResult

Podemos crear un record:

``` csharp
public record PagedResult<T>(
    List<T> Items,
    int Page,
    int PageSize,
    int TotalCount)
{
    public int TotalPages =>
        (int)Math.Ceiling(
            TotalCount / (double)PageSize);
}
```

Entonces:

``` csharp
var result = new PagedResult<TaskItem>(
    items,
    page,
    pageSize,
    totalCount);
```

Esto es un primer ejemplo de un tipo genérico.

------------------------------------------------------------------------

# 35. Un endpoint más completo

Podríamos terminar teniendo:

``` http
GET /api/tasks
    ?search=aspnet
    &status=Pending
    &sortBy=createdAt
    &sortDirection=desc
    &page=1
    &pageSize=20
```

El flujo:

``` text
HTTP Query
     ↓
TaskQuery
     ↓
Filter
     ↓
Sort
     ↓
Pagination
     ↓
PagedResult
```

Esto se parece bastante a lo que encontramos en APIs reales.

------------------------------------------------------------------------

# 36. Pero hay un problema

Podríamos terminar con un método enorme:

``` csharp
GetAll(
    string? search,
    TaskStatus? status,
    string? sortBy,
    string? sortDirection,
    int page,
    int pageSize)
```

y muchas reglas dentro.

No necesariamente es incorrecto.

Pero a medida que la aplicación crece, necesitamos mejores mecanismos
para representar las operaciones.

Esto nos llevará progresivamente hacia:

``` text
Services
Repositories
CQRS
Queries
Handlers
```

No debemos introducirlos todavía.

Primero necesitamos entender bien el problema.

------------------------------------------------------------------------

# 37. Diseño final del TaskManager del módulo

Una versión conceptual podría contener:

``` text
Create
GetAll
GetById
Update
Delete
Search
```

Y una consulta:

``` text
TaskQuery
```

que contenga:

``` text
Search
Status
SortBy
SortDirection
Page
PageSize
```

La implementación puede mantenerse simple.

No necesitamos crear una abstracción para cada pequeño detalle.

------------------------------------------------------------------------

# 38. Controller completo

Una versión inicial podría verse así:

``` csharp
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
    public ActionResult<List<TaskItem>> GetAll(
        [FromQuery] TaskQuery query)
    {
        return Ok(_manager.GetAll(query));
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

    [HttpPut("{id:guid}")]
    public ActionResult<TaskItem> Update(
        Guid id,
        UpdateTaskRequest request)
    {
        var task = _manager.Update(
            id,
            request.Title,
            request.Description);

        if (task is null)
        {
            return NotFound();
        }

        return Ok(task);
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

Observá algo importante:

El Controller no implementa:

``` text
filter
sort
pagination
```

Solamente recibe:

``` text
TaskQuery
```

y delega.

------------------------------------------------------------------------

# 39. FromQuery

Utilizamos:

``` csharp
[FromQuery] TaskQuery query
```

Esto indica que los valores deben obtenerse de los query parameters.

Por ejemplo:

``` http
GET /api/tasks?search=aspnet&page=2
```

se transforma conceptualmente en:

``` csharp
TaskQuery
{
    Search = "aspnet",
    Page = 2
}
```

Esto es model binding.

------------------------------------------------------------------------

# 40. FromBody

Para un POST:

``` csharp
public ActionResult<TaskItem> Create(
    CreateTaskRequest request)
```

ASP.NET Core obtiene el objeto desde el body.

Podríamos escribir explícitamente:

``` csharp
public ActionResult<TaskItem> Create(
    [FromBody] CreateTaskRequest request)
```

Con `[ApiController]`, ASP.NET Core normalmente puede inferirlo.

Es útil conocer ambas formas.

------------------------------------------------------------------------

# 41. FromRoute

También podemos indicar:

``` csharp
[FromRoute] Guid id
```

Por ejemplo:

``` csharp
public ActionResult<TaskItem> GetById(
    [FromRoute] Guid id)
```

Esto deja explícito que:

``` text
id
```

proviene de la ruta.

------------------------------------------------------------------------

# 42. FromQuery vs FromBody vs FromRoute

Modelo mental:

``` text
GET /api/tasks/123?includeComments=true
                         │
                         └── Query

123
│
└── Route

POST body
{
  "title": "Learn C#"
}
        │
        └── Body
```

En C#:

``` csharp
[FromRoute]
[FromQuery]
[FromBody]
```

------------------------------------------------------------------------

# 43. API Contract

Los endpoints que acabamos de diseñar forman parte del contrato de
nuestra API.

Por ejemplo:

``` text
POST /api/tasks
```

espera:

``` json
{
  "title": "...",
  "description": "..."
}
```

y devuelve:

``` text
201 Created
```

Mientras:

``` text
GET /api/tasks/{id}
```

puede devolver:

``` text
200 OK
404 Not Found
```

Este contrato será muy importante cuando nuestro frontend consuma el
backend.

------------------------------------------------------------------------

# 44. Idempotencia: concepto inicial

Hay un concepto importante asociado a HTTP.

Una operación es idempotente cuando repetirla produce el mismo estado
final.

Por ejemplo:

``` http
PUT /api/tasks/123
```

con:

``` json
{
  "title": "Learn C#"
}
```

Si lo ejecutamos una vez o varias veces, el recurso termina con el mismo
estado.

En cambio:

``` http
POST /api/tasks
```

normalmente crea nuevos recursos cada vez.

No necesitamos profundizar más ahora, pero es importante conocer el
concepto.

------------------------------------------------------------------------

# 45. PUT vs PATCH

Recordemos:

``` text
PUT
```

se utiliza normalmente para reemplazar o actualizar la representación
completa que estamos gestionando.

``` text
PATCH
```

se utiliza para cambios parciales.

Ejemplo:

``` http
PUT /api/tasks/123
```

``` json
{
  "title": "Learn C#",
  "description": "Full description"
}
```

Mientras:

``` http
PATCH /api/tasks/123
```

podría modificar únicamente:

``` json
{
  "title": "New title"
}
```

Para nuestro curso utilizaremos `PUT` inicialmente para mantener el CRUD
simple.

------------------------------------------------------------------------

# 46. Manejo de errores

Por ahora tenemos:

``` csharp
if (task is null)
{
    return NotFound();
}
```

Esto funciona.

Pero imaginemos una aplicación con:

``` text
100 controllers
```

y cada uno contiene:

``` text
try/catch
if
return BadRequest
return NotFound
return Unauthorized
...
```

La aplicación puede volverse inconsistente.

En el módulo 9 y especialmente en el módulo 10 veremos una estrategia
mucho más centralizada.

------------------------------------------------------------------------

# 47. Ejercicio 1 --- CRUD completo

Verificá que funcionen:

``` text
POST   /api/tasks
GET    /api/tasks
GET    /api/tasks/{id}
PUT    /api/tasks/{id}
DELETE /api/tasks/{id}
```

Probá también casos de error:

``` text
GET task inexistente
PUT task inexistente
DELETE task inexistente
```

Todos deberían responder:

``` http
404 Not Found
```

cuando corresponda.

------------------------------------------------------------------------

# 48. Ejercicio 2 --- Search

Implementá:

``` http
GET /api/tasks?search=angular
```

La búsqueda debe ser:

-   case insensitive;
-   aplicada al título;
-   opcional.

------------------------------------------------------------------------

# 49. Ejercicio 3 --- Filter

Implementá:

``` http
GET /api/tasks?status=Completed
```

Debe filtrar por:

``` text
Pending
InProgress
Completed
```

Probá también un status inválido.

Preguntate:

> ¿Qué debería responder la API?

Todavía no existe una única respuesta obligatoria. Lo importante es
justificar la decisión.

------------------------------------------------------------------------

# 50. Ejercicio 4 --- Sorting

Implementá:

``` http
GET /api/tasks?sortBy=title
```

y:

``` http
GET /api/tasks?sortBy=createdAt
```

Agregá:

``` text
asc
desc
```

como dirección.

------------------------------------------------------------------------

# 51. Ejercicio 5 --- Pagination

Implementá:

``` http
GET /api/tasks?page=1&pageSize=10
```

La respuesta debería incluir:

``` text
items
page
pageSize
totalCount
totalPages
```

Usá:

``` csharp
Skip()
Take()
```

------------------------------------------------------------------------

# 52. Challenge --- combinación de filtros

Implementá una request como:

``` http
GET /api/tasks
    ?search=api
    &status=Pending
    &sortBy=createdAt
    &sortDirection=desc
    &page=1
    &pageSize=10
```

El resultado debe aplicar en este orden conceptual:

``` text
Search
   ↓
Status filter
   ↓
Sort
   ↓
Pagination
   ↓
Response
```

Pensá por qué el orden importa.

Por ejemplo, generalmente queremos paginar **después** de filtrar y
ordenar.

------------------------------------------------------------------------

# 53. Challenge --- límite de pageSize

Impedí:

``` text
pageSize > 100
```

Por ejemplo:

``` http
GET /api/tasks?pageSize=1000
```

debería limitarse a:

``` text
100
```

o rechazar la request.

Implementá una de las dos estrategias y justificá tu elección.

------------------------------------------------------------------------

# 54. Preguntas de comprensión

### 1

¿Qué significa CRUD?

### 2

¿Por qué utilizamos:

``` text
POST
```

para crear?

### 3

¿Por qué utilizamos:

``` text
404
```

si una tarea no existe?

### 4

¿Por qué un DELETE exitoso puede devolver:

``` text
204
```

### 5

¿Cuál es la diferencia entre:

``` text
Route Parameter
Query Parameter
Body
```

### 6

¿Por qué usamos:

``` csharp
private set
```

en la entidad?

### 7

¿Qué hacen:

``` csharp
Skip()
Take()
```

### 8

¿Por qué debemos paginar después de filtrar?

### 9

¿Por qué no queremos colocar toda la lógica en el Controller?

### 10

¿Qué problema empieza a aparecer cuando nuestro método `GetAll` recibe
demasiados parámetros?

------------------------------------------------------------------------

# 55. Relación con frontend

Como desarrollador frontend, ya conocés este problema.

Por ejemplo:

``` typescript
tasks
  .filter(...)
  .sort(...)
  .slice(...)
  .map(...)
```

En backend estamos haciendo algo conceptualmente similar:

``` csharp
tasks
    .Where(...)
    .OrderBy(...)
    .Skip(...)
    .Take(...)
    .Select(...)
```

La diferencia importante es que más adelante:

``` text
LINQ
 ↓
EF Core
 ↓
SQL
```

hará que estas operaciones se ejecuten en la base de datos en lugar de
traer todo a memoria.

Ese cambio será uno de los puntos más importantes del módulo 6.

------------------------------------------------------------------------

# 56. Estado del proyecto

Al terminar el módulo:

``` text
Task Management API
│
├── Domain
│   ├── TaskItem.cs
│   └── TaskStatus.cs
│
├── Services
│   ├── TaskManager.cs
│   └── TaskQuery.cs
│
├── Controllers
│   ├── TasksController.cs
│   ├── CreateTaskRequest.cs
│   └── UpdateTaskRequest.cs
│
└── Program.cs
```

Funcionalidades:

``` text
Create
Read
Update
Delete
Search
Filter
Sort
Pagination
```

Todo todavía en memoria.

------------------------------------------------------------------------

# 57. Arquitectura actual

Nuestro backend:

``` text
                 HTTP
                  │
                  ↓
          TasksController
                  │
                  ↓
             TaskManager
                  │
                  ↓
           List<TaskItem>
```

Todavía no tenemos:

``` text
Database
Repository
CQRS
Mediator
Authentication
Middleware
Caching
Tests
```

Eso es correcto.

Estamos construyendo la aplicación progresivamente.

------------------------------------------------------------------------

# 58. ¿Qué problema aparece ahora?

Nuestro Controller depende de:

``` csharp
TaskManager
```

y `TaskManager` depende de:

``` csharp
List<TaskItem>
```

Tenemos una aplicación funcional, pero la creación y administración de
dependencias todavía es bastante manual.

Además, a medida que crezcan:

``` text
Controllers
Services
Repositories
Clients
Validators
Handlers
```

necesitaremos una forma consistente de construir y administrar esas
dependencias.

Ese problema será el foco del próximo módulo.

------------------------------------------------------------------------

# 59. Próximo módulo

El módulo 4 será:

# Dependency Injection

Vamos a entender:

-   qué problema resuelve DI;
-   inversión de control;
-   constructor injection;
-   `IServiceCollection`;
-   `AddSingleton`;
-   `AddScoped`;
-   `AddTransient`;
-   interfaces y DI;
-   lifetimes;
-   testing;
-   por qué `Scoped` es tan importante en APIs;
-   cómo registrar nuestros servicios correctamente.

El flujo comenzará a evolucionar de:

``` text
Controller
   ↓
TaskManager
```

a:

``` text
Controller
   ↓
ITaskService
   ↓
TaskService
```

y ASP.NET Core será responsable de construir las dependencias.
