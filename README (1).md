# Laporan Code Defense - Nusantara Weather Explorer

**Mata Kuliah:** Teknologi Informasi - Universitas Udayana
**Nama Proyek:** Nusantara Weather Explorer (React + Vite + Tailwind CSS)

---

## 1. Endpoint API & Struktur JSON

Pada aplikasi cuaca ini, terdapat beberapa API yang digunakan sebagai sumber data dari aplikasi tersebut. API tersebut memiliki fungsi yang berbeda, yaitu mengambil data wilayah Indonesia, mengubah nama wilayah menjadi koordinat, dan mengambil data cuaca berdasarkan koordinat. Kemudian, setelah kecamatan dipilih, nama wilayah tersebut dikirim ke Geocoding API untuk mendapatkan koordinat, dan koordinat itu digunakan untuk memanggil Weather API.

### 1.1 API Data Wilayah

Sederhananya, API data wilayah ini digunakan untuk mendapatkan daftar wilayah Indonesia, mulai dari provinsi, kabupaten/kota, hingga kecamatan. Maka dari itu dengan adanya API ini pengguna tidak perlu memasukkan nama wilayah secara manual karena pilihan wilayah sudah disediakan dalam bentuk dropdown. Berikut merupakan beberapa endpoint yang digunakan.

```
https://www.emsifa.com/api-wilayah-indonesia/api/provinces.json
https://www.emsifa.com/api-wilayah-indonesia/api/regencies/{provinceId}.json
https://www.emsifa.com/api-wilayah-indonesia/api/districts/{regencyId}.json
```

```json
[
  {
    "id": "11",
    "name": "Aceh"
  },
  {
    "id": "12",
    "name": "Sumatera Utara"
  }
]
```

_Kode Program 1.1 - Daftar Endpoint dan Contoh Struktur JSON yang Digunakan_

Melalui kode di atas terdapat beberapa endpoint yang digunakan, seperti endpoint yang pertama untuk mengambil seluruh daftar provinsi di Indonesia. Kemudian, endpoint kedua untuk mengambil daftar kabupaten/kota berdasarkan ID provinsi yang dipilih. Terakhir, endpoint untuk mengambil daftar kecamatan berdasarkan ID kabupaten/kota yang dipilih. Pada bagian akhir kode ini juga terdapat struktur JSON yang berisi `id` untuk menentukan wilayah pada tingkat berikutnya dan `name` sebagai nama wilayah yang ditampilkan kepada pengguna.

### 1.2 Geocoding API

Lalu ada Geocoding API yang digunakan setelah pengguna selesai memilih wilayah sampai tingkat kecamatan. API ini berfungsi untuk mengubah nama wilayah yang dipilih menjadi koordinat geografis, yaitu latitude dan longitude. Koordinat tersebut nantinya menjadi informasi penting yang digunakan untuk meminta data cuaca dari Weather API. Berikut merupakan endpoint hingga struktur JSON yang digunakan.

```
https://geocoding-api.open-meteo.com/v1/search
```

```json
{
  "results": [
    {
      "name": "Singaraja",
      "latitude": -8.112,
      "longitude": 115.088,
      "country_code": "ID"
    }
  ]
}
```

_Kode Program 1.2 - Endpoint dan Struktur JSON_

Melalui kode di atas terdapat endpoint yang digunakan bagi API tersebut. Nantinya, akan ada beberapa parameter yang dikirimkan dalam request, seperti `name`, `count`, `language`, `format`. Selain itu, di bagian akhir terdapat juga struktur JSON yang di mana aplikasi tersebut akan mengambil nilai `latitude` dan `longitude`. Aplikasi juga memeriksa `country_code` agar hasil lokasi yang digunakan benar-benar berada di Indonesia.

### 1.3 Weather API

