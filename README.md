Baik, mari kita satukan semua langkah pengembangan aplikasi web CRUD e-commerce yang telah dibahas sebelumnya, termasuk semua perbaikan terbaru dan fitur tambahan. Panduan ini akan mencakup **Next.js 14**, **TypeScript**, **Bun**, **MariaDB** di **Laragon**, **Shadcn** sebagai penyedia komponen UI, dan **Tailwind CSS** dengan palet warna khusus. Selain itu, kita juga akan menerapkan fitur-fitur tambahan seperti **halaman registrasi yang lebih menarik**, **navigasi bawah untuk versi mobile**, dan **tampilan responsif** untuk semua perangkat.

## **1. Persiapan Lingkungan**

Sebelum memulai, pastikan Anda telah menginstal hal-hal berikut:

- **Bun**: Sebagai runtime JavaScript/TypeScript. [Panduan Instalasi Bun](https://bun.sh/docs/installation)
- **Laragon**: Sudah terinstal dan menjalankan MariaDB secara lokal.
- **Node.js dan npm**: Meskipun kita akan menggunakan Bun, beberapa tools mungkin memerlukannya.
- **Git**: Untuk kontrol versi (opsional tetapi disarankan).
- **Editor Kode**: Seperti VSCode.

## **2. Membuat Proyek Next.js dengan Bun dan TypeScript**

### **a. Inisialisasi Proyek Baru**

Buka terminal dan jalankan perintah berikut untuk membuat proyek Next.js baru menggunakan Bun:

```bash
bun create next-app ecommerce-app --typescript
```

Gantilah `ecommerce-app` dengan nama proyek Anda.

### **b. Navigasi ke Direktori Proyek**

```bash
cd ecommerce-app
```

### **c. Instalasi Dependensi**

Kita akan menggunakan beberapa dependensi tambahan untuk fungsionalitas aplikasi:

```bash
bun add mysql2 dotenv classnames react-toastify bcryptjs jsonwebtoken @heroicons/react
```

## **3. Mengkonfigurasi Database MariaDB di Laragon**

### **a. Membuat Database dan Tabel**

1. Buka **phpMyAdmin** melalui Laragon atau gunakan tool MariaDB lainnya.
2. Buat database baru, misalnya `ecommerce_db`.
3. Di dalam database `ecommerce_db`, buat tabel `users` dan `products` dengan struktur berikut:

```sql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    role ENUM('admin', 'user') DEFAULT 'user',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### **b. Menambahkan Data Admin**

Masukkan data admin ke tabel `users` dengan password yang di-hash. Anda bisa menggunakan skrip berikut di Bun untuk menghasilkan hash password:

```typescript:src/scripts/hashPassword.ts
import bcrypt from 'bcryptjs';

const password = 'admin123';
bcrypt.hash(password, 10).then(hash => {
    console.log(`Hashed Password: ${hash}`);
});
```

Jalankan skrip tersebut menggunakan Bun:

```bash
bun run src/scripts/hashPassword.ts
```

Salin hasil hash dan masukkan ke tabel `users`:

```sql
INSERT INTO users (username, password, role) VALUES ('admin', 'HASUHASH', 'admin');
```

Gantilah `'HASUHASH'` dengan hasil hash yang Anda peroleh dari skrip.

## **4. Mengatur Koneksi Database dengan `mysql2`**

### **a. Membuat File Konfigurasi Koneksi**

Buat file `db.ts` di dalam folder `src/lib/` untuk mengatur koneksi ke database.

```typescript:src/lib/db.ts
import mysql from 'mysql2/promise';
import dotenv from 'dotenv';

dotenv.config();

const pool = mysql.createPool({
    host: process.env.DB_HOST || 'localhost',
    user: process.env.DB_USER || 'root',
    password: process.env.DB_PASSWORD || '',
    database: process.env.DB_NAME || 'ecommerce_db',
    waitForConnections: true,
    connectionLimit: 10,
    queueLimit: 0,
});

export default pool;
```

### **b. Menyimpan Kredensial Database di Variabel Lingkungan**

Buat file `.env.local` di root proyek Anda dan tambahkan kredensial database serta `JWT_SECRET`:

```
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=
DB_NAME=ecommerce_db
DB_PORT=3306
JWT_SECRET=your_jwt_secret
```

**Catatan:** Gantilah `your_jwt_secret` dengan string rahasia yang kuat untuk keamanan JWT.

## **5. Mengatur Tailwind CSS dan Shadcn untuk Komponen UI**

### **a. Instalasi Tailwind CSS**

Jalankan perintah berikut untuk menginstal Tailwind CSS dan konfigurasi PostCSS:

```bash
bun add -D tailwindcss postcss autoprefixer
```

Inisialisasi konfigurasi Tailwind:

```bash
bun tailwindcss init -p
```

### **b. Konfigurasi `tailwind.config.js`**

Buka file `tailwind.config.js` dan modifikasi seperti berikut untuk menambahkan palet warna khusus dan pengaturan font:

```javascript:tailwind.config.js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    './src/**/*.{js,ts,jsx,tsx}',
    './src/components/**/*.{js,ts,jsx,tsx}',
  ],
  theme: {
    extend: {
      colors: {
        'bg-color': '#f7efe3',
        'inactive-bg': '#c3b9ab',
        'icon-color': '#0d3c26',
      },
      fontFamily: {
        sans: ['Inter', 'sans-serif'], // Ganti 'Inter' dengan font yang diinginkan
      },
    },
  },
  plugins: [],
}
```

**Komentar untuk Mengubah Font:**
Untuk mengganti font aplikasi secara keseluruhan, ubah `fontFamily` di atas sesuai dengan font yang diinginkan. Anda dapat menambahkan font baru melalui Google Fonts atau sumber lainnya dan mengimpornya di `globals.css`.

### **c. Tambahkan Direktif Tailwind ke CSS Global**

Buka file `src/app/globals.css` dan tambahkan:

```css:src/app/globals.css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Mengubah font aplikasi secara keseluruhan */
/* Untuk mengganti font, ubah 'font-sans' dengan kelas font yang telah ditentukan di tailwind.config.js */
body {
  @apply font-sans;
}
```

### **d. Mengatur Shadcn UI**

Kita akan membuat komponen dasar seperti Button, Input, Label, Modal, dan BottomNavigation menggunakan Tailwind CSS dengan skema warna yang telah ditentukan.

#### **i. Membuat Komponen UI**

Buat folder `src/components/ui/` dan tambahkan komponen berikut:

**Button.tsx**

```typescript:src/components/ui/button.tsx
import React from 'react';
import classNames from 'classnames';

interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
    variant?: 'default' | 'outline' | 'destructive' | 'secondary';
}

export const Button: React.FC<ButtonProps> = ({ variant = 'default', className, ...props }) => {
    const baseStyle = 'px-4 py-2 rounded-md font-semibold focus:outline-none';
    const variantStyles = {
        default: 'bg-blue-500 text-white hover:bg-blue-600',
        outline: 'border border-blue-500 text-blue-500 hover:bg-blue-500 hover:text-white',
        destructive: 'bg-red-500 text-white hover:bg-red-600',
        secondary: 'bg-gray-500 text-white hover:bg-gray-600',
    };

    return (
        <button
            className={classNames(baseStyle, variantStyles[variant], className)}
            {...props}
        />
    );
};
```

**Input.tsx**

```typescript:src/components/ui/input.tsx
import React from 'react';
import classNames from 'classnames';

interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {}

export const Input: React.FC<InputProps> = ({ className, ...props }) => (
    <input
        className={classNames(
            'border rounded-md px-3 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500',
            className
        )}
        {...props}
    />
);
```

**Label.tsx**

```typescript:src/components/ui/label.tsx
import React from 'react';
import classNames from 'classnames';

interface LabelProps extends React.LabelHTMLAttributes<HTMLLabelElement> {}

export const Label: React.FC<LabelProps> = ({ className, ...props }) => (
    <label className={classNames('text-sm font-medium text-icon-color', className)} {...props} />
);
```

**Modal.tsx**

```typescript:src/components/ui/modal.tsx
import React, { useEffect } from 'react';
import ReactDOM from 'react-dom';
import { Button } from './button';
import classNames from 'classnames';

interface ModalProps {
    isOpen: boolean;
    onClose: () => void;
    children: React.ReactNode;
}

export const Modal: React.FC<ModalProps> = ({ isOpen, onClose, children }) => {
    useEffect(() => {
        const handleEscape = (event: KeyboardEvent) => {
            if (event.key === 'Escape') {
                onClose();
            }
        };
        document.addEventListener('keydown', handleEscape);
        return () => {
            document.removeEventListener('keydown', handleEscape);
        };
    }, [onClose]);

    if (!isOpen) return null;

    return ReactDOM.createPortal(
        <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
            <div className={classNames('bg-white rounded-lg p-6 w-full max-w-md', 'bg-bg-color')}>
                {children}
                <Button variant="secondary" onClick={onClose} className="mt-4">
                    Tutup
                </Button>
            </div>
        </div>,
        document.body
    );
};
```

**BottomNavigation.tsx**

```typescript:src/components/ui/bottomNavigation.tsx
import React from 'react';
import Link from 'next/link';
import { HomeIcon, UserIcon, ShoppingCartIcon } from '@heroicons/react/24/outline'; // Gunakan Heroicons untuk ikon monochrome

export const BottomNavigation: React.FC = () => {
    return (
        <nav className="fixed bottom-0 left-0 right-0 bg-inactive-bg shadow-md md:hidden">
            <ul className="flex justify-around">
                <li>
                    <Link href="/user">
                        <a className="flex flex-col items-center py-2">
                            <HomeIcon className="h-6 w-6 text-icon-color" />
                            <span className="text-xs text-icon-color">Home</span>
                        </a>
                    </Link>
                </li>
                <li>
                    <Link href="/cart">
                        <a className="flex flex-col items-center py-2">
                            <ShoppingCartIcon className="h-6 w-6 text-icon-color" />
                            <span className="text-xs text-icon-color">Keranjang</span>
                        </a>
                    </Link>
                </li>
                <li>
                    <Link href="/profile">
                        <a className="flex flex-col items-center py-2">
                            <UserIcon className="h-6 w-6 text-icon-color" />
                            <span className="text-xs text-icon-color">Profil</span>
                        </a>
                    </Link>
                </li>
            </ul>
        </nav>
    );
};
```

**Komentar untuk Mengubah Font:**
Untuk memudahkan pengubahan font aplikasi secara keseluruhan, berikut adalah petunjuk pada kode:

### **tailwind.config.js**

```javascript:tailwind.config.js
// ...
theme: {
  extend: {
    // Menambahkan font kustom
    fontFamily: {
      sans: ['Inter', 'sans-serif'], // Ganti 'Inter' dengan font yang diinginkan
    },
    // Tambahkan pengaturan warna lain jika diperlukan
  },
},
// ...
```

### **globals.css**

```css:src/app/globals.css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Mengubah font aplikasi secara keseluruhan */
/* Untuk mengganti font, ubah 'font-sans' dengan kelas font yang telah ditentukan di tailwind.config.js */
body {
  @apply font-sans;
}
```

**Catatan:**
- Jika ingin menggunakan font lain, tambahkan font tersebut melalui Google Fonts atau download dan impor ke proyek Anda.
- Setelah menambahkan font baru, pastikan untuk memperbarui `fontFamily` di `tailwind.config.js` dan gunakan kelas Tailwind yang sesuai di `globals.css`.

## **6. Membuat API Routes untuk CRUD Produk, Autentikasi, dan Registrasi**

### **a. Struktur Direktori API**

Buat struktur direktori berikut di dalam folder `src/app/api/`:

```
src/
├── app/
│   └── api/
│       ├── auth/
│       │   └── route.ts
│       ├── register/
│       │   └── route.ts
│       └── products/
│           ├── [id]/
│           │   └── route.ts
│           └── route.ts
```

### **b. API Route untuk Autentikasi (Login)**

**File:** `src/app/api/auth/route.ts`

```typescript:src/app/api/auth/route.ts
import { NextResponse } from 'next/server';
import pool from '../../../lib/db';
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';

