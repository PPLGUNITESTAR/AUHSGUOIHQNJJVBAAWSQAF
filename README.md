Baik, mari kita buat aplikasi web CRUD e-commerce yang kokoh menggunakan **Next.js 14** dengan **TypeScript** dan **Bun** sebagai runtime. Kita akan menggunakan **MariaDB** yang berjalan secara lokal melalui **Laragon** sebagai database, serta **Shadcn** sebagai penyedia komponen UI. Aplikasi ini akan mencakup fitur dasar seperti menambah, menampilkan, mengedit, dan menghapus produk tanpa menggunakan ORM seperti Prisma, melainkan langsung mengelola koneksi database dengan **`mysql2`**.

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

Kita akan menggunakan **`mysql2`** untuk menghubungkan ke MariaDB, **`dotenv`** untuk manajemen variabel lingkungan, dan **Shadcn** untuk komponen UI.

```bash
bun add mysql2 dotenv
```

## **3. Mengkonfigurasi Database MariaDB di Laragon**

### **a. Membuat Database dan Tabel**

1. Buka **phpMyAdmin** melalui Laragon atau gunakan tool MariaDB lainnya.
2. Buat database baru, misalnya `ecommerce_db`.
3. Di dalam database `ecommerce_db`, buat tabel `products` dengan struktur berikut:

```sql
CREATE TABLE products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
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
```

**Catatan:** Jika Laragon menggunakan password untuk `root`, tambahkan di `DB_PASSWORD`.

## **5. Mengatur Shadcn untuk Komponen UI**

### **a. Instalasi Shadcn**

Shadcn merupakan koleksi komponen UI yang estetis dan mudah dikustomisasi. Untuk mengintegrasikannya, kita akan menggunakan **Shadcn UI** dengan **Tailwind CSS**.

#### **i. Instalasi Tailwind CSS dan Shadcn UI**

```bash
bun add -D tailwindcss postcss autoprefixer
```

Inisialisasi konfigurasi Tailwind:

```bash
bun tailwindcss init -p
```

#### **ii. Konfigurasi `tailwind.config.js`**

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

#### **iii. Tambahkan Direktif Tailwind ke CSS Global**

Buka file `src/app/globals.css` dan tambahkan:

```css:src/app/globals.css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### **b. Menginstal Shadcn UI**

Ikuti panduan resmi [Shadcn UI](https://shadcn.com/docs/getting-started) untuk instalasi dan konfigurasi. Secara umum, Anda dapat menggunakan paket yang sudah tersedia atau mengkonfigurasi sesuai kebutuhan.

## **6. Membuat API Routes untuk CRUD Produk**

### **a. Struktur Direktori API**

Buat struktur direktori berikut di dalam folder `src/app/api/`:

```
src/
├── app/
│   └── api/
│       └── products/
│           ├── [id]/
│           │   └── route.ts
│           └── route.ts
```

### **b. API Route untuk Mengelola Produk**

#### **i. Mengambil dan Menambahkan Produk**

Buat file `route.ts` di `src/app/api/products/`:

```typescript:src/app/api/products/route.ts
import { NextResponse } from 'next/server';
import pool from '../../../lib/db';

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

## **7. Membuat Komponen Frontend untuk CRUD Produk dengan Shadcn**

### **a. Menyusun Halaman Daftar dan Formulir Produk**

Buka atau buat file `src/app/page.tsx` dan tambahkan kode berikut untuk menampilkan daftar produk serta formulir tambah dan edit produk menggunakan komponen dari Shadcn.

