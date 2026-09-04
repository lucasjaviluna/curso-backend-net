# Módulo 5 --- Configuration en .NET

## Objetivo

En este módulo aprenderemos cómo ASP.NET Core maneja la configuración de
una aplicación y cómo separar los valores variables del código.

Al terminar podremos:

-   entender el sistema de Configuration de .NET;
-   usar `appsettings.json`;
-   trabajar con configuraciones por ambiente;
-   leer valores mediante `IConfiguration`;
-   usar el Options Pattern;
-   registrar opciones mediante Dependency Injection;
-   usar variables de entorno;
-   manejar secretos de desarrollo con User Secrets;
-   preparar la aplicación para incorporar una base de datos en el
    próximo módulo.

------------------------------------------------------------------------

## 1. ¿Por qué necesitamos Configuration?

Hasta ahora nuestra aplicación tiene valores directamente escritos en el
código:

``` csharp
var pageSize = 20;
```

Esto funciona, pero rápidamente aparecen problemas.

Imaginemos:

-   Development → página de 20 elementos;
-   Staging → página de 50;
-   Production → página de 100.

No queremos cambiar el código cada vez que desplegamos.

La idea es separar:

``` text
Código
   +
Configuración
```

La aplicación contiene la lógica.

La configuración contiene valores que pueden cambiar según el ambiente.

------------------------------------------------------------------------

## 2. Analogía con Angular

Si vienes de Angular, probablemente ya viste algo parecido:

``` text
environment.ts
environment.prod.ts
```

Por ejemplo:

``` typescript
export const environment = {
  production: false,
  apiUrl: 'http://localhost:5000'
};
```

En ASP.NET Core existe un concepto equivalente, pero el sistema de
configuración es más flexible.

Podemos obtener configuración desde:

-   `appsettings.json`;
-   `appsettings.{Environment}.json`;
-   variables de entorno;
-   User Secrets;
-   argumentos de línea de comandos;
-   otros providers.

La idea general es:

``` text
Configuration
      ↓
  aplicación
```

------------------------------------------------------------------------

## 3. appsettings.json

Una aplicación ASP.NET Core normalmente tiene:

``` text
appsettings.json
```

Podemos colocar:

``` json
{
  "Application": {
    "Name": "Task Management API",
    "Version": "1.0"
  },
  "TaskManagement": {
    "DefaultPageSize": 20,
    "MaxPageSize": 100
  }
}
```

Tenemos dos grupos:

``` text
Application
TaskManagement
```

Y dentro de `TaskManagement`:

``` text
DefaultPageSize
MaxPageSize
```

------------------------------------------------------------------------

## 4. Jerarquía de configuración

.NET representa las propiedades anidadas mediante `:`.

Por ejemplo:

``` json
{
  "TaskManagement": {
    "DefaultPageSize": 20
  }
}
```

puede identificarse como:

``` text
TaskManagement:DefaultPageSize
```

Esto será importante cuando trabajemos con variables de entorno.

------------------------------------------------------------------------

## 5. IConfiguration

ASP.NET Core proporciona:

``` csharp
IConfiguration
```

para acceder a la configuración.

Por ejemplo:

``` csharp
public class MyService
{
    private readonly IConfiguration _configuration;

    public MyService(IConfiguration configuration)
    {
        _configuration = configuration;
    }
}
```

Podemos obtener:

``` csharp
var pageSize =
    _configuration["TaskManagement:DefaultPageSize"];
```

Pero hay un detalle importante.

Ese valor se obtiene como string.

Si necesitamos un entero podemos usar:

``` csharp
var pageSize =
    _configuration.GetValue<int>(
        "TaskManagement:DefaultPageSize");
```

------------------------------------------------------------------------

## 6. ¿Debemos usar IConfiguration en todos lados?

No.

Podríamos hacer esto:

``` csharp
public class TaskService
{
    private readonly IConfiguration _configuration;

    public TaskService(IConfiguration configuration)
    {
        _configuration = configuration;
    }
}
```

Pero si cada servicio empieza a conocer todas las claves de
configuración:

