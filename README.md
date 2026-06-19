# Демо-экзамен КОД 09.02.07-5-2026

## Содержание
- [Установка Laravel](#установка-laravel)
- [База данных](#база-данных)
- [Миграции](#миграции)
- [Модели](#модели)
- [Контроллеры](#контроллеры)
- [Маршруты](#маршруты)
- [Views (Blade)](#views-blade)
- [ 6 модуль ](#6-модуль)

---

## Установка Laravel

```bash
composer create-project laravel/laravel --prefer-dist .
```

---

## База данных

```sql
-- ------------------------------------------------------------
-- 1. Клиенты (из Заказчики.json)
-- ------------------------------------------------------------
CREATE TABLE clients (
    id VARCHAR(20) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    inn VARCHAR(20) DEFAULT NULL,
    address VARCHAR(255) DEFAULT NULL,
    phone VARCHAR(20) DEFAULT NULL,
    salesman BOOLEAN NOT NULL DEFAULT FALSE,
    buyer BOOLEAN NOT NULL DEFAULT FALSE
);

-- ------------------------------------------------------------
-- 2. Номенклатура — единый справочник продукции и материалов
-- type: 'product' = готовая продукция, 'material' = сырьё
-- ------------------------------------------------------------
CREATE TABLE nomenclature (
    id INT PRIMARY KEY AUTO_INCREMENT,
    code VARCHAR(50) DEFAULT NULL,
    name VARCHAR(255) NOT NULL,
    type ENUM('product', 'material') NOT NULL,
    unit VARCHAR(20) NOT NULL DEFAULT 'шт',
    price DECIMAL(10,2) DEFAULT NULL
);

-- ------------------------------------------------------------
-- 3. Спецификации — заголовок (одна спецификация на продукт)
-- ------------------------------------------------------------
CREATE TABLE specifications (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    product_id INT NOT NULL,
    quantity DECIMAL(10,4) NOT NULL DEFAULT 1,
    unit VARCHAR(20) NOT NULL DEFAULT 'шт',
    FOREIGN KEY (product_id) REFERENCES nomenclature(id)
);

-- ------------------------------------------------------------
-- 4. Состав спецификации — какие материалы и в каком количестве
-- ------------------------------------------------------------
CREATE TABLE specification_lines (
    id INT PRIMARY KEY AUTO_INCREMENT,
    specification_id INT NOT NULL,
    material_id INT NOT NULL,
    quantity DECIMAL(10,4) NOT NULL,
    unit VARCHAR(20) NOT NULL DEFAULT 'кг',
    FOREIGN KEY (specification_id) REFERENCES specifications(id),
    FOREIGN KEY (material_id) REFERENCES nomenclature(id)
);

-- ------------------------------------------------------------
-- 5. Заказы — заголовок
-- ------------------------------------------------------------
CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    number INT NOT NULL,
    order_date DATE NOT NULL,
    client_id VARCHAR(20) NOT NULL,
    FOREIGN KEY (client_id) REFERENCES clients(id)
);

-- ------------------------------------------------------------
-- 6. Позиции заказа — продукция, количество, цены
-- ------------------------------------------------------------
CREATE TABLE order_lines (
    id INT PRIMARY KEY AUTO_INCREMENT,
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity DECIMAL(10,4) NOT NULL,
    unit VARCHAR(20) NOT NULL DEFAULT 'шт',
    price DECIMAL(10,2) NOT NULL,
    total DECIMAL(10,2) GENERATED ALWAYS AS (quantity * price) STORED,
    FOREIGN KEY (order_id) REFERENCES orders(id),
    FOREIGN KEY (product_id) REFERENCES nomenclature(id)
);

-- ------------------------------------------------------------
-- 7. Производство — заголовок
-- ------------------------------------------------------------
CREATE TABLE production (
    id INT PRIMARY KEY AUTO_INCREMENT,
    number INT NOT NULL,
    production_date DATE NOT NULL
);

-- ------------------------------------------------------------
-- 8. Выпуск продукции — что было произведено
-- ------------------------------------------------------------
CREATE TABLE production_outputs (
    id INT PRIMARY KEY AUTO_INCREMENT,
    production_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity DECIMAL(10,4) NOT NULL,
    unit VARCHAR(20) NOT NULL DEFAULT 'шт',
    FOREIGN KEY (production_id) REFERENCES production(id),
    FOREIGN KEY (product_id) REFERENCES nomenclature(id)
);

-- ------------------------------------------------------------
-- 9. Расход материалов — что было списано в производство
-- ------------------------------------------------------------
CREATE TABLE production_inputs (
    id INT PRIMARY KEY AUTO_INCREMENT,
    production_id INT NOT NULL,
    material_id INT NOT NULL,
    quantity DECIMAL(10,4) NOT NULL,
    unit VARCHAR(20) NOT NULL DEFAULT 'кг',
    FOREIGN KEY (production_id) REFERENCES production(id),
    FOREIGN KEY (material_id) REFERENCES nomenclature(id)
);


-- ------------------------------------------------------------
-- Импорт данных из Заказчики.json в таблицу clients
-- ------------------------------------------------------------
INSERT INTO clients (id, name, inn, address, phone, salesman, buyer) VALUES
('000000001', 'ООО "Поставка"', NULL, 'г.Пятигорск', '+79198634592', TRUE, TRUE),
('000000002', 'ООО "Кинотеатр Квант"', '26320045123', 'г. Железноводск, ул. Мира, 123', '+79884581555', TRUE, FALSE),
('000000008', 'ООО "Новый JDTO"', '26320045111', 'г. Железноводсу', '+79884581555', TRUE, FALSE),
('000000003', 'ООО "Ромашка"', '4140784214', 'г. Омск, ул. Строителей, 294', '+79882584546', FALSE, TRUE),
('000000009', 'ООО "Ипподром"', '5874045632', 'г. Уфа, ул. Набережная, 37', '+79627486389', TRUE, TRUE),
('000000010', 'ООО "Ассоль"', '2629011278', 'г. Калуга, ул. Пушкина, 94', '+79184572398', FALSE, TRUE);


-- Детализация стоимости по каждой позиции заказа
SELECT
    o.number AS order_number,
    o.order_date,
    c.name AS client_name,
    prod.name AS product_name,
    ol.quantity AS order_quantity,
    unit_costs.unit_cost AS cost_per_unit,
    ol.quantity * unit_costs.unit_cost AS line_total
FROM orders o
JOIN clients c ON c.id = o.client_id
JOIN order_lines ol ON ol.order_id = o.id
JOIN nomenclature prod ON prod.id = ol.product_id
JOIN (
    SELECT
        s.product_id,
        SUM(sl.quantity * n.price) AS unit_cost
    FROM specifications s
    JOIN specification_lines sl ON sl.specification_id = s.id
    JOIN nomenclature n ON n.id = sl.material_id
    GROUP BY s.product_id
) AS unit_costs ON unit_costs.product_id = ol.product_id;
```

---

## Миграции

### create_users_table

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('email')->unique();
            $table->string('password');
            $table->boolean('need_to_change_password')->default(true);
            $table->unsignedTinyInteger('wrong_counter')->default(0);
            $table->boolean('is_admin')->default(false);
            $table->timestamps();
        });

        Schema::create('password_reset_tokens', function (Blueprint $table) {
            $table->string('email')->primary();
            $table->string('token');
            $table->timestamp('created_at')->nullable();
        });

        Schema::create('sessions', function (Blueprint $table) {
            $table->string('id')->primary();
            $table->foreignId('user_id')->nullable()->index();
            $table->string('ip_address', 45)->nullable();
            $table->text('user_agent')->nullable();
            $table->longText('payload');
            $table->integer('last_activity')->index();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('users');
        Schema::dropIfExists('password_reset_tokens');
        Schema::dropIfExists('sessions');
    }
};
```

---

## Модели

### User.php

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use HasFactory, Notifiable;

    protected $fillable = [
        'name',
        'email',
        'password',
        'need_to_change_password',
        'wrong_counter',
        'is_admin',
    ];

    protected $hidden = [
        'password',
        'remember_token',
    ];

    protected function casts(): array
    {
        return [
            'email_verified_at'        => 'datetime',
            'password'                 => 'hashed',
            'need_to_change_password'  => 'boolean',
        ];
    }
}
```

---

## Контроллеры

### Создание контроллеров

```bash
php artisan make:controller ActionsController
php artisan make:controller ViewsController
```

### ActionsController.php

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class ActionsController extends Controller
{
    public function login(Request $request)
    {
            $request->validate([
                'email' => 'required|email',
                'password' => 'required|string|min:6',
            ]);

            if (Auth::attempt($request->only(['email','password']))){
                $user = Auth::user();
                if ($user->wrong_counter === 3) {
                    Auth::logout();

                    return redirect()
                        ->route('login')
                        ->withErrors(['email' => 'Your account is locked due to too many failed login attempts.']);
                }

                if ($user->need_to_change_password) {
                    return redirect()
                    ->route('login.change-password')
                    ->with('status','You need to change your password.');
                }
                return redirect()->route('dashboard');
            }

             if ($user = User::firstWhere('email', $request->input('email'))) {
                if ($user->wrong_counter != 3){
                    $user->wrong_counter++;
                    $user->save();
                }
                if ($user->wrong_counter === 3) {
                    return redirect()
                        ->route('login')
                        ->withErrors(['email' => 'Your account has been locked due to too many failed login attempts.']);
                }
             };

            return redirect()
                ->route('login')
                ->withErrors(['email' => 'The provided credentials do not match our records.'])
                ->withInput($request->only('email'));
    }


    public function changePassword(Request  $request)
    {
        $request->validate([
            'current_password' => 'required|string|min:6|current_password',
            'new_password' => 'required|string|min:6|confirmed:new_password_repeat',
        ]);

        $user = Auth::user();
        $user->password = $request->input('new_password');
        $user->need_to_change_password = false;
        $user->save();

        return redirect()
            ->route('dashboard')
            ->with('status', 'Your password has been changed successfully.');

    }

    public function editUser(User $user, Request $request)
    {
        $request->validate([
            'name' => 'required|string|max:255',
            'email' => "required|email|unique:users,email,{$user->id}",
            'password' => 'nullable|string|min:6',
        ]);

        $user->name = $request->input('name');
        $user->email = $request->input('email');
        if ($request->filled('password')) {
            $user->password = $request->input('password');
        }
        $user->save();

        return redirect()
            ->route('dashboard')
            ->with('status', 'User has been updated successfully.');

    }

    public function createUser(Request $request)
    {
        $request->validate([
            'name' => 'required|string|max:255',
            'email' => 'required|email|unique:users,email',
            'password' => 'required|string|min:6',
        ]);

        $user = new User();
        $user->name = $request->input('name');
        $user->email = $request->input('email');
        $user->password = $request->input('password');
        $user->save();

        return redirect()
            ->route('dashboard')
            ->with('status', 'User has been created successfully.');
    }

    public function deleteUser(User $user)
    {
        if($user->id === Auth::id()){
            return redirect()
                ->route('dashboard')
                ->withErrors(['user' => 'You cannot delete your own account.']);
        }
        $user->delete();
        return redirect()
            ->route('dashboard')
            ->with('status', 'User has been deleted successfully.');
    }

    public function unlockUser(User $user)
    {
        if($user->id === Auth::id()){
            return redirect()
                ->route('dashboard')
                ->withErrors(['user' => 'You cannot lock/unlock your own account.']);
        }
        if ($user->wrong_counter !== 3){
            $user->wrong_counter = 3;
            $user->save();

            return redirect()
                ->route('dashboard')
                ->with('status', 'User  account has been locked successfully.');
        }
        $user->wrong_counter = 0;
        $user->save();
        return redirect()
            ->route('dashboard')
            ->with('status', 'User account has been unlocked successfully.');
        }

    public function logout()
    {
        Auth::logout();
        return redirect()->route('login');
    }
}
```

### ViewsController.php

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;

class ViewsController extends Controller
{
    public function index()
    {
        return view ('index', [
            'users' => User::all()
        ]);
    }

    public function login()
        {
            return view ('login');
        }
    public function logout()
    {
        return view ('logout');
    }

    public function changePassword()
    {
        return view ('change_password');
    }

    public function editUser(?User $user = null)
    {
        return view('user_form' , [
            'user' => $user
        ]);
    }
}
```

---

## Маршруты

### web.php

```php
<?php

use App\Http\Controllers\ViewsController;
use App\Http\Controllers\ActionsController;
use Illuminate\Support\Facades\Route;

Route::get ('/',[ViewsController::class, 'index'])
    ->name ('dashboard')
    ->middleware(['auth']);

Route::get ('/login',[ViewsController::class, 'login'])
    ->name ('login')
    ->middleware(['guest']);
Route::post ('/login',[ActionsController::class, 'login'])
    ->middleware('guest');

Route::get ('/login/change-password',[ViewsController::class, 'changePassword'])
    ->name ('login.change-password')
    ->middleware(['auth']);
Route::post ('/login/change-password',[ActionsController::class, 'changePassword'])
    ->middleware('auth');

Route::get ('/user/{user?}',[ViewsController::class, 'editUser'])
    ->name ('user')
    ->middleware(['auth']);
Route::delete ('/user/{user}',[ActionsController::class, 'deleteUser'])
    ->name ('user.delete')
    ->middleware('auth');
Route::patch ('/user/{user?}',[ActionsController::class, 'unlockUser'])
    ->name ('user.unlock')
    ->middleware('auth');
Route::post ('/user/{user?}',[ActionsController::class, 'createUser'])
    ->name ('user.save')
    ->middleware('auth');
Route::put ('/user/{user}',[ActionsController::class, 'editUser'])
    ->name ('user.update')
    ->middleware('auth');
Route::post ('/logout',[ActionsController::class, 'logout'])
    ->name ('logout')
    ->middleware('auth');
```

---

## Views (Blade)

### login.blade.php

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Login</title>
</head>
<body>
    <form action="{{ route('login') }}" method="POST">
        @csrf
        <div>
            <label for="email">Email:</label>
            <input type="email" id="email" name="email" value="{{ old('email') }}" required>
            @error('email')
                <div class="error">{{ $message }}</div>
            @enderror
        </div>
        <div>
            <label for="password">Password:</label>
            <input type="password" id="password" name="password" required>
            @error('password')
                <div class="error">{{ $message }}</div>
            @enderror
        </div>
        <button type="submit">Login</button>
    </form>
</body>
</html>
```

### change_password.blade.php

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Need to change password</title>
</head>
<body>
    <form action="{{ route('login.change-password') }}" method="POST">
        @csrf
        <div>
            <label for="current_password">Current Password:</label>
            <input type="password" id="current_password" name="current_password" required>
            @error('current_password')
                <div class="error">{{ $message }}</div>
            @enderror
        </div>
        <div>
            <label for="new_password">New Password:</label>
            <input type="password" id="new_password" name="new_password" required>
            @error('new_password')
                <div class="error">{{ $message }}</div>
            @enderror
        </div>
        <div>
            <label for="new_password_repeat">Repeat New Password:</label>
            <input type="password" id="new_password_repeat" name="new_password_repeat" required>
        </div>
        <button type="submit">Change Password</button>
    </form>
</body>
</html>
```

### index.blade.php

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Dashboard</title>
</head>
<body>
    <h1>Welcome to the Dashboard</h1>
    <h2>List of Users</h2>
    <table border = '1'>
        <thead>
            <tr>
                <th>Name</th>
                <th>Email</th>
                <th>Need to change password</th>
                <th>User is blocked</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            @foreach ($users as $user)
                <tr>
                    <td>{{ $user->name }}</td>
                    <td>{{ $user->email }}</td>
                    <td>{{ $user->need_to_change_password ? 'Yes' : 'No' }}</td>
                    <td>{{ $user->wrong_counter === 3 ? 'Yes' : 'No' }}</td>
                    <td>
                       <a href = "{{ route('user', ['user' => $user->id]) }}">Edit</a>
                       <form action="{{ route('user.delete', ['user' => $user->id]) }}" method="POST" style="display:inline;">
                            @csrf
                            @method('DELETE')
                            <button type="submit">Delete</button>
                        </form>
                        <form action="{{ route('user.unlock', ['user' => $user->id]) }}" method="POST" style="display:inline;">
                            @csrf
                            @method('PATCH')
                            <button type="submit">{{ $user->wrong_counter === 3 ? 'Unlock' : 'Block' }}</button>
                        </form>
                    </td>
                </tr>
            @endforeach
        </tbody>
        <tfoot>
            <tr>
                <td colspan="5">
                    <a href="{{ route('user') }}">Create New User</a>
                    <form action="{{ route('logout') }}" method="POST" style="display:inline;">
                        @csrf
                        <button type="submit">Logout</button>
                    </form>
                </td>

            </tr>
        </tfoot>
    </table>
    @if(session('message'))
        <div class="status">
            {{ session('message') }}
        </div>
    @endif
</body>
</html>
```

### user_form.blade.php

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>{{ $user ? 'Edit user' : 'Create user' }}</title>
</head>
<body>
    <h1>{{ $user ? 'Edit user' : 'Create user' }}</h1>
    <form action="{{ $user ? route('user.update', $user->id) : route('user.save') }}" method="POST">
        @csrf
        @if ($user)
            @method('PUT')
        @endif
        <div>
            <label for="name">Name:</label>
            <input type="text" id="name" name="name" value="{{ old('name', $user->name ?? '') }}" required>
        </div>
        <div>
            <label for="email">Email:</label>
            <input type="email" id="email" name="email" value="{{ old('email', $user->email ?? '') }}" required>
        </div>
        <div>
            <label for="password">Password:</label>
            <input type="password" id="password" name="password" {{ $user ? '' : 'required' }}>
        </div>
        <button type="submit">{{ $user ? 'Update user' : 'Create user' }}</button>
        <a href="{{ route('dashboard') }}">Cancel</a>
    </form>
    <form>
        @if($errors->any())
          <div>
            <h2>Errors:</h2>
            <ul>
                @foreach ($errors->all() as $error)
                    <li>{{ $error }}</li>
                @endforeach
            </ul>
          </div>
        @endif
    </form>
</body>
</html>
```
### Создание пользователя

### Обновление миграций (Если не создается пользователь)
```
php artisan migrate:refresh
```
```
php artisan tinker
```
```
App\Models\User::create(['email' => 'repev.egor@nttek.ru' , 'password' => '12345678' , 'need_to_change_password' => true , 'name' => 'George Repev']);
```

## 6 модуль

```
composer create-project laravel/laravel --prefer-dist .
```
### Контроллеры
```
php artisan make:controller TestCaseController
Удалить welcome.blade.php, создать index.blade.php
```
### TestCaseController.php:
```
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class TestCaseController extends Controller
{
    public function show(Request $request)
    {
        return view('index');
    }

    public function getData()
    {
        $data = file_get_contents('http://localhost:8080/api/fullName');

        if ($data) {
            $data = json_decode($data)->value;
        }

        return redirect()->route('test-case')->with('value', $data);
    }

    public function checkData(Request $request)
    {
        $value = $request->input('value');

        if (preg_match('/^[А-Яа-яЁё]+ [А-Яа-яЁё]+ [А-Яа-яЁё]+$/u', $value)) {
            return redirect()->route('test-case')
                ->with('value', $value)
                ->with('message', 'ФИО корректно');
        } else {
            return redirect()->route('test-case')
                ->with('value', $value)
                ->with('message', 'ФИО содержит некорректные символы');
        }
    }
}
```

### index.blade.php:
```
<p>
    <form action="{{ route('test-case.get') }}">
        <button type="submit">Получить данные</button>
        <span>{{ session('value') }}</span>
    </form>
</p>

<p>
    <form method="POST" action="{{ route('test-case.check') }}">
        @csrf
        <input type="hidden" name="value" value="{{ session('value') }}">
        <button type="submit">Отправить результаты теста</button>
        <span>{{ session('message') }}</span>
    </form>
</p>
```
### web.php:
```
<?php

use App\Http\Controllers\TestCaseController;
use Illuminate\Support\Facades\Route;

Route::get('/', [TestCaseController::class, 'show'])
    ->name('test-case');

Route::get('/get', [TestCaseController::class, 'getData'])
    ->name('test-case.get');

Route::post('/check', [TestCaseController::class, 'checkData'])
    ->name('test-case.check');
```

### кейсы
```
Получить данные от эмулятора. ФИО содержит только буквы русского алфавита и пробелы (например: Иванов Иван Иванович). Нажать кнопку "Отправить результат теста".
Система подтверждает корректность данных. Валидация пройдена успешно.
Успешно
Получить данные от эмулятора. ФИО содержит запрещённые символы (например: Ив@нов !Иван №1). Нажать кнопку "Отправить результат теста".
Система обнаруживает недопустимые символы. Валидация не пройдена. Выводится сообщение об ошибке.
Успешно
```
### modul 5
```
Проектная документация
Информационная система — Авторизация и управление пользователями
1. Функциональное назначение
Информационная система предназначена для авторизации зарегистрированных пользователей и управления их учётными записями. Система обеспечивает защищённый вход по связке логин/пароль, защиту от перебора паролей с автоматической блокировкой учётной записи, принудительную смену пароля и административный функционал управления пользователями.
2. Структура базы данных
Поле	Описание
id	Уникальный идентификатор
name	Имя пользователя
email	Логин (уникальный)
password	Хэш пароля
need_to_change_password	Флаг принудительной смены пароля при следующем входе
wrong_counter	Счётчик неверных попыток входа
is_admin	Признак роли пользователя
remember_token	Токен сессии
email_verified_at	Дата подтверждения почты

3. Методы системы
3.1. Авторизация
Параметр	Значение
Метод	login
Параметр: email	string, обязательный, формат email
Параметр: password	string, обязательный, минимум 6 символов
Логика	Проверка связки логин/пароль. При совпадении — переход в систему
Условие блокировки	Если счётчик неверных попыток (wrong_counter) равен 3 — доступ запрещён, сообщение о блокировке учётной записи
Условие смены пароля	Если для пользователя установлен флаг need_to_change_password — переход на страницу смены пароля
Ошибка	При неверных данных — сообщение о несоответствии введённых данных записям системы, увеличение счётчика неверных попыток


3.2. Смена пароля
Параметр	Значение
Метод	changePassword
Параметр: current_password	string, обязательный — проверка соответствия текущему паролю
Параметр: new_password	string, обязательный, минимум 6 символов
Параметр: new_password_repeat	string, обязательный — подтверждение нового пароля
Результат	Установка нового пароля, снятие флага принудительной смены пароля

3.3. Создание пользователя
Параметр	Значение
Метод	createUser
Параметр: name	string, обязательный, максимум 255 символов
Параметр: email	string, обязательный, формат email, проверка уникальности в таблице users
Параметр: password	string, обязательный, минимум 6 символов
Результат	Создание новой учётной записи
Ошибка	Сообщение, если введённый логин уже занят

3.4. Редактирование пользователя
Параметр	Значение
Метод	editUser
Параметр: name	string, обязательный, максимум 255 символов
Параметр: email	string, обязательный, формат email, проверка уникальности (с исключением текущего пользователя)
Параметр: password	string, необязательный, минимум 6 символов
Результат	Обновление данных пользователя. Пароль обновляется только если поле заполнено

3.5. Удаление пользователя
Параметр	Значение
Метод	deleteUser
Логика	Удаление учётной записи по идентификатору
Ограничение	Запрет удаления собственной учётной записи
3.6. Блокировка / разблокировка пользователя
Параметр	Значение
Метод	unlockUser
Логика	Переключение статуса блокировки учётной записи (счётчик неверных попыток)
Ограничение	Запрет блокировки/разблокировки собственной учётной записи

3.7. Выход из системы
Параметр	Значение
Метод	logout
Результат	Завершение сессии пользователя

4. Безопасность
Механизм	Описание
Хэширование паролей	Пароль хранится в хэшированном виде, не в открытом тексте
Защита от перебора	Блокировка учётной записи после 3 неверных попыток входа
Защита от ошибок администрирования	Запрет удаления и блокировки собственной учётной записи
Принудительная смена пароля	Возможность установить требование смены пароля при следующем входе
Валидация данных	Проверка обязательности и формата всех вводимых полей на стороне сервера
```
