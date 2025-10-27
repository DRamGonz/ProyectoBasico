# ProyectoBasico
Proyecto api rest básica para practicar y recordar conocimientos.

¡Claro! Aquí tienes todo en **formato Markdown**, listo para guardar en tu repositorio Git. He unificado el **enunciado original replanteado** y la **solución paso a paso** para Laravel 12 con capa de servicios.

---

# API de Gestión de Proyectos y Tareas (Laravel 12)

## 🧩 Enunciado del Ejercicio

### 🎯 Objetivo

Desarrollar una API REST en Laravel para gestionar **usuarios, proyectos y tareas**, implementando **roles básicos**, utilizando una **capa de servicios** para manejar la lógica de negocio y acceso a la base de datos.

---

### 📘 Requisitos funcionales

#### 1️⃣ Entidades principales

**Usuarios (`users`)**

* Campos: `name`, `email`, `password`, `role` (`admin`, `project_manager`, `developer`)
* Relaciones:

  * Un `project_manager` puede tener muchos proyectos.
  * Un `developer` puede tener muchas tareas asignadas.

**Proyectos (`projects`)**

* Campos: `name`, `description`, `start_date`, `end_date`, `user_id` (responsable/project_manager)
* Relaciones:

  * Un proyecto tiene muchas tareas.

**Tareas (`tasks`)**

* Campos: `title`, `description`, `status` (`pendiente`, `en_progreso`, `completada`), `project_id`, `assigned_to` (usuario asignado)
* Relaciones:

  * Una tarea pertenece a un proyecto.
  * Una tarea puede estar asignada a un usuario (developer).

---

#### 2️⃣ Relaciones resumidas

* `User (project_manager)` → tiene muchos `Project`.
* `Project` → tiene muchas `Task`.
* `Task` → pertenece a `Project` y puede estar asignada a `User`.

---

#### 3️⃣ Endpoints principales (sin autenticación)

| Endpoint        | Método                 | Descripción       |
| --------------- | ---------------------- | ----------------- |
| `/api/users`    | GET, POST, PUT, DELETE | CRUD de usuarios  |
| `/api/projects` | GET, POST, PUT, DELETE | CRUD de proyectos |
| `/api/tasks`    | GET, POST, PUT, DELETE | CRUD de tareas    |

---

#### 4️⃣ Validaciones importantes

* No se puede eliminar un proyecto si tiene tareas asociadas.
* Solo se puede asignar tareas si existe el usuario y el proyecto.
* Validar campos obligatorios (`name`, `title`, `status`, etc.)

---

#### 5️⃣ Capa de servicios

Cada entidad tendrá un **Service** que encapsula la lógica de base de datos:

```
app/Services/
 ├── UserService.php
 ├── ProjectService.php
 └── TaskService.php
```

Ejemplo de responsabilidades:

* **UserService**: crear, actualizar, eliminar, obtener usuarios.
* **ProjectService**: CRUD de proyectos, validación de tareas antes de eliminar.
* **TaskService**: CRUD de tareas, asignación a usuario, filtros opcionales.

---

#### 6️⃣ Flujo básico

1. Controller recibe la request.
2. Controller pasa los datos al Service correspondiente.
3. Service valida y accede a la base de datos.
4. Controller retorna la respuesta JSON al cliente.

---

#### 7️⃣ Extras opcionales

* Filtros por estado de tareas: `/api/tasks?status=pendiente`
* Paginación: `/api/projects?page=2&per_page=10`
* Preparar base para autenticación y roles en el futuro.

---

# **Solución Paso a Paso en Laravel 12**

---

## Paso 0: Crear el proyecto

```bash
composer create-project laravel/laravel:^12.0 gestion-proyectos
cd gestion-proyectos
php artisan serve
```
También si es la última versión no hace falta indicar el 12: composer create-project laravel/laravel gestion-proyectos

---

## Paso 1: Configurar la base de datos

Editar `.env`:

```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=gestion_proyectos
DB_USERNAME=root
DB_PASSWORD=tu_password
```

Limpiar caché de configuración:

```bash
php artisan config:clear
```

---

## Paso 2: Crear migraciones y modelos

