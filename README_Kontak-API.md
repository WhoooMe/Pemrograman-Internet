# Kontak App — REST API Laravel 11 & Frontend React

**Laporan Praktikum Pemrograman Internet**
Suplemen Pertemuan 5: Hands-On Laravel REST API & SQLite

| | |
|---|---|
| **Nama** | Kelvin |
| **NIM** | 2505551007 |
| **Mata Kuliah** | Pemrograman Internet (B) |

---

## Daftar Isi

1. [Instalasi Project Laravel 11 dan Database SQLite](#1-instalasi-project-laravel-11-dan-database-sqlite)
2. [Setup API & Sanctum Authentication](#2-setup-api--sanctum-authentication)
3. [Database Migration](#3-database-migration)
4. [Pembuatan Controller](#4-pembuatan-controller)
5. [Registrasi API Route](#5-registrasi-api-route)
6. [Integrasi ke Frontend (React + Vite)](#6-integrasi-ke-frontend-react--vite)
7. [Pengujian & Debugging Environment](#7-pengujian--debugging-environment)
8. [Kesimpulan & Dokumentasi Prompt](#8-kesimpulan--dokumentasi-prompt)

---

## Gambaran Umum Aplikasi

Pada kesempatan kali ini penulis hendak membangun sebuah aplikasi Kontak API, di mana aplikasi ini akan melakukan manajemen kontak berbasis web yang dibangun dengan arsitektur terpisah antara backend dan frontend. Backend berperan murni sebagai penyedia REST API menggunakan Laravel 11 dengan database SQLite, sedangkan frontend dibangun sebagai Single Page Application menggunakan React yang dikompilasi oleh Vite. Keduanya berkomunikasi lewat pertukaran data JSON dengan autentikasi berbasis API token dari Laravel Sanctum.

Fitur utama yang tersedia meliputi registrasi dan login pengguna, serta operasi CRUD penuh terhadap data kontak milik pengguna yang sedang masuk. Setiap kontak dapat menyimpan lebih dari satu nomor telepon melalui relasi one-to-many, mengikuti pola relasi `kontak` dan `kontak_phones`. Seluruh data kontak terisolasi per pengguna, sehingga pengguna A tidak dapat melihat maupun mengubah kontak milik pengguna B.

### Teknologi yang Digunakan

| Lapisan | Teknologi | Versi |
|---|---|---|
| Backend Framework | Laravel | 11 |
| Database | SQLite | - |
| Autentikasi | Laravel Sanctum (API Token) | - |
| Frontend Library | React | 19 |
| Build Tool | Vite | 8 |
| Routing Frontend | React Router DOM | 7 |
| HTTP Client | Axios | 1.20 |
| Styling | Tailwind CSS | 4 |

### Struktur Direktori Utama

```
kontak-api/
├── app/
│   ├── Models/
│   │   ├── User.php                 # Model pengguna + trait HasApiTokens
│   │   ├── Contact.php              # Model kontak, relasi hasMany ke phones
│   │   └── ContactPhone.php         # Model nomor telepon, relasi belongsTo
│   └── Http/
│       ├── Controllers/Api/
│       │   ├── AuthController.php   # Register, Login, Logout
│       │   └── ContactController.php# CRUD kontak
│       └── Resources/
│           ├── ContactResource.php      # Format JSON kontak
│           └── ContactPhoneResource.php # Format JSON nomor telepon
├── database/
│   ├── db_kontak.sqlite             # File database SQLite
│   └── migrations/                  # Skema tabel
├── routes/
│   ├── api.php                      # Endpoint REST API
│   └── web.php                      # Catch-all route untuk SPA React
└── resources/
    ├── css/app.css                  # Design token Tailwind v4
    ├── views/welcome.blade.php      # Satu-satunya Blade, wadah React
    └── js/
        ├── app.jsx                  # Entry point + definisi routing
        ├── api.js                   # Konfigurasi Axios + interceptor
        ├── context/                 # AuthContext & ToastContext
        ├── components/              # Komponen reusable
        └── pages/                   # Halaman sesuai route
```

Pada kode di atas dapat dilihat bahwa telah dilakukannya pemisahan tanggung jawab yang jelas antara lapisan data, lapisan logika, dan lapisan tampilan. Folder `app/` berisi seluruh logika backend, sementara `resources/js/` menampung keseluruhan aplikasi React yang berjalan di sisi klien. Pemisahan semacam ini memudahkan penelusuran ketika terjadi kesalahan, karena kita langsung tahu di lapisan mana masalah harus dicari.

---

## 1. Instalasi Project Laravel 11 dan Database SQLite

### 1.1 Membuat Project Laravel

Langkah pertama adalah membuat kerangka project Laravel menggunakan Composer. Perintah `create-project` akan mengunduh seluruh dependensi Laravel beserta struktur folder standarnya ke dalam direktori bernama `kontak-api`.

```bash
composer create-project laravel/laravel kontak-api
cd kontak-api
```

Setelah perintah selesai dijalankan, Composer secara otomatis menyalin file `.env.example` menjadi `.env` dan membuatkan `APP_KEY` baru. Seluruh perintah pada tahap-tahap berikutnya dijalankan dari dalam direktori `kontak-api` ini, karena di situlah file `artisan`, `composer.json`, dan `package.json` berada.

> **Dokumentasi 1**
>
> <img width="959" height="545" alt="Screenshot 2026-09-15 143059" src="https://github.com/user-attachments/assets/c052f351-6e66-4f83-a8d5-f2dc5e7d17b1" />
>
> *Gambar 1.1 Proses instalasi project Laravel 11 melalui Composer.*
>
> Tangkapan layar di atas memperlihatkan Composer sedang mengunduh paket-paket dependensi Laravel satu per satu ke dalam folder `vendor/`. Proses ini biasanya memakan waktu beberapa menit tergantung kecepatan koneksi internet. Baris terakhir yang menandakan keberhasilan adalah pesan pembuatan `APP_KEY` secara otomatis oleh Laravel.

### 1.2 Konfigurasi Database SQLite

Berbeda dengan MySQL yang memerlukan server database terpisah, SQLite menyimpan seluruh data di dalam satu file tunggal. Pendekatan ini sangat cocok untuk praktikum karena tidak perlu instalasi dan konfigurasi server tambahan, serta file databasenya mudah dipindahkan bersama project.

Buat file database kosong terlebih dahulu:

```bash
touch database/db_kontak.sqlite
```

Pada Windows dengan PowerShell, perintah `touch` tidak tersedia sehingga digantikan dengan:

```powershell
New-Item database/db_kontak.sqlite -ItemType File
```

Selanjutnya arahkan konfigurasi koneksi pada file `.env` ke file SQLite yang baru dibuat:

```dotenv
DB_CONNECTION=sqlite
DB_DATABASE=database/db_kontak.sqlite
```

Konfigurasi ini memberi tahu Laravel bahwa driver database yang dipakai adalah SQLite, bukan MySQL bawaan. Nilai `DB_DATABASE` diisi dengan path relatif menuju file database, dihitung dari root project. Pada file `config/database.php`, nilai tersebut dibaca melalui `env('DB_DATABASE', database_path('database.sqlite'))` sehingga pengaturan di `.env` akan menimpa nilai bawaannya.

> **Dokumentasi 2**
>
> <img width="701" height="445" alt="Screenshot 2026-09-16 225550" src="https://github.com/user-attachments/assets/a7927240-fb5e-4a06-8917-e5e15f5743bd" />
>
> *Gambar 1.2 Konfigurasi koneksi SQLite pada file `.env`.*
>
> Gambar ini menampilkan potongan file `.env` yang sudah diubah pada bagian konfigurasi database. Baris `DB_CONNECTION` diisi `sqlite` dan `DB_DATABASE` menunjuk langsung ke file `database/db_kontak.sqlite`. Baris-baris konfigurasi MySQL seperti `DB_HOST`, `DB_PORT`, `DB_USERNAME`, dan `DB_PASSWORD` tidak lagi diperlukan dan bisa dikomentari atau dibiarkan saja.

---

## 2. Setup API & Sanctum Authentication

### 2.1 Menjalankan Perintah `install:api`

Laravel 11 tidak lagi menyertakan file `routes/api.php` secara bawaan. File tersebut baru dibuat ketika kita menjalankan perintah berikut:

```bash
php artisan install:api
```

Satu perintah ini mengerjakan tiga hal sekaligus. Pertama, ia membuat file `routes/api.php` dan mendaftarkannya pada `bootstrap/app.php` sehingga seluruh rute di dalamnya otomatis mendapat prefix `/api`. Kedua, ia memasang paket Laravel Sanctum melalui Composer. Ketiga, ia menyiapkan file migration untuk tabel `personal_access_tokens` yang nantinya menyimpan token milik setiap pengguna.

### 2.2 Memasang Trait `HasApiTokens`

Langkah ini bersifat wajib dan sering terlewat. Tanpa trait `HasApiTokens`, model `User` tidak memiliki method `createToken()`, sehingga proses register maupun login akan gagal dengan error `BadMethodCallException`.

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Laravel\Sanctum\HasApiTokens;   // 1. Import trait

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;   // 2. Pasang trait

    protected function casts(): array
    {
        return [
            'email_verified_at' => 'datetime',
            'password' => 'hashed',
        ];
    }

    /**
     * Relasi: satu User memiliki banyak Contact.
     */
    public function contacts()
    {
        return $this->hasMany(Contact::class);
    }
}
```

Potongan kode di atas menunjukkan dua hal penting sekaligus. Selain memasang `HasApiTokens` untuk kemampuan menerbitkan token, model `User` juga diberi method `contacts()` yang mendefinisikan relasi one-to-many ke model `Contact`. Relasi inilah yang nantinya dipakai di controller untuk memastikan seorang pengguna hanya bisa mengakses kontak miliknya sendiri.

Perlu diperhatikan juga penggunaan cast `'password' => 'hashed'`. Dengan cast ini, Laravel otomatis melakukan hashing setiap kali atribut `password` diisi, sehingga password tidak pernah tersimpan dalam bentuk teks biasa di database.

### 2.3 Cara Kerja Autentikasi Token

Alur autentikasi pada aplikasi ini berjalan sebagai berikut. Ketika pengguna berhasil register atau login, server menerbitkan sebuah token teks acak dan mengirimkannya dalam response JSON. Frontend menyimpan token tersebut di `localStorage` browser, lalu menyertakannya pada header `Authorization: Bearer <token>` di setiap permintaan berikutnya. Middleware `auth:sanctum` di sisi server akan memeriksa token itu dan menolak permintaan dengan status 401 apabila tokennya tidak valid atau sudah dihapus.

> **Dokumentasi 3**
>
> <img width="782" height="221" alt="Screenshot 2026-09-15 143126" src="https://github.com/user-attachments/assets/8289c4dd-d473-46b3-aac3-282c0da6fe2d" />
>
> *Gambar 2.1 Keluaran perintah `php artisan install:api`.*
>
> Tangkapan layar ini memperlihatkan keluaran terminal saat perintah `install:api` dijalankan. Terlihat Laravel meminta konfirmasi untuk menjalankan migration, kemudian melaporkan bahwa file `routes/api.php` berhasil dibuat. Setelah tahap ini, seluruh endpoint yang kita definisikan di `routes/api.php` otomatis dapat diakses dengan awalan `/api`.

---

## 3. Database Migration

### 3.1 Skema Tabel `contacts`

Tabel `contacts` menyimpan data induk dari setiap kontak. Kolom `user_id` berfungsi sebagai foreign key yang menghubungkan kontak dengan pemiliknya, dan diberi `cascadeOnDelete()` agar seluruh kontak ikut terhapus apabila akun penggunanya dihapus.

```php
Schema::create('contacts', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->string('name');
    $table->string('email')->nullable();
    $table->text('address')->nullable();
    $table->timestamps();
});
```

Kolom `name` dibuat wajib diisi karena sebuah kontak tanpa nama tidak memiliki arti. Sebaliknya, `email` dan `address` dibuat `nullable` mengingat tidak semua kontak yang kita simpan memiliki kedua data tersebut. Method `timestamps()` secara otomatis menambahkan kolom `created_at` dan `updated_at` yang berguna untuk menampilkan informasi kapan sebuah kontak dibuat dan terakhir diubah.

### 3.2 Skema Tabel `contact_phones`

Inilah bagian yang mewujudkan relasi one-to-many sesuai slide suplemen. Alih-alih menyimpan satu nomor telepon pada kolom `phone` di tabel `contacts`, nomor telepon dipindahkan ke tabel terpisah sehingga satu kontak dapat memiliki banyak nomor sekaligus.

```php
Schema::create('contact_phones', function (Blueprint $table) {
    $table->id();
    $table->foreignId('contact_id')->constrained('contacts')->cascadeOnDelete();
    $table->enum('type', ['home', 'mobile', 'office'])->default('mobile');
    $table->string('phone_number');
    $table->timestamps();
});
```

Kolom `type` menggunakan tipe enum dengan tiga pilihan tetap, setara dengan kolom `jenis` bernilai `Rumah`, `HP`, dan `Kantor` pada slide suplemen. Penamaan kolom sengaja dibuat dalam bahasa Inggris dengan format snake_case agar konsisten dengan kolom-kolom di tabel `contacts` serta mengikuti konvensi penamaan standar Laravel. Label bahasa Indonesia tetap ditampilkan kepada pengguna, tetapi penerjemahannya dilakukan di sisi frontend, bukan di database.

Sama seperti tabel sebelumnya, foreign key `contact_id` diberi `cascadeOnDelete()`. Artinya ketika sebuah kontak dihapus, seluruh nomor telepon miliknya ikut terhapus secara otomatis tanpa perlu kode tambahan di controller.

### 3.3 Catatan Penyesuaian terhadap Spesifikasi

Spesifikasi tugas menyebutkan bahwa field kontak *minimal* mencakup `id`, `user_id`, `name`, `email`, `phone`, `address`, dan `timestamps`. Pada implementasi ini, kolom `phone` tunggal digantikan oleh tabel relasional `contact_phones` demi mengikuti pola relasi 1:N yang diajarkan pada slide suplemen. Keputusan ini diambil karena kata "minimal" pada spesifikasi membuka ruang untuk pengembangan, sementara relasi 1:N justru merupakan materi inti pertemuan tersebut.

Konsekuensinya, dibuat dua migration tambahan: satu untuk menghapus kolom `phone` dari tabel `contacts`, dan satu lagi untuk membangun ulang tabel `contact_phones` dengan penamaan kolom yang konsisten. Kedua migration tersebut dilengkapi komentar penjelas di dalam kodenya agar alasan perubahan tetap terdokumentasi bagi siapa pun yang membaca repositori ini.

### 3.4 Model Eloquent dan Definisi Relasi

Relasi yang sudah dirancang di level database perlu dideklarasikan ulang di level model agar Eloquent dapat memahaminya.

```php
// app/Models/Contact.php
class Contact extends Model
{
    protected $fillable = ['user_id', 'name', 'email', 'address'];

    public function user()
    {
        return $this->belongsTo(User::class);
    }

    // Relasi 1:N — satu Contact punya banyak nomor telepon
    public function phones()
    {
        return $this->hasMany(ContactPhone::class);
    }
}

// app/Models/ContactPhone.php
class ContactPhone extends Model
{
    protected $table = 'contact_phones';
    protected $fillable = ['contact_id', 'type', 'phone_number'];

    // Relasi Inverse: setiap nomor dimiliki oleh 1 Contact
    public function contact()
    {
        return $this->belongsTo(Contact::class);
    }
}
```

Properti `$fillable` menentukan kolom mana saja yang boleh diisi secara massal melalui method seperti `create()` atau `update()`. Pembatasan ini merupakan lapisan keamanan bawaan Laravel untuk mencegah serangan mass assignment, yaitu ketika penyerang mengirim field tak terduga untuk mengubah data yang seharusnya terlindungi.

Relasi didefinisikan dua arah: `phones()` pada model `Contact` dan `contact()` pada model `ContactPhone`. Dengan begitu kita bisa mengambil daftar nomor dari sebuah kontak, maupun sebaliknya menelusuri kontak pemilik dari sebuah nomor.

### 3.5 Eksekusi Migration

```bash
php artisan migrate
```

Perintah ini membaca seluruh file di folder `database/migrations/` secara berurutan berdasarkan timestamp pada nama filenya, lalu mengeksekusi method `up()` masing-masing. Apabila terjadi kesalahan skema di tengah pengembangan, `php artisan migrate:fresh` dapat digunakan untuk menghapus seluruh tabel dan membangunnya kembali dari nol.

> **Dokumentasi 4**
>
> <img width="787" height="229" alt="Screenshot 2026-09-16 230041" src="https://github.com/user-attachments/assets/85ec03b2-2b02-43cd-8310-0796cd28dbed" />
>
> *Gambar 3.1 Keluaran perintah `php artisan migrate`.*
>
> Gambar ini menampilkan daftar migration yang berhasil dijalankan beserta durasi eksekusi masing-masing dalam milidetik. Terlihat tabel `users`, `cache`, `jobs`, `personal_access_tokens`, `contacts`, dan `contact_phones` terbentuk secara berurutan. Status `DONE` di sisi kanan setiap baris menandakan tidak ada migration yang gagal dieksekusi.

---

## 4. Pembuatan Controller

### 4.1 AuthController

`AuthController` menangani tiga proses autentikasi: registrasi akun baru, login, dan logout.

```php
public function register(Request $request)
{
    $req = $request->validate([
        'name' => ['required', 'string', 'max:255'],
        'email' => ['required', 'email', 'unique:users'],
        'password' => ['required', 'min:8', 'confirmed'],
    ]);

    $user = User::create([
        'name' => $req['name'],
        'email' => $req['email'],
        'password' => Hash::make($req['password'])
    ]);

    $token = $user->createToken('auth_token')->plainTextToken;

    return response()->json([
        'message' => 'Register berhasil',
        'token' => $token,
        'user' => $user
    ], 201);
}
```

Method `validate()` pada baris pertama berperan sebagai penjaga gerbang. Apabila data yang dikirim tidak memenuhi aturan, Laravel otomatis menghentikan eksekusi dan mengembalikan response 422 berisi detail kesalahan per field, tanpa perlu kita menulis pengecekan manual. Aturan `unique:users` memastikan tidak ada dua akun dengan email yang sama, sedangkan `confirmed` mewajibkan adanya field `password_confirmation` yang nilainya identik.

Setelah akun terbentuk, `createToken('auth_token')` menerbitkan token baru dan menyimpan bentuk ter-hash-nya di tabel `personal_access_tokens`. Properti `plainTextToken` mengembalikan token dalam bentuk asli, dan inilah satu-satunya kesempatan token tersebut dapat dibaca karena server hanya menyimpan versi ter-hash.

```php
public function login(Request $request)
{
    $credentials = $request->validate([
        'email' => ['required', 'email'],
        'password' => ['required', 'string'],
    ]);

    if (!Auth::attempt($credentials)) {
        return response()->json(['message' => 'Email atau password salah.'], 422);
    }

    $user = User::where('email', $request->email)->firstOrFail();
    $token = $user->createToken('auth_token')->plainTextToken;

    return response()->json([
        'message' => 'Login berhasil',
        'token' => $token,
        'user' => $user
    ]);
}

public function logout(Request $request)
{
    $request->user()->currentAccessToken()?->delete();

    return response()->json(['message' => 'Logout Berhasil']);
}
```

Pada method `login`, `Auth::attempt()` mencocokkan email dan password yang dikirim dengan data di database, termasuk membandingkan hash password secara aman. Status 422 dipilih alih-alih 401 untuk kasus kredensial salah, dengan alasan praktis: frontend memperlakukan status 401 sebagai penanda token kedaluwarsa dan akan melakukan redirect paksa ke halaman login. Jika login gagal juga mengembalikan 401, pesan kesalahan justru hilang tersapu redirect sebelum sempat terbaca pengguna.

Method `logout` menghapus token yang sedang dipakai saja, bukan seluruh token milik pengguna. Dengan begitu, apabila pengguna login di dua perangkat berbeda, keluar dari satu perangkat tidak memaksa perangkat lainnya ikut keluar.

### 4.2 ContactController

`ContactController` dibuat sebagai API Resource Controller yang menyediakan lima method standar untuk operasi CRUD.

```php
/**
 * GET /api/contacts
 * Menampilkan semua kontak milik user yang sedang login beserta
 * daftar nomor telepon masing-masing (eager load relasi phones).
 */
public function index(Request $request)
{
    $contacts = $request->user()
        ->contacts()
        ->with('phones')
        ->latest()
        ->get();

    return ContactResource::collection($contacts);
}
```

Perhatikan bahwa query dimulai dari `$request->user()->contacts()`, bukan dari `Contact::all()`. Perbedaan ini krusial untuk keamanan: dengan memulai dari relasi milik pengguna yang sedang login, Laravel otomatis menambahkan kondisi `where user_id = ...` sehingga mustahil kontak milik pengguna lain ikut terambil.

Method `with('phones')` melakukan eager loading, yaitu mengambil seluruh nomor telepon dalam satu query tambahan alih-alih satu query per kontak. Tanpa ini, menampilkan 50 kontak akan memicu 51 query ke database, sebuah masalah performa yang dikenal sebagai N+1 query problem.

```php
/**
 * POST /api/contacts
 */
public function store(Request $request)
{
    $data = $request->validate([
        'name' => ['required', 'string', 'max:255'],
        'email' => ['nullable', 'email', 'max:255'],
        'address' => ['nullable', 'string'],
        'phones' => ['nullable', 'array'],
        'phones.*.type' => ['required_with:phones', 'in:home,mobile,office'],
        'phones.*.phone_number' => ['required_with:phones', 'string', 'max:40'],
    ]);

    $contact = $request->user()->contacts()->create([
        'name' => $data['name'],
        'email' => $data['email'] ?? null,
        'address' => $data['address'] ?? null,
    ]);

    if (!empty($data['phones'])) {
        $contact->phones()->createMany($data['phones']);
    }

    return (new ContactResource($contact->load('phones')))
        ->response()
        ->setStatusCode(201);
}
```

Aturan validasi `phones.*.type` menggunakan notasi wildcard, yang berarti aturan tersebut diterapkan ke setiap elemen dalam array `phones`. Notasi ini sangat berguna untuk memvalidasi data berbentuk daftar tanpa perlu menuliskan aturan satu per satu secara manual.

Pembuatan data dilakukan dua tahap. Kontak induk dibuat lebih dulu melalui relasi `contacts()` agar kolom `user_id` terisi otomatis, kemudian `createMany()` menyisipkan seluruh nomor telepon sekaligus dengan `contact_id` yang juga terisi otomatis. Status 201 dikembalikan sebagai penanda standar HTTP bahwa sebuah resource baru berhasil dibuat.

```php
/**
 * PUT/PATCH /api/contacts/{id}
 * Data phones dikirim ulang secara penuh (replace semua nomor lama
 * dengan yang baru) supaya konsisten dengan form edit di frontend.
 */
public function update(Request $request, Contact $contact)
{
    abort_unless($contact->user_id === $request->user()->id, 404);

    $data = $request->validate([ /* aturan sama seperti store */ ]);

    $contact->update([
        'name' => $data['name'],
        'email' => $data['email'] ?? null,
        'address' => $data['address'] ?? null,
    ]);

    if (array_key_exists('phones', $data)) {
        $contact->phones()->delete();
        if (!empty($data['phones'])) {
            $contact->phones()->createMany($data['phones']);
        }
    }

    return new ContactResource($contact->refresh()->load('phones'));
}
```

Baris `abort_unless()` merupakan pemeriksaan kepemilikan yang wajib ada pada method `show`, `update`, dan `destroy`. Tanpa baris ini, siapa pun yang memiliki token valid bisa mengubah kontak milik orang lain hanya dengan menebak ID-nya. Status 404 sengaja dipilih daripada 403, karena 403 secara tidak langsung mengonfirmasi bahwa data dengan ID tersebut memang ada.

Strategi pembaruan nomor telepon menggunakan pendekatan hapus-lalu-buat-ulang. Pendekatan ini dipilih karena jauh lebih sederhana dibanding melacak nomor mana yang ditambah, diubah, atau dihapus, sekaligus sejalan dengan perilaku form di frontend yang selalu mengirim daftar nomor secara lengkap.

### 4.3 API Resource

API Resource berperan sebagai lapisan penerjemah antara model database dan response JSON yang dikirim ke klien.

```php
// app/Http/Resources/ContactResource.php
class ContactResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'address' => $this->address,
            // Relasi 1:N — daftar nomor telepon milik kontak ini
            'phones' => ContactPhoneResource::collection($this->whenLoaded('phones')),
            'created_at' => $this->created_at,
            'updated_at' => $this->updated_at,
        ];
    }
}
```

Manfaat utama pendekatan ini adalah kontrol penuh atas struktur JSON yang keluar. Kolom `user_id` sengaja tidak disertakan karena informasi tersebut tidak berguna bagi frontend dan justru membocorkan detail internal database. Method `whenLoaded('phones')` memastikan kunci `phones` hanya muncul apabila relasinya benar-benar sudah di-eager-load, sehingga tidak memicu query tambahan yang tidak disengaja.

Perlu diketahui bahwa API Resource membungkus hasilnya di dalam kunci `data`. Itulah sebabnya di sisi frontend kita mengakses `res.data.data` alih-alih `res.data` secara langsung.

---

## 5. Registrasi API Route

Seluruh endpoint didefinisikan pada file `routes/api.php` dan otomatis mendapat prefix `/api`.

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\Api\AuthController;
use App\Http\Controllers\Api\ContactController;

// Public Routes
Route::post('/register', [AuthController::class, 'register']);
Route::post('/login', [AuthController::class, 'login']);

// Protected Routes (Bearer Token Sanctum)
Route::middleware('auth:sanctum')->group(function () {
    Route::post('/logout', [AuthController::class, 'logout']);
    Route::apiResource('contacts', ContactController::class);
});

Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```

Rute dibagi menjadi dua kelompok berdasarkan kebutuhan autentikasi. Endpoint `register` dan `login` harus dapat diakses publik, sebab pada titik tersebut pengguna memang belum memiliki token. Seluruh endpoint lainnya dibungkus middleware `auth:sanctum` yang akan menolak permintaan tanpa token valid dengan status 401.

Penggunaan `Route::apiResource()` merupakan bentuk ringkas dari pendefinisian lima rute CRUD sekaligus. Satu baris tersebut setara dengan menulis lima `Route::get`, `Route::post`, `Route::put`, dan `Route::delete` secara manual, sekaligus menjamin penamaan rute mengikuti konvensi RESTful Laravel.

### Daftar Endpoint

| Method | Endpoint | Deskripsi | Auth |
|---|---|---|:---:|
| `POST` | `/api/register` | Mendaftarkan akun baru, mengembalikan token | ✗ |
| `POST` | `/api/login` | Masuk, mengembalikan token | ✗ |
| `POST` | `/api/logout` | Menghapus token yang sedang aktif | ✓ |
| `GET` | `/api/contacts` | Menampilkan seluruh kontak milik user | ✓ |
| `POST` | `/api/contacts` | Menambah kontak baru | ✓ |
| `GET` | `/api/contacts/{id}` | Menampilkan detail satu kontak | ✓ |
| `PUT` | `/api/contacts/{id}` | Memperbarui data kontak | ✓ |
| `DELETE` | `/api/contacts/{id}` | Menghapus kontak beserta nomornya | ✓ |

### Catch-All Route untuk SPA

Karena routing halaman ditangani React Router di sisi klien, Laravel perlu diberi tahu agar mengembalikan halaman React untuk setiap URL yang bukan endpoint API.

```php
// routes/web.php
Route::get('/', function () {
    return view('welcome');
});

Route::view('/{any}', 'welcome')->where('any', '^(?!api).*$');
```

Pola regex `^(?!api).*$` menggunakan negative lookahead untuk mencocokkan semua URL kecuali yang diawali kata `api`. Tanpa aturan ini, membuka `127.0.0.1:8000/contacts/create` secara langsung atau menekan tombol refresh pada halaman tersebut akan menghasilkan error 404 dari Laravel, karena Laravel tidak mengenali rute yang sebenarnya hanya ada di sisi React.

---

## 6. Integrasi ke Frontend (React + Vite)

Bagian ini merupakan porsi pekerjaan terbesar dalam project. Frontend tidak dibangun sebagai project terpisah, melainkan menumpang di dalam struktur Laravel dan dikompilasi oleh Vite yang sudah tersedia secara bawaan pada Laravel 11.

### 6.1 Arsitektur Frontend

Seluruh aplikasi React berjalan di dalam satu file Blade tunggal, yaitu `resources/views/welcome.blade.php`. File tersebut hanya menyediakan sebuah div kosong dengan id `app` sebagai tempat React melakukan mounting, ditambah pemanggilan aset melalui directive `@vite`.

```blade
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="theme-color" content="#1f6b53">
    <meta name="csrf-token" content="{{ csrf_token() }}">
    <title>Kontak App | Manajemen Kontak</title>

    @viteReactRefresh
    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
<body>
    <div id="app"></div>
</body>
</html>
```

Directive `@viteReactRefresh` wajib diletakkan sebelum `@vite` dan berfungsi menyuntikkan skrip preamble yang dibutuhkan plugin React untuk mengaktifkan Fast Refresh. Tanpa baris ini, aplikasi akan gagal dimuat dengan error yang akan dibahas pada bagian debugging. Directive `@vite` sendiri secara cerdas memilih sumber aset: mengarah ke dev server Vite ketika `npm run dev` berjalan, atau ke file hasil build di `public/build/` ketika mode produksi.

### 6.2 Routing dengan React Router DOM

Definisi seluruh halaman terpusat di `resources/js/app.jsx`. Route dikelompokkan berdasarkan kebutuhan autentikasinya menggunakan dua komponen pembungkus.

```jsx
<Routes>
  {/* Hanya untuk tamu (belum login) */}
  <Route element={<GuestRoute />}>
    <Route path="/login" element={<Login />} />
    <Route path="/register" element={<Register />} />
  </Route>

  {/* Butuh token Sanctum — Navbar & layout utama ada di sini */}
  <Route element={<ProtectedRoute />}>
    <Route path="/" element={<Dashboard />} />
    <Route path="/contacts/create" element={<ContactCreate />} />
    <Route path="/contacts/:id" element={<ContactDetail />} />
    <Route path="/contacts/:id/edit" element={<ContactEdit />} />
    <Route path="/404" element={<NotFound />} />
  </Route>

  <Route path="*" element={<Navigate to="/404" replace />} />
</Routes>
```

Pengelompokan semacam ini membuat aturan akses menjadi eksplisit dan mudah dibaca. `GuestRoute` melempar pengguna yang sudah login ke dashboard apabila mencoba membuka halaman login, sementara `ProtectedRoute` melakukan kebalikannya. Route dengan path `*` di baris terakhir menangkap seluruh URL yang tidak dikenali dan mengarahkannya ke halaman 404 khusus.

```jsx
// components/ProtectedRoute.jsx
export default function ProtectedRoute() {
  const { isAuthenticated } = useAuth();
  const location = useLocation();

  if (!isAuthenticated) {
    return <Navigate to="/login" replace state={{ from: location }} />;
  }

  return (
    <div className="flex min-h-screen flex-col bg-slate-50">
      <Navbar />
      <main className="mx-auto w-full max-w-6xl flex-1 px-4 py-8 sm:px-6">
        <Outlet />
      </main>
      <footer>...</footer>
    </div>
  );
}
```

Selain menjaga akses, `ProtectedRoute` juga merangkap sebagai layout utama aplikasi. Navbar dan footer cukup ditulis satu kali di sini, lalu `<Outlet />` menjadi tempat halaman anak dirender. Pendekatan ini menghilangkan duplikasi yang akan terjadi bila setiap halaman harus memanggil Navbar-nya sendiri.

Detail kecil yang penting adalah `state={{ from: location }}`. Halaman yang hendak dituju pengguna disimpan sebelum redirect, sehingga setelah berhasil login pengguna dikembalikan ke halaman tersebut, bukan selalu ke dashboard.

### 6.3 Konfigurasi Axios dan Interceptor

File `resources/js/api.js` memusatkan seluruh konfigurasi komunikasi dengan API. Dengan pendekatan ini, tidak ada satu pun halaman yang perlu mengurus token secara manual.

```js
export const TOKEN_KEY = 'kontak_token';
export const USER_KEY = 'kontak_user';

const api = axios.create({
    baseURL: '/api',
    headers: { Accept: 'application/json' },
});

/* Interceptor request: sisipkan Bearer Token otomatis di setiap request. */
api.interceptors.request.use((config) => {
    const token = localStorage.getItem(TOKEN_KEY);
    if (token) config.headers.Authorization = `Bearer ${token}`;
    return config;
});
```

Interceptor request berjalan tepat sebelum setiap permintaan dikirim. Ia membaca token dari `localStorage` dan menyisipkannya ke header `Authorization`, sehingga kode di halaman cukup menulis `api.get('/contacts')` tanpa memikirkan autentikasi sama sekali.

```js
/*
 * Interceptor response: kalau token invalid/expired (401), bersihkan sesi
 * lalu lempar user ke /login. Hanya kunci milik aplikasi ini yang dihapus,
 * jadi data localStorage lain tidak ikut terhapus.
 */
api.interceptors.response.use(
    (response) => response,
    (error) => {
        if (error.response?.status === 401) {
            localStorage.removeItem(TOKEN_KEY);
            localStorage.removeItem(USER_KEY);

            const onLoginPage = window.location.pathname === '/login';
            const isLoginRequest = error.config?.url?.includes('/login');

            // Jangan redirect saat user memang sedang gagal login,
            // biar pesan "email atau password salah" tetap terlihat.
            if (!onLoginPage && !isLoginRequest) {
                window.location.href = '/login?expired=1';
            }
        }
        return Promise.reject(error);
    },
);
```

Interceptor response menangani kasus token kedaluwarsa secara terpusat. Ketika server membalas 401, sesi lokal dibersihkan dan pengguna diarahkan kembali ke halaman login dengan parameter `?expired=1`, yang kemudian dibaca halaman Login untuk menampilkan notifikasi bahwa sesi telah berakhir.

Dua pengecekan tambahan di dalamnya penting untuk menghindari perilaku yang membingungkan. Pengecekan `isLoginRequest` mencegah redirect ketika yang gagal justru permintaan login itu sendiri, dan hanya dua kunci milik aplikasi yang dihapus alih-alih mengosongkan seluruh isi `localStorage` domain tersebut.

```js
/** Ubah error Axios jadi satu kalimat yang enak dibaca user. */
export function extractErrorMessage(error) {
    if (error.response) {
        const { status, data } = error.response;
        if (status === 422 && data?.errors) {
            return Object.values(data.errors).flat().join(' ');
        }
        if (status === 404) return 'Data yang kamu cari tidak ditemukan.';
        if (status >= 500) return 'Server sedang bermasalah. Coba lagi beberapa saat lagi.';
        if (data?.message) return data.message;
        return 'Permintaan gagal diproses.';
    }
    if (error.request) return 'Tidak bisa menghubungi server. Periksa koneksi internetmu.';
    return 'Terjadi kesalahan yang tidak diketahui.';
}
```

Fungsi pembantu ini menerjemahkan objek error Axios yang rumit menjadi satu kalimat yang dapat dipahami pengguna awam. Tanpa fungsi ini, setiap halaman harus menulis ulang logika pengecekan status yang sama berulang kali.

### 6.4 Manajemen State dengan Context API

Dua Context dipakai untuk data yang dibutuhkan lintas halaman: `AuthContext` untuk sesi pengguna, dan `ToastContext` untuk notifikasi.

```jsx
export function AuthProvider({ children }) {
  const [user, setUser] = useState(readStoredUser);
  const [token, setToken] = useState(() => localStorage.getItem(TOKEN_KEY));

  useEffect(() => {
    if (token) localStorage.setItem(TOKEN_KEY, token);
    else localStorage.removeItem(TOKEN_KEY);
  }, [token]);

  const login = useCallback(async (email, password) => {
    const res = await api.post('/login', { email, password });
    setToken(res.data.token);
    setUser(res.data.user);
    return res.data;
  }, []);

  const value = useMemo(
    () => ({ user, token, isAuthenticated: Boolean(token), login, register, logout }),
    [user, token, login, register, logout],
  );

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}
```

State awal dibaca langsung dari `localStorage` melalui lazy initializer, sehingga pengguna tetap dalam keadaan login meski halaman di-refresh. Sinkronisasi dua arah dijaga oleh `useEffect`: setiap perubahan pada state token otomatis ditulis atau dihapus dari penyimpanan browser.

Penggunaan `useMemo` pada objek `value` merupakan optimasi yang penting. Tanpanya, objek baru akan terbentuk pada setiap render dan memaksa seluruh komponen yang mengonsumsi context ikut render ulang, meskipun nilai di dalamnya sama sekali tidak berubah.

### 6.5 Komponen Reusable

Sesuai pedoman desain, elemen antarmuka dipecah menjadi komponen-komponen kecil yang dapat dipakai ulang. Berikut daftar lengkapnya.

| Komponen | Fungsi |
|---|---|
| `InputField` | Field form serbaguna: input, textarea, atau select dalam satu komponen, lengkap dengan label dan pesan error per field |
| `PrimaryButton` | Tombol dengan empat varian (primary, danger, ghost, outline), dua ukuran, dan indikator loading |
| `ContactCard` | Kartu kontak untuk tampilan grid di dashboard |
| `Navbar` | Navigasi atas dengan menu hamburger responsif |
| `ContactForm` | Form kontak yang dipakai bersama halaman Tambah dan Edit |
| `PhoneFieldsEditor` | Editor daftar nomor telepon dinamis (relasi 1:N) |
| `ConfirmDialog` | Dialog konfirmasi hapus, dipakai di Dashboard dan Detail |
| `Alert` | Pesan error, sukses, atau info inline |
| `EmptyState` | Tampilan saat data kosong atau pencarian nihil |
| `PageHeader` | Header halaman konsisten (link kembali, judul, aksi) |
| `AuthLayout` | Layout dua kolom untuk halaman Login dan Register |
| `Avatar` | Inisial nama dengan warna deterministik |
| `Spinner` & `ContactCardSkeleton` | Indikator dan placeholder saat memuat data |
| `ProtectedRoute` & `GuestRoute` | Pembungkus route berdasarkan status autentikasi |

Berikut contoh implementasi `PrimaryButton` yang memusatkan seluruh gaya tombol dalam satu berkas.

```jsx
const variants = {
  primary: 'bg-brand-600 text-white shadow-sm hover:bg-brand-700 active:bg-brand-800',
  danger: 'bg-rose-600 text-white shadow-sm hover:bg-rose-700 active:bg-rose-800',
  ghost: 'bg-transparent text-slate-700 border border-slate-300 hover:bg-slate-100',
  outline: 'bg-white text-brand-700 border border-brand-300 hover:bg-brand-50',
};

export default function PrimaryButton({
  children, type = 'button', onClick, loading = false,
  disabled = false, variant = 'primary', size = 'md',
  fullWidth = false, className = '', ...rest
}) {
  return (
    <button
      type={type}
      onClick={onClick}
      disabled={disabled || loading}
      aria-busy={loading || undefined}
      className={`inline-flex items-center justify-center gap-2 rounded-lg font-semibold
        transition disabled:cursor-not-allowed disabled:opacity-60
        ${variants[variant]} ${sizes[size]} ${fullWidth ? 'w-full' : ''} ${className}`}
      {...rest}
    >
      {loading && <svg className="h-4 w-4 animate-spin">...</svg>}
      {children}
    </button>
  );
}
```

Keuntungan pendekatan ini terasa saat ada permintaan perubahan desain. Mengubah sudut lengkung atau warna seluruh tombol di aplikasi cukup dilakukan pada satu objek `variants`, tanpa perlu menyisir puluhan file halaman satu per satu.

Prop `loading` juga menyatukan dua perilaku sekaligus, yaitu menampilkan ikon berputar dan menonaktifkan tombol. Penggabungan ini mencegah bug umum berupa pengguna menekan tombol simpan berkali-kali sehingga data terkirim ganda.

### 6.6 Editor Nomor Telepon Dinamis

Komponen `PhoneFieldsEditor` adalah perwujudan relasi 1:N di sisi antarmuka. Pengguna dapat menambah atau menghapus baris nomor telepon secara bebas sebelum menyimpan.

```jsx
export default function PhoneFieldsEditor({ phones, onChange, errors = {} }) {
  function updatePhone(index, field, value) {
    onChange(phones.map((phone, i) => (i === index ? { ...phone, [field]: value } : phone)));
  }

  function addPhone() {
    onChange([...phones, { type: 'mobile', phone_number: '' }]);
  }

  function removePhone(index) {
    onChange(phones.filter((_, i) => i !== index));
  }
  // ...
}
```

Ketiga fungsi di atas selalu menghasilkan array baru alih-alih memodifikasi array yang ada. Prinsip immutability ini wajib dipatuhi di React, sebab React mendeteksi perubahan state dengan membandingkan referensi objek, bukan isinya.

Komponen ini dirancang sebagai controlled component, artinya ia tidak menyimpan state sendiri melainkan menerima data dan fungsi pengubah dari halaman induk. Dengan begitu, halaman Tambah dan halaman Edit dapat menggunakan komponen yang sama persis meski sumber data awalnya berbeda.

### 6.7 State Visual: Loading, Error, dan Sukses

Pedoman desain mensyaratkan adanya umpan balik visual yang jelas untuk setiap kondisi. Implementasinya tersebar di beberapa titik sebagai berikut.

**Kondisi loading** ditangani dengan dua cara berbeda sesuai konteks. Dashboard menampilkan skeleton berbentuk kartu agar tata letak tidak melompat ketika data akhirnya tiba, sementara halaman detail dan edit menggunakan spinner biasa.

```jsx
{loading ? (
  <div className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-3">
    {Array.from({ length: 6 }).map((_, i) => (
      <ContactCardSkeleton key={i} />
    ))}
  </div>
) : error ? (
  <div className="flex flex-col items-start gap-3">
    <Alert type="error">{error}</Alert>
    <PrimaryButton variant="outline" onClick={fetchContacts}>
      Coba lagi
    </PrimaryButton>
  </div>
) : contacts.length === 0 ? (
  <EmptyState title="Belum ada kontak" />
) : (
  <div className="grid">{/* daftar kartu kontak */}</div>
)}
```

Rangkaian kondisional bertingkat di atas mencakup empat keadaan yang mungkin terjadi pada sebuah daftar data. Urutannya disusun dari yang paling spesifik ke paling umum: sedang memuat, gagal memuat, berhasil tetapi kosong, dan terakhir berhasil dengan data. Pola ini membuat antarmuka tidak pernah menampilkan layar kosong tanpa penjelasan.

**Kondisi error** disampaikan pada dua tingkat. Kesalahan umum ditampilkan lewat komponen `Alert` di bagian atas form, sedangkan kesalahan validasi per field muncul tepat di bawah input yang bersangkutan dengan bingkai berwarna merah.

**Kondisi sukses** menggunakan notifikasi toast yang muncul di pojok kanan atas lalu menghilang otomatis setelah beberapa detik.

```jsx
const showToast = useCallback(
  (message, type = 'success', duration = 3500) => {
    const id = ++idCounter;
    setToasts((prev) => [...prev, { id, message, type }]);
    if (duration) setTimeout(() => removeToast(id), duration);
    return id;
  },
  [removeToast],
);
```

Setiap toast diberi id unik dari penghitung yang terus bertambah, sehingga beberapa notifikasi dapat tampil bersamaan tanpa saling menimpa. Timer penghapusan otomatis dipasang saat toast dibuat, namun pengguna tetap dapat menutupnya lebih cepat secara manual.

### 6.8 Desain Responsif

Antarmuka dirancang dengan pendekatan mobile-first dan diuji pada tiga rentang lebar layar. Pada layar kecil, Navbar menyembunyikan tombol-tombolnya ke dalam menu hamburger, tombol tambah kontak berubah menjadi tombol mengambang di pojok kanan bawah, dan grid kartu kontak menyusut menjadi satu kolom.

```jsx
<div className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-3">
```

Satu baris kelas di atas mengatur perilaku grid pada tiga ukuran layar sekaligus. Secara bawaan grid menggunakan satu kolom, lalu menjadi dua kolom pada lebar `sm` ke atas, dan tiga kolom pada lebar `lg` ke atas. Pendekatan mobile-first semacam ini membuat tampilan di perangkat kecil menjadi kondisi dasar, bukan sekadar pengecualian yang ditambahkan belakangan.

Tombol aksi pada form juga disusun menggunakan `flex-col-reverse` di layar sempit, sehingga tombol utama selalu berada di posisi paling atas dan mudah dijangkau ibu jari.

> **Dokumentasi 5**
>
> <img width="959" height="498" alt="Screenshot 2026-09-16 230318" src="https://github.com/user-attachments/assets/14d3e220-4fbb-4111-959b-4d8f0337fca1" />
>
> *Gambar 6.1 Tampilan dashboard pada layar desktop.*
>
> Gambar ini menampilkan dashboard dengan grid tiga kolom, kolom pencarian di bagian atas, dan Navbar lengkap berisi identitas pengguna. Setiap kartu kontak memuat avatar inisial, nama, email, nomor telepon utama beserta labelnya, dan potongan alamat. Tiga tombol aksi di bagian bawah kartu menyediakan akses cepat ke detail, edit, dan hapus.

> **Dokumentasi 6**
>
> <img width="959" height="499" alt="Screenshot 2026-09-16 230433" src="https://github.com/user-attachments/assets/246f1999-1e73-45f5-aaa3-d9ddcc35504b" />

>
> *Gambar 6.2 Halaman login dengan layout dua kolom.*
>
> Halaman login menggunakan `AuthLayout` yang membagi layar menjadi dua bagian pada perangkat lebar. Panel kiri berisi identitas aplikasi dan deskripsi singkat, sedangkan panel kanan memuat form beserta tautan menuju halaman registrasi. Panel kiri otomatis disembunyikan pada layar kecil supaya form tetap menjadi fokus utama.

> **Dokumentasi 7**
>
> <img width="950" height="499" alt="Screenshot 2026-09-16 230518" src="https://github.com/user-attachments/assets/bd9f9a28-5943-4e27-98b6-362a73b282ad" />
>
> *Gambar 6.3 Form tambah kontak beserta editor nomor telepon dinamis.*
>
> Gambar ini memperlihatkan form dengan field nama, email, alamat, serta bagian nomor telepon yang dapat ditambah secara dinamis. Setiap baris nomor terdiri atas dropdown jenis dan input nomor, dilengkapi tombol hapus di sisi kanan. Tautan tambah nomor di bagian atas memungkinkan pengguna menyisipkan baris baru sebanyak yang dibutuhkan.

> **Dokumentasi 8**
>
> <img width="896" height="292" alt="Screenshot 2026-09-16 230642" src="https://github.com/user-attachments/assets/5721d661-91bf-4e02-9ef3-57321809d916" />
>
> *Gambar 6.4 Notifikasi toast setelah kontak berhasil disimpan.*
>
> Setelah proses penyimpanan berhasil, notifikasi hijau muncul di pojok kanan atas berisi nama kontak yang baru ditambahkan. Notifikasi ini menghilang sendiri setelah beberapa detik, namun tetap dapat ditutup lebih cepat melalui tombol silang. Secara bersamaan, pengguna langsung diarahkan kembali ke dashboard sehingga dapat melihat hasilnya.

> **Dokumentasi 9**
>
> <img width="449" height="496" alt="Screenshot 2026-09-16 230722" src="https://github.com/user-attachments/assets/cf8b531d-4620-4411-a5da-ea938d04e8d8" />
>
> *Gambar 6.5 Dialog konfirmasi sebelum kontak dihapus.*
>
> Dialog ini muncul sebagai lapisan di atas halaman dengan latar belakang yang digelapkan dan sedikit diburamkan. Isi pesannya menyebutkan nama kontak secara eksplisit serta memperingatkan bahwa seluruh nomor telepon miliknya akan ikut terhapus. Dialog dapat ditutup melalui tombol batal, menekan tombol Escape, maupun mengklik area gelap di luar kotak.

---

## 7. Pengujian & Debugging Environment

Bagian ini mendokumentasikan kendala teknis nyata yang ditemui selama pengembangan beserta cara mengatasinya. Pencatatan semacam ini bermanfaat sebagai rujukan apabila masalah serupa terulang di project berikutnya.

### 7.1 Error Operator `&&` pada PowerShell

**Gejala.** Saat menjalankan perintah gabungan dari panduan yang ditulis dalam sintaks bash, PowerShell menolak dengan pesan berikut.

```
At line:1 char:22
+ cp .env.example .env && php artisan key:generate
+                      ~~
The token '&&' is not a valid statement separator in this version.
    + CategoryInfo          : ParserError: (:) [], ParentContainsErrorRecordException
    + FullyQualifiedErrorId : InvalidEndOfLine
```

**Analisis.** Operator `&&` yang berfungsi menjalankan perintah kedua hanya bila perintah pertama sukses merupakan sintaks khas bash pada Linux dan macOS. Windows PowerShell versi 5.1, yang masih menjadi bawaan banyak instalasi Windows, tidak mengenali operator tersebut. Dukungan untuk `&&` baru ditambahkan pada PowerShell 7 ke atas.

**Solusi.** Perintah dipisah menjadi baris-baris terpisah, atau digabung menggunakan titik koma sebagai pemisah khas PowerShell.

```powershell
# Cara 1 — pisah baris (paling aman, berlaku di PowerShell maupun CMD)
cp .env.example .env
php artisan key:generate

# Cara 2 — pakai titik koma (khusus PowerShell)
cp .env.example .env; php artisan key:generate
```

Perlu dicatat bahwa `;` pada PowerShell tidak sepenuhnya setara dengan `&&`. Titik koma menjalankan perintah berikutnya tanpa peduli apakah perintah sebelumnya berhasil atau gagal, sehingga untuk perintah yang saling bergantung, memisahkannya per baris tetap menjadi pilihan yang lebih aman.

### 7.2 Error `@vitejs/plugin-react can't detect preamble`

**Gejala.** Setelah seluruh proses instalasi selesai dan kedua server dijalankan, halaman `127.0.0.1:8000` terbuka dalam keadaan kosong sepenuhnya. Konsol browser menampilkan pesan berikut.

```
Uncaught Error: @vitejs/plugin-react can't detect preamble. Something is wrong.
    at AuthContext.jsx:68:1
```

**Analisis.** Plugin `@vitejs/plugin-react` membutuhkan sebuah skrip pendahulu, yang disebut preamble, untuk mengaktifkan fitur React Fast Refresh. Pada project Laravel, skrip tersebut disuntikkan oleh directive Blade `@viteReactRefresh`. Directive ini tidak tersertakan pada file `welcome.blade.php`, sehingga plugin gagal menemukan preamble yang dicarinya dan menghentikan eksekusi seluruh aplikasi React.

Nama file yang disebut dalam pesan error, yaitu `AuthContext.jsx`, sempat menyesatkan karena mengesankan masalah ada pada file tersebut. Kenyataannya file itu hanyalah modul pertama yang diproses plugin, dan kesalahan sebenarnya berada di lapisan template Blade, bukan pada kode React mana pun.

**Solusi.** Tambahkan directive `@viteReactRefresh` tepat sebelum directive `@vite` pada file `resources/views/welcome.blade.php`.

```blade
@viteReactRefresh
@vite(['resources/css/app.css', 'resources/js/app.js'])
```

Urutan penulisan bersifat wajib: `@viteReactRefresh` harus berada di atas `@vite`, sebab preamble perlu dimuat lebih dahulu sebelum modul React apa pun dieksekusi. Setelah perubahan disimpan, cukup lakukan hard refresh pada browser menggunakan `Ctrl + Shift + R` tanpa perlu menghidupkan ulang kedua server.

> **Dokumentasi 10**
>
> <img width="902" height="515" alt="Screenshot 2026-09-16 174658" src="https://github.com/user-attachments/assets/ac726458-3b92-468b-a2b8-7ac337aba9f7" />
>
> *Gambar 7.1 Pesan error preamble pada konsol DevTools.*
>
> Tangkapan layar ini memperlihatkan halaman yang tampil putih sepenuhnya di sisi kiri, berdampingan dengan panel konsol DevTools di sisi kanan. Pesan error berwarna merah menunjuk ke `AuthContext.jsx` baris 68, meski penyebab sebenarnya berada pada file Blade. Kasus ini menjadi pengingat bahwa lokasi yang disebut pada pesan error tidak selalu merupakan sumber masalah yang sesungguhnya.

### 7.3 Ringkasan Perintah Menjalankan Aplikasi

Seluruh perintah di bawah ini dijalankan dari dalam direktori `kontak-api`.

```powershell
# Persiapan awal (cukup sekali)
composer install
cp .env.example .env
php artisan key:generate
php artisan migrate
npm install
```

Setelah persiapan selesai, aplikasi dijalankan menggunakan dua terminal terpisah yang keduanya berada di direktori yang sama.

```powershell
# Terminal 1 - Vite dev server (mengompilasi React & CSS)
npm run dev

# Terminal 2 - Laravel server (menyajikan API & halaman)
php artisan serve
```

Kedua proses harus tetap berjalan selama pengembangan dan tidak boleh ditutup. Aplikasi diakses melalui alamat dari `php artisan serve`, umumnya `http://127.0.0.1:8000`, bukan melalui alamat yang ditampilkan `npm run dev`. Alasannya, Vite dev server di sini hanya bertugas memasok aset ke Laravel dan tidak menyajikan halaman secara mandiri.

---

## 8. Dokumentasi Prompt

### 8.1 Peran AI dalam Pembangunan Frontend

Tentunya pada bagian frontend project ini penulis menggunakan bantuan AI melalui pendekatan iteratif. Alurnya berjalan dalam empat tahap: penyampaian spesifikasi lengkap di awal, pembangunan struktur dan komponen, perapian serta perbaikan bug, dan terakhir penelusuran kendala saat aplikasi dijalankan.

Pola yang terbukti efektif adalah memberikan konteks selengkap mungkin di prompt pertama, mencakup file project, materi rujukan, dan spesifikasi yang terperinci. Dengan konteks semacam itu, AI dapat memeriksa kode yang sudah ada dan menemukan masalah yang belum disadari, alih-alih sekadar menghasilkan kode baru yang belum tentu nyambung dengan struktur project.

### 8.2 Dokumentasi Prompt yang Digunakan

Berikut prompt-prompt penting yang dipakai sepanjang pembangunan frontend, disusun berurutan sesuai tahapannya.

---

#### Prompt 1 Spesifikasi Awal (Prompt Utama)

**Tahap:** Pembangunan dan perapian struktur frontend
**Lampiran:** `kontak-app.zip`, `Slide_Suplemen_Pertemuan_5_Pemrograman_Internet.html`

```
aku kan ada file .zip kontak api, sekarang coba bantu aku dalam
merapikan frontendnya agar sesuai dengan yang aku maksud seperti di bawah ini:

Spesifikasi Backend (Laravel):
- Buatkan struktur database (Migration) dan Model untuk User dan Contact.
  Field untuk kontak minimal mencakup: id, user_id, name, email, phone,
  address, dan timestamps.
- Gunakan Laravel Sanctum untuk sistem autentikasi (API Token) agar
  endpoint aman.
- Buatkan AuthController untuk proses Register, Login, dan Logout.
- Buatkan ContactController (RESTful) untuk sistem CRUD: GET /api/contacts,
  POST /api/contacts, GET /api/contacts/{id}, PUT /api/contacts/{id},
  DELETE /api/contacts/{id}
- Pastikan semua response dikembalikan dalam format JSON yang rapi
  (menggunakan API Resource jika memungkinkan).

Namun dalam membuat backend ini, sesuaikan dengan slide suplemen yang aku
berikan

Spesifikasi Frontend (React):
- Gunakan React Router DOM untuk navigasi halaman: /register, /login, /
  (Dashboard/View Kontak), /contacts/create, /contacts/{id}, dan
  /contacts/{id}/edit.
- Gunakan Axios untuk konsumsi API. Buatkan konfigurasi Axios interceptor
  untuk menyisipkan Bearer Token secara otomatis pada setiap request, serta
  menangani error (misal: redirect ke login jika token expired).
- Gunakan Tailwind CSS (atau CSS murni yang rapi) untuk styling.

Pedoman Desain UI/UX (Komponen):
- Desain antarmuka harus terlihat modern, clean, dan responsif (mobile-friendly).
- Pisahkan elemen UI menjadi komponen-komponen kecil (Reusable Components)
  seperti: InputField, PrimaryButton, ContactCard (untuk tampilan grid/list),
  dan Navbar.
- Berikan state visual yang jelas (misalnya: status loading saat memuat data,
  pesan error saat validasi form gagal, dan alert/toast success saat kontak
  berhasil ditambah/dihapus).
```

**Mengapa prompt ini efektif.** Prompt ini menyertakan tiga elemen sekaligus, yaitu file project yang sudah ada, materi rujukan berupa slide suplemen, dan spesifikasi yang diuraikan per butir. Kelengkapan konteks tersebut memungkinkan AI memeriksa kode yang sudah ditulis sebelumnya alih-alih memulai dari nol.

---

#### Prompt 2 Klarifikasi Prosedur Menjalankan Aplikasi

**Tahap:** Persiapan menjalankan aplikasi

```
kamu kan bilang ini cara jalanin nya itu aku jalaninnya di terminal dan
berada di dalam folder /kontak-api nya kan?, setelahnya aku baru jalanin
npm run dev dan artisan serve di 2 terminal yang berbeda tapi di folder
yang sama?
```

**Mengapa prompt ini penting.** Prompt ini mengonfirmasi pemahaman sebelum menjalankan perintah, sehingga kekeliruan direktori kerja dapat dihindari sejak awal. Kesalahan semacam ini sering menghasilkan pesan error yang membingungkan dan memakan waktu untuk ditelusuri.

**Hasil yang diperoleh.** Diperoleh kepastian bahwa seluruh perintah dijalankan dari direktori `kontak-api`, serta pemahaman bahwa aplikasi diakses melalui alamat `php artisan serve`, bukan alamat dari `npm run dev`.

---

#### Prompt 3 Penelusuran Error PowerShell

**Tahap:** Debugging environment

```
At line:1 char:22
+ cp .env.example .env && php artisan key:generate
+                      ~~
The token '&&' is not a valid statement separator in this version.
    + CategoryInfo          : ParserError: (:) [], ParentContainsErrorRecordException
    + FullyQualifiedErrorId : InvalidEndOfLine

di urutan ke 2 ada error
```

**Mengapa prompt ini efektif.** Pesan error ditempelkan secara utuh apa adanya, termasuk bagian `CategoryInfo` dan `FullyQualifiedErrorId` yang sering dianggap tidak penting.

**Hasil yang diperoleh.** Teridentifikasi bahwa penyebabnya adalah perbedaan sintaks shell antara bash dan Windows PowerShell 5.1, bukan kesalahan pada kode maupun konfigurasi project.

---

#### Prompt 4 Penelusuran Error Preamble React

**Tahap:** Debugging runtime
**Lampiran:** Tangkapan layar konsol DevTools

```
react-dom_client.js?v=13981f39:15805 Download the React DevTools for a better
development experience: https://react.dev/link/react-devtools
AuthContext.jsx:68 Uncaught Error: @vitejs/plugin-react can't detect preamble.
Something is wrong.
    at AuthContext.jsx:68:1

Ada terjadi error di mana websitenya tidak bisa dibuka, ini pesan error yang ditampilkan di DevTools.
```

**Mengapa prompt ini efektif.** Prompt ini melampirkan tangkapan layar konsol sekaligus menegaskan bahwa solusi sebelumnya belum berhasil. Informasi bahwa upaya pertama gagal sangat membantu karena mempersempit kemungkinan penyebab dan mencegah saran yang berulang.

**Hasil yang diperoleh.** Ditemukan bahwa directive `@viteReactRefresh` hilang dari file `welcome.blade.php`. Penting dicatat bahwa kesalahan ini berasal dari kode hasil bantuan AI pada tahap perapian sebelumnya, yang menegaskan perlunya pengujian langsung terhadap setiap kode yang dihasilkan.

---

## Lampiran: Referensi

1. Slide Suplemen Pertemuan 5 - Tutorial Hands-On Laravel REST API & SQLite, Pemrograman Internet, Teknologi Informasi Universitas Udayana.
2. Dokumentasi Laravel 11 — <https://laravel.com/docs/11.x>
3. Dokumentasi Laravel Sanctum — <https://laravel.com/docs/11.x/sanctum>
4. Dokumentasi React Router — <https://reactrouter.com>
5. Dokumentasi Tailwind CSS v4 (Theme Variables) — <https://tailwindcss.com/docs/theme>
6. Dokumentasi Axios (Interceptors) — <https://axios-http.com/docs/interceptors>

---

*Laporan ini disusun sebagai dokumentasi praktikum mata kuliah Pemrograman Internet, Program Studi Teknologi Informasi, Fakultas Teknik, Universitas Udayana.*
