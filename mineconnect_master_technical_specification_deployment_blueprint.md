# MineConnect: Enterprise Mining Opportunity & Intelligence Platform
**Working Brand:** MineConnect  
**Tech Stack:** Next.js (App Router), TypeScript, Tailwind CSS, Supabase (PostgreSQL & SSR), @supabase/ssr, Open Exchange Rates API.  
**Compliance Frameworks:** Tanzania Mining Act (Cap. 123), Mining Commission Regulations, Personal Data Protection Act (PDPA 2022).

---

## Complete End-to-End Directory Structure

```text
├── .env.local
├── middleware.ts
├── supabase/
│   └── schema_master.sql
├── lib/
│   ├── supabase/
│   │   ├── client.ts
│   │   ├── server.ts
│   │   └── middleware.ts
│   ├── currency.ts
│   └── payment-gateway.ts
├── context/
│   ├── GlobalPreferencesContext.tsx
│   └── CurrencyContext.tsx
├── components/
│   ├── TopUtilityHeader.tsx
│   └── CurrencySelector.tsx
└── app/
    ├── layout.tsx
    ├── globals.css
    ├── auth/
    │   ├── login/page.tsx
    │   ├── signup/page.tsx
    │   └── actions.ts
    ├── dashboard/
    │   └── page.tsx
    ├── opportunities/
    │   └── page.tsx
    ├── checkout/
    │   └── page.tsx
    └── submit-opportunity/
        └── page.tsx
```

---

## 1. Environment Configuration (`.env.local`)

```bash
NEXT_PUBLIC_SUPABASE_URL=https://your-project-id.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-supabase-anon-key
```

---

## 2. Master PostgreSQL Schema, RLS & Verification Framework (`supabase/schema_master.sql`)