export async function POST(request: Request) {
    try {
        const { username, password } = await request.json();

        if (!username || !password) {
            return NextResponse.json({ error: 'Username dan password wajib diisi.' }, { status: 400 });
        }

        const [rows]: any = await pool.query('SELECT * FROM users WHERE username = ?', [username]);

        if (rows.length === 0) {
            return NextResponse.json({ error: 'Pengguna tidak ditemukan.' }, { status: 404 });
        }

        const user = rows[0];
        const isPasswordValid = await bcrypt.compare(password, user.password);

        if (!isPasswordValid) {
            return NextResponse.json({ error: 'Password salah.' }, { status: 401 });
        }

        const token = jwt.sign(
            { userId: user.id, username: user.username, role: user.role },
            process.env.JWT_SECRET || 'default_secret',
            { expiresIn: '1h' }
        );

        return NextResponse.json({ token });
    } catch (error: any) {
        return NextResponse.json({ error: 'Gagal melakukan login.' }, { status: 500 });
    }
}
```

### **c. API Route untuk Registrasi Pengguna**

**File:** `src/app/api/register/route.ts`

```typescript:src/app/api/register/route.ts
import { NextResponse } from 'next/server';
import pool from '../../../lib/db';
import bcrypt from 'bcryptjs';

export async function POST(request: Request) {
    try {
        const { username, password } = await request.json();

        if (!username || !password) {
            return NextResponse.json({ error: 'Username dan password wajib diisi.' }, { status: 400 });
        }

        // Cek apakah username sudah ada
        const [existingUsers]: any = await pool.query('SELECT * FROM users WHERE username = ?', [username]);
        if (existingUsers.length > 0) {
            return NextResponse.json({ error: 'Username sudah digunakan.' }, { status: 409 });
        }

        // Hash password
        const hashedPassword = await bcrypt.hash(password, 10);

        // Tambahkan pengguna baru
        await pool.query(
            'INSERT INTO users (username, password, role) VALUES (?, ?, ?)',
            [username, hashedPassword, 'user']
        );

        return NextResponse.json({ message: 'Registrasi berhasil.' }, { status: 201 });
    } catch (error: any) {
        console.error('Error:', error);
        return NextResponse.json({ error: 'Gagal melakukan registrasi.' }, { status: 500 });
    }
}
```

### **d. Middleware untuk Melindungi Route Admin**

Buat file `middleware.ts` di root proyek Anda untuk mengamankan route admin:

```typescript:middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import jwt from 'jsonwebtoken';

export function middleware(request: NextRequest) {
    const url = request.nextUrl.clone();
    const token = request.cookies.get('token')?.value;

    // Halaman admin
    if (url.pathname.startsWith('/admin')) {
        if (!token) {
            url.pathname = '/login';
            return NextResponse.redirect(url);
        }

        try {
            const decoded = jwt.verify(token, process.env.JWT_SECRET || 'default_secret') as any;
            if (decoded.role !== 'admin') {
                url.pathname = '/';
                return NextResponse.redirect(url);
            }
        } catch {
            url.pathname = '/login';
            return NextResponse.redirect(url);
        }
    }

    return NextResponse.next();
}

export const config = {
    matcher: ['/admin/:path*'],
}
```

**Catatan:** Pastikan untuk menambahkan `middleware.ts` di root proyek Anda.

### **e. API Route untuk Mengelola Produk**

#### **i. Mengambil dan Menambahkan Produk**

**File:** `src/app/api/products/route.ts`

```typescript:src/app/api/products/route.ts
import { NextResponse } from 'next/server';
import pool from '../../../lib/db';
import jwt from 'jsonwebtoken';

export async function GET() {
    try {
        const [rows] = await pool.query('SELECT * FROM products');
        return NextResponse.json(rows);
    } catch (error: any) {
        return NextResponse.json({ error: 'Gagal mengambil produk.' }, { status: 500 });
    }
}

export async function POST(request: Request) {
    try {
        const token = request.headers.get('Authorization')?.split(' ')[1];
        if (!token) {
            return NextResponse.json({ error: 'Tidak ada token yang diberikan.' }, { status: 401 });
        }

        const decoded = jwt.verify(token, process.env.JWT_SECRET || 'default_secret') as any;
        if (decoded.role !== 'admin') {
            return NextResponse.json({ error: 'Akses ditolak.' }, { status: 403 });
        }

        const { name, description, price } = await request.json();

        if (!name || !price) {
            return NextResponse.json({ error: 'Nama dan harga wajib diisi.' }, { status: 400 });
        }

        const [result]: any = await pool.query(
            'INSERT INTO products (name, description, price) VALUES (?, ?, ?)',
            [name, description || null, price]
        );

        const newProduct = {
            id: result.insertId,
            name,
            description,
            price,
            created_at: new Date(),
            updated_at: new Date(),
        };

        return NextResponse.json(newProduct, { status: 201 });
    } catch (error: any) {
        return NextResponse.json({ error: 'Gagal menambahkan produk.' }, { status: 500 });
    }
}
```

#### **ii. Mengambil, Memperbarui, dan Menghapus Produk Berdasarkan ID**

**File:** `src/app/api/products/[id]/route.ts`

```typescript:src/app/api/products/[id]/route.ts
import { NextResponse } from 'next/server';
import pool from '../../../../lib/db';
import jwt from 'jsonwebtoken';

