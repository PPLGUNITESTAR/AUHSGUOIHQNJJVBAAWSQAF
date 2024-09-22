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

Kita akan menggunakan **`mysql2`** untuk menghubungkan ke MariaDB, **`dotenv`** untuk manajemen variabel lingkungan, dan **Shadcn** untuk komponen UI. Selain itu, kita akan menginstal **`classnames`** untuk mengelola kelas CSS kondisional, **`react-toastify`** untuk notifikasi, dan **`bcryptjs`** serta **`jsonwebtoken`** untuk autentikasi.

```bash
bun add mysql2 dotenv classnames react-toastify bcryptjs jsonwebtoken
```

Untuk komponen Shadcn, kita akan mengaturnya secara manual karena tidak ada paket resmi.

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

Masukkan data admin ke tabel `users` dengan password yang di-hash. Anda bisa menggunakan skrip berikut di Node.js untuk menghasilkan hash password:

```typescript
// scripts/hashPassword.ts
import bcrypt from 'bcryptjs';

const password = 'admin123';
bcrypt.hash(password, 10).then(hash => {
    console.log(`Hashed Password: ${hash}`);
});
```

Jalankan skrip tersebut menggunakan Bun:

```bash
bun run scripts/hashPassword.ts
```

Salin hasil hash dan masukkan ke tabel `users`:

```sql
INSERT INTO users (username, password, role) VALUES ('admin', 'hasil_hash_password', 'admin');
```

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

Buat file `.env.local` di root proyek Anda dan tambahkan kredensial database:

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

Buka file `tailwind.config.js` dan modifikasi seperti berikut:

```javascript:tailwind.config.js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    './src/**/*.{js,ts,jsx,tsx}',
    './src/components/**/*.{js,ts,jsx,tsx}',
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

### **c. Tambahkan Direktif Tailwind ke CSS Global**

Buka file `src/app/globals.css` dan tambahkan:

```css:src/app/globals.css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### **d. Mengatur Shadcn UI**

Shadcn merupakan koleksi komponen UI yang estetis dan mudah dikustomisasi. Kita akan membuat komponen dasar seperti Button, Input, Label, dan Modal menggunakan Tailwind CSS.

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
    <label className={classNames('text-sm font-medium', className)} {...props} />
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
            <div className="bg-white rounded-lg p-6 w-full max-w-md">
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

#### **ii. Instalasi `classnames` dan `react-toastify`**

Sudah diinstal sebelumnya dengan perintah:

```bash
bun add classnames react-toastify
```

#### **iii. Menyiapkan Toast Notifications**

Tambahkan `ToastContainer` ke dalam aplikasi Anda. Perbarui `src/app/layout.tsx`:

```typescript:src/app/layout.tsx
'use client';

import '../globals.css';
import { ReactNode } from 'react';
import { ToastContainer } from 'react-toastify';
import 'react-toastify/dist/ReactToastify.css';

interface LayoutProps {
    children: ReactNode;
}

const Layout = ({ children }: LayoutProps) => {
    return (
        <>
            {children}
            <ToastContainer position="top-right" autoClose={3000} />
        </>
    );
};

export default Layout;
```

## **6. Membuat API Routes untuk CRUD Produk dan Autentikasi**

### **a. Struktur Direktori API**

Buat struktur direktori berikut di dalam folder `src/app/api/`:

```
src/
├── app/
│   └── api/
│       ├── auth/
│       │   └── route.ts
│       └── products/
│           ├── [id]/
│           │   └── route.ts
│           └── route.ts
```

### **b. API Route untuk Autentikasi**

#### **i. Membuat API Route untuk Login**

Buat file `route.ts` di `src/app/api/auth/`:

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

#### **ii. Membuat Middleware untuk Melindungi Route**

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

### **c. API Route untuk Mengelola Produk**

#### **i. Mengambil dan Menambahkan Produk**

Buat file `route.ts` di `src/app/api/products/`:

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

Buat file `route.ts` di `src/app/api/products/[id]/`:

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

        // Update produk
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