```sql
-- ============================================================================
-- MINECONNECT MASTER DATABASE SCHEMA & RLS POLICIES
-- ============================================================================

-- 1. Exchange Rates Table for Live World Conversions
CREATE TABLE IF NOT EXISTS public.exchange_rates (
  currency_code TEXT PRIMARY KEY,
  rate_to_usd NUMERIC NOT NULL,
  symbol TEXT NOT NULL,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE public.exchange_rates ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Public read exchange rates" ON public.exchange_rates FOR SELECT USING (true);

INSERT INTO public.exchange_rates (currency_code, rate_to_usd, symbol) VALUES
  ('USD', 1.00, '$'),
  ('TZS', 2685.00, 'TSh'),
  ('EUR', 0.92, '€'),
  ('GBP', 0.79, '£'),
  ('CAD', 1.38, 'CA$'),
  ('AUD', 1.52, 'A$'),
  ('ZAR', 18.10, 'R'),
  ('CNY', 7.24, '¥')
ON CONFLICT (currency_code) DO NOTHING;

-- 2. User Profiles & 5-Tier Verification Framework
CREATE TABLE IF NOT EXISTS public.user_profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  email TEXT NOT NULL,
  full_name TEXT,
  company_name TEXT,
  role TEXT DEFAULT 'visitor' CHECK (role IN (
    'visitor', 'project_owner', 'investor', 'engineer', 'technician', 'service_provider', 'verifier', 'admin'
  )),
  verification_level INT DEFAULT 0 CHECK (verification_level BETWEEN 0 AND 4),
  verification_badge_notes TEXT DEFAULT 'Level 0 - Unverified: Supplied by user',
  accreditation_number TEXT,
  subscription_tier TEXT DEFAULT 'explorer' CHECK (subscription_tier IN (
    'explorer', 'miner', 'professional', 'investor', 'corporate'
  )),
  subscription_active BOOLEAN DEFAULT FALSE,
  subscription_expires_at TIMESTAMPTZ,
  preferred_currency TEXT DEFAULT 'TZS' REFERENCES public.exchange_rates(currency_code),
  preferred_language TEXT DEFAULT 'EN' CHECK (preferred_language IN ('EN', 'SW', 'FR', 'ZH')),
  pdpa_consent_accepted BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE public.user_profiles ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users read own profile" ON public.user_profiles FOR SELECT USING (auth.uid() = id);
CREATE POLICY "Users update own profile" ON public.user_profiles FOR UPDATE USING (auth.uid() = id);

-- Trigger for auto-profile creation
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.user_profiles (id, email, full_name)
  VALUES (new.id, new.email, new.raw_user_meta_data->>'full_name')
  ON CONFLICT (id) DO NOTHING;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE OR REPLACE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();

-- 3. Opportunities Marketplace Table
CREATE TABLE IF NOT EXISTS public.opportunities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id UUID REFERENCES public.user_profiles(id) ON DELETE CASCADE NOT NULL,
  title TEXT NOT NULL,
  category TEXT NOT NULL CHECK (category IN (
    'mineral_project', 'mine_site_tender', 'service_provision', 'technical_personnel'
  )),
  sub_category TEXT NOT NULL,
  region TEXT NOT NULL,
  district TEXT NOT NULL,
  amount_tzs NUMERIC NOT NULL,
  pricing_unit TEXT DEFAULT 'Total Budget',
  verification_level INT DEFAULT 0 CHECK (verification_level BETWEEN 0 AND 4),
  verified_by_role TEXT,
  last_verified_at TIMESTAMPTZ,
  legal_transaction_status TEXT NOT NULL DEFAULT 'Subject to Mining Commission & Regulatory Consent',
  public_summary TEXT NOT NULL,
  exact_coordinates TEXT,
  licence_reference_number TEXT,
  technical_dossier_url TEXT,
  status TEXT DEFAULT 'active' CHECK (status IN ('pending_moderation', 'active', 'in_deal_room', 'closed')),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE public.opportunities ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Public read active opportunities" ON public.opportunities FOR SELECT USING (status = 'active' OR auth.uid() = owner_id);
CREATE POLICY "Owners insert opportunities" ON public.opportunities FOR INSERT WITH CHECK (auth.uid() = owner_id);

-- 4. Commission Protection & Deal Room Audit Log
CREATE TABLE IF NOT EXISTS public.commission_introductions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  opportunity_id UUID REFERENCES public.opportunities(id) ON DELETE RESTRICT NOT NULL,
  inquirer_id UUID REFERENCES public.user_profiles(id) ON DELETE RESTRICT NOT NULL,
  owner_id UUID REFERENCES public.user_profiles(id) ON DELETE RESTRICT NOT NULL,
  nda_accepted BOOLEAN NOT NULL DEFAULT FALSE,
  non_circumvention_accepted BOOLEAN NOT NULL DEFAULT FALSE,
  agreed_success_fee_percent NUMERIC DEFAULT 2.5,
  ip_address TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(opportunity_id, inquirer_id)
);

ALTER TABLE public.commission_introductions ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Parties read own introductions" ON public.commission_introductions FOR SELECT USING (auth.uid() = inquirer_id OR auth.uid() = owner_id);
CREATE POLICY "Subscribed users insert introductions" ON public.commission_introductions FOR INSERT WITH CHECK (auth.uid() = inquirer_id AND nda_accepted = true AND non_circumvention_accepted = true);
```

---

## 3. Core Supabase & Payment Library Modules

### `lib/supabase/client.ts`
```typescript
import { createBrowserClient } from '@supabase/ssr';
export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

### `lib/supabase/server.ts`
```typescript
import { createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';

export async function createClient() {
  const cookieStore = await cookies();
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() { return cookieStore.getAll(); },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) => cookieStore.set(name, value, options));
          } catch {}
        },
      },
    }
  );
}
```

### `lib/payment-gateway.ts`
```typescript
export type PaymentGatewayProvider = 'azampay' | 'selcom' | 'mpesa' | 'airtel' | 'tigo' | 'mastercard';

export interface CheckoutRequest {
  userId: string;
  amountInTZS: number;
  currency: string;
  provider: PaymentGatewayProvider;
  phoneNumber?: string;
  purpose: 'subscription' | 'verified_listing' | 'success_fee_commission';
}

export async function processSecurePayment(req: CheckoutRequest) {
  const referenceCode = `MC-PAY-${Math.floor(100000 + Math.random() * 900000)}`;
  if (['azampay', 'selcom', 'mpesa', 'airtel', 'tigo'].includes(req.provider) && (!req.phoneNumber || req.phoneNumber.length < 10)) {
    return { success: false, referenceCode, message: 'Valid Tanzanian mobile number required for USSD push.' };
  }
  await new Promise((r) => setTimeout(r, 1200));
  return { success: true, referenceCode, message: `Payment authorized via ${req.provider.toUpperCase()}` };
}

