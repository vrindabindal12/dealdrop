# DealDrop - Smart Product Price Tracker
Track product prices across e-commerce sites and get alerts on price drops. Built with Next.js, Firecrawl, and Supabase.

## 🎯 Features

- 🔍 **Track Any Product** - Works with Amazon, Zara, Walmart, and more
- 📊 **Price History Charts** - Interactive graphs showing price trends over time
- 🔐 **Google Authentication** - Secure sign-in with Google OAuth
- 🔄 **Automated Daily Checks** - Scheduled cron jobs check prices automatically
- 📧 **Email Alerts** - Get notified when prices drop via Resend

## 🛠️ Tech Stack

- **Next.js 16** - React framework with App Router
- **Firecrawl** - Web data extraction API
  - Handles JavaScript rendering
  - Rotating proxies & anti-bot bypass
  - Structured data extraction with AI
  - Works across different e-commerce sites
- **Supabase** - Backend platform
  - PostgreSQL Database
  - Google Authentication
  - Row Level Security (RLS)
  - pg_cron for scheduled jobs
- **Resend** - Transactional emails
- **shadcn/ui** - UI component library
- **Recharts** - Interactive charts
- **Tailwind CSS** - Styling

## 📋 Prerequisites

Before you begin, ensure you have:

- Node.js 18+ installed
- A [Supabase](https://supabase.com) account
- A [Firecrawl](https://firecrawl.dev) account
- A [Resend](https://resend.com) account
- Google OAuth credentials from [Google Cloud Console](https://console.cloud.google.com/)

## 🚀 Setup Instructions

### 1. Clone and Install

```bash
git clone https://github.com/piyush-eon/smart-product-price-tracker.git
cd smart-product-price-tracker
npm install
```

### 2. Supabase Setup

#### Create Project

1. Create a new project at [supabase.com](https://supabase.com)
2. Wait for the project to be ready

#### Run Database Migrations

Go to SQL Editor in your Supabase dashboard and run these migrations:

**Migration 1: Database Schema** (`supabase/migrations/001_schema.sql`)

```sql
-- Enable UUID extension
create extension if not exists "uuid-ossp";

-- Products table
create table products (
  id uuid primary key default uuid_generate_v4(),
  user_id uuid references auth.users(id) on delete cascade not null,
  url text not null,
  name text not null,
  current_price numeric(10,2) not null,
  currency text not null default 'USD',
  image_url text,
  created_at timestamp with time zone default now(),
  updated_at timestamp with time zone default now()
);

-- Price history table
create table price_history (
  id uuid primary key default uuid_generate_v4(),
  product_id uuid references products(id) on delete cascade not null,
  price numeric(10,2) not null,
  currency text not null,
  checked_at timestamp with time zone default now()
);

-- Add unique constraint for upsert functionality
ALTER TABLE products
ADD CONSTRAINT products_user_url_unique UNIQUE (user_id, url);

-- Enable Row Level Security
alter table products enable row level security;
alter table price_history enable row level security;

-- Policies for products
create policy "Users can view their own products"
  on products for select
  using (auth.uid() = user_id);

create policy "Users can insert their own products"
  on products for insert
  with check (auth.uid() = user_id);

create policy "Users can update their own products"
  on products for update
  using (auth.uid() = user_id);

create policy "Users can delete their own products"
  on products for delete
  using (auth.uid() = user_id);

-- Policies for price_history
create policy "Users can view price history for their products"
  on price_history for select
  using (
    exists (
      select 1 from products
      where products.id = price_history.product_id
      and products.user_id = auth.uid()
    )
  );

-- Indexes for performance
create index products_user_id_idx on products(user_id);
create index price_history_product_id_idx on price_history(product_id);
create index price_history_checked_at_idx on price_history(checked_at desc);
```

**Migration 2: Setup Cron Job** (`supabase/migrations/002_setup_cron.sql`)

```sql
-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS pg_cron;
CREATE EXTENSION IF NOT EXISTS pg_net;

-- Create function to trigger price check via HTTP
CREATE OR REPLACE FUNCTION trigger_price_check()
RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
BEGIN
  PERFORM net.http_post(
    url := 'https://your-app-url.vercel.app/api/cron/check-prices',
    headers := jsonb_build_object(
      'Content-Type', 'application/json',
      'Authorization', 'Bearer YOUR_CRON_SECRET_HERE'
    )
  );
END;
$$;

-- Schedule cron job to run daily at 9 AM UTC
SELECT cron.schedule(
  'daily-price-check',
  '0 9 * * *',
  'SELECT trigger_price_check();'
);
```

**Note:** Update the URL and Authorization Bearer token in the function after deployment.

#### Enable Google Authentication

1. Go to **Authentication** → **Providers** in Supabase
2. Enable **Google** provider
3. Get OAuth credentials from [Google Cloud Console](https://console.cloud.google.com/):
   - Create a new project or select existing
   - Enable Google+ API
   - Create OAuth 2.0 credentials
   - Add authorized redirect URI: `https://<your-project>.supabase.co/auth/v1/callback`
4. Copy Client ID and Client Secret to Supabase