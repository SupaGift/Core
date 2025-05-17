# Breakout Gifts 🎁

---

## Table of Contents
1. Quick Start
2. Tech Stack
3. Folder Structure & DDD Layers
4. Domain Model
5. Sender Flow (UX & Code Walk-Through)
6. Receiver Flow (UX & Code Walk-Through)
7. On-chain Payment Lifecycle
8. Environment Variables
9. NPM Scripts
10. Deployment (Vercel ✈️)
11. Backlog / Inactive Buttons

---

## 1  Quick Start

```bash
git clone https://github.com/your-org/breakout-gifts.git
cd breakout-gifts
cp .env.example .env.local            # fill creds – see §8
npm i
npm run dev
```

Open http://localhost:3000 and create your first gift.

---

## 2  Tech Stack

| Layer          | Library / Service                      |
| -------------- | -------------------------------------- |
| Front-end      | Next.js 14 (App Router) + Tailwind CSS |
| Auth & Wallets | `@privy-io/react-auth`                 |
| Database       | Supabase (PostgreSQL + Realtime)       |
| Edge Fns       | Supabase Edge Functions (TypeScript)   |
| Blockchain     | Solana (`@solana/web3.js`)             |
| Payments       | Solana Pay (`@solana/pay`) + QRCode    |
| Forms/Validation | React Hook Form + Zod                |

---

## 3  Folder Structure & DDD Layers
src/
├─ app/ # Next.js Route Handlers & Pages
│ ├─ sender/ # Sender journey (amount ➜ type ➜ checkout ➜ pay ➜ success)
│ ├─ gift/ # Receiver journey ([giftId]/index + withdraw)
│ └─ api/ # Route handlers (Privy create-user, etc.)
├─ entities/ # Pure domain logic
│ ├─ gift/ # DTOs + Supabase adapter
│ └─ user/
├─ features/ # Re-usable UX slices
│ ├─ auth/ # Privy login components
│ └─ sender-flow/ # Global React Context for wizard
├─ shared/ # Cross-cutting utilities & UI atoms
│ ├─ lib/ # supabaseClient, solana.ts helpers…
│ └─ ui/ # Button, AuthHeader, etc.

*FSD keeps UI/State logic close to pages while DDD isolates business rules in `entities/`.*

---

## 4  Domain Model

### `gifts` table

| column            | type      | purpose                           |
| ----------------- | --------- | --------------------------------- |
| `id` (pk)         | `uuid`    | Internal DB id                    |
| `gift_id`         | `text`    | Public, embedded in link / QR     |
| `sender_email`    | `text`    | Bound to Privy identity           |
| `receiver_email`  | `text`    | Gift recipient                    |
| `receiver_wallet` | `text`    | Privy auto-generated wallet       |
| `amount`          | `numeric` | Gift size (SOL)                   |
| `token`           | `text`    | `SOL` or `USDC`                   |
| `status`          | `text`    | `pending` ➜ `paid` ➜ `claimed`    |
| `claimed`         | `ts`      | When receiver clicked *Claim*     |
| `last_update`     | `ts`      | Row audit                         |
| `is_staked`       | `bool`    | Placeholder (inactive)            |

A minimal `users` table stores `{ email, wallet }` for analytics.

---

![Sender diagram flow](supagift_1.png)

## 5  Sender Flow 🔄

| Page                 | Component                                   | Key Logic |
| -------------------- | ------------------------------------------- | --------- |
| `/sender/amount`     | `AmountPage.tsx`                            | React Hook Form + Zod – stores amount in `SenderFlowContext` |
| `/sender/type`       | `TypePage.tsx`                              | Select ⬩ Token ⬩ Staking* ⬩ NFT* |
| `/sender/checkout`   | `CheckoutPage.tsx`                          | Calls **Supabase Edge Fn** ➜ creates Privy wallet for receiver & `gifts` row |
| `/sender/pay`        | `PayPage.tsx`                               | • Generate `reference` + Solana Pay URL<br>• Render QR via `qrcode.react`<br>• Poll chain + listen to Supabase realtime to mark `paid` |
| `/sender/success`    | Simple confirmation                         | Shows shareable QR / link |

\* Staking & NFT remain **inactive buttons**.

The wizard state persists in `<SenderFlowProvider>` enabling back/forward navigation without DB writes until checkout.

---

![Receiver diagram flow](supagift_2.png)

## 6  Receiver Flow 🎉

1. User opens `/gift/[giftId]` from link or QR.
2. Gift fetched using `fetchGiftById()` -> locked view.
3. Privy email login verifies `user.email.address === gift.receiver_email`.
4. On first claim ➜ `updateGiftStatus(gift_id, "claimed")`.
5. Receiver can  
   • **Withdraw**: dynamic import of `withdraw.tsx` uses Privy `sendTransaction` to transfer SOL/USDC.  
   • **Off-ramp**: inactive pink button (future fiat).

All branch assets live under the same page – routing stays shallow to keep links clean.

---

## 7  On-chain Payment Lifecycle

```mermaid
sequenceDiagram
  participant Sender
  participant Wallet
  participant SolanaChain
  participant Supabase

  Sender->>Wallet: Scan Solana Pay QR
  Wallet->>SolanaChain: Transfer SOL with `reference`
  loop until confirmed
    PayPage->>SolanaChain: findReference(reference)
  end
  SolanaChain-->>PayPage: tx signature
  PayPage->>Supabase: update gifts.status = 'paid'
  Supabase-->>PayPage (realtime): row update
  PayPage->>Sender: redirect /success
```

Realtime DB push removes need for webhooks locally; in production you can mirror the same logic in a Supabase Edge Function trigger.

---

## 8  Environment Variables (`.env.local`)

```env
# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://xxxxxxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=public_anon_key
SUPABASE_SERVICE_ROLE_KEY=service_role_key   # only for Edge Fns

# Privy
NEXT_PUBLIC_PRIVY_APP_ID=prvy_app_id
PRIVY_APP_SECRET=prvy_app_secret

# Blockchain
NEXT_PUBLIC_SOLANA_NETWORK=devnet            # devnet | testnet | mainnet-beta
```

---

## 9  Deployment

The repository is **Vercel-ready** – environment variables must be set in the dashboard.  
Supabase Edge Functions are auto-deployed via `supabase functions deploy` in CI/CD.

---

## 10  Backlog / Inactive Buttons

* Staking integrations (Marinade / Jito)
* NFT upload & minting
* Fiat off-ramp
* Gift link expiration & tracking

These features are visible in the UI but intentionally disabled pending spec.

---

Built with ❤️ for the Solana Breakpoint hackathon – fork, extend and ship your own gifting experience!