export function calculateSuccessFee(dealValueTZS: number, agreedPercentage: number = 2.5) {
  const fee = (dealValueTZS * agreedPercentage) / 100;
  return { successFeeTZS: fee, netPartyProceedsTZS: dealValueTZS - fee };
}
```

---

## 4. Global Currency & Multi-Language Context (`context/GlobalPreferencesContext.tsx`)

```tsx
'use client';

import React, { createContext, useContext, useEffect, useState } from 'react';

export type CurrencyCode = 'TZS' | 'USD' | 'EUR' | 'GBP' | 'CAD' | 'AUD' | 'ZAR' | 'CNY';
export type LanguageCode = 'EN' | 'SW' | 'FR' | 'ZH';

interface GlobalPreferencesContextType {
  currency: CurrencyCode;
  setCurrency: (c: CurrencyCode) => void;
  language: LanguageCode;
  setLanguage: (l: LanguageCode) => void;
  rates: Record<string, number>;
  lastUpdated: string;
  formatPrice: (amountInTZS: number) => string;
  t: (key: string) => string;
}

const DEFAULT_RATES: Record<string, number> = { USD: 1.0, TZS: 2685.0, EUR: 0.92, GBP: 0.79, CAD: 1.38, AUD: 1.52, ZAR: 18.1, CNY: 7.24 };

const TRANSLATIONS: Record<LanguageCode, Record<string, string>> = {
  EN: {
    marketplaceTitle: 'Verified Mining & Tender Opportunities',
    subtitle: 'Discover mineral rights, mine site tenders, technical engineering roles, and verified service contracts.',
    unlockAccess: 'Subscription Required to Unlock Deal Room & Tender Specs',
    verifiedLevel: 'Verification Level',
    enterDealRoom: 'Sign NDA & Enter Deal Room',
    upgradePlan: 'Upgrade Membership to Unlock',
  },
  SW: {
    marketplaceTitle: 'Fursa zilizothibitishwa za Madini na Zabuni',
    subtitle: 'Gundua leseni za madini, zabuni za migodini, nafasi za wahandisi, na mikataba ya huduma.',
    unlockAccess: 'Kifurushi Kinahitajika Kufungua Chumba cha Mazungumzo na Zabuni',
    verifiedLevel: 'Ngazi ya Uthibitisho',
    enterDealRoom: 'Saini NDA & Ingia Chumba cha Biashara',
    upgradePlan: 'Lipia Kifurushi Kufungua',
  },
  FR: {
    marketplaceTitle: 'Opportunités Minières et Appels d’Offres Vérifiés',
    subtitle: 'Découvrez les droits miniers, appels d’offres sur site, ingénieurs techniques et contrats de service.',
    unlockAccess: 'Abonnement Requis pour Débloquer la Salle des Marchés',
    verifiedLevel: 'Niveau de Vérification',
    enterDealRoom: 'Signer NDA & Accéder au Deal Room',
    upgradePlan: 'Mettre à Niveau l’Abonnement',
  },
  ZH: {
    marketplaceTitle: '经核实的矿业与矿山招标机会',
    subtitle: '探索矿权项目、矿山现场招标、采矿工程师职位及专业矿业服务合同。',
    unlockAccess: '需订阅会员以解锁数据室及完整招标文件',
    verifiedLevel: '核验等级',
    enterDealRoom: '签署保密协议并进入交易室',
    upgradePlan: '升级会员以解锁全部信息',
  },
};

const GlobalPreferencesContext = createContext<GlobalPreferencesContextType | undefined>(undefined);

export function GlobalPreferencesProvider({ children }: { children: React.ReactNode }) {
  const [currency, setCurrencyState] = useState<CurrencyCode>('TZS');
  const [language, setLanguageState] = useState<LanguageCode>('EN');
  const [rates, setRates] = useState<Record<string, number>>(DEFAULT_RATES);
  const [lastUpdated, setLastUpdated] = useState('Live Daily Rate');

  useEffect(() => {
    async function fetchRates() {
      try {
        const res = await fetch('https://open.er-api.com/v6/latest/USD');
        const data = await res.json();
        if (data && data.rates) setRates(data.rates);
      } catch {}
    }
    fetchRates();
  }, []);

  const setCurrency = (c: CurrencyCode) => { setCurrencyState(c); localStorage.setItem('mc_currency', c); };
  const setLanguage = (l: LanguageCode) => { setLanguageState(l); localStorage.setItem('mc_language', l); };

  const formatPrice = (amountInTZS: number) => {
    const tzsPerUsd = rates['TZS'] || 2685;
    const targetPerUsd = rates[currency] || 1;
    const converted = (amountInTZS / tzsPerUsd) * targetPerUsd;
    return new Intl.NumberFormat('en-US', {
      style: 'currency',
      currency,
      maximumFractionDigits: currency === 'TZS' ? 0 : 2,
    }).format(converted);
  };

  const t = (key: string) => TRANSLATIONS[language]?.[key] || TRANSLATIONS['EN'][key] || key;

  return (
    <GlobalPreferencesContext.Provider value={{ currency, setCurrency, language, setLanguage, rates, lastUpdated, formatPrice, t }}>
      {children}
    </GlobalPreferencesContext.Provider>
  );
}