Weather API merupakan API yang digunakan untuk mendapatkan informasi kondisi cuaca berdasarkan koordinat yang telah diperoleh dari proses geocoding. Data yang diambil berupa suhu, kecepatan angin, kode kondisi cuaca, dan waktu pengukuran. Hasil data tersebut kemudian diolah dan ditampilkan dalam bentuk kartu informasi cuaca pada antarmuka aplikasi. Berikut merupakan endpoint dan struktur JSON yang digunakan.

```
https://api.open-meteo.com/v1/forecast
```

```json
{
  "current": {
    "temperature_2m": 27.5,
    "wind_speed_10m": 8.2,
    "weather_code": 2,
    "time": "2026-09-02T16:00"
  },
  "current_units": {
    "temperature_2m": "°C",
    "wind_speed_10m": "km/h"
  }
}
```

_Kode Program 1.3 - Endpoint dan Struktur JSON Dari Weather API_

Pada kode di atas, terdapat endpoint yang digunakan sebagai alamat ketika menerima request dan mengirimkan kembali respons. Selain itu parameter yang akan dikirimkan sendiri itu berupa latitude, longitude, temperatur, kecepatan angin, kode cuaca, dan zona waktu. Hal ini dapat dilihat pada struktur JSON yang terdapat `temperature_2m` digunakan untuk menampilkan suhu, `wind_speed_10m` digunakan untuk menampilkan kecepatan angin, sedangkan `weather_code` digunakan untuk menentukan kondisi cuaca. Nilai `weather_code` kemudian diterjemahkan oleh aplikasi menjadi keterangan yang lebih mudah dipahami pengguna, seperti cerah, berawan, hujan, atau badai petir.

---

## 2. Bedah Code

Pada bagian ini dilakukan pembedahan terhadap kode-kode utama yang membangun logika aplikasi, khususnya penerapan `useEffect`, tiga state wajib (`data`, `loading`, `error`), serta fungsi-fungsi handler yang menjembatani interaksi pengguna dengan proses pengambilan data.

### 2.1 Struktur Komponen

Aplikasi dipecah menjadi beberapa bagian agar tidak monolitik:

- `App.jsx` - komponen induk yang menyimpan state `region` dan menampilkan `RegionSelector`, `LoadingUI`, `ErrorMsg`, atau `WeatherCard` secara kondisional.
- `RegionSelector.jsx` - dropdown bertingkat Provinsi → Kabupaten/Kota → Kecamatan.
- `WeatherCard.jsx` - kartu hasil cuaca (suhu, ikon, angin, waktu).
- `LoadingUI.jsx` - tampilan skeleton/spinner saat data sedang dimuat.
- `ErrorMsg.jsx` - tampilan peringatan saat terjadi kegagalan fetch.
- `hooks/useFetch.js` - custom hook generik untuk fetch data wilayah.
- `hooks/useWeatherByRegion.js` - custom hook untuk rangkaian fetch geocoding → cuaca.
- `utils/api.js` - kumpulan base URL API.
- `utils/weatherCodes.js` - pemetaan kode cuaca WMO ke label dan ikon.

### 2.2 Tiga State Wajib: `data`, `loading`, `error`

Setiap proses fetching di aplikasi ini - baik untuk data wilayah (`useFetch`) maupun data cuaca (`useWeatherByRegion`) - konsisten menerapkan tiga state berikut:

```js
const [data, setData] = useState(null);
const [loading, setLoading] = useState(false);
const [error, setError] = useState("");
```

- **`data`** menampung hasil respons API yang berhasil diambil, digunakan untuk merender `WeatherCard` atau isi dropdown.
- **`loading`** bernilai `true` selama request berlangsung, dipakai untuk menampilkan `LoadingUI` (skeleton shimmer) dan menonaktifkan (`disabled`) elemen `<select>` agar pengguna tidak memicu request ganda.
- **`error`** menyimpan pesan kegagalan dalam bentuk string, dipakai untuk menampilkan `ErrorMsg` dengan UI peringatan merah.

