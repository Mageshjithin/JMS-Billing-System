# JMS Textiles — Billing System

GST billing, stock and customer management for JMS Textiles.

- **Backend:** Node.js + Express + MongoDB (Mongoose), JWT login
- **Frontend:** React + Vite (in the shop's maroon and gold colours)

## Features

| Area | What it does |
| --- | --- |
| **New Bill (POS)** | Barcode scan or name/SKU search, qty, per-item and whole-bill discount, live GST, split payments (Cash / UPI / Card / Bank), credit bills, change to return. Shortcuts: `F2` search, `F9` save |
| **GST** | CGST + SGST, or IGST for inter-state sales. Prices can include or exclude GST. Garments get their rate from the per-piece slab (default: up to ₹2,500 → 5%, above → 18%), or you can set a fixed rate per product |
| **Invoices** | Number format `JMS/2026-27/0001`, restarting each financial year. A4 or 80 mm thermal print, amount in words, GST breakup, WhatsApp share, take payments on credit bills later, cancel a bill (puts the stock back) |
| **Products & stock** | One row per size/colour with SKU, barcode, HSN and stock. Low-stock alerts, add/remove stock, CSV export |
| **Customers** | Saved automatically from the phone number on a bill. Shows purchase history and amount due |
| **Reports** | Today / month dashboard, daily sales, GST summary by rate and HSN (for GSTR-1 / 3B), top products, CSV export |
| **Users** | Admin and Staff roles. Only admins can cancel bills, delete records or change settings |

## Quick start (try it without installing MongoDB)

Requires Node.js 18 or newer.

```bash
cd billing-system/backend
npm install
cp .env.example .env
npm run dev:memory
```

`dev:memory` starts a temporary in-memory MongoDB and loads sample products. **Nothing is saved when you stop it.** The first run downloads a MongoDB binary (about 600 MB).

In a second terminal:

```bash
cd billing-system/frontend
npm install
npm run dev
```

Open http://localhost:5173 and log in with the admin account from `.env` (`SEED_ADMIN_USERNAME` / `SEED_ADMIN_PASSWORD`). **Change that password in Settings straight away.**

## Real setup with MongoDB

1. Get MongoDB, either:
   - **Local:** install [MongoDB Community Server](https://www.mongodb.com/try/download/community) and keep `MONGO_URI=mongodb://127.0.0.1:27017/jms_billing`, or
   - **Cloud:** create a free [MongoDB Atlas](https://www.mongodb.com/atlas) cluster and paste its connection string into `MONGO_URI`.
2. In `backend/.env`, set `JWT_SECRET` to a long random string:
   ```bash
   node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
   ```
3. Create the admin user and sample products. Skip this if you want to start with no products; the admin user is also created on the first server start.
   ```bash
   cd backend
   npm run seed
   ```
4. Start it:
   ```bash
   npm run dev
   ```
   Run the frontend with `npm run dev` as shown above.

## Production (single server)

Build the frontend once. The backend then serves it, so everything runs on one port:

```bash
cd frontend && npm run build
cd ../backend && npm start
```

Open http://localhost:5000. To keep it running on a shop PC or VPS, use a process manager such as `pm2 start src/server.js --name jms-billing`.

## First things to do

1. **Settings → Shop details:** address, phone, **GSTIN** (bills print "TAX INVOICE" once a GSTIN is set), and the footer note.
2. **Settings → Billing & GST:** check the GST slab and the "prices include GST" option with your accountant.
3. **Products:** delete the sample items and add your real stock. Give each size its own SKU, and add a barcode if you use a scanner.
4. **Staff accounts:** create a Staff login for each person at the counter.

## Project layout

```
billing-system/
├── backend/
│   └── src/
│       ├── server.js          # start-up
│       ├── app.js             # Express app and routes; serves frontend/dist in production
│       ├── models/            # User, Product, Customer, Invoice, Counter, Setting
│       ├── routes/            # auth, users, products, customers, invoices, reports, settings
│       ├── services/tax.js    # GST calculation (the server's numbers are the ones saved)
│       ├── seed.js            # admin user and sample products
│       └── dev-memory.js      # in-memory MongoDB for trying it out
└── frontend/
    └── src/
        ├── pages/             # Dashboard, Billing, Invoices, InvoiceView, Products, Customers, Reports, Settings
        ├── components/        # Layout, modal, toasts, icons
        └── lib/               # api client, auth, formatting, tax.js (live preview copy)
```

> `frontend/src/lib/tax.js` is a copy of `backend/src/services/tax.js`, used for the live total on the billing screen. If you change the GST logic, change both files.

## API summary

All routes are under `/api` and need `Authorization: Bearer <token>`, except login.

| Method | Route | Purpose |
| --- | --- | --- |
| POST | `/auth/login` | Log in, returns a token |
| GET/POST/PUT/DELETE | `/products` | Products; `/products/lookup/:code` matches a SKU or barcode exactly; `/products/:id/stock` adjusts stock |
| GET/POST/PUT/DELETE | `/customers` | Customers with their totals |
| GET/POST | `/invoices` | List or create bills (stock is checked and reduced in the same step) |
| POST | `/invoices/:id/payments` | Take a payment on a credit bill |
| POST | `/invoices/:id/cancel` | Cancel a bill and return its stock (admin only) |
| GET | `/reports/dashboard`, `/reports/sales`, `/reports/gst`, `/reports/top-products` | Reports; take `?from=YYYY-MM-DD&to=YYYY-MM-DD` |
| GET/PUT | `/settings` | Shop settings |
| GET/POST/PUT | `/users` | Staff accounts (admin only) |