### **d. API Route untuk Mendaftarkan Pengguna (Opsional)**

Jika Anda ingin menyediakan fitur registrasi pengguna, Anda bisa menambahkan API route berikut. Namun, dalam contoh ini kita fokus pada login dan admin management.

## **7. Membuat Halaman Admin dan Pengguna**

### **a. Membuat Halaman Login**

Buat file `src/app/login/page.tsx`:

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
                document.cookie = `token=${data.token}; path=/`;
                toast.success('Login berhasil!');
                router.push('/admin');
            } else {
                toast.error(data.error || 'Gagal login.');
            }
        } catch (error) {
            console.error('Error:', error);
            toast.error('Terjadi kesalahan saat login.');
        } finally {
            setLoading(false);
        }
    };

    return (
        <div className="flex items-center justify-center min-h-screen bg-gray-100">
            <form onSubmit={handleLogin} className="bg-white p-6 rounded-md shadow-md w-full max-w-sm">
                <h2 className="text-2xl font-semibold mb-4">Login Admin</h2>
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
                <Button type="submit" disabled={loading}>
                    {loading ? 'Proses...' : 'Login'}
                </Button>
            </form>
        </div>
    );
};

export default Login;
```

### **b. Membuat Halaman Admin**

Buat folder `src/app/admin/` dan buat file `page.tsx`:

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
            setLoading(false);
        } catch (error) {
            console.error('Gagal mengambil produk:', error);
            setLoading(false);
        }
    };

    const getToken = () => {
        const match = document.cookie.match(new RegExp('(^| )token=([^;]+)'));
        if (match) return match[2];
        return null;
    };

    const addProduct = async (e: FormEvent) => {
        e.preventDefault();
        try {
            const token = getToken();
            if (!token) {
                toast.error('Token tidak ditemukan. Silakan login lagi.');
                router.push('/login');
                return;
            }

            const res = await fetch('/api/products', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                    'Authorization': `Bearer ${token}`,
                },
                body: JSON.stringify({ name, description, price }),
            });

            if (res.ok) {
                const newProduct: Product = await res.json();
                setProducts([...products, newProduct]);
                setName('');
                setDescription('');
                setPrice(0);
                toast.success('Produk berhasil ditambahkan!');
            } else {
                const errorData = await res.json();
                toast.error(errorData.error || 'Gagal menambahkan produk.');
            }
        } catch (error) {
            console.error('Gagal menambahkan produk:', error);
            toast.error('Terjadi kesalahan saat menambahkan produk.');
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

    const updateProduct = async (e: FormEvent) => {
        e.preventDefault();
        if (!currentProduct) return;
        try {
            setEditLoading(true);
            const token = getToken();
            if (!token) {
                toast.error('Token tidak ditemukan. Silakan login lagi.');
                router.push('/login');
                return;
            }

            const res = await fetch(`/api/products/${currentProduct.id}`, {
                method: 'PUT',
                headers: {
                    'Content-Type': 'application/json',
                    'Authorization': `Bearer ${token}`,
                },
                body: JSON.stringify({ name: editName, description: editDescription, price: editPrice }),
            });

            if (res.ok) {
                setProducts(products.map(p => p.id === currentProduct.id ? { ...p, name: editName, description: editDescription, price: editPrice } : p));
                toast.success('Produk berhasil diperbarui!');
                closeEditModal();
            } else {
                const errorData = await res.json();
                toast.error(errorData.error || 'Gagal memperbarui produk.');
            }
        } catch (error) {
            console.error('Gagal memperbarui produk:', error);
            toast.error('Terjadi kesalahan saat memperbarui produk.');
        } finally {
            setEditLoading(false);
        }
    };

    const deleteProduct = async (id: number) => {
        if (!confirm('Apakah Anda yakin ingin menghapus produk ini?')) return;
        try {
            const token = getToken();
            if (!token) {
                toast.error('Token tidak ditemukan. Silakan login lagi.');
                router.push('/login');
                return;
            }

            const res = await fetch(`/api/products/${id}`, {
                method: 'DELETE',
                headers: {
                    'Authorization': `Bearer ${token}`,
                },
            });

            if (res.ok) {
                setProducts(products.filter(p => p.id !== id));
                toast.success('Produk berhasil dihapus!');
            } else {
                const errorData = await res.json();
                toast.error(errorData.error || 'Gagal menghapus produk.');
            }
        } catch (error) {
            console.error('Gagal menghapus produk:', error);
            toast.error('Terjadi kesalahan saat menghapus produk.');
        }
    };

    const logout = () => {
        document.cookie = 'token=; path=/; expires=Thu, 01 Jan 1970 00:00:00 GMT';
        router.push('/login');
    };

    return (
        <div className="min-h-screen bg-gray-100 p-4">
            <header className="flex justify-between items-center mb-6">
                <h1 className="text-3xl font-bold">Admin Dashboard</h1>
                <Button variant="destructive" onClick={logout}>
                    Logout
                </Button>
            </header>

            <section className="mb-8">
                <h2 className="text-2xl font-semibold mb-4">Tambah Produk Baru</h2>
                <form onSubmit={addProduct} className="space-y-4 bg-white p-6 rounded-md shadow-md">
                    <div>
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
                    <div>
                        <Label htmlFor="description">Deskripsi</Label>
                        <Input
                            id="description"
                            type="text"
                            value={description}
                            onChange={(e) => setDescription(e.target.value)}
                            placeholder="Masukkan deskripsi produk"
                        />
                    </div>
                    <div>
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
                    <Button type="submit">Tambah Produk</Button>
                </form>
            </section>

            <section>
                <h2 className="text-2xl font-semibold mb-4">Daftar Produk</h2>
                {loading ? (
                    <p>Memuat...</p>
                ) : (
                    <ul className="space-y-4">
                        {products.map((product) => (
                            <li key={product.id} className="bg-white p-4 rounded-md shadow-md flex justify-between items-center">
                                <div>
                                    <h3 className="text-xl font-bold">{product.name}</h3>
                                    <p>{product.description}</p>
                                    <p className="font-semibold">Harga: Rp {product.price.toFixed(2)}</p>
                                </div>
                                <div className="space-x-2">
                                    <Button variant="secondary" onClick={() => openEditModal(product)}>
                                        Edit
                                    </Button>
                                    <Button variant="destructive" onClick={() => deleteProduct(product.id)}>
                                        Hapus
                                    </Button>
                                </div>
                            </li>
                        ))}
                    </ul>
                )}
            </section>

            {/* Modal Edit Produk */}
            {isEditOpen && currentProduct && (
                <Modal isOpen={isEditOpen} onClose={closeEditModal}>
                    <h2 className="text-2xl font-semibold mb-4">Edit Produk</h2>
                    <form onSubmit={updateProduct} className="space-y-4">
                        <div>
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
                        <div>
                            <Label htmlFor="edit-description">Deskripsi</Label>
                            <Input
                                id="edit-description"
                                type="text"
                                value={editDescription}
                                onChange={(e) => setEditDescription(e.target.value)}
                                placeholder="Masukkan deskripsi produk"
                            />
                        </div>
                        <div>
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

### **c. Membuat Halaman Pengguna**

Buat file `src/app/user/page.tsx`:

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
        <div className="min-h-screen bg-gray-100 p-4">
            <header className="flex justify-between items-center mb-6">
                <h1 className="text-3xl font-bold">E-commerce Store</h1>
                <Button variant="secondary" onClick={() => { /* Implementasi Keranjang Belanja */ }}>
                    Keranjang
                </Button>
            </header>

            <section className="mb-8">
                <Input
                    type="text"
                    value={search}
                    onChange={(e) => setSearch(e.target.value)}
                    placeholder="Cari produk..."
                    className="w-full max-w-md"
                />
            </section>

            <section>
                {filteredProducts.length === 0 ? (
                    <p>Tidak ada produk yang ditemukan.</p>
                ) : (
                    <ul className="grid grid-cols-1 md:grid-cols-3 gap-6">
                        {filteredProducts.map(product => (
                            <li key={product.id} className="bg-white p-4 rounded-md shadow-md">
                                <h3 className="text-xl font-bold mb-2">{product.name}</h3>
                                <p className="mb-2">{product.description}</p>
                                <p className="font-semibold mb-4">Harga: Rp {product.price.toFixed(2)}</p>
                                <Button variant="default" onClick={() => { /* Implementasi Tambah ke Keranjang */ }}>
                                    Beli
                                </Button>
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

### **d. Menambahkan Routing untuk Halaman Admin dan Pengguna**

Pastikan Anda memiliki navigasi yang sesuai antara halaman admin dan pengguna. Anda bisa menambahkan link di header atau membuat halaman utama dengan navigasi.

## **8. Menambahkan Validasi Input dan Penanganan Error**

### **a. Validasi Input di Frontend**

Sudah diterapkan melalui atribut `required`, `min`, dan `step` pada elemen input. Anda bisa menambahkan lebih banyak validasi sesuai kebutuhan, seperti regex untuk format tertentu.

### **b. Validasi Input di Backend**

Sudah diterapkan dengan memeriksa apakah `name` dan `price` diisi sebelum menambahkan atau memperbarui produk.

### **c. Penanganan Error**

Penanganan error sudah diterapkan di seluruh API routes dengan mengembalikan pesan error yang sesuai dan status HTTP yang tepat. Di frontend, error ditampilkan menggunakan `react-toastify` untuk notifikasi.

## **9. Implementasi Autentikasi dan Otorisasi**

Autentikasi dilakukan menggunakan JWT. Token disimpan dalam cookie dan diverifikasi di setiap request ke API routes yang memerlukan autentikasi, terutama untuk route admin.

### **a. Menambahkan Middleware**

Sudah dibuat di bagian sebelumnya (`middleware.ts`) untuk melindungi route admin.

### **b. Menjaga Keamanan JWT**

Pastikan `JWT_SECRET` di file `.env.local` adalah string yang kuat dan rahasia. Jangan membagikan atau mempublikasikannya.

## **10. Menjalankan Aplikasi**

Setelah semua langkah di atas selesai, jalankan server pengembangan dengan Bun:

```bash
bun dev
```

Buka browser dan navigasikan ke `http://localhost:3000` untuk melihat aplikasi e-commerce Anda berfungsi.