``` text
TaskManagement:DefaultPageSize
TaskManagement:MaxPageSize
Application:Name
Application:Version
...
```

terminamos distribuyendo detalles de configuración por toda la
aplicación.

Para solucionar esto existe el:

# Options Pattern

------------------------------------------------------------------------

## 7. Options Pattern

El Options Pattern permite representar una sección de configuración
mediante una clase fuertemente tipada.

Creamos:

``` text
Options/
    TaskManagementOptions.cs
```

Con:

``` csharp
namespace TaskManagement.Api.Options;

public class TaskManagementOptions
{
    public int DefaultPageSize { get; set; }

    public int MaxPageSize { get; set; }
}
```

Ahora tenemos una representación fuertemente tipada de:

``` json
"TaskManagement": {
  "DefaultPageSize": 20,
  "MaxPageSize": 100
}
```

------------------------------------------------------------------------

## 8. Registrar Options

En `Program.cs`:

``` csharp
builder.Services.Configure<TaskManagementOptions>(
    builder.Configuration.GetSection("TaskManagement"));
```

Estamos diciendo:

> Toma la sección `TaskManagement` de la configuración y conviértela en
> `TaskManagementOptions`.

Podemos imaginar:

``` text
appsettings.json
       ↓
TaskManagement
       ↓
TaskManagementOptions
       ↓
Dependency Injection
```

------------------------------------------------------------------------

## 9. IOptions`<T>`{=html}

Ahora un servicio puede recibir:

``` csharp
IOptions<TaskManagementOptions>
```

Por ejemplo:

``` csharp
using Microsoft.Extensions.Options;
using TaskManagement.Api.Options;

public class TaskService : ITaskService
{
    private readonly List<TaskItem> _tasks = [];
    private readonly TaskManagementOptions _options;

    public TaskService(
        IOptions<TaskManagementOptions> options)
    {
        _options = options.Value;
    }
}
```

Ahora podemos acceder a:

``` csharp
_options.DefaultPageSize
```

y:

``` csharp
_options.MaxPageSize
```

sin conocer los nombres de las claves de configuración.

------------------------------------------------------------------------

## 10. Aplicándolo al proyecto

En el módulo 3 agregamos paginación.

Podemos tener:

``` text
DefaultPageSize = 20
MaxPageSize = 100
```

Nuestro servicio puede aplicar:

``` csharp
var effectivePageSize =
    pageSize ?? _options.DefaultPageSize;

effectivePageSize = Math.Min(
    effectivePageSize,
    _options.MaxPageSize);
```

Esto significa:

Si el cliente no envía `pageSize`:

``` text
20
```

Si envía:

``` text
50
```

usamos:

``` text
50
```

Si envía:

``` text
500
```

aplicamos el máximo:

``` text
100
```

------------------------------------------------------------------------

## 11. IOptionsSnapshot y IOptionsMonitor

Existen varias interfaces relacionadas.

### IOptions`<T>`{=html}

``` csharp
IOptions<TaskManagementOptions>
```

Es la opción más sencilla.

Es adecuada para configuraciones que no necesitamos reevaluar durante la
ejecución de una request.

### IOptionsSnapshot`<T>`{=html}

``` csharp
IOptionsSnapshot<TaskManagementOptions>
```

Está orientada a escenarios donde las opciones pueden ser reevaluadas
por request.

Es especialmente útil en aplicaciones web.

### IOptionsMonitor`<T>`{=html}

``` csharp
IOptionsMonitor<TaskManagementOptions>
```

Permite observar cambios en las opciones cuando el provider de
configuración admite reload.

Por ejemplo:

``` csharp
_optionsMonitor.CurrentValue
```

Para nuestro proyecto utilizaremos inicialmente:

``` csharp
IOptions<T>
```

No necesitamos introducir complejidad adicional todavía.

------------------------------------------------------------------------

## 12. Environments

ASP.NET Core trabaja con ambientes.

Los más habituales son:

``` text
Development
Staging
Production
```

Podemos saber el ambiente mediante:

``` text
ASPNETCORE_ENVIRONMENT
```

Por ejemplo:

``` text
Development
```