export async function GET(request: Request, { params }: { params: { id: string } }) {
    try {
        const { id } = params;
        const [rows] = await pool.query('SELECT * FROM products WHERE id = ?', [id]);

        if ((rows as any[]).length === 0) {
            return NextResponse.json({ error: 'Produk tidak ditemukan.' }, { status: 404 });
        }

        return NextResponse.json((rows as any[])[0]);
    } catch (error: any) {
        return NextResponse.json({ error: 'Gagal mengambil produk.' }, { status: 500 });
    }
}

export async function PUT(request: Request, { params }: { params: { id: string } }) {
    try {
        const token = request.headers.get('Authorization')?.split(' ')[1];
        if (!token) {
            return NextResponse.json({ error: 'Tidak ada token yang diberikan.' }, { status: 401 });
        }

        const decoded = jwt.verify(token, process.env.JWT_SECRET || 'default_secret') as any;
        if (decoded.role !== 'admin') {
            return NextResponse.json({ error: 'Akses ditolak.' }, { status: 403 });
        }

        const { id } = params;
        const { name, description, price } = await request.json();

        if (!name || !price) {
            return NextResponse.json({ error: 'Nama dan harga wajib diisi.' }, { status: 400 });
        }

        // Cek apakah produk ada
        const [rows]: any = await pool.query('SELECT * FROM products WHERE id = ?', [id]);
        if (rows.length === 0) {
            return NextResponse.json({ error: 'Produk tidak ditemukan.' }, { status: 404 });
        }

        // Perbarui produk
        await pool.query(
            'UPDATE products SET name = ?, description = ?, price = ?, updated_at = NOW() WHERE id = ?',
            [name, description || null, price, id]
        );

        return NextResponse.json({ message: 'Produk berhasil diperbarui.' });
    } catch (error: any) {
        return NextResponse.json({ error: 'Gagal memperbarui produk.' }, { status: 500 });
    }
}

export async function DELETE(request: Request, { params }: { params: { id: string } }) {
    try {
        const token = request.headers.get('Authorization')?.split(' ')[1];
        if (!token) {
            return NextResponse.json({ error: 'Tidak ada token yang diberikan.' }, { status: 401 });
        }

        const decoded = jwt.verify(token, process.env.JWT_SECRET || 'default_secret') as any;
        if (decoded.role !== 'admin') {
            return NextResponse.json({ error: 'Akses ditolak.' }, { status: 403 });
        }

        const { id } = params;

        // Cek apakah produk ada
        const [rows]: any = await pool.query('SELECT * FROM products WHERE id = ?', [id]);
        if (rows.length === 0) {
            return NextResponse.json({ error: 'Produk tidak ditemukan.' }, { status: 404 });
        }

        // Hapus produk
        await pool.query('DELETE FROM products WHERE id = ?', [id]);

        return NextResponse.json({ message: 'Produk berhasil dihapus.' });
    } catch (error: any) {
        return NextResponse.json({ error: 'Gagal menghapus produk.' }, { status: 500 });
    }
}
```

## **7. Membuat Halaman Admin, Pengguna, Login, dan Registrasi**

### **a. Membuat Halaman Login**

**File:** `src/app/login/page.tsx`

```typescript:src/app/login/page.tsx
'use client';

import { useState, FormEvent } from 'react';
import { useRouter } from 'next/navigation';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { toast } from 'react-toastify';

const Login = () => {
    const router = useRouter();
    const [username, setUsername] = useState('');
    const [password, setPassword] = useState('');
    const [loading, setLoading] = useState(false);

    const handleLogin = async (e: FormEvent) => {
        e.preventDefault();
        setLoading(true);
        try {
            const res = await fetch('/api/auth', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ username, password }),
            });

            const data = await res.json();

            if (res.ok) {
                // Simpan token di localStorage atau cookie
                // Dalam contoh ini, kita akan menggunakan cookie
                document.cookie = `token=${data.token}; path=/;`;
                toast.success('Login berhasil!');
                router.push('/admin');
            } else {
                toast.error(data.error || 'Gagal melakukan login.');
            }
        } catch (error) {
            console.error('Error:', error);
            toast.error('Terjadi kesalahan saat login.');
        } finally {
            setLoading(false);
        }
    };

    return (
        <div className="flex items-center justify-center min-h-screen bg-bg-color">
            <form onSubmit={handleLogin} className="bg-inactive-bg p-6 rounded-md shadow-md w-full max-w-sm">
                <h2 className="text-2xl font-semibold mb-4 text-icon-color">Login Admin</h2>
                <div className="mb-4">
                    <Label htmlFor="username">Username</Label>
                    <Input
                        id="username"
                        type="text"
                        value={username}
                        onChange={(e) => setUsername(e.target.value)}
                        required
                        placeholder="Masukkan username"
                    />
                </div>
                <div className="mb-4">
                    <Label htmlFor="password">Password</Label>
                    <Input
                        id="password"
                        type="password"
                        value={password}
                        onChange={(e) => setPassword(e.target.value)}
                        required
                        placeholder="Masukkan password"
                    />
                </div>
                <Button type="submit" disabled={loading} className="w-full">
                    {loading ? 'Proses...' : 'Login'}
                </Button>
                <p className="mt-4 text-center text-icon-color">
                    Belum memiliki akun?{' '}
                    <a href="/register" className="text-blue-500 hover:underline">
                        Register di sini
                    </a>
                </p>
            </form>
        </div>
    );
};

export default Login;
```

### **b. Membuat Halaman Registrasi**

**File:** `src/app/register/page.tsx`

```typescript:src/app/register/page.tsx
'use client';

import { useState, FormEvent } from 'react';
import { useRouter } from 'next/navigation';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { toast } from 'react-toastify';