```typescript:src/app/page.tsx
'use client';

import { useEffect, useState, FormEvent } from 'react';
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

const Home = () => {
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

    // Mengambil semua produk saat komponen dimuat
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

    const addProduct = async (e: FormEvent) => {
        e.preventDefault();
        try {
            const res = await fetch('/api/products', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ name, description, price }),
            });
            if (res.ok) {
                const newProduct: Product = await res.json();
                setProducts([...products, newProduct]);
                // Reset form
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

    const deleteProduct = async (id: number) => {
        if (!confirm('Apakah Anda yakin ingin menghapus produk ini?')) return;
        try {
            const res = await fetch(`/api/products/${id}`, {
                method: 'DELETE',
            });
            if (res.ok) {
                setProducts(products.filter((product) => product.id !== id));
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
            const res = await fetch(`/api/products/${currentProduct.id}`, {
                method: 'PUT',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ name: editName, description: editDescription, price: editPrice }),
            });
            if (res.ok) {
                setProducts(
                    products.map((product) =>
                        product.id === currentProduct.id
                            ? { ...product, name: editName, description: editDescription, price: editPrice }
                            : product
                    )
                );
                closeEditModal();
                toast.success('Produk berhasil diperbarui!');
            } else {
                const errorData = await res.json();
                toast.error(errorData.error || 'Gagal memperbarui produk.');
            }
        } catch (error) {
            console.error('Gagal memperbarui produk:', error);
            toast.error('Terjadi kesalahan saat memperbarui produk.');
        }
    };

    return (
        <div className="container mx-auto p-4">
            <h1 className="text-3xl font-bold mb-6">E-Commerce CRUD App</h1>

            {/* Daftar Produk */}
            <section>
                <h2 className="text-2xl font-semibold mb-4">Daftar Produk</h2>
                {loading ? (
                    <p>Memuat...</p>
                ) : (
                    <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
                        {products.map((product) => (
                            <div key={product.id} className="border rounded p-4 shadow">
                                <h3 className="text-xl font-bold mb-2">{product.name}</h3>
                                <p className="mb-2">{product.description}</p>
                                <p className="font-semibold mb-4">Harga: Rp {product.price.toFixed(2)}</p>
                                <div className="flex justify-between">
                                    <Button variant="outline" onClick={() => openEditModal(product)}>
                                        Edit
                                    </Button>
                                    <Button variant="destructive" onClick={() => deleteProduct(product.id)}>
                                        Hapus
                                    </Button>
                                </div>
                            </div>
                        ))}
                    </div>
                )}
            </section>

            {/* Form Tambah Produk */}
            <section className="mt-8">
                <h2 className="text-2xl font-semibold mb-4">Tambah Produk Baru</h2>
                <form onSubmit={addProduct} className="space-y-4">
                    <div>
                        <Label htmlFor="name" className="block mb-1">
                            Nama Produk
                        </Label>
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
                        <Label htmlFor="description" className="block mb-1">
                            Deskripsi
                        </Label>
                        <Input
                            id="description"
                            type="text"
                            value={description}
                            onChange={(e) => setDescription(e.target.value)}
                            placeholder="Masukkan deskripsi produk"
                        />
                    </div>
                    <div>
                        <Label htmlFor="price" className="block mb-1">
                            Harga
                        </Label>
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

            {/* Modal Edit Produk */}
            {isEditOpen && currentProduct && (
                <Modal isOpen={isEditOpen} onClose={closeEditModal}>
                    <h2 className="text-2xl font-semibold mb-4">Edit Produk</h2>
                    <form onSubmit={updateProduct} className="space-y-4">
                        <div>
                            <Label htmlFor="edit-name" className="block mb-1">
                                Nama Produk
                            </Label>
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
                            <Label htmlFor="edit-description" className="block mb-1">
                                Deskripsi
                            </Label>
                            <Input
                                id="edit-description"
                                type="text"
                                value={editDescription}
                                onChange={(e) => setEditDescription(e.target.value)}
                                placeholder="Masukkan deskripsi produk"
                            />
                        </div>
                        <div>
                            <Label htmlFor="edit-price" className="block mb-1">
                                Harga
                            </Label>
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
                            <Button variant="secondary" onClick={closeEditModal}>
                                Batal
                            </Button>
                            <Button type="submit">Perbarui Produk</Button>
                        </div>
                    </form>
                </Modal>
            )}

            {/* Toast Notifications */}
            <ToastContainer />
        </div>
    );
};

export default Home;
```

### **b. Membuat Komponen UI dengan Shadcn**

Untuk menggunakan komponen dari Shadcn, Anda perlu membuat atau mengimpor komponen tersebut. Berikut contoh struktur sederhana untuk komponen `Button`, `Input`, `Label`, dan `Modal`.

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

Untuk mengelola kelas CSS kondisional dan notifikasi, instal `classnames` dan `react-toastify`:

```bash
bun add classnames react-toastify
```

#### **iii. Menyiapkan Toast Notifications**

Tambahkan `ToastContainer` ke dalam aplikasi Anda. Perbarui `src/app/layout.tsx` atau file layout utama Anda:

```typescript:src/app/layout.tsx
'use client';

import '../globals.css';
import { ToastContainer } from 'react-toastify';
import 'react-toastify/dist/ReactToastify.css';
import { ReactNode } from 'react';

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

## **8. Menjalankan Aplikasi**

Setelah semua langkah di atas selesai, jalankan server pengembangan dengan Bun:

```bash
bun dev
```

Buka browser dan navigasikan ke `http://localhost:3000` untuk melihat aplikasi CRUD e-commerce Anda berfungsi.

## **9. Alur Keseluruhan Aplikasi**

1. **Frontend (Next.js dengan Shadcn UI)**:
    - Pengguna berinteraksi dengan antarmuka pengguna (UI) di halaman web.
    - Aksi pengguna (seperti menambah, mengedit, atau menghapus produk) dikirim ke API route melalui HTTP request.

2. **API Route (Next.js)**:
    - Menerima request dari frontend.
    - Menggunakan **`mysql2`** untuk berinteraksi dengan database MariaDB secara manual.

3. **Database (MariaDB)**:
    - Menyimpan data aplikasi.
    - Merespons request dari API sesuai dengan operasi CRUD yang dilakukan.

### **Diagram Alur**

```
[ Frontend (Next.js + Shadcn UI) ]
          |
          | HTTP Request (API)
          |
[ API Route (Next.js) ]
          |
          | mysql2
          |
[ Database (MariaDB) ]
```

## **10. Konsep Koneksi Database Lokal saat Deploy ke Vercel**

### **a. Koneksi ke Database Lokal Tidak Bisa Diakses dari Vercel**

Saat Anda mendeploy aplikasi ke **Vercel**, database lokal yang berjalan di Laragon **tidak bisa diakses langsung** dari Vercel karena Vercel berjalan di cloud dan tidak terhubung ke jaringan lokal Anda.

### **b. Solusi untuk Koneksi Database setelah Deploy**

Untuk menghubungkan aplikasi yang dideploy ke Vercel dengan database, Anda perlu menggunakan database yang dapat diakses secara publik atau database yang disediakan oleh layanan cloud. Berikut adalah beberapa opsi:

1. **Menggunakan Database Cloud:**
    - **PlanetScale**, **Amazon RDS**, **Google Cloud SQL**, atau layanan database lainnya yang mendukung akses dari internet.
    - Migrasikan database Anda dari lokal ke layanan cloud ini dan perbarui konfigurasi koneksi di aplikasi Anda.

2. **Menggunakan VPS atau Server Publik:**
    - Jika Anda memiliki VPS atau server publik, Anda bisa memasang MariaDB di sana dan mengatur akses dari luar.
    - Pastikan untuk mengamankan database Anda dengan benar (firewall, user permissions, SSL, dll).

3. **Menggunakan Layanan Database Terhosting:**
    - Layanan seperti **ClearDB**, **JawsDB**, atau **Supabase** menawarkan MariaDB terhosting yang dapat diintegrasikan dengan mudah.

### **c. Langkah-langkah Menghubungkan Database Cloud ke Aplikasi di Vercel**

1. **Migrasi Database:**
    - Pindahkan database dari Laragon ke layanan database cloud pilihan Anda.
    - Ekspor database dari Laragon menggunakan tools seperti phpMyAdmin atau `mysqldump` dan import ke database cloud.

2. **Dapatkan Kredensial Database Cloud:**
    - Setelah database berhasil di-migrasi, catat kredensial koneksi seperti `host`, `port`, `nama_database`, `username`, dan `password`.

