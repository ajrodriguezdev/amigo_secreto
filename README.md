# Amigo Secreto

Página de lista de deseos y sorteo para amigo secreto. Sitio estático (HTML/CSS/JS, sin build), conectado a Supabase.

## 1. Configurar Supabase

1. Crea un proyecto en [supabase.com](https://supabase.com).
2. En **SQL Editor**, corre este script completo (es idempotente: se puede volver a correr sin romper nada ni borrar datos existentes):

```sql
-- Tabla base de listas de deseos (si no existe todavía)
create table if not exists entries (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  options jsonb not null default '[]'::jsonb,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

alter table entries enable row level security;

-- Sin políticas de select/insert/update directas: todo pasa por las
-- funciones de abajo.
drop policy if exists "Cualquiera puede ver la lista" on entries;
drop policy if exists "Cualquiera puede publicar su lista" on entries;
drop policy if exists "Cualquiera puede actualizar su propia lista" on entries;

do $$
begin
  if not exists (select 1 from pg_constraint where conname = 'entries_name_unique') then
    alter table entries add constraint entries_name_unique unique (name);
  end if;
end $$;

-- Roster fijo de participantes (solo nombres, sin PIN propio)
create table if not exists participants (
  name text primary key,
  created_at timestamptz not null default now()
);

insert into participants (name) values
  ('Miguel Becerra'), ('Larry Ceballos'), ('Mayra Lozano'), ('Yeison Ramirez'),
  ('Miguel Villamizar'), ('Néstor Lozano'), ('Néstor Salazar'),
  ('Fernando Rodriguez'), ('Liseth Sandoval')
on conflict (name) do nothing;

alter table participants enable row level security;
alter table participants drop column if exists pin;
-- Sin políticas: nunca se lee directo desde el cliente.

-- Registro de "quién eligió a quién" + el PIN generado al elegir
create table if not exists claims (
  picker_name text primary key,
  assigned_name text not null,
  pin text,
  claimed_at timestamptz not null default now()
);

alter table claims enable row level security;
alter table claims add column if not exists pin text;

create unique index if not exists claims_assigned_name_unique_idx
  on claims (lower(trim(assigned_name)));
create unique index if not exists claims_pin_unique_idx
  on claims (pin) where pin is not null;

drop function if exists save_entry_with_pin(text, text, jsonb);
drop function if exists claim_or_get_assignment(text, text, text);
drop function if exists claim_or_get_assignment(text, text);
drop function if exists get_entry_by_name(text);

-- Publicar/actualizar tu lista: solo exige que tu nombre esté en el roster
create or replace function save_entry(p_name text, p_options jsonb)
returns table (id uuid, name text)
language plpgsql
security definer
set search_path = public
as $$
begin
  if not exists (select 1 from participants where lower(trim(name)) = lower(trim(p_name))) then
    raise exception 'unknown_participant';
  end if;

  return query
  insert into entries (name, options, updated_at)
  values (trim(p_name), p_options, now())
  on conflict (name) do update
    set options = excluded.options, updated_at = now()
  returning entries.id, entries.name;
end;
$$;

revoke all on function save_entry(text, jsonb) from public;
grant execute on function save_entry(text, jsonb) to anon, authenticated;

-- Elegir a tu amigo secreto UNA VEZ: genera un PIN aleatorio como comprobante
create or replace function draw_secret_santa(p_picker_name text, p_target_name text)
returns table (id uuid, name text, options jsonb, pin text)
language plpgsql
security definer
set search_path = public
as $$
declare
  v_picker text := lower(trim(p_picker_name));
  v_target text := trim(p_target_name);
  v_pin text;
  v_attempts int := 0;
begin
  if v_picker = '' then raise exception 'picker_name_required'; end if;
  if v_target = '' then raise exception 'target_required'; end if;
  if not exists (select 1 from participants where lower(trim(name)) = v_picker) then
    raise exception 'unknown_participant';
  end if;
  if exists (select 1 from claims where picker_name = v_picker) then
    raise exception 'already_drawn';
  end if;
  if exists (select 1 from claims where lower(trim(assigned_name)) = lower(v_target)) then
    raise exception 'target_already_taken';
  end if;

  loop
    v_pin := lpad(floor(random()*1000000)::text, 6, '0');
    v_attempts := v_attempts + 1;
    begin
      insert into claims (picker_name, assigned_name, pin) values (v_picker, v_target, v_pin);
      exit;
    exception
      when unique_violation then
        if v_attempts > 20 then
          raise exception 'target_already_taken';
        end if;
    end;
  end loop;

  return query
  select e.id, e.name, e.options, v_pin
  from entries e
  where lower(trim(e.name)) = lower(v_target)
  limit 1;
end;
$$;

revoke all on function draw_secret_santa(text, text) from public;
grant execute on function draw_secret_santa(text, text) to anon, authenticated;

-- Consultar tu asignación más tarde, SOLO con tu PIN (sin nombre)
create or replace function get_assignment_by_pin(p_pin text)
returns table (id uuid, name text, options jsonb)
language sql
security definer
set search_path = public
as $$
  select e.id, c.assigned_name as name, coalesce(e.options, '[]'::jsonb) as options
  from claims c
  left join entries e on lower(trim(e.name)) = lower(trim(c.assigned_name))
  where c.pin = trim(p_pin)
  limit 1;
$$;

revoke all on function get_assignment_by_pin(text) from public;
grant execute on function get_assignment_by_pin(text) to anon, authenticated;

-- Nombres que todavía nadie ha elegido, para el desplegable de destino
create or replace function get_available_targets()
returns table (name text)
language sql
security definer
set search_path = public
as $$
  select p.name
  from participants p
  where lower(trim(p.name)) not in (
    select lower(trim(assigned_name)) from claims
  )
  order by p.name asc;
$$;

revoke all on function get_available_targets() from public;
grant execute on function get_available_targets() to anon, authenticated;

-- Los 9 nombres, para llenar los desplegables "Tu nombre"
create or replace function get_participant_names()
returns table (name text)
language sql
security definer
set search_path = public
as $$
  select name from participants order by name asc;
$$;

revoke all on function get_participant_names() from public;
grant execute on function get_participant_names() to anon, authenticated;
```

3. En **Project Settings → API**, copia la **Project URL** y la **anon public key** (o **publishable key**, en proyectos nuevos).

## Cómo funciona la seguridad

- **Roster fijo**: solo los 9 nombres en `participants` pueden usar el sitio — ambos desplegables ("Tu nombre") se llenan desde ahí.
- **Publicar tu lista**: no requiere PIN, solo que tu nombre esté en el roster (`save_entry`).
- **Elegir a tu amigo secreto**: es un evento único por persona (`draw_secret_santa`). Al elegir, el servidor genera un **PIN aleatorio de 6 dígitos** y lo devuelve **una sola vez** — el sitio lo muestra en un campo bloqueado para copiar/guardar.
- **Consultar después**: la única forma de volver a ver tu asignación es con ese PIN, en la sección "¿Ya elegiste? Consulta con tu PIN" (`get_assignment_by_pin`) — no hace falta volver a identificarte por nombre.
- **Un solo objetivo por persona**: el índice único en `claims.assigned_name` impide que dos personas terminen con el mismo asignado, incluso si eligen al mismo tiempo.
- **Elección fija**: si alguien intenta elegir de nuevo con el mismo nombre, `draw_secret_santa` rechaza con `already_drawn`.

**Límite honesto**: el PIN se muestra **una sola vez**. Si se pierde, la persona no tiene forma de recuperarlo por sí misma — solo tú, como administrador, puedes buscarlo directo en Supabase:
```sql
select picker_name, assigned_name, pin from claims where picker_name = 'Nombre de la persona';
```
No lo publiques ni lo compartas fuera de ese caso puntual.

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

Listo — la URL que le compartes a todos es la misma para los 9; cada quien elige su nombre, publica su lista, y cuando elige a su amigo secreto recibe su propio PIN para volver.