const Register = () => {
    const router = useRouter();
    const [username, setUsername] = useState('');
    const [password, setPassword] = useState('');
    const [confirmPassword, setConfirmPassword] = useState('');
    const [loading, setLoading] = useState(false);

    const handleRegister = async (e: FormEvent) => {
        e.preventDefault();
        if (password !== confirmPassword) {
            toast.error('Password dan konfirmasi password tidak cocok.');
            return;
        }
        setLoading(true);
        try {
            const res = await fetch('/api/register', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ username, password }),
            });

            const data = await res.json();

            if (res.ok) {
                toast.success('Registrasi berhasil! Silakan login.');
                router.push('/login');
            } else {
                toast.error(data.error || 'Gagal melakukan registrasi.');
            }
        } catch (error) {
            console.error('Error:', error);
            toast.error('Terjadi kesalahan saat registrasi.');
        } finally {
            setLoading(false);
        }
    };

    return (
        <div className="flex items-center justify-center min-h-screen bg-bg-color">
            <form onSubmit={handleRegister} className="bg-inactive-bg p-6 rounded-md shadow-md w-full max-w-sm">
                <h2 className="text-2xl font-semibold mb-4 text-icon-color">Register Pengguna</h2>
                <div className="mb-4">
                    <Label htmlFor="username">Username</Label>
                    <Input
                        id="username"
                        type="text"
                        value={username}
                        onChange={(e) => setUsername(e.target.value)}
                        required
                        placeholder="Masukkan username"
                    />
                </div>
                <div className="mb-4">
                    <Label htmlFor="password">Password</Label>
                    <Input
                        id="password"
                        type="password"
                        value={password}
                        onChange={(e) => setPassword(e.target.value)}
                        required
                        placeholder="Masukkan password"
                    />
                </div>
                <div className="mb-4">
                    <Label htmlFor="confirmPassword">Konfirmasi Password</Label>
                    <Input
                        id="confirmPassword"
                        type="password"
                        value={confirmPassword}
                        onChange={(e) => setConfirmPassword(e.target.value)}
                        required
                        placeholder="Ulangi password"
                    />
                </div>
                <Button type="submit" disabled={loading} className="w-full">
                    {loading ? 'Proses...' : 'Register'}
                </Button>
                <p className="mt-4 text-center text-icon-color">
                    Sudah memiliki akun?{' '}
                    <a href="/login" className="text-blue-500 hover:underline">
                        Login di sini
                    </a>
                </p>
            </form>
        </div>
    );
};

export default Register;
```

### **c. Membuat Halaman Admin**

**File:** `src/app/admin/page.tsx`

```typescript:src/app/admin/page.tsx
'use client';

import { useEffect, useState, FormEvent } from 'react';
import { useRouter } from 'next/navigation';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { Modal } from '@/components/ui/modal';
import { toast } from 'react-toastify';

interface Product {
    id: number;
    name: string;
    description?: string;
    price: number;
    created_at: string;
    updated_at: string;
}