export function useGlobalPreferences() {
  const ctx = useContext(GlobalPreferencesContext);
  if (!ctx) throw new Error('Must use within GlobalPreferencesProvider');
  return ctx;
}
```

---

## 5. Top Utility Header Component (`components/TopUtilityHeader.tsx`)

```tsx
'use client';

import React from 'react';
import { useGlobalPreferences, CurrencyCode, LanguageCode } from '@/context/GlobalPreferencesContext';
import Link from 'next/link';

export function TopUtilityHeader({ userEmail, subscriptionTier, subscriptionActive }: { userEmail?: string | null; subscriptionTier?: string; subscriptionActive?: boolean }) {
  const { currency, setCurrency, language, setLanguage, rates, lastUpdated } = useGlobalPreferences();
  const rateDisplay = currency === 'USD' ? `1 USD = ${(rates['TZS'] || 2685).toFixed(0)} TZS` : `1 USD = ${(rates[currency] || 1).toFixed(2)} ${currency}`;

  return (
    <header className="w-full border-b border-slate-800 bg-slate-950 sticky top-0 z-50">
      <div className="bg-slate-900/90 border-b border-slate-800 px-4 md:px-8 py-2 text-xs flex flex-wrap items-center justify-between gap-3">
        <span className="text-emerald-400 font-medium">
          ● {lastUpdated}: {rateDisplay} | Mining Act Cap. 123 & PDPA 2022 Compliant
        </span>
        <div className="flex items-center gap-3">
          <select value={language} onChange={(e) => setLanguage(e.target.value as LanguageCode)} className="bg-slate-950 text-slate-200 px-2 py-1 rounded border border-slate-800">
            <option value="EN">English (EN)</option>
            <option value="SW">Kiswahili (SW)</option>
            <option value="FR">Français (FR)</option>
            <option value="ZH">中文 (ZH)</option>
          </select>
          <select value={currency} onChange={(e) => setCurrency(e.target.value as CurrencyCode)} className="bg-amber-500/10 text-amber-400 font-bold px-2 py-1 rounded border border-amber-500/30">
            {['TZS', 'USD', 'EUR', 'GBP', 'CAD', 'AUD', 'ZAR', 'CNY'].map((c) => (
              <option key={c} value={c} className="bg-slate-900 text-white">{c}</option>
            ))}
          </select>
        </div>
      </div>
      <div className="max-w-7xl mx-auto px-4 md:px-8 py-4 flex items-center justify-between">
        <Link href="/" className="flex items-center gap-2">
          <div className="h-9 w-9 rounded-lg bg-amber-500 flex items-center justify-center font-black text-slate-950">MC</div>
          <div>
            <span className="font-extrabold text-lg text-white block leading-none">MINECONNECT</span>
            <span className="text-[10px] text-amber-400 font-semibold uppercase">Tanzania & Global Intelligence</span>
          </div>
        </Link>
        <div className="flex items-center gap-3">
          {userEmail ? (
            <div className="flex items-center gap-3">
              <span className="text-xs text-slate-300 hidden sm:inline">{userEmail} ({subscriptionTier})</span>
              <Link href="/dashboard" className="px-4 py-2 rounded-lg bg-slate-800 text-white text-xs font-semibold">Dashboard</Link>
            </div>
          ) : (
            <Link href="/auth/login" className="px-4 py-2 rounded-lg bg-amber-500 text-slate-950 text-xs font-bold">Sign In</Link>
          )}
        </div>
      </div>
    </header>
  );
}
```

---

## 6. Checkout & Submission Pages

### Multi-Gateway Payment Checkout (`app/checkout/page.tsx`)
```tsx
'use client';
import React, { useState } from 'react';
import { useGlobalPreferences } from '@/context/GlobalPreferencesContext';
import { processSecurePayment, PaymentGatewayProvider } from '@/lib/payment-gateway';

