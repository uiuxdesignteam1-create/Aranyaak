# Aranyaak · Payment hand-over for the developer

The website already has the complete payment UI. It runs in **demo mode** today:
a test payment page lets anyone try success, failure, cancel and hold-expiry.
To go live with SSLCOMMERZ, only a small backend and two config values are needed.
No design or front-end flow changes.

Design reference: Figma file `GjdnQt1uHRObFO7arYL2Mc`, section 21 "Payment flow — all states".

## 1. Switch to live

In `index.html` find:

```js
const PAYMENT={mode:'demo', initUrl:''};
```

and change it to:

```js
const PAYMENT={mode:'live', initUrl:'https://YOUR-BACKEND/pay/init'};
```

## 2. What the site sends to `initUrl` (POST, JSON)

```json
{
  "ref": "ARK-26-40177",
  "amount": 13050,
  "currency": "BDT",
  "total": 26100,
  "method": "card | bkash | nagad | rocket | netbank",
  "cottage": "Camellia Cottage",
  "checkin": "2026-12-19",
  "checkout": "2026-12-21",
  "guests": 2, "kids": 0,
  "name": "Rahim Uddin", "email": "guest@example.com", "phone": "+8801711000000",
  "extras": ["airport"],
  "success_url": "https://site/?pay=success&ref=ARK-26-40177",
  "fail_url":    "https://site/?pay=fail&ref=ARK-26-40177",
  "cancel_url":  "https://site/?pay=cancel&ref=ARK-26-40177"
}
```

The backend must reply `{"GatewayPageURL": "https://..."}`. The site then sends the guest there.
Any other reply (or a network error) shows the "Payment failed" screen.

## 3. What the backend does

1. **Re-calculate the amount on the server.** Never trust `amount` from the browser.
2. Create the SSLCOMMERZ session (store_id / store_passwd stay on the server only):
   - Sandbox: `https://sandbox.sslcommerz.com/gwprocess/v4/api.php`
   - Live: `https://securepay.sslcommerz.com/gwprocess/v4/api.php`
   - Use `ref` as `tran_id`. Point SSLCOMMERZ's `success_url`, `fail_url`, `cancel_url`
     and `ipn_url` at **your backend**, not at the website.
3. Return `GatewayPageURL` to the site.
4. When SSLCOMMERZ calls back (success URL or IPN), validate with the Order Validation API
   (`val_id`, `store_id`, `store_passwd`) and check status `VALID`, amount, currency and `tran_id`.
   - Sandbox: `https://sandbox.sslcommerz.com/validator/api/validationserverAPI.php`
   - Live: `https://securepay.sslcommerz.com/validator/api/validationserverAPI.php`
5. Only after validation, mark the booking paid, send the email and SMS, then redirect the guest
   to the website's `success_url`. On failure use `fail_url`; on cancel use `cancel_url`.

## 4. Screens the site already handles

| URL / action | Screen |
|---|---|
| Guest presses "Confirm and pay" | Redirecting to secure payment |
| `?pay=success&ref=…` | Booking confirmed (booking number, receipt download, calendar, WhatsApp) |
| `?pay=fail&ref=…` | Payment failed (try again / another method, 20-min hold countdown) |
| `?pay=cancel&ref=…` | Payment cancelled (return to payment / change dates) |
| Hold timer reaches 0 | Hold expired (check availability again) |
| Bank transfer chosen | Booking requested · awaiting payment (bank details placeholder) |
| Menu → My booking | Find my booking (booking number + mobile) |

Before redirecting, the site saves the booking in `sessionStorage` (`ark-pay`) and restores it on return.

## 5. Still needed from the client

- SSLCOMMERZ merchant account (sandbox first, then live store ID and password)
- Bank account details for the bank-transfer screen (currently "To be added")
- A real "My booking" lookup needs a database on the backend. Today it only finds the booking
  made in the same browser session.