### 1️⃣ Usuario

```bash
php artisan make:model User -m
```

Migración `xxxx_create_users_table.php`:

```php
$table->id();
$table->string('name');
$table->string('email')->unique();
$table->string('password');
$table->enum('role', ['admin','project_manager','developer'])->default('developer');
$table->timestamps();
```

Modelo `User.php`:

```php
protected $fillable = ['name','email','password','role'];
```

---

### 2️⃣ Proyecto

```bash
php artisan make:model Project -m
```

Migración:

```php
$table->id();
$table->string('name');
$table->text('description')->nullable();
$table->date('start_date');
$table->date('end_date')->nullable();
$table->foreignId('user_id')->constrained('users')->onDelete('cascade');
$table->timestamps();
```

Modelo `Project.php`:

```php
protected $fillable = ['name','description','start_date','end_date','user_id'];

public function tasks(){ return $this->hasMany(Task::class); }
public function manager(){ return $this->belongsTo(User::class,'user_id'); }
```

---

### 3️⃣ Tarea

```bash
php artisan make:model Task -m
```

Migración:

```php
$table->id();
$table->string('title');
$table->text('description')->nullable();
$table->enum('status',['pendiente','en_progreso','completada'])->default('pendiente');
$table->foreignId('project_id')->constrained('projects')->onDelete('cascade');
$table->foreignId('assigned_to')->nullable()->constrained('users')->onDelete('set null');
$table->timestamps();
```

Modelo `Task.php`:

```php
protected $fillable = ['title','description','status','project_id','assigned_to'];

public function project(){ return $this->belongsTo(Project::class); }
public function assignedUser(){ return $this->belongsTo(User::class,'assigned_to'); }
```

---

### 4️⃣ Ejecutar migraciones

```bash
php artisan migrate
```

---

## Paso 3: Crear carpeta Services

```bash
mkdir app/Services
```

### UserService.php

```php
namespace App\Services;

use App\Models\User;
use Illuminate\Support\Facades\Hash;

class UserService {
    public function all(){ return User::all(); }
    public function find($id){ return User::findOrFail($id); }
    public function create(array $data){ 
        $data['password'] = Hash::make($data['password']);
        return User::create($data);
    }
    public function update($id,array $data){
        $user = User::findOrFail($id);
        if(isset($data['password'])) $data['password'] = Hash::make($data['password']);
        $user->update($data);
        return $user;
    }
    public function delete($id){ return User::findOrFail($id)->delete(); }
}
```

---

### ProjectService.php

```php
namespace App\Services;

use App\Models\Project;

class ProjectService {
    public function all(){ return Project::with('tasks','manager')->get(); }
    public function find($id){ return Project::with('tasks','manager')->findOrFail($id); }
    public function create(array $data){ return Project::create($data); }
    public function update($id,array $data){
        $project = Project::findOrFail($id);
        $project->update($data);
        return $project;
    }
    public function delete($id){
        $project = Project::findOrFail($id);
        if($project->tasks()->count()>0) throw new \Exception("No se puede eliminar un proyecto con tareas asociadas.");
        $project->delete();
        return true;
    }
}
```

---

### TaskService.php

```php
namespace App\Services;

use App\Models\Task;

class TaskService {
    public function all(){ return Task::with('project','assignedUser')->get(); }
    public function find($id){ return Task::with('project','assignedUser')->findOrFail($id); }
    public function create(array $data){ return Task::create($data); }
    public function update($id,array $data){
        $task = Task::findOrFail($id);
        $task->update($data);
        return $task;
    }
    public function delete($id){ return Task::findOrFail($id)->delete(); }
}
```

---

## Paso 4: Crear controladores

```bash
php artisan make:controller UserController --api
php artisan make:controller ProjectController --api
php artisan make:controller TaskController --api
```

Ejemplo UserController:

```php
protected $service;
public function __construct(UserService $service){ $this->service = $service; }

public function index(){ return response()->json($this->service->all()); }
public function show($id){ return response()->json($this->service->find($id)); }
public function store(Request $request){
    $validated = $request->validate([
        'name'=>'required|string',
        'email'=>'required|email|unique:users',
        'password'=>'required|string|min:6',
        'role'=>'required|in:admin,project_manager,developer'
    ]);
    return response()->json($this->service->create($validated),201);
}
public function update(Request $request,$id){ ... }
public function destroy($id){ ... }
```