const Admin = () => {
    const router = useRouter();
    const [products, setProducts] = useState<Product[]>([]);
    const [name, setName] = useState('');
    const [description, setDescription] = useState('');
    const [price, setPrice] = useState<number>(0);
    const [loading, setLoading] = useState(false);

    // Untuk modal edit
    const [isEditOpen, setIsEditOpen] = useState(false);
    const [currentProduct, setCurrentProduct] = useState<Product | null>(null);
    const [editName, setEditName] = useState('');
    const [editDescription, setEditDescription] = useState('');
    const [editPrice, setEditPrice] = useState<number>(0);
    const [editLoading, setEditLoading] = useState(false);

    useEffect(() => {
        fetchProducts();
    }, []);

    const fetchProducts = async () => {
        try {
            setLoading(true);
            const res = await fetch('/api/products');
            const data: Product[] = await res.json();
            setProducts(data);
        } catch (error) {
            console.error('Gagal mengambil produk:', error);
            toast.error('Terjadi kesalahan saat mengambil produk.');
        } finally {
            setLoading(false);
        }
    };

    const handleAddProduct = async (e: FormEvent) => {
        e.preventDefault();
        setLoading(true);
        try {
            const token = getToken();
            const res = await fetch('/api/products', {
                method: 'POST',
                headers: { 
                    'Content-Type': 'application/json',
                    'Authorization': `Bearer ${token}`
                },
                body: JSON.stringify({ name, description, price }),
            });

            const data = await res.json();

            if (res.ok) {
                toast.success('Produk berhasil ditambahkan.');
                setProducts([...products, data]);
                setName('');
                setDescription('');
                setPrice(0);
            } else {
                toast.error(data.error || 'Gagal menambahkan produk.');
            }
        } catch (error) {
            console.error('Error:', error);
            toast.error('Terjadi kesalahan saat menambahkan produk.');
        } finally {
            setLoading(false);
        }
    };

    const openEditModal = (product: Product) => {
        setCurrentProduct(product);
        setEditName(product.name);
        setEditDescription(product.description || '');
        setEditPrice(product.price);
        setIsEditOpen(true);
    };

    const closeEditModal = () => {
        setIsEditOpen(false);
        setCurrentProduct(null);
    };

    const handleEditProduct = async (e: FormEvent) => {
        e.preventDefault();
        if (!currentProduct) return;
        setEditLoading(true);
        try {
            const token = getToken();
            const res = await fetch(`/api/products/${currentProduct.id}`, {
                method: 'PUT',
                headers: { 
                    'Content-Type': 'application/json',
                    'Authorization': `Bearer ${token}`
                },
                body: JSON.stringify({ name: editName, description: editDescription, price: editPrice }),
            });

            const data = await res.json();

            if (res.ok) {
                toast.success('Produk berhasil diperbarui.');
                setProducts(products.map(prod => prod.id === currentProduct.id ? { ...prod, name: editName, description: editDescription, price: editPrice } : prod));
                closeEditModal();
            } else {
                toast.error(data.error || 'Gagal memperbarui produk.');
            }
        } catch (error) {
            console.error('Error:', error);
            toast.error('Terjadi kesalahan saat memperbarui produk.');
        } finally {
            setEditLoading(false);
        }
    };

    const handleDeleteProduct = async (id: number) => {
        if (!confirm('Apakah Anda yakin ingin menghapus produk ini?')) return;
        try {
            const token = getToken();
            const res = await fetch(`/api/products/${id}`, {
                method: 'DELETE',
                headers: { 
                    'Authorization': `Bearer ${token}`
                },
            });

            const data = await res.json();

            if (res.ok) {
                toast.success('Produk berhasil dihapus.');
                setProducts(products.filter(prod => prod.id !== id));
            } else {
                toast.error(data.error || 'Gagal menghapus produk.');
            }
        } catch (error) {
            console.error('Error:', error);
            toast.error('Terjadi kesalahan saat menghapus produk.');
        }
    };

    const getToken = (): string => {
        if (typeof window !== 'undefined') {
            const match = document.cookie.match(new RegExp('(^| )token=([^;]+)'));
            if (match) return match[2];
        }
        return '';
    };

    return (
        <div className="min-h-screen bg-bg-color p-4">
            <header className="flex justify-between items-center mb-6">
                <h1 className="text-3xl font-bold text-icon-color">Admin Dashboard</h1>
                <Button variant="outline" onClick={() => {
                    // Hapus token dan redirect ke login
                    document.cookie = 'token=; Max-Age=0; path=/;';
                    router.push('/login');
                }}>
                    Logout
                </Button>
            </header>

            <section className="mb-8">
                <form onSubmit={handleAddProduct} className="bg-inactive-bg p-4 rounded-md shadow-md">
                    <h2 className="text-xl font-semibold mb-4 text-icon-color">Tambah Produk Baru</h2>
                    <div className="mb-4">
                        <Label htmlFor="name">Nama Produk</Label>
                        <Input
                            id="name"
                            type="text"
                            value={name}
                            onChange={(e) => setName(e.target.value)}
                            required
                            placeholder="Masukkan nama produk"
                        />
                    </div>
                    <div className="mb-4">
                        <Label htmlFor="description">Deskripsi</Label>
                        <Input
                            id="description"
                            type="text"
                            value={description}
                            onChange={(e) => setDescription(e.target.value)}
                            placeholder="Masukkan deskripsi produk"
                        />
                    </div>
                    <div className="mb-4">
                        <Label htmlFor="price">Harga</Label>
                        <Input
                            id="price"
                            type="number"
                            value={price}
                            onChange={(e) => setPrice(parseFloat(e.target.value))}
                            required
                            placeholder="Masukkan harga produk"
                            min="0"
                            step="0.01"
                        />
                    </div>
                    <Button type="submit" disabled={loading}>
                        {loading ? 'Menambahkan...' : 'Tambah Produk'}
                    </Button>
                </form>
            </section>

            <section>
                <h2 className="text-2xl font-semibold mb-4 text-icon-color">Daftar Produk</h2>
                {loading ? (
                    <p className="text-icon-color">Memuat produk...</p>
                ) : (
                    <table className="w-full bg-inactive-bg rounded-md shadow-md">
                        <thead>
                            <tr>
                                <th className="px-4 py-2 text-left">Nama</th>
                                <th className="px-4 py-2 text-left">Deskripsi</th>
                                <th className="px-4 py-2 text-left">Harga</th>
                                <th className="px-4 py-2 text-center">Aksi</th>
                            </tr>
                        </thead>
                        <tbody>
                            {products.map(product => (
                                <tr key={product.id} className="border-t border-icon-color">
                                    <td className="px-4 py-2">{product.name}</td>
                                    <td className="px-4 py-2">{product.description}</td>
                                    <td className="px-4 py-2">Rp {product.price.toFixed(2)}</td>
                                    <td className="px-4 py-2 text-center">
                                        <Button variant="outline" onClick={() => openEditModal(product)} className="mr-2">
                                            Edit
                                        </Button>
                                        <Button variant="destructive" onClick={() => handleDeleteProduct(product.id)}>
                                            Hapus
                                        </Button>
                                    </td>
                                </tr>
                            ))}
                        </tbody>
                    </table>
                )}
            </section>

            {/* Modal Edit */}
            {isEditOpen && currentProduct && (
                <Modal isOpen={isEditOpen} onClose={closeEditModal}>
                    <form onSubmit={handleEditProduct} className="flex flex-col">
                        <h2 className="text-xl font-semibold mb-4 text-icon-color">Edit Produk</h2>
                        <div className="mb-4">
                            <Label htmlFor="edit-name">Nama Produk</Label>
                            <Input
                                id="edit-name"
                                type="text"
                                value={editName}
                                onChange={(e) => setEditName(e.target.value)}
                                required
                                placeholder="Masukkan nama produk"
                            />
                        </div>
                        <div className="mb-4">
                            <Label htmlFor="edit-description">Deskripsi</Label>
                            <Input
                                id="edit-description"
                                type="text"
                                value={editDescription}
                                onChange={(e) => setEditDescription(e.target.value)}
                                placeholder="Masukkan deskripsi produk"
                            />
                        </div>
                        <div className="mb-4">
                            <Label htmlFor="edit-price">Harga</Label>
                            <Input
                                id="edit-price"
                                type="number"
                                value={editPrice}
                                onChange={(e) => setEditPrice(parseFloat(e.target.value))}
                                required
                                placeholder="Masukkan harga produk"
                                min="0"
                                step="0.01"
                            />
                        </div>
                        <div className="flex justify-end space-x-2">
                            <Button variant="secondary" onClick={closeEditModal} type="button">
                                Batal
                            </Button>
                            <Button type="submit" disabled={editLoading}>
                                {editLoading ? 'Memperbarui...' : 'Perbarui Produk'}
                            </Button>
                        </div>
                    </form>
                </Modal>
            )}
        </div>
    );
};

export default Admin;
```

### **d. Membuat Halaman Pengguna**

**File:** `src/app/user/page.tsx`

```typescript:src/app/user/page.tsx
'use client';

import { useEffect, useState } from 'react';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { toast } from 'react-toastify';

interface Product {
    id: number;
    name: string;
    description?: string;
    price: number;
    created_at: string;
    updated_at: string;
}