- **Halaman Pengguna:** `http://localhost:3000/user`
- **Halaman Admin:** `http://localhost:3000/admin`
- **Halaman Login Admin:** `http://localhost:3000/login`

## **11. Alur Keseluruhan Aplikasi**

1. **Frontend (Next.js dengan Shadcn UI):**
    - Pengguna mengakses halaman pengguna untuk melihat dan mencari produk.
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

## **12. Hierarki Folder Proyek**

Berikut adalah struktur folder lengkap dari proyek CRUD e-commerce yang kokoh ini:

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
│   │   ├── login/
│   │   │   └── page.tsx
│   │   ├── user/
│   │   │   └── page.tsx
│   │   ├── api/
│   │   │   ├── auth/
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
│   │       └── modal.tsx
│   ├── lib/
│   │   └── db.ts
│   └── ...
├── styles/
│   └── ...
├── scripts/
│   └── hashPassword.ts
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

- **`node_modules/`**: Berisi paket-paket dependensi yang diinstal.
- **`public/`**: Berisi aset publik seperti favicon dan gambar.
- **`src/`**: Sumber kode aplikasi.
  - **`app/`**: Folder utama untuk halaman dan API.
    - **`admin/`**: Halaman admin untuk mengelola produk.
    - **`auth/`**: API routes untuk autentikasi.
    - **`login/`**: Halaman login admin.
    - **`user/`**: Halaman pengguna untuk melihat dan membeli produk.
    - **`api/`**: Berisi API routes untuk autentikasi dan produk.
      - **`auth/route.ts`**: API route untuk login.
      - **`products/`**: API routes untuk mengelola produk.
        - **`[id]/route.ts`**: API route untuk operasi spesifik berdasarkan ID produk (GET, PUT, DELETE).
        - **`route.ts`**: API route untuk operasi umum (GET all, POST).
    - **`globals.css`**: CSS global untuk Tailwind.
    - **`layout.tsx`**: Layout utama aplikasi, termasuk pengaturan ToastContainer.
    - **`page.tsx`**: Halaman utama (bisa diarahkan ke pengguna atau admin).
  - **`components/ui/`**: Komponen UI yang dibuat menggunakan Shadcn.
    - **`button.tsx`**, **`input.tsx`**, **`label.tsx`**, **`modal.tsx`**: Komponen UI yang dapat digunakan ulang.
  - **`lib/`**: Library atau helper functions.
    - **`db.ts`**: Konfigurasi koneksi database menggunakan `mysql2`.
  - **`scripts/`**: Skrip tambahan, seperti untuk meng-hash password.
    - **`hashPassword.ts`**: Skrip untuk menghasilkan hash password.