---

## Paso 5: Rutas (`routes/api.php`)

```php
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\UserController;
use App\Http\Controllers\ProjectController;
use App\Http\Controllers\TaskController;

Route::apiResource('users',UserController::class);
Route::apiResource('projects',ProjectController::class);
Route::apiResource('tasks',TaskController::class);
```

---

## Paso 6: Configurar `bootstrap/app.php`

```php
return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        api: __DIR__.'/../routes/api.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up'
    )
    ->withMiddleware(fn($middleware) => null)
    ->withExceptions(fn($exceptions) => null)
    ->create();
```

---

## Paso 7: Probar en Postman

### Crear usuario

`POST http://127.0.0.1:8000/api/users`

```json
{
  "name
```

Perfecto 🔥
Aquí tienes una **guía completa y actualizada para Laravel 12**, paso a paso, con todo el proceso de **autenticación (Sanctum)** + **autorización (Policies)**
💾 Formato **Markdown (.md)** — lista para documentar o compartir con tu equipo.

---

# 🛡️ Autenticación y Autorización en Laravel 12 (Sanctum + Policies)

> Guía práctica para proteger APIs con Laravel 12, sin usar `Kernel.php`,
> aprovechando la nueva configuración en `bootstrap/app.php`.

---

## ⚙️ 1. Instalación y configuración de Sanctum

Sanctum es el sistema recomendado por Laravel para autenticar APIs usando **tokens personales**.

### 🔹 Instalar Sanctum

```bash
composer require laravel/sanctum
```

### 🔹 Publicar configuración y migraciones

```bash
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
php artisan migrate
```

Esto creará la tabla `personal_access_tokens`.

---

## ⚙️ 2. Configurar Sanctum en `app/Http/Kernel.php` (Laravel 11 o anterior)

En **Laravel 12**, el `Kernel` ya **no se usa directamente**.
En su lugar, configuramos los middlewares en `bootstrap/app.php`.

### 📁 `bootstrap/app.php`

Agrega el middleware de Sanctum:

```php
use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        api: __DIR__.'/../routes/api.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware): void {
        // 🔹 Middleware globales o alias
        $middleware->api(prepend: [
            \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
        ]);

        // 🔸 Alias personalizados
        $middleware->alias([
            'role' => \App\Http\Middleware\RoleMiddleware::class,
        ]);
    })
    ->withExceptions(function (Exceptions $exceptions): void {
        //
    })
    ->create();
```

---

## 👤 3. Configurar el modelo `User`

Asegúrate de que el modelo `User` use el trait de Sanctum.

### 📁 `app/Models/User.php`

```php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;

    protected $fillable = [
        'name',
        'email',
        'password',
        'role',
    ];

    protected $hidden = [
        'password',
        'remember_token',
    ];

    protected function casts(): array
    {
        return [
            'email_verified_at' => 'datetime',
            'password' => 'hashed',
        ];
    }
}
```

---

## 🔐 4. Crear endpoints de autenticación

Creamos un **AuthController** para manejar login, registro y logout.

### 📁 `app/Http/Controllers/AuthController.php`

```php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use App\Models\User;

class AuthController extends Controller
{
    // 🔹 Registro
    public function register(Request $request)
    {
        $validated = $request->validate([
            'name' => 'required|string',
            'email' => 'required|email|unique:users',
            'password' => 'required|min:6',
        ]);

        $user = User::create([
            'name' => $validated['name'],
            'email' => $validated['email'],
            'password' => $validated['password'],
            'role' => 'user', // Por defecto
        ]);

        return response()->json(['user' => $user], 201);
    }

    // 🔹 Login
    public function login(Request $request)
    {
        $user = User::where('email', $request->email)->first();

        if (! $user || ! Hash::check($request->password, $user->password)) {
            return response()->json(['message' => 'Credenciales inválidas'], 401);
        }

        $token = $user->createToken('api_token')->plainTextToken;

        return response()->json([
            'token' => $token,
            'user' => $user,
        ]);
    }

    // 🔹 Logout
    public function logout(Request $request)
    {
        $request->user()->currentAccessToken()->delete();

        return response()->json(['message' => 'Sesión cerrada correctamente']);
    }
}
```