const User = () => {
    const [products, setProducts] = useState<Product[]>([]);
    const [search, setSearch] = useState('');

    useEffect(() => {
        fetchProducts();
    }, []);

    const fetchProducts = async () => {
        try {
            const res = await fetch('/api/products');
            const data: Product[] = await res.json();
            setProducts(data);
        } catch (error) {
            console.error('Gagal mengambil produk:', error);
            toast.error('Terjadi kesalahan saat mengambil produk.');
        }
    };

    const filteredProducts = products.filter(product =>
        product.name.toLowerCase().includes(search.toLowerCase())
    );

    return (
        <div className="min-h-screen bg-bg-color p-4">
            <header className="flex justify-between items-center mb-6">
                <h1 className="text-3xl font-bold text-icon-color">E-commerce Store</h1>
                <Button variant="outline" onClick={() => { /* Implementasi Keranjang Belanja */ }}>
                    Keranjang
                </Button>
            </header>

            <section className="mb-8">
                <Input
                    type="text"
                    value={search}
                    onChange={(e) => setSearch(e.target.value)}
                    placeholder="Cari produk..."
                    className="w-full max-w-md bg-inactive-bg"
                />
            </section>

            <section>
                {filteredProducts.length === 0 ? (
                    <p className="text-icon-color">Tidak ada produk yang ditemukan.</p>
                ) : (
                    <ul className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6">
                        {filteredProducts.map(product => (
                            <li key={product.id} className="bg-inactive-bg p-4 rounded-md shadow-md flex flex-col justify-between">
                                <div>
                                    <h3 className="text-xl font-bold text-icon-color">{product.name}</h3>
                                    <p className="text-icon-color mt-2">{product.description}</p>
                                </div>
                                <div className="mt-4">
                                    <p className="font-semibold text-icon-color">Harga: Rp {product.price.toFixed(2)}</p>
                                    <Button variant="default" className="mt-2" onClick={() => { /* Implementasi Tambah ke Keranjang */ }}>
                                        Beli
                                    </Button>
                                </div>
                            </li>
                        ))}
                    </ul>
                )}
            </section>
        </div>
    );
};

export default User;
```

## **8. Menambahkan Navigasi Bawah untuk Versi Mobile**

### **a. Membuat Komponen BottomNavigation**

**File:** `src/components/ui/bottomNavigation.tsx`

```typescript:src/components/ui/bottomNavigation.tsx
import React from 'react';
import Link from 'next/link';
import { HomeIcon, UserIcon, ShoppingCartIcon } from '@heroicons/react/24/outline'; // Gunakan Heroicons untuk ikon monochrome

export const BottomNavigation: React.FC = () => {
    return (
        <nav className="fixed bottom-0 left-0 right-0 bg-inactive-bg shadow-md md:hidden">
            <ul className="flex justify-around">
                <li>
                    <Link href="/user">
                        <a className="flex flex-col items-center py-2">
                            <HomeIcon className="h-6 w-6 text-icon-color" />
                            <span className="text-xs text-icon-color">Home</span>
                        </a>
                    </Link>
                </li>
                <li>
                    <Link href="/cart">
                        <a className="flex flex-col items-center py-2">
                            <ShoppingCartIcon className="h-6 w-6 text-icon-color" />
                            <span className="text-xs text-icon-color">Keranjang</span>
                        </a>
                    </Link>
                </li>
                <li>
                    <Link href="/profile">
                        <a className="flex flex-col items-center py-2">
                            <UserIcon className="h-6 w-6 text-icon-color" />
                            <span className="text-xs text-icon-color">Profil</span>
                        </a>
                    </Link>
                </li>
            </ul>
        </nav>
    );
};
```

**Catatan:**
Pastikan Anda telah menginstal `@heroicons/react`:

```bash
bun add @heroicons/react
```

### **b. Menambahkan BottomNavigation ke Layout**

Update file `src/app/layout.tsx` untuk menyertakan komponen navigasi bawah.

```typescript:src/app/layout.tsx
'use client';

import '../globals.css';
import { ReactNode } from 'react';
import { ToastContainer } from 'react-toastify';
import 'react-toastify/dist/ReactToastify.css';
import { BottomNavigation } from '@/components/ui/bottomNavigation';

interface LayoutProps {
    children: ReactNode;
}

const Layout = ({ children }: LayoutProps) => {
    return (
        <>
            {children}
            <ToastContainer position="top-right" autoClose={3000} />
            <BottomNavigation />
        </>
    );
};

export default Layout;
```

## **9. Menjamin Responsivitas untuk Semua Perangkat**

Tailwind CSS menyediakan utilitas responsif yang dapat digunakan untuk menyesuaikan tampilan aplikasi di berbagai perangkat. Pastikan Anda menggunakan kelas responstif seperti `sm:`, `md:`, `lg:`, dan `xl:` sesuai kebutuhan.

**Contoh:**

```html
<div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6">
    <!-- Konten -->
</div>
```

Selain itu, pastikan komponen seperti navigasi bawah hanya terlihat pada perangkat mobile dengan menggunakan kelas Tailwind berikut:

```html
<nav className="fixed bottom-0 left-0 right-0 bg-inactive-bg shadow-md md:hidden">
    <!-- Konten Navigasi -->
