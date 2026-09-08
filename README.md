# Amigo Secreto

Página de lista de deseos para amigo secreto. Sitio estático (HTML/CSS/JS, sin build), conectado a Supabase.

## 1. Configurar Supabase

1. Crea un proyecto en [supabase.com](https://supabase.com).
2. En **SQL Editor**, corre:

```sql
create table entries (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  options jsonb not null default '[]'::jsonb,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

alter table entries enable row level security;

create policy "Cualquiera puede ver la lista"
  on entries for select
  using (true);

create policy "Cualquiera puede publicar su lista"
  on entries for insert
  with check (true);

create policy "Cualquiera puede actualizar su propia lista"
  on entries for update
  using (true);
```

3. En **Project Settings → API**, copia la **Project URL** y la **anon public key**.

## 2. Configurar el sitio

Abre `index.html`, busca este bloque cerca del inicio del `<script>` y pega tus valores:

```js
var SUPABASE_URL = 'PON_AQUI_TU_SUPABASE_URL';
var SUPABASE_ANON_KEY = 'PON_AQUI_TU_SUPABASE_ANON_KEY';
```

Mientras no los pongas, la página muestra un aviso y el formulario queda deshabilitado — así sabrás si falta este paso.

## 3. Subir a GitHub

```bash
git init
git add .
git commit -m "Amigo secreto: lista de deseos"
git branch -M main
git remote add origin https://github.com/<tu-usuario>/amigo-secreto.git
git push -u origin main
```

(O usa `gh repo create amigo-secreto --public --source=. --push` si ya tienes `gh` logueado.)

## 4. Desplegar en Vercel

1. Entra a [vercel.com](https://vercel.com), inicia sesión (GitHub o Google).
2. **Add New Project** → conecta tu cuenta de GitHub si no lo has hecho → selecciona el repo `amigo-secreto`.
3. Vercel detecta que es un sitio estático — no hay que tocar ninguna configuración.
4. **Deploy**.

Listo — la URL que te da Vercel es la que compartes con todos.