---

## 🌐 5. Rutas API protegidas

### 📁 `routes/api.php`

```php
use App\Http\Controllers\AuthController;
use App\Http\Controllers\ProjectController;
use App\Http\Controllers\TaskController;
use App\Http\Controllers\UserController;
use App\Http\Controllers\ReportController;
use Illuminate\Support\Facades\Route;

// 🧱 Autenticación
Route::post('/register', [AuthController::class, 'register']);
Route::post('/login', [AuthController::class, 'login']);

// 🔐 Rutas protegidas
Route::middleware('auth:sanctum')->group(function () {

    Route::post('/logout', [AuthController::class, 'logout']);

    Route::apiResource('users', UserController::class);
    Route::apiResource('projects', ProjectController::class);
    Route::apiResource('tasks', TaskController::class);

    // Reportes
    Route::get('/reports/tasks-summary', [ReportController::class, 'tasksSummary'])
        ->middleware('role:admin,manager');
});
```

---

## 🧱 6. Middleware de roles

### 📁 `app/Http/Middleware/RoleMiddleware.php`

```php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class RoleMiddleware
{
    public function handle(Request $request, Closure $next, ...$roles)
    {
        $user = $request->user();

        if (! $user || ! in_array($user->role, $roles)) {
            return response()->json(['message' => 'No autorizado'], 403);
        }

        return $next($request);
    }
}
```

✅ Registrado como alias en `bootstrap/app.php`:

```php
$middleware->alias([
    'role' => \App\Http\Middleware\RoleMiddleware::class,
]);
```

---

## 🧭 7. Autorización con Policies

Las **Policies** controlan la autorización a nivel de modelo (quién puede modificar, ver o eliminar algo).

### 🔹 Crear una Policy

```bash
php artisan make:policy ProjectPolicy --model=Project
```

### 📁 `app/Policies/ProjectPolicy.php`

```php
namespace App\Policies;

use App\Models\User;
use App\Models\Project;

class ProjectPolicy
{
    public function viewAny(User $user): bool
    {
        return true;
    }

    public function view(User $user, Project $project): bool
    {
        return $user->id === $project->user_id || $user->role === 'admin';
    }

    public function update(User $user, Project $project): bool
    {
        return $user->id === $project->user_id || $user->role === 'admin';
    }

    public function delete(User $user, Project $project): bool
    {
        return $user->role === 'admin';
    }
}
```

### 📁 Registrar la Policy

`AuthServiceProvider`:

```php
namespace App\Providers;

use App\Models\Project;
use App\Policies\ProjectPolicy;
use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;

class AuthServiceProvider extends ServiceProvider
{
    protected $policies = [
        Project::class => ProjectPolicy::class,
    ];

    public function boot(): void
    {
        //
    }
}
```

---

## 🔒 8. Usar Policies en controladores

Ejemplo en `ProjectController`:

```php
public function update(Request $request, Project $project)
{
    $this->authorize('update', $project);

    $project->update($request->all());

    return response()->json($project);
}
```

Laravel verificará automáticamente la `ProjectPolicy`.

---

## 🧠 9. Flujo de autenticación (para Postman)

1. **POST** `/api/register` → crear usuario
2. **POST** `/api/login` → obtener token
3. Añadir el token en el **header**:

   ```
   Authorization: Bearer TU_TOKEN_AQUI
   ```
4. Ya puedes acceder a rutas protegidas (`/projects`, `/tasks`, `/reports/...`).

---

## ✅ Conclusión

Con esta configuración tienes:

* 🔐 Autenticación con **Sanctum**
* 🧱 Control de acceso con **roles personalizados**
* 🧭 Autorización a nivel de modelo con **Policies**
* 🧩 Todo adaptado a **Laravel 12** sin `Kernel.php`

---


