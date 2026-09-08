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

create policy "Cualquiera puede publicar su lista"
  on entries for insert
  with check (true);

create policy "Cualquiera puede actualizar su propia lista"
  on entries for update
  using (true);

alter table entries add constraint entries_name_unique unique (name);

-- Sin política de SELECT a propósito: nadie puede leer la tabla completa.

-- Registro de "quién ya eligió a quién" — es lo que impide elegir dos veces,
-- aunque se borre el navegador o se entre desde otro dispositivo: el bloqueo
-- vive en el servidor, atado al nombre de quien busca (picker_name).
create table if not exists claims (
  picker_name text primary key,
  assigned_name text not null,
  claimed_at timestamptz not null default now()
);

alter table claims enable row level security;
-- Sin políticas de select/insert directas: solo se toca vía la función de abajo.

create or replace function claim_or_get_assignment(p_picker_name text, p_target_name text default null)
returns table (id uuid, name text, options jsonb, already_claimed boolean)
language plpgsql
security definer
set search_path = public
as $$
declare
  v_picker text := lower(trim(p_picker_name));
  v_target text;
  v_was_existing boolean;
begin
  if v_picker is null or v_picker = '' then
    raise exception 'picker_name_required';
  end if;

  select assigned_name into v_target from claims where picker_name = v_picker;

  if v_target is null then
    if p_target_name is null or trim(p_target_name) = '' then
      raise exception 'target_required';
    end if;

    insert into claims (picker_name, assigned_name)
    values (v_picker, trim(p_target_name))
    on conflict (picker_name) do nothing;

    select assigned_name into v_target from claims where picker_name = v_picker;
  end if;

  v_was_existing := (v_target is distinct from trim(coalesce(p_target_name, '')));

  return query
  select e.id, e.name, e.options, v_was_existing
  from entries e
  where lower(trim(e.name)) = lower(trim(v_target))
  limit 1;
end;
$$;

revoke all on function claim_or_get_assignment(text, text) from public;
grant execute on function claim_or_get_assignment(text, text) to anon, authenticated;
```

**Cómo funciona el bloqueo**: quien busca escribe su propio nombre (`picker_name`) y el nombre de su amigo secreto (`target_name`). La primera vez, el servidor guarda esa pareja en `claims` y la devuelve. Cualquier búsqueda posterior con el mismo `picker_name` ignora el `target_name` que se escriba y siempre devuelve la asignación original. `localStorage` en el navegador solo recuerda el nombre de quien buscó para no tener que volver a escribirlo — el bloqueo real está en la base de datos, no en el navegador.

3. En **Project Settings → API**, copia la **Project URL** y la **anon public key** (o **publishable key**, en proyectos nuevos).

**Diseño de privacidad**: nadie puede leer la tabla `entries` directamente (no hay política de `select`). La única forma de ver una lista es llamando a `get_entry_by_name` con el nombre exacto de la persona — así cada quien solo ve la lista de a quien le tocó, no la de todos.

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
