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
php artisan make:controller UserController
php artisan make:controller ProjectController
php artisan make:controller TaskController
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