3. **Perbarui Variabel Lingkungan di Vercel:**
    - Di dashboard Vercel, buka pengaturan proyek Anda.
    - Tambahkan variabel lingkungan yang sesuai (`DB_HOST`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`, `DB_PORT`) dengan nilai dari database cloud Anda.

4. **Perbarui File `.env.local` untuk Pengembangan:**
    - Pastikan file `.env.local` di lingkungan pengembangan Anda memiliki kredensial yang benar untuk mengakses database cloud.
    - Contoh `.env.local`:
      ```
      DB_HOST=your_cloud_host
      DB_USER=your_cloud_user
      DB_PASSWORD=your_cloud_password
      DB_NAME=your_cloud_db
      DB_PORT=3306
      ```

5. **Deploy Ulang Aplikasi ke Vercel:**
    - Setelah konfigurasi selesai, deploy ulang aplikasi Anda agar Vercel menggunakan kredensial baru untuk menghubungkan ke database cloud.

### **d. Keamanan dan Optimasi**

- **Keamanan:**
    - Gunakan SSL untuk koneksi ke database jika tersedia.
    - Batasi akses IP ke database cloud untuk hanya mengizinkan alamat IP tertentu jika memungkinkan.
    - Gunakan pengguna database dengan hak akses minimal yang diperlukan.

- **Optimasi:**
    - Pastikan database cloud Anda berada di region yang sama dengan Vercel untuk mengurangi latensi.
    - Monitor penggunaan dan skalabilitas database cloud sesuai kebutuhan aplikasi Anda.

## **11. Hierarki Folder Proyek**

Berikut adalah struktur folder lengkap dari proyek CRUD e-commerce sederhana ini:

```
ecommerce-app/
├── node_modules/
├── public/
│   ├── favicon.ico
│   └── ...
├── src/
│   ├── app/
│   │   ├── api/
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
├── .env.local
├── .gitignore
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
    - **`api/`**: Berisi API routes untuk produk.
      - **`products/`**: API routes untuk mengelola produk.
        - **`[id]/route.ts`**: API route untuk operasi spesifik berdasarkan ID produk (GET, PUT, DELETE).
        - **`route.ts`**: API route untuk operasi umum (GET all, POST).
    - **`globals.css`**: CSS global untuk Tailwind.
    - **`layout.tsx`**: Layout utama aplikasi, termasuk pengaturan ToastContainer.
    - **`page.tsx`**: Halaman utama yang menampilkan dan mengelola produk.
  - **`components/ui/`**: Komponen UI yang dibuat menggunakan Shadcn.
    - **`button.tsx`**, **`input.tsx`**, **`label.tsx`**, **`modal.tsx`**: Komponen UI yang dapat digunakan ulang.
  - **`lib/`**: Library atau helper functions.
    - **`db.ts`**: Konfigurasi koneksi database menggunakan `mysql2`.
- **`styles/`**: Berisi file CSS tambahan jika diperlukan.
- **`.env.local`**: File variabel lingkungan untuk menyimpan kredensial database.
- **`.gitignore`**: Mengabaikan file atau folder tertentu saat menggunakan Git.
- **`package.json`**: File manajemen dependensi dan skrip.
- **`tailwind.config.js`**: Konfigurasi Tailwind CSS.
- **`postcss.config.js`**: Konfigurasi PostCSS.
- **`tsconfig.json`**: Konfigurasi TypeScript.
- **`bun.lockb`**: Lockfile Bun untuk memastikan versi dependensi konsisten.

## **12. Penutup**

Dengan mengikuti langkah-langkah di atas, Anda telah berhasil membuat aplikasi CRUD e-commerce yang kokoh menggunakan **Next.js 14**, **TypeScript**, **Bun**, dan **MariaDB** sebagai database. Aplikasi ini memanfaatkan **`mysql2`** untuk mengelola koneksi database dan **Shadcn** sebagai penyedia komponen UI, memberikan tampilan yang estetis dan responsif.

### **Tips Tambahan:**

- **Validasi Input:** Pastikan untuk selalu memvalidasi data yang diterima dari pengguna baik di frontend maupun backend untuk mencegah masalah keamanan seperti SQL Injection.
- **Error Handling:** Implementasikan penanganan error yang lebih komprehensif untuk meningkatkan pengalaman pengguna dan memudahkan debugging.
- **Optimasi Performa:** Gunakan teknik caching atau optimasi query database untuk meningkatkan performa aplikasi.
- **Autentikasi dan Otorisasi:** Tambahkan fitur autentikasi pengguna jika diperlukan untuk mengamankan bagian tertentu dari aplikasi.
- **Testing:** Implementasikan testing (unit tests, integration tests) untuk memastikan kualitas kode dan fungsionalitas aplikasi.
- **Deployment:** Pastikan Anda mengikuti praktik terbaik saat mendeploy aplikasi ke platform produksi seperti Vercel, termasuk pengaturan variabel lingkungan yang aman dan optimasi performa.

Jika Anda mengalami kendala atau memiliki pertanyaan lebih lanjut, jangan ragu untuk menanyakannya!