Pola ini diterapkan secara konsisten di kedua custom hook, sehingga komponen UI (`App.jsx`, `RegionSelector.jsx`) cukup membaca ketiga state tersebut tanpa perlu tahu detail implementasi fetch di baliknya.

### 2.3 `useFetch.js` - Hook Generik untuk Data Wilayah

Hook ini digunakan tiga kali oleh `RegionSelector` (untuk provinsi, kabupaten/kota, dan kecamatan). Bagian inti dari implementasinya:

```js
useEffect(() => {
  if (!url) {
    setData(null);
    setError("");
    setLoading(false);
    return;
  }

  const controller = new AbortController();

  async function fetchData() {
    setLoading(true);
    setError("");
    try {
      const response = await fetch(url, { signal: controller.signal });
      if (!response.ok) {
        throw new Error(
          `Gagal mengambil data wilayah (status: ${response.status})`,
        );
      }
      const json = await response.json();
      setData(json);
    } catch (err) {
      if (err.name === "AbortError") return;
      setError(err.message || "Terjadi kesalahan saat mengambil data wilayah");
      setData(null);
    } finally {
      if (!controller.signal.aborted) {
        setLoading(false);
      }
    }
  }

  fetchData();
  return () => controller.abort();
}, [url]);
```

Penjelasan penerapan `useEffect` dan penanganan error pada kode di atas:

- **Guard clause `if (!url)`** - dropdown Kabupaten/Kota dan Kecamatan diberi `url` bernilai `null` selama level di atasnya belum dipilih (`provinceId ? ... : null`). Hook akan berhenti tanpa melakukan fetch apa pun sampai `url` tersedia.
- **Dependency array `[url]`** - inilah yang membuat fetch otomatis berjalan ulang setiap kali `url` berubah. Karena `url` dibentuk dari `provinceId` atau `regencyId`, maka setiap kali Provinsi berganti, `url` untuk Kabupaten ikut berubah, dan `useEffect` otomatis memicu fetch baru.
- **`AbortController`** - dibuat baru setiap kali efek berjalan. Fungsi _cleanup_ (`return () => controller.abort()`) dipanggil React sebelum efek berikutnya dieksekusi atau saat komponen unmount. Ini mencegah _race condition_: jika pengguna mengganti Provinsi dengan cepat, request Kabupaten dari pilihan Provinsi sebelumnya akan dibatalkan sebelum sempat "menimpa" data dari pilihan Provinsi yang baru.
- **`try...catch...finally`** - `fetch()` tidak otomatis melempar error untuk status HTTP 4xx/5xx, sehingga `!response.ok` diperiksa secara manual dan dilempar sebagai `Error`. Blok `catch` membedakan error asli dari error `AbortError` (yang sengaja dipicu oleh pembatalan, bukan kegagalan sungguhan) agar pesan error yang salah tidak sempat tampil ke pengguna. Blok `finally` tetap memastikan `loading` kembali `false`, dengan guard tambahan `!controller.signal.aborted` supaya request lama yang sudah dibatalkan tidak menimpa `loading` milik request baru yang sedang berjalan.

### 2.4 `useWeatherByRegion.js` - Rangkaian Fetch Geocoding → Cuaca

Hook ini dipanggil sekali di `App.jsx` dan bertugas menjalankan dua tahap fetch berurutan setiap kali `region` berubah:

```js
useEffect(() => {
  if (!region) {
    setData(null);
    setError("");
    setLoading(false);
    return;
  }

  const controller = new AbortController();

  async function fetchWeather() {
    setLoading(true);
    setError("");
    setData(null);
    try {
      const bestMatch = await geocodeWithFallback(region, controller.signal);
      if (!bestMatch) {
        throw new Error(`Koordinat untuk "${region.label}" tidak ditemukan...`);
      }
      const {
        latitude,
        longitude,
        name: matchedName,
        matchedLevel,
      } = bestMatch;

      const weatherUrl = `${WEATHER_URL}?latitude=${latitude}&longitude=${longitude}&current=temperature_2m,wind_speed_10m,weather_code&timezone=auto`;
      const weatherRes = await fetch(weatherUrl, { signal: controller.signal });
      if (!weatherRes.ok) {
        throw new Error(
          `Layanan cuaca bermasalah (status: ${weatherRes.status})`,
        );
      }

      const weatherJson = await weatherRes.json();
      if (!weatherJson.current) {
        throw new Error("Data cuaca tidak lengkap dari server.");
      }

      setData({
        /* ...susun object cuaca siap pakai untuk WeatherCard */
      });
    } catch (err) {
      if (err.name === "AbortError") return;
      setError(
        err.message ||
          "Terjadi kesalahan tak terduga saat mengambil data cuaca",
      );
    } finally {
      if (!controller.signal.aborted) setLoading(false);
    }
  }

  fetchWeather();
  return () => controller.abort();
}, [region]);
```

Beberapa poin penting:

- **Dependency array `[region]`** - `region` adalah object `{ district, regency, province, label }` yang baru terisi setelah pengguna melengkapi pilihan Kecamatan di `RegionSelector`. Efek ini otomatis berjalan ulang setiap kali `region` berganti, sehingga proses geocoding + cuaca selalu mengikuti wilayah terbaru yang dipilih.
- **Satu `AbortController` untuk dua tahap fetch** - karena geocoding dan cuaca dianggap satu rangkaian proses, satu `signal` yang sama dipakai untuk kedua `fetch()`. Jika pengguna berpindah kecamatan sebelum tahap pertama (geocoding) selesai, kedua request - baik yang sedang berjalan maupun yang belum sempat dikirim - otomatis batal.
- **Fungsi `geocodeWithFallback`** - merupakan fungsi handler tambahan yang mengatasi keterbatasan data Open-Meteo Geocoding (berbasis GeoNames, sering tidak memiliki data setingkat kecamatan). Fungsi ini mencoba query secara bertahap dari yang paling spesifik ke paling umum: `kecamatan, kabupaten` → `kabupaten, provinsi` → `provinsi, Indonesia`, dan berhenti begitu salah satu query menghasilkan data. Hasil yang dipilih juga diprioritaskan yang memiliki `country_code === 'ID'` agar tidak salah mengambil lokasi dari negara lain yang kebetulan memiliki nama serupa.
- **`fallbackNote`** - ketika hasil geocoding didapat bukan dari tingkat kecamatan (melainkan kabupaten/kota atau provinsi), aplikasi menyimpan catatan (`fallbackNote`) yang kemudian ditampilkan di `WeatherCard` sebagai peringatan kecil kepada pengguna bahwa data yang ditampilkan adalah estimasi, bukan data persis di kecamatan yang dipilih.

### 2.5 `useEffect` pada `RegionSelector.jsx` untuk Reset Dropdown Bertingkat

Selain dua hook fetch di atas, `RegionSelector.jsx` juga memakai `useEffect` untuk menjaga konsistensi antar-dropdown:

```js
useEffect(() => {
  setRegencyId("");
  setDistrictId("");
}, [provinceId]);

useEffect(() => {
  setDistrictId("");
}, [regencyId]);
```

Kedua efek ini memastikan tidak ada kombinasi data basi - misalnya kecamatan lama yang tetap terpilih padahal provinsinya sudah diganti. Setiap kali `provinceId` berubah, pilihan Kabupaten/Kota dan Kecamatan direset; begitu pula saat `regencyId` berubah, pilihan Kecamatan direset.

Ada juga efek ketiga yang berfungsi sebagai _handler_ pengirim data ke komponen induk:

```js
useEffect(() => {
  if (!districtId) {
    onRegionChange(null);
    return;
  }
  const province = provinces?.find((p) => p.id === provinceId);
  const regency = regencies?.find((r) => r.id === regencyId);
  const district = districts?.find((d) => d.id === districtId);

  if (province && regency && district) {
    onRegionChange({
      district: district.name,
      regency: regency.name,
      province: province.name,
      label: `${district.name}, ${regency.name}, ${province.name}`,
    });
  }
}, [districtId]);
```