- **`styles/`**: Berisi file CSS tambahan jika diperlukan.
- **`.env.local`**: File variabel lingkungan untuk menyimpan kredensial database dan JWT.
- **`.gitignore`**: Mengabaikan file atau folder tertentu saat menggunakan Git.
- **`middleware.ts`**: Middleware untuk melindungi route admin.
- **`package.json`**: File manajemen dependensi dan skrip.
- **`tailwind.config.js`**: Konfigurasi Tailwind CSS.
- **`postcss.config.js`**: Konfigurasi PostCSS.
- **`tsconfig.json`**: Konfigurasi TypeScript.
- **`bun.lockb`**: Lockfile Bun untuk memastikan versi dependensi konsisten.

## **13. Penutup**

Dengan mengikuti langkah-langkah di atas, Anda telah berhasil membuat aplikasi CRUD e-commerce yang kokoh menggunakan **Next.js 14**, **TypeScript**, **Bun**, dan **MariaDB** sebagai database. Aplikasi ini memiliki fitur autentikasi, validasi input, penanganan error, serta halaman admin dan pengguna yang terpisah. **Shadcn** dan **Tailwind CSS** digunakan untuk memberikan tampilan yang estetis dan responsif.

### **Tips Tambahan yang Diterapkan:**

- **Validasi Input:** Validasi dilakukan baik di frontend menggunakan atribut HTML dan di backend dengan memeriksa kehadiran data yang diperlukan sebelum memproses request.
- **Error Handling:** Penanganan error diterapkan di seluruh API routes dengan mengembalikan pesan error yang sesuai dan status HTTP yang tepat. Di frontend, error ditampilkan menggunakan `react-toastify` untuk notifikasi yang informatif.
- **Styling:** Styling aplikatif dikembangkan menggunakan **Tailwind CSS** dan komponen UI dari **Shadcn** untuk tampilan yang konsisten dan menarik.
- **Autentikasi Pengguna:** Implementasi autentikasi menggunakan JWT untuk melindungi route admin dan memastikan hanya admin yang dapat mengelola produk.
- **Halaman Admin dan Pengguna:** Terpisahnya halaman admin dan pengguna untuk memisahkan fungsi pengelolaan produk dari tampilan produk untuk pengguna umum.

### **Fitur Tambahan yang Bisa Dikembangkan:**

- **Autentikasi Lanjutan:** Menambahkan fitur registrasi pengguna, reset password, dan profil pengguna.
- **Keranjang Belanja:** Implementasi fitur keranjang belanja untuk pengguna.
- **Pembayaran:** Integrasi dengan layanan pembayaran seperti Stripe atau PayPal.
- **Optimasi Performa:** Menggunakan teknik caching seperti SWR atau React Query untuk fetching data yang lebih efisien.
- **Testing:** Menambahkan unit tests dan integration tests untuk memastikan kualitas kode dan fungsionalitas aplikasi.
- **Deployment:** Mengoptimalkan proses deployment ke platform produksi seperti Vercel, termasuk pengaturan variabel lingkungan yang aman dan skalabilitas aplikasi.
