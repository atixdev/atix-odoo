# ATIX Payment Gateway for Odoo

Integrates the [ATIX](https://www.atix.com.pe) payment gateway into Odoo's e-commerce checkout. Supports credit/debit card payments in **PEN (Peruvian Soles)** and **USD (US Dollars)** via a popup widget.

---

## Table of Contents

1. [Features](#features)
2. [Requirements](#requirements)
3. [Installation](#installation)
4. [Configuration](#configuration)
5. [Payment Flow](#payment-flow)
6. [Automatic Status Polling](#automatic-status-polling)
7. [Manual Status Check](#manual-status-check)
8. [Staging / Database Copies](#staging--database-copies)
9. [Supported Currencies](#supported-currencies)
10. [Uninstallation](#uninstallation)

---

## Features

- Popup-based payment widget (no page redirect for card entry)
- Dual-currency support: separate API keys for PEN and USD
- Automatic switching between Test (Sandbox) and Production endpoints
- Token-based transaction tracking
- Automatic status polling via cron job (every 2 minutes)
- Manual status refresh action from the backend
- Partial refund support
- Database neutralization script (wipes API keys on DB copy)
- Computed Return URL helper for ATIX Dashboard configuration

---

## Requirements

- Odoo 18.0
- Odoo modules: `payment`, `base_automation`, `website`, `website_sale`
- An active ATIX account with API keys for PEN and/or USD

---

## Installation

1. Copy the `payment_atix` folder into your Odoo addons directory.

2. Update your addons list:
   - Go to **Settings > Technical > Activate Developer Mode**
   - Go to **Settings > Apps > Update Apps List**

3. Search for `payment_atix` (or "ATIX") in the Apps list and click **Install**.

   The module will automatically register and activate the ATIX payment provider on install.

---

## Configuration

### 1. Open the ATIX provider settings

Go to **Website > Configuration > Payment Providers** (or **Accounting > Configuration > Payment Providers**) and click on **ATIX**.

### 2. Enter your API Keys

In the **Credentials** tab, fill in:

| Field | Description |
|---|---|
| **API Key (PEN)** | API Key for Peruvian Sol transactions, provided by ATIX |
| **API Key (USD)** | API Key for US Dollar transactions, provided by ATIX |

> These fields are restricted to users with the **Administrator** (`base.group_system`) role.

### 3. Configure the Return URL in the ATIX Dashboard

The **Return URL** field (read-only, auto-computed) displays the exact URL that must be registered in your ATIX Dashboard for both the **Approved** and **Rejected** redirect fields. Copy this value and paste it in your ATIX merchant portal settings.

The URL has the format:
```
https://<your-domain>/payment_atix/status/{token}
```

### 4. Set the provider state

- Set the provider to **Test** to use the ATIX Sandbox environment.
- Set to **Enabled** to process real payments via the Production endpoint.

### 5. Publish the provider

Make sure the ATIX provider is set to **Published** so it appears on the checkout page.

---

## Payment Flow

1. The customer reaches the checkout on your Odoo website and selects ATIX as the payment method.
2. A payment popup opens in the browser (loaded from the ATIX JS SDK). The customer enters their card details directly with ATIX.
3. Upon completion, the ATIX widget returns a session token and a redirect URL.
4. The token is saved on the Odoo transaction (status: **Pending**).
5. The customer is redirected to the ATIX-hosted confirmation or 3DS authentication page.
6. After ATIX processes the payment, the customer is redirected back to Odoo via the Return URL.
7. Odoo fetches the final transaction status from the ATIX API:
   - Result code `00` → transaction marked **Done**
   - Result code `-99` → transaction marked **Cancelled**
8. The customer is redirected to their order page in the portal.

---

## Automatic Status Polling

A scheduled cron job runs **every 2 minutes** and checks the status of any ATIX transaction that:
- Has a stored token (payment was initiated)
- Does **not** yet have a final reference code
- Was created within the **last 6 hours**

This acts as a safety net for cases where the ATIX callback was not received (e.g., the customer closed the browser before being redirected back).

The cron job is installed and enabled automatically and requires no manual setup.

---

## Manual Status Check

Administrators can manually trigger a status check for any ATIX transaction from the backend:

1. Go to **Accounting > Payments > Transactions** (or the equivalent payment transaction list).
2. Select one or more ATIX transactions.
3. From the **Action** menu, run **"Consulta estado de pago por pasarela ATIX"** (Query payment status via ATIX gateway).

This immediately polls the ATIX API for the current status of the selected transaction(s).

---

## Staging / Database Copies

When duplicating a production database (e.g., to create a staging environment), Odoo automatically runs `data/neutralize.sql`, which sets both ATIX API keys to `NULL`:

```sql
UPDATE payment_provider SET atix_apikey_pen = NULL, atix_apikey_usd = NULL;
```

This prevents the staging database from accidentally processing real payments with production credentials. You must enter new (test) API keys after restoring the database on a staging server.

---

## Supported Currencies

| Currency | Code |
|---|---|
| Peruvian Sol | PEN |
| US Dollar | USD |

Transactions in any other currency will not be offered the ATIX payment option at checkout.

---

## Uninstallation

Uninstalling the module from **Settings > Apps** will automatically disable and remove the ATIX provider registration from your Odoo instance. No manual cleanup is required.
