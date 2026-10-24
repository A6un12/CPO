# Make admin-added coupon codes work for every user

The quiz is a static page, so shared coupon codes need a small free online database. This guide uses Supabase (free plan). It takes about 10 minutes.

With this setup:
- Coupon codes live in the database, not in the page source.
- The admin password is checked by Supabase, not stored in the page.
- Only the admin can see the full list, add codes or delete codes.
- Students can only check whether the code they typed is valid.

## 1. Create the project
1. Go to supabase.com and sign up, then choose New project (any name, pick a region near India, set a database password and keep it safe).
2. Wait until the project finishes setting up.

## 2. Create the table and the check function
Open SQL Editor, choose New query, paste everything below and press Run.

```sql
create table public.coupons (
  code text primary key,
  created_at timestamptz not null default now()
);

alter table public.coupons enable row level security;

create policy "admin can read coupons" on public.coupons
  for select to authenticated using (true);
create policy "admin can add coupons" on public.coupons
  for insert to authenticated with check (true);
create policy "admin can delete coupons" on public.coupons
  for delete to authenticated using (true);

create or replace function public.check_coupon(c text)
returns boolean
language sql
security definer
set search_path = public
as $$ select exists (select 1 from public.coupons where code = upper(c)) $$;

grant execute on function public.check_coupon(text) to anon, authenticated;

insert into public.coupons (code) values ('CPO661') on conflict do nothing;
```

## 3. Create the admin login
1. Open Authentication, then Users, then Add user, then Create new user.
2. Enter an email (for example `admin@yourdomain.com`) and the admin password you want. Tick Auto Confirm User, then create it.
3. Open Authentication, then Sign In / Providers (or Settings) and turn OFF "Allow new users to sign up". This makes sure nobody else can create an account.

On the quiz page the admin username stays `admin`. The password is the one you set in Supabase.

## 4. Connect the page
1. Open Project Settings, then API (or API Keys). Copy the Project URL and the anon public key (it is meant to be public; never use the service_role key).
2. In your GitHub repository open `index.html`, press the pencil icon, and find this line near the bottom:

   `const BACKEND={url:'',key:'',adminEmail:''};`

3. Fill it in, for example:

   `const BACKEND={url:'https://abcd1234.supabase.co',key:'eyJ...your-anon-key...',adminEmail:'admin@yourdomain.com'};`

4. Commit the change. GitHub Pages updates in about a minute.

## 5. Use it
- Tap Admin on the coupon page, log in as `admin` with your Supabase password, and add codes. Each code shows the date and time it was added, and works for every student straight away.
- Students enter a code on the coupon page. The page asks the database whether it is valid.

## Good to know
- Until step 4 is done, the hosted page shows "Admin sign-in is not set up yet" and no coupon code works. That is intentional, so nothing secret is in the page.
- The admin session lasts about an hour. If it expires, the page asks you to log in again.
- Students need an internet connection to enter a code. Once unlocked, a device stays unlocked until the power icon is tapped or browser data is cleared.
- A determined person could try many guesses against the check function. Use codes that are not easy to guess.