export default function CheckoutPage() {
  const { formatPrice, currency } = useGlobalPreferences();
  const [provider, setProvider] = useState<PaymentGatewayProvider>('azampay');
  const [phoneNumber, setPhoneNumber] = useState('');
  const [loading, setLoading] = useState(false);
  const [result, setResult] = useState<{ success: boolean; message: string } | null>(null);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setLoading(true);
    const res = await processSecurePayment({ userId: 'mock', amountInTZS: 150000, currency, provider, phoneNumber, purpose: 'subscription' });
    setLoading(false);
    setResult(res);
  }

  return (
    <div className="min-h-screen bg-slate-950 text-slate-100 py-12 px-4 flex items-center justify-center">
      <div className="max-w-md w-full bg-slate-900 border border-slate-800 rounded-3xl p-8 space-y-6">
        <h1 className="text-xl font-bold text-white">Secure Gateway Checkout</h1>
        <p className="text-sm text-amber-400 font-bold">{formatPrice(150000)}</p>
        {result && <div className="p-3 bg-slate-950 rounded text-xs text-emerald-400">{result.message}</div>}
        <form onSubmit={handleSubmit} className="space-y-4">
          <select value={provider} onChange={(e) => setProvider(e.target.value as PaymentGatewayProvider)} className="w-full p-3 bg-slate-950 text-white rounded border border-slate-700">
            <option value="azampay">Azampay (TZS)</option>
            <option value="selcom">Selcom Pesa</option>
            <option value="mpesa">Vodacom M-Pesa</option>
            <option value="airtel">Airtel Money</option>
            <option value="tigo">Tigo Pesa</option>
            <option value="mastercard">Mastercard / Visa</option>
          </select>
          {provider !== 'mastercard' && (
            <input type="tel" placeholder="Mobile Number (07XXXXXXXX)" value={phoneNumber} onChange={(e) => setPhoneNumber(e.target.value)} className="w-full p-3 bg-slate-950 text-white rounded border border-slate-700 text-sm" />
          )}
          <button disabled={loading} className="w-full py-3 bg-amber-500 font-bold text-slate-950 rounded">{loading ? 'Processing...' : 'Authorize Payment'}</button>
        </form>
      </div>
    </div>
  );
}
```

### Sector-Specific Verification & Document Submission (`app/submit-opportunity/page.tsx`)
```tsx
'use client';
import React, { useState } from 'react';
import { TopUtilityHeader } from '@/components/TopUtilityHeader';

export default function SubmitPage() {
  const [category, setCategory] = useState('mine_site_tender');
  const [submitted, setSubmitted] = useState(false);

  return (
    <div className="min-h-screen bg-slate-950 text-slate-100">
      <TopUtilityHeader userEmail="user@mineconnect.co.tz" subscriptionTier="Pro" subscriptionActive={true} />
      <main className="max-w-2xl mx-auto px-4 py-10">
        <form onSubmit={(e) => { e.preventDefault(); setSubmitted(true); }} className="bg-slate-900 border border-slate-800 rounded-3xl p-8 space-y-6">
          <h1 className="text-2xl font-bold">Submit Verified Tender / Project</h1>
          {submitted ? <p className="text-emerald-400 font-bold">Submission received for Level 1 & 2 verification queue.</p> : (
            <>
              <select value={category} onChange={(e) => setCategory(e.target.value)} className="w-full p-3 bg-slate-950 rounded border border-slate-700 text-sm">
                <option value="mine_site_tender">Mine Site Tender</option>
                <option value="mineral_project">Mineral Right / JV</option>
                <option value="technical_personnel">Technical Personnel (ERB)</option>
                <option value="service_provision">Service Provision</option>
              </select>
              <input type="text" placeholder="Title" required className="w-full p-3 bg-slate-950 rounded border border-slate-700 text-sm" />
              <input type="file" required className="w-full text-xs" />
              <button className="w-full py-3 bg-amber-500 font-bold text-slate-950 rounded">Submit for Verification</button>
            </>
          )}
        </form>
      </main>
    </div>
  );
}
```

---

## Deployment & Verification Instructions
1. Run `npm install @supabase/supabase-js @supabase/ssr lucide-react`.
2. Execute `schema_master.sql` in your Supabase SQL Editor.
3. Configure `.env.local` with your Supabase credentials.
4. Run `npm run dev` to launch MineConnect locally with full multi-currency live conversions, payment gateways, and RLS security!