Efek ini hanya dipicu oleh perubahan `districtId` (bukan `provinces`/`regencies`/`districts`, yang hanya dipakai untuk mencari nama). Begitu Kecamatan lengkap dipilih, fungsi `onRegionChange` (dikirim dari `App.jsx` sebagai props, berupa `setRegion`) dipanggil untuk mengirim object wilayah ke komponen induk, yang kemudian memicu `useWeatherByRegion` untuk mulai mengambil data cuaca.

### 2.6 Fungsi Handler Lainnya

- **`onChange={(e) => onChange(e.target.value)}`** pada komponen `SelectField` - handler standar untuk elemen `<select>`, meneruskan `id` wilayah yang dipilih pengguna ke state induk (`setProvinceId`, `setRegencyId`, atau `setDistrictId`).
- **`getWeatherInfo(code)`** pada `utils/weatherCodes.js` - bukan event handler, melainkan fungsi pemetaan (mapping) yang menerjemahkan `weather_code` numerik dari WMO menjadi label berbahasa Indonesia (misalnya "Cerah", "Hujan Lebat") beserta ikon `lucide-react` yang sesuai, dipakai langsung oleh `WeatherCard`.
- **`onRetry`** pada `ErrorMsg.jsx` - komponen ini sudah menyediakan slot handler `onRetry` untuk tombol "Coba lagi", meskipun pada versi saat ini `App.jsx` belum menyambungkannya ke fungsi refetch; ini menjadi salah satu ruang pengembangan lanjutan.

---

## 3. Screenshot UI

Pada bagian ini akan diberikan beberapa tangkapan layar atau screenshot dari aplikasi yang sebelumnya telah di bangun. Screenshot ini digunakan untuk menunjukkan bahwa fitur pemilihan wilayah dan tampilan informasi cuaca telah berhasil diterapkan pada sisi antarmuka.

### 3.1 Tampilan Awal

Bagian ini akan menampilkan halaman utama dengan dropdown Provinsi, Kabupaten/Kota, dan Kecamatan dalam kondisi kosong, serta pesan "Pilih wilayah di atas untuk melihat cuacanya."

<img width="334" height="420" alt="Screenshot 2026-09-02 193747" src="https://github.com/user-attachments/assets/c03a481a-f12d-4d79-80f7-f70bdc0f4404" />

### 3.2 Kondisi Loading

Bagian ini akan menampilkan spinner atau LoadingUI saat aplikasi sedang mengambil data wilayah dari dropdown atau saat sedang memproses geocoding dan data cuaca setelah kecamatan dipilih.

<img width="364" height="280" alt="Screenshot 2026-09-02 193725" src="https://github.com/user-attachments/assets/9f7c9ea2-76fa-49e0-8b68-1334fd75b1e0" />

### 3.3 Kondisi Sukses

Bagian ini akan menampilkan WeatherCard dengan data cuaca hasil fetch nama lokasi, suhu, ikon kondisi cuaca, kecepatan angin, dan waktu pengukuran, untuk salah satu wilayah yang dipilih, misalnya kecamatan di Bali). Untuk lebih jelasnya berikut merupakan tampilan yang dimaksud.

<img width="284" height="422" alt="Screenshot 2026-09-02 194820" src="https://github.com/user-attachments/assets/d8f4226f-9f3a-461a-afae-53686be83288" />

### 3.4 Kondisi Error

Bagian ini akan menampilkan ErrorMsg dengan UI peringatan merah, misalnya saat koneksi internet diputus atau salah satu endpoint API tidak dapat diakses. Untuk lebih jelasnya berikut merupakan tampilan yang dimaksud.

<img width="370" height="454" alt="Screenshot 2026-09-02 164458" src="https://github.com/user-attachments/assets/091b7a31-7896-490a-a8c4-485d59b2c0f0" />

---

## 4. Log Prompt AI

Bagian ini berisikan prompt persis yang digunakan untuk menghasilkan source code awal proyek hingga menjadi sebuah web app utuh yang dapat bekerja dengan baik. Untuk lebih jelasnya berikut merupakan prompt yang dimaksud.

**Peran Anda:** Bertindaklah sebagai Senior React.js Web Developer yang ahli dalam membangun arsitektur aplikasi modular dan integrasi REST API.

**Tugas:** Buatkan saya source code lengkap untuk proyek React Vite bernama "Nusantara Weather Explorer". Aplikasi ini adalah pengembangan dari Weather App sederhana menjadi aplikasi pemantau cuaca interaktif dengan pemilihan wilayah bertingkat khusus untuk daerah di Indonesia.

**Spesifikasi Logika & API:**

1. **Dropdown Wilayah Bertingkat** - Aplikasi harus memiliki dropdown yang saling bergantung (Provinsi → Kabupaten/Kota → Kecamatan). Gunakan Public API Wilayah Indonesia (misalnya `emsifa/api-wilayah-indonesia` atau alternatif yang valid) untuk mengambil data ini.
2. **Geocoding & Cuaca** - Setelah user memilih Kecamatan (atau tingkat wilayah terdalam), ambil koordinat (Latitude & Longitude) dari nama wilayah tersebut menggunakan Geocoding API (seperti Open-Meteo Geocoding API).
3. **Data Cuaca Utama** - Gunakan koordinat tersebut untuk melakukan fetch ke Open-Meteo Weather API guna menampilkan Suhu (°C), Kecepatan Angin (km/h), dan Kode Cuaca (untuk ikon).

**Spesifikasi Teknis (Sesuai Konsep Dasar):**

- **Arsitektur Modular** - Jangan buat aplikasi monolitik. Pecah menjadi beberapa komponen di dalam folder `src/components/` (misal: `App.jsx`, `RegionSelector.jsx`, `WeatherCard.jsx`, `LoadingUI.jsx`, `ErrorMsg.jsx`).
- **3 State Wajib** - Setiap kali melakukan proses fetching (baik API wilayah maupun API cuaca), pastikan menggunakan tiga state utama: `data`, `loading` (boolean), dan `error` (string pesan error).
- **Penggunaan `useEffect`** - Gunakan `useEffect` dengan dependency array yang tepat. Dropdown Kabupaten harus di-fetch ulang ketika Provinsi berubah, dst.
- **Keamanan & Best Practice** - Terapkan blok `try...catch...finally`. Periksa `!response.ok` secara manual dan lemparkan error. Gunakan `AbortController` di dalam `useEffect` sebagai cleanup function untuk mencegah race condition saat user mengganti wilayah dengan cepat.

**Spesifikasi UI/UX (Desain Menarik):**

- Terapkan gaya Glassmorphism (efek kaca buram transparan, border tipis) untuk card cuaca dan form wilayah.
- Buat background web menjadi dinamis atau setidaknya menggunakan gradasi warna estetik yang terlihat modern (seperti paduan deep blue dan soft cyan).
- Tampilkan animasi loading (Spinner/Skeleton) yang menarik saat `loading` bernilai `true`.
- Tampilkan pesan error dengan UI peringatan merah yang rapi jika API gagal diakses.
- Gunakan Tailwind CSS untuk styling (berikan kode kelas Tailwind-nya langsung di dalam JSX).

**Output yang Diharapkan:** Berikan struktur folder project dan kode lengkap untuk masing-masing file komponen (`.jsx`) yang bisa langsung dijalankan, lengkap dengan komentar singkat dalam Bahasa Indonesia pada bagian implementasi `useEffect` dan penanganan error untuk keperluan laporan pertahanan kode (code defense).