------------------------------------------------------------------------

## 13. appsettings por ambiente

Podemos tener:

``` text
appsettings.json
appsettings.Development.json
appsettings.Staging.json
appsettings.Production.json
```

Por ejemplo:

### appsettings.json

``` json
{
  "TaskManagement": {
    "DefaultPageSize": 20,
    "MaxPageSize": 100
  }
}
```

### appsettings.Development.json

``` json
{
  "TaskManagement": {
    "DefaultPageSize": 10,
    "MaxPageSize": 50
  }
}
```

Cuando ejecutamos en Development, la configuración específica del
ambiente puede sobrescribir los valores generales.

Conceptualmente:

``` text
appsettings.json
       +
appsettings.Development.json
       ↓
configuración final
```

------------------------------------------------------------------------

## 14. ¿Cómo sabe ASP.NET Core el ambiente?

Podemos establecer:

``` bash
ASPNETCORE_ENVIRONMENT=Development
```

En Visual Studio o en perfiles de ejecución normalmente ya existe una
configuración para Development.

También podemos consultar el ambiente mediante:

``` csharp
builder.Environment.EnvironmentName
```

O:

``` csharp
builder.Environment.IsDevelopment()
```

Por ejemplo:

``` csharp
if (builder.Environment.IsDevelopment())
{
    // comportamiento específico de desarrollo
}
```

No debemos convertir esto en una forma de distribuir lógica de negocio
por ambiente.

El ambiente debería utilizarse principalmente para infraestructura y
configuración.

------------------------------------------------------------------------

## 15. Variables de entorno

Otra fuente muy importante de configuración son las variables de
entorno.

Por ejemplo:

``` text
TaskManagement__MaxPageSize=50
```

Los dos guiones bajos:

``` text
__
```

representan niveles de configuración.

Por ejemplo:

``` text
TaskManagement__MaxPageSize
```

corresponde a:

``` text
TaskManagement:MaxPageSize
```

Esto es especialmente importante en Docker y Kubernetes.

Podemos tener:

``` text
Container
   ↓
Environment Variable
   ↓
ASP.NET Configuration
```

------------------------------------------------------------------------

## 16. ¿Por qué son importantes las variables de entorno?

Porque permiten cambiar configuración sin modificar el código ni
necesariamente modificar el archivo de configuración.

Por ejemplo:

``` text
Development
MaxPageSize = 50
```

Production:

``` text
MaxPageSize = 100
```

La misma aplicación puede ejecutarse en ambos ambientes.

------------------------------------------------------------------------

## 17. Secrets

Nunca deberíamos colocar secretos reales en:

``` text
appsettings.json
```

Por ejemplo:

``` json
{
  "ConnectionStrings": {
    "Default": "Server=...;Password=123456"
  }
}
```

No es una buena práctica si ese archivo termina versionado en Git.

Ejemplos de información sensible:

-   passwords;
-   API keys;
-   tokens;
-   connection strings con credenciales;
-   client secrets.

------------------------------------------------------------------------

## 18. User Secrets

Para desarrollo local podemos utilizar:

``` bash
dotnet user-secrets init
```

Esto agrega soporte para User Secrets al proyecto.

Luego podemos guardar:

``` bash
dotnet user-secrets set "ConnectionStrings:Default" "..."
```

Y consultar:

``` bash
dotnet user-secrets list
```

La ventaja es que el secreto no necesita estar dentro del repositorio.

------------------------------------------------------------------------

## 19. Connection Strings

En el próximo módulo utilizaremos una base de datos.

Podemos preparar ahora la configuración:

``` json
{
  "ConnectionStrings": {
    "Default": "..."
  }
}
```

Podemos obtenerla con:

``` csharp
var connectionString =
    builder.Configuration.GetConnectionString("Default");
```

Pero nuevamente:

> La connection string no debería contener credenciales reales dentro de
> un archivo versionado.

En desarrollo podemos utilizar User Secrets.

En producción normalmente utilizaremos mecanismos de secretos
proporcionados por la infraestructura.

------------------------------------------------------------------------

## 20. Configuration + DI