</nav>
```

## **10. Hierarki Folder Proyek Terbaru**

Berikut adalah struktur folder lengkap setelah melakukan pembaruan:

```
ecommerce-app/
├── node_modules/
├── public/
│   ├── favicon.ico
│   └── ...
├── src/
│   ├── app/
│   │   ├── admin/
│   │   │   └── page.tsx
│   │   ├── auth/
│   │   │   └── route.ts
│   │   ├── register/
│   │   │   └── route.ts
│   │   ├── login/
│   │   │   └── page.tsx
│   │   ├── user/
│   │   │   └── page.tsx
│   │   ├── cart/
│   │   │   └── page.tsx
│   │   ├── profile/
│   │   │   └── page.tsx
│   │   ├── api/
│   │   │   ├── auth/
│   │   │   │   └── route.ts
│   │   │   ├── register/
│   │   │   │   └── route.ts
│   │   │   └── products/
│   │   │       ├── [id]/
│   │   │       │   └── route.ts
│   │   │       └── route.ts
│   │   ├── globals.css
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── components/
│   │   └── ui/
│   │       ├── button.tsx
│   │       ├── input.tsx
│   │       ├── label.tsx
│   │       ├── modal.tsx
│   │       └── bottomNavigation.tsx
│   ├── lib/
│   │   └── db.ts
│   ├── scripts/
│   │   └── hashPassword.ts
│   └── ...
├── styles/
│   └── ...
├── .env.local
├── .gitignore
├── middleware.ts
├── package.json
├── tailwind.config.js
├── postcss.config.js
├── tsconfig.json
└── bun.lockb
```

### **Penjelasan Struktur Folder:**

- **`src/app/register/`**: Berisi API route dan halaman registrasi pengguna.
- **`src/components/ui/bottomNavigation.tsx`**: Komponen navigasi bawah untuk perangkat mobile.
- **`src/app/api/register/route.ts`**: API route untuk registrasi pengguna.
- **`tailwind.config.js`**: Telah diperbarui dengan palet warna khusus dan pengaturan font.
- **`globals.css`**: Telah diperbarui untuk menggunakan font kustom secara keseluruhan.

## **11. Menjalankan Aplikasi**

Setelah melakukan semua pembaruan di atas, jalankan aplikasi Anda dengan perintah berikut:

```bash
bun dev
```

Buka browser dan navigasikan ke:

- **Halaman Pengguna:** `http://localhost:3000/user`
- **Halaman Admin:** `http://localhost:3000/admin`
- **Halaman Login Admin:** `http://localhost:3000/login`
- **Halaman Registrasi Pengguna:** `http://localhost:3000/register`

## **12. Alur Keseluruhan Aplikasi yang Lebih Kokoh**

1. **Frontend (Next.js dengan Shadcn UI):**
    - Pengguna mengakses halaman pengguna untuk melihat, mencari, dan membeli produk.
    - Pengguna baru dapat mendaftar melalui halaman registrasi.
    - Admin mengakses halaman admin setelah login untuk mengelola produk (menambah, mengedit, menghapus).

2. **API Route (Next.js):**
    - Menerima request dari frontend.
    - Untuk operasi admin, memverifikasi JWT untuk autentikasi dan otorisasi.
    - Menggunakan **`mysql2`** untuk berinteraksi dengan database MariaDB secara manual.

3. **Database (MariaDB):**
    - Menyimpan data pengguna dan produk.
    - Merespons request dari API sesuai dengan operasi CRUD yang dilakukan.

### **Diagram Alur**

```
[ Frontend (Next.js + Shadcn UI) ]
          |
          | HTTP Request (API) + JWT
          |
[ API Route (Next.js) ]
          |
          | mysql2
          |
[ Database (MariaDB) ]
```

## **13. Penutup**

Dengan mengikuti langkah-langkah di atas, Anda telah berhasil membuat aplikasi CRUD e-commerce yang lebih kokoh, estetis, dan responsif menggunakan **Next.js 14**, **TypeScript**, **Bun**, dan **MariaDB** sebagai database. Aplikasi ini memiliki fitur autentikasi pengguna, halaman registrasi, navigasi bawah khusus untuk perangkat mobile, serta desain yang responsif untuk berbagai perangkat. **Shadcn** dan **Tailwind CSS** digunakan untuk memberikan tampilan yang konsisten dan menarik sesuai dengan palet warna yang telah ditentukan.

### **Fitur Tambahan yang Diterapkan:**

- **Validasi Input:** Validasi dilakukan baik di frontend menggunakan atribut HTML dan di backend dengan memeriksa kehadiran data yang diperlukan sebelum memproses request.
- **Error Handling:** Penanganan error diterapkan di seluruh API routes dengan mengembalikan pesan error yang sesuai dan status HTTP yang tepat. Di frontend, error ditampilkan menggunakan `react-toastify` untuk notifikasi yang informatif.
- **Styling:** Styling aplikatif dikembangkan menggunakan **Tailwind CSS** dengan palet warna khusus dan komponen UI dari **Shadcn** untuk tampilan yang konsisten dan menarik.
- **Autentikasi Pengguna:** Implementasi autentikasi menggunakan JWT untuk melindungi route admin dan memastikan hanya admin yang dapat mengelola produk.
- **Halaman Admin dan Pengguna:** Terpisahnya halaman admin dan pengguna untuk memisahkan fungsi pengelolaan produk dari tampilan produk untuk pengguna umum.
- **Navigasi Bawah untuk Mobile:** Menambahkan navigasi bawah khusus untuk perangkat mobile agar navigasi lebih mudah diakses.
- **Responsivitas:** Memastikan tampilan aplikasi responsif di berbagai perangkat menggunakan utilitas responsif dari Tailwind CSS.

### **Fitur Tambahan yang Bisa Dikembangkan:**

- **Autentikasi Lanjutan:** Menambahkan fitur reset password, verifikasi email, dan profil pengguna.
- **Keranjang Belanja:** Implementasi fitur keranjang belanja yang memungkinkan pengguna menambah produk sebelum melakukan pembelian.
- **Pembayaran:** Integrasi dengan layanan pembayaran seperti Stripe atau PayPal untuk proses transaksi.
- **Optimasi Performa:** Menggunakan teknik caching seperti SWR atau React Query untuk fetching data yang lebih efisien dan cepat.
- **Testing:** Menambahkan unit tests dan integration tests untuk memastikan kualitas kode dan fungsionalitas aplikasi.
- **Deployment:** Mengoptimalkan proses deployment ke platform produksi seperti Vercel, termasuk pengaturan variabel lingkungan yang aman dan skalabilitas aplikasi.

Jika Anda mengalami kendala atau memiliki pertanyaan lebih lanjut, jangan ragu untuk menanyakannya!