Hasta ahora vimos dos conceptos:

``` text
Dependency Injection
Configuration
```

Ahora podemos ver cómo se relacionan.

Configuration puede proporcionar datos.

DI puede proporcionar objetos.

Por ejemplo:

``` text
appsettings.json
       ↓
Configuration
       ↓
Options
       ↓
Dependency Injection
       ↓
TaskService
```

Esto es muy importante porque permite que nuestros servicios no tengan
que preocuparse por saber de dónde viene la configuración.

------------------------------------------------------------------------

## 21. Estructura del proyecto

Podemos organizar:

``` text
TaskManagement.Api/
│
├── Controllers/
│   └── TasksController.cs
│
├── Options/
│   └── TaskManagementOptions.cs
│
├── Services/
│   ├── ITaskService.cs
│   └── TaskService.cs
│
├── Domain/
│   └── TaskItem.cs
│
├── appsettings.json
├── appsettings.Development.json
└── Program.cs
```

Todavía es una estructura sencilla.

No queremos introducir capas adicionales sin necesidad.

------------------------------------------------------------------------

## 22. Configuración completa

Nuestro `appsettings.json` puede quedar:

``` json
{
  "Application": {
    "Name": "Task Management API",
    "Version": "1.0"
  },
  "TaskManagement": {
    "DefaultPageSize": 20,
    "MaxPageSize": 100
  }
}
```

Creamos:

``` text
Options/TaskManagementOptions.cs
```

``` csharp
namespace TaskManagement.Api.Options;

public class TaskManagementOptions
{
    public int DefaultPageSize { get; set; }

    public int MaxPageSize { get; set; }
}
```

Y en `Program.cs`:

``` csharp
using TaskManagement.Api.Options;

builder.Services.Configure<TaskManagementOptions>(
    builder.Configuration.GetSection("TaskManagement"));
```

------------------------------------------------------------------------

## 23. Validación de Options

Nuestra configuración tiene reglas.

Por ejemplo:

``` text
DefaultPageSize > 0
MaxPageSize >= DefaultPageSize
```

Una configuración como:

``` json
{
  "TaskManagement": {
    "DefaultPageSize": 100,
    "MaxPageSize": 20
  }
}
```

no tiene demasiado sentido.

Más adelante podemos utilizar validación de Options para detectar este
tipo de errores al iniciar la aplicación.

Por ahora alcanza con entender el concepto:

> La configuración también tiene reglas y puede necesitar validación.

------------------------------------------------------------------------

## 24. No mezclar configuración con lógica de negocio

Evitemos cosas como:

``` csharp
if (_configuration["Environment"] == "Production")
{
    // lógica de negocio
}
```

Preferimos:

``` text
Configuration
      ↓
Infrastructure / Application configuration
      ↓
servicios
```

La lógica de negocio debería depender de comportamientos y contratos, no
de strings que representan ambientes.

------------------------------------------------------------------------

## 25. Ejemplo completo del flujo

Tenemos:

``` json
{
  "TaskManagement": {
    "DefaultPageSize": 20,
    "MaxPageSize": 100
  }
}
```

↓

``` csharp
builder.Configuration
    .GetSection("TaskManagement")
```

↓

``` csharp
TaskManagementOptions
```

↓

``` csharp
IOptions<TaskManagementOptions>
```

↓

``` csharp
TaskService
```

↓

``` csharp
_options.MaxPageSize
```

Esta cadena es una de las ideas más importantes del módulo.

------------------------------------------------------------------------

## 26. Analogía con Angular

Podemos compararlo conceptualmente:

  -----------------------------------------------------------------------
  Angular                             ASP.NET Core
  ----------------------------------- -----------------------------------
  `environment.ts`                    `appsettings.json`

  `environment.prod.ts`               `appsettings.Production.json`

  `providers`                         DI container

  `InjectionToken`                    Options / configuración tipada

  `inject()`                          constructor injection

  environment variables del           environment variables del proceso
  build/deploy                        

  service                             service
  -----------------------------------------------------------------------

No son equivalentes técnicamente, pero el modelo mental es parecido:

``` text
configuración
     ↓
inyección
     ↓
servicio
```

------------------------------------------------------------------------

## 27. Ejercicios

### Ejercicio 1 --- Crear Options

Crea:

``` text
ApplicationOptions
```

con:

``` csharp
public string Name { get; set; }

public string Version { get; set; }
```

Configúralo desde:

``` json
"Application": {
  "Name": "Task Management API",
  "Version": "1.0"
}
```

Y obtén la configuración mediante:

``` csharp
IOptions<ApplicationOptions>
```

### Ejercicio 2 --- Configuración por ambiente

Crea:

``` text
appsettings.Development.json
```

con:

``` json
{
  "TaskManagement": {
    "DefaultPageSize": 10,
    "MaxPageSize": 50
  }
}
```

Ejecuta la aplicación en Development y comprueba qué valores recibe
`TaskService`.

### Ejercicio 3 --- Variable de entorno

Configura:

``` text
TaskManagement__MaxPageSize=10
```

y comprueba que el valor sobrescribe el valor definido en
`appsettings.json`.

Esto te ayudará a entender cómo funcionará la configuración cuando
posteriormente ejecutemos la API dentro de Docker.

### Ejercicio 4 --- Endpoint temporal

Crea temporalmente:

``` http
GET /api/config
```

que devuelva:

``` json
{
  "defaultPageSize": 20,
  "maxPageSize": 100
}
```

Puedes utilizar:

``` csharp
[FromServices]
IOptions<TaskManagementOptions>
```

Este endpoint es solamente para aprendizaje.

No expongas secretos mediante una API real.

### Ejercicio 5 --- Connection String

Agrega:

``` json
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=TaskManagement"
  }
}
```

y obtén el valor mediante:

``` csharp
builder.Configuration.GetConnectionString("Default");
```

No necesitas conectar todavía una base de datos.

Eso será parte del módulo siguiente.

------------------------------------------------------------------------

## 28. Preguntas de comprensión

Antes de continuar intenta responder:

1.  ¿Qué problema resuelve Configuration?

2.  ¿Cuál es la diferencia entre `appsettings.json` y
    `appsettings.Development.json`?

3.  ¿Qué representa `TaskManagement:MaxPageSize`?

4.  ¿Qué ventaja tiene Options Pattern frente a usar `IConfiguration`
    directamente en todos los servicios?

5.  ¿Qué representa `TaskManagement__MaxPageSize` como variable de
    entorno?

6.  ¿Por qué no deberíamos guardar passwords en Git?

7.  ¿Para qué sirven User Secrets?

8.  ¿Qué relación existe entre Configuration y Dependency Injection?

------------------------------------------------------------------------

## 29. Estado del proyecto

Al terminar este módulo tenemos:

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

Y ahora aparece una nueva dependencia transversal:

``` text
              Configuration
                    │
                    ↓
            TaskManagementOptions
                    │
                    ↓
                TaskService
```

La aplicación todavía guarda las tareas en memoria.

El siguiente gran salto será persistirlas.

------------------------------------------------------------------------

## 30. Qué aprendimos

En este módulo aprendimos:

-   qué es Configuration;
-   `appsettings.json`;
-   configuración jerárquica;
-   `IConfiguration`;
-   Options Pattern;
-   `IOptions<T>`;
-   ambientes;
-   `appsettings.Development.json`;
-   variables de entorno;
-   User Secrets;
-   connection strings;
-   integración entre Configuration y DI.

La idea fundamental es:

``` text
No hardcodear valores variables.
```

En lugar de:

``` csharp
var maxPageSize = 100;
```

tenemos:

``` text
Configuration
      ↓
Options
      ↓
DI
      ↓
Service
```

------------------------------------------------------------------------

## 31. Próximo módulo

Hasta ahora tenemos:

``` text
HTTP
 ↓
Controller
 ↓
Service
 ↓
Memory
```

En el próximo módulo vamos a sustituir:

``` text
Memory
```

por:

``` text
EF Core
   ↓
DbContext
   ↓
SQL Database
```

Y nuestro **Task Management API** comenzará a comportarse como una
aplicación backend real.
