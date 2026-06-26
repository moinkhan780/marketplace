# Multi-Vendor Delivery & Services Marketplace

**Project Specification — Version 4 (Production Ready)**
**Flutter Web · Firebase · M-Pesa · Twilio WhatsApp**

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [System Components](#2-system-components)
3. [Technology Stack](#3-technology-stack)
4. [Architecture — Folder Structure](#4-architecture--folder-structure)
5. [Database Schema (Firestore)](#5-database-schema-firestore)
6. [Firestore Security Rules](#6-firestore-security-rules)
7. [Cloud Functions — Specifications](#7-cloud-functions--specifications)
8. [Scheduled Functions](#8-scheduled-functions)
9. [Firebase Cost Optimisation](#9-firebase-cost-optimisation)
10. [Customer Ordering Process](#10-customer-ordering-process)
11. [Payment Flow](#11-payment-flow)
12. [Notification Flow (WhatsApp)](#12-notification-flow-whatsapp)
13. [Vendor Workflow](#13-vendor-workflow)
14. [Rider Workflow](#14-rider-workflow)
15. [Service Provider Workflow](#15-service-provider-workflow)
16. [Admin Workflow](#16-admin-workflow)
17. [Money Flow Scenarios](#17-money-flow-scenarios)
18. [Order Cancellation Detail](#18-order-cancellation-detail)
19. [UI Mockups](#19-ui-mockups)
20. [Implementation Phases](#20-implementation-phases)
21. [Success Metrics](#21-success-metrics)

---

## 1. Project Overview

A Flutter Web marketplace platform connecting customers with multiple independent vendors (laundry, fresh produce, quick bites, etc.) and independent service providers (e.g. home services), supported by a dedicated rider network for deliveries. Customers can add items from multiple vendors into a single cart — one order, one rider, one delivery fee, one payment. Vendors manage their own products and a single business location, and receive their portion of incoming orders directly. Service providers list bookable services that customers can request — customers contact providers directly via WhatsApp to arrange bookings; no payment is collected through the platform for service bookings. Riders pick up from all vendors in the order then deliver to the customer. Admin manually pays out riders and vendors via M-Pesa app and marks payments settled in the dashboard.

---

## 2. System Components

### 2.1 Customer Web (Flutter Web — Customer Facing)

- **Purpose:** Browse vendors/products and services, add items from multiple vendors into one cart, place orders, pay (M-Pesa or COD), track orders, order history. Contact service providers via WhatsApp directly from service listings.
- **Access:** Web browser (responsive design), guest checkout supported

### 2.2 Admin Web (Flutter Web — Platform Admin Facing)

- **Purpose:** Approve/manage vendors, service providers, and riders; set commission and subscription rates; manage admin-defined categories; platform-wide reporting (tables only); manually mark rider/vendor payouts as paid; process refunds manually
- **Access:** Secure admin login; re-authentication required for all financial write operations

### 2.3 Vendor Web (Flutter Web — Vendor Facing)

- **Purpose:** Vendor self-registration, managing own products/pricing (within admin-defined categories), managing a single business location, receiving and actioning their portion of incoming orders, viewing commission deductions and payout status
- **Access:** Secure vendor login, gated by admin activation
- **Note:** Vendors do not assign riders. Rider is auto-assigned by the system once ALL vendors in the order accept.

### 2.4 Service Provider Web (Flutter Web — Service Provider Facing)

- **Purpose:** Service provider self-registration, listing services + pricing (within admin-defined categories), viewing subscription status. Customers contact providers via WhatsApp directly from the service listing — no in-app booking flow.
- **Access:** Secure service provider login, gated by admin activation

### 2.5 Rider Web (Mobile Web — Rider Facing)

- **Purpose:** Rider self-registration, viewing assigned orders (auto-assigned by system), updating delivery status per pickup and drop-off, collecting COD cash, managing wallet/recharge, viewing earnings and payout history
- **Access:** Mobile browser — login gated by admin activation; PWA manifest enabled for home screen installation

---

## 3. Technology Stack

### 3.1 Frontend

- **Framework:** Flutter Web (single codebase, single repository, role-based routing via GoRouter)
- **State Management:** Provider package (per-feature ChangeNotifiers; `context.select()` for scoped rebuilds — per-vendor ChangeNotifiers for cart to prevent full cart tree rebuilds)
- **Authentication:** Firebase Authentication (phone auth + custom claims per role)
- **Responsive:** Mobile-first responsive design
- **Image Loading:** `cached_network_image` with disk cache; `ListView.builder` for lazy loading on browse screens

### 3.2 Backend

- **Database:** Firebase Firestore (NoSQL) — `vendorSubOrders` stored as subcollection `orders/{orderId}/subOrders/{vendorId}`
- **Cloud Functions:** Firebase Cloud Functions v2 (all credentials via Secret Manager — never environment config)
- **File Storage:** Firebase Storage (uploads quarantined via Cloud Function trigger validation; public product images served via Firebase Hosting CDN — not Storage download URLs)

### 3.3 Third-party Integrations

- **WhatsApp Notifications:** Twilio API (only notification channel; credentials in Secret Manager)
- **Payment:** M-Pesa Daraja API — STK Push with IP allowlist + secret token validation on callback
- **Distance:** Haversine formula computed in Cloud Functions (no external API)
- **Map Picker:** Google Maps JavaScript API (map pin selection only; manual lat/lng entry fields provided as fallback when Maps API is unavailable)

---

## 4. Architecture — Folder Structure

Single Flutter project. All five user roles are organised as top-level features under `lib/features/`. Shared domain logic, models, and services live in `lib/core/`. Role-based routing via GoRouter with Firebase Auth custom claims guards. Firestore offline persistence is disabled for the Admin Web build target.

```
lib/
├── core/
│   ├── models/           order.dart, vendor.dart, rider.dart, ...
│   ├── services/         firestore_service.dart, auth_service.dart,
│   │                     twilio_service.dart, mpesa_service.dart
│   ├── providers/        auth_provider.dart  (shared role detection)
│   ├── routing/          app_router.dart  (GoRouter, role guards)
│   └── widgets/          shared UI components
├── features/
│   ├── customer/
│   │   ├── browse/       model/ view/ provider/
│   │   ├── cart/         model/ view/ provider/  ← per-vendor ChangeNotifiers
│   │   ├── checkout/     model/ view/ provider/
│   │   ├── orders/       model/ view/ provider/
│   │   └── services/     model/ view/ provider/
│   ├── vendor/
│   │   ├── products/     model/ view/ provider/
│   │   ├── orders/       model/ view/ provider/
│   │   ├── location/     model/ view/ provider/
│   │   └── earnings/     model/ view/ provider/
│   ├── rider/
│   │   ├── deliveries/   model/ view/ provider/
│   │   ├── wallet/       model/ view/ provider/
│   │   └── earnings/     model/ view/ provider/
│   ├── service_provider/
│   │   ├── services/     model/ view/ provider/
│   │   └── subscription/ model/ view/ provider/
│   └── admin/
│       ├── vendors/      model/ view/ provider/
│       ├── riders/       model/ view/ provider/
│       ├── orders/       model/ view/ provider/
│       ├── payouts/      model/ view/ provider/
│       ├── refunds/      model/ view/ provider/
│       └── settings/     model/ view/ provider/
└── main.dart
```

---

## 5. Database Schema (Firestore)

### 5.1 Collection Overview

`vendorSubOrders` is a **subcollection** — not an array on the orders document. Path: `orders/{orderId}/subOrders/{vendorId}`. This is required because:

- Firestore cannot index into array-of-map fields — the poll function cannot query `subOrderStatus` or `autoDeclineWarningSent` inside an array
- Vendor order streams would return all vendors' data if the array remained on the parent doc
- A 10-vendor order with 20 items each approaches the 1 MB Firestore document limit

A separate `pending_suborders` index collection (shallow docs: `orderId`, `vendorId`, `createdAt`, `subOrderStatus`, `warningSent`) exists solely for the poll function's queries.

| Collection / Subcollection | Key Fields | Notes |
|---|---|---|
| `users` | userId, name, phone, role, isActive | Customer, vendor, rider, service_provider, admin |
| `categories` | categoryId, name, isActive | Admin-managed only |
| `vendors` | vendorId, status, commissionPercent, location{}, locationConfigured | Single location embedded |
| `products` | productId, vendorId, categoryId, price, isAvailable | Hidden if vendor locationConfigured=false |
| `orders` | orderId, userId, paymentMethod, orderStatus, codAmountDue, deliveryFeeBaseAmount, cancellationWindowExpiresAt, cancellationWindowClosed, refundedAmount | vendorSubOrders moved to subcollection |
| `orders/{id}/subOrders/{vendorId}` | subOrderStatus, subTotal, vendorNetAmount, autoDeclineWarningSent, codConfirmedAt | Was array on orders doc |
| `pending_suborders` | orderId, vendorId, createdAt, subOrderStatus, warningSent | Lightweight index for poll queries |
| `riders` | riderId, isOnline, activeDeliveryCount, lastAssignedAt, pendingCODBalance, riderAccountStatus | lastAssignedAt initialised to epoch on activation |
| `rider_wallet_transactions` | type (cod_collection/recharge_payment/monthly_settlement), amount, balanceAfter | Cloud Functions only |
| `payouts` | payeeType, grossAmount, netPayoutAmount, payoutStatus | Created manually by admin |
| `refunds` | refundType, refundStatus, declinedVendorId (optional) | Managed by admin |
| `service_providers` | providerId, subscriptionStatus, subscriptionDueDate | |
| `services` | serviceId, providerPhone (denormalized — auto-updated by trigger) | |
| `failed_notifications` | orderId, recipient, message, failedAt, retryCount | Dead-letter for Twilio failures |
| `payment_attempts` | orderId+uid composite key, count, hourlyCount | STK Push rate limiting |
| `upload_rejections` | originalPath, quarantinePath, contentType, size | Admin review of rejected uploads |
| `platform_settings` | deliveryZones[], riderSharePercent, customerCancelWindowMinutes, bankDetails{} | |

### 5.2 `users` Collection

```json
{
  "userId":    "string (Firebase Auth UID)",
  "name":      "string",
  "phone":     "string (verified)",
  "email":     "string (optional)",
  "apartment": "string",
  "wing":      "string",
  "floor":     "string",
  "role":      "string (customer/admin/vendor/service_provider/rider)",
  "isActive":  "boolean",
  "isGuestOrder": "boolean (false for registered accounts)",
  "createdAt": "timestamp",
  "updatedAt": "timestamp"
}
```

### 5.3 `categories` Collection

```json
{
  "categoryId": "string",
  "name":       "string",
  "isActive":   "boolean",
  "createdAt":  "timestamp",
  "updatedAt":  "timestamp"
}
```

### 5.4 `vendors` Collection

```json
{
  "vendorId":          "string (Firebase Auth UID)",
  "businessName":      "string",
  "ownerName":         "string",
  "phone":             "string (verified)",
  "businessRegDocUrl": "string (Firebase Storage URL)",
  "ownerIdPhotoUrl":   "string (Firebase Storage URL)",
  "categoryIds":       "array of strings",
  "status":            "string (pending_activation/active/rejected/blocked)",
  "commissionPercent": "number",
  "location": {
    "name":    "string",
    "address": "string",
    "lat":     "number",
    "lng":     "number"
  },
  "locationConfigured": "boolean (false until vendor sets lat/lng)",
  "rejectionReason":    "string (optional)",
  "blockedReason":      "string (optional)",
  "createdAt":          "timestamp",
  "activatedAt":        "timestamp (optional)",
  "updatedAt":          "timestamp"
}
```

### 5.5 `products` Collection

```json
{
  "productId":     "string",
  "vendorId":      "string",
  "categoryId":    "string",
  "name":          "string",
  "description":   "string",
  "price":         "number",
  "unit":          "string (kg/piece/serving)",
  "imageUrl":      "string (Firebase Hosting CDN URL — 800px resized)",
  "thumbUrl":      "string (Firebase Hosting CDN URL — 200px thumbnail)",
  "isAvailable":   "boolean",
  "stockQuantity": "number",
  "createdAt":     "timestamp",
  "updatedAt":     "timestamp"
}
```

### 5.6 `orders/{orderId}/subOrders/{vendorId}` Subcollection

```json
{
  "vendorId":                "string",
  "vendorName":              "string",
  "items": [
    {
      "itemId":              "string",
      "name":                "string",
      "quantity":            "number",
      "price":               "number",
      "category":            "string",
      "specialInstructions": "string"
    }
  ],
  "subTotal":                "number",
  "vendorCommissionPercent": "number",
  "vendorCommissionAmount":  "number",
  "vendorNetAmount":         "number",
  "subOrderStatus":          "string (pending|accepted|declined|preparing|ready|picked_up|cancelled)",
  "autoDeclineWarningSent":  "boolean (default false)",
  "declineReason":           "string (optional)",
  "acceptedAt":              "timestamp (optional)",
  "readyAt":                 "timestamp (optional)",
  "pickedUpAt":              "timestamp (optional)",
  "codConfirmedAt":          "timestamp (optional)"
}
```

> `vendorCommissionPercent` is **snapshotted at order creation and never mutated**. It is the source of truth for this order's commission calculation regardless of any subsequent admin rate changes.

### 5.7 `orders` Document

```json
{
  "orderId":                     "string",
  "userId":                      "string (Firebase Auth UID or anonymous)",
  "isGuestOrder":                "boolean",
  "userName":                    "string",
  "userPhone":                   "string",
  "vendorSubOrderIds":           "array of vendorIds (for transaction getAll)",
  "overallSubtotal":             "number (sum of all vendor subTotals at checkout — original, never mutated)",
  "deliveryFee":                 "number (0 if waived; locked at checkout)",
  "deliveryFeeWaived":           "boolean",
  "deliveryFeeBaseAmount":       "number (pre-waiver fee — always stored; used for rider payout)",
  "riderSharePercent":           "number (snapshotted at order creation — never mutated)",
  "farthestVendorDistanceKm":    "number (Haversine, used to set fee)",
  "deliveryZone":                "string (Zone 1-5 or beyond_zone)",
  "totalAmountCharged":          "number (overallSubtotal + deliveryFee at checkout)",
  "refundedAmount":              "number (running total — prevents double-refund, default 0)",
  "codAmountDue":                "number (set at rider assignment — sum of accepted subTotals + deliveryFee; uses subTotal NOT vendorNetAmount)",
  "codAmountCollected":          "number (optional — set when rider confirms cash)",
  "codSettlementStatus":         "string (n/a | awaiting_rider_recharge | collected)",
  "paymentMethod":               "string (mpesa | cod)",
  "paymentStatus":               "string (pending/pending_collection/confirmed/failed/partially_refunded/fully_refunded)",
  "paymentReference":            "string (M-Pesa transaction ID, optional for cod)",
  "orderStatus":                 "string (pending/all_accepted_awaiting_rider/partially_declined/out_for_delivery/delivered/cancelled)",
  "cancellationWindowExpiresAt": "timestamp (set to createdAt + window on creation; set to now() on first vendor accept)",
  "cancellationWindowClosed":    "boolean (atomic flag — prevents duplicate window-close writes)",
  "cancelledAt":                 "timestamp (optional)",
  "cancellationReason":          "string (optional)",
  "deliveryLocation": {
    "method":      "string (current_location/manual_address/map_pin)",
    "lat":         "number",
    "lng":         "number",
    "apartment":   "string (optional)",
    "wing":        "string (optional)",
    "floor":       "string (optional)",
    "addressText": "string (optional)"
  },
  "riderId":      "string (null until auto-assigned)",
  "riderName":    "string (optional)",
  "assignedAt":   "timestamp (optional)",
  "adminAlert":   "string (optional)",
  "createdAt":    "timestamp",
  "updatedAt":    "timestamp",
  "deliveredAt":  "timestamp (optional)"
}
```

### 5.8 `refunds` Collection

```json
{
  "refundId":             "string",
  "orderId":              "string",
  "userId":               "string",
  "userPhone":            "string",
  "amount":               "number",
  "refundType":           "string (full / partial_vendor_decline / partial_item_adjust)",
  "reason":               "string (customer_cancelled/vendor_declined/all_vendors_declined/order_issue)",
  "initiatedBy":          "string (customer/vendor/admin/system)",
  "declinedVendorId":     "string (optional — for partial vendor decline refunds)",
  "paymentReference":     "string (original M-Pesa transaction ID)",
  "refundStatus":         "string (pending_manual/completed/failed)",
  "mpesaManualReference": "string (entered by admin when marking paid)",
  "adminNote":            "string (optional — e.g. 'Delivery was waived at checkout; no delivery fee to refund')",
  "createdAt":            "timestamp",
  "completedAt":          "timestamp (optional)"
}
```

### 5.9 `service_providers` Collection

```json
{
  "providerId":                "string",
  "businessName":              "string",
  "ownerName":                 "string",
  "phone":                     "string (verified)",
  "idDocUrl":                  "string",
  "categoryIds":               "array of strings",
  "status":                    "string (pending_activation/active/rejected/blocked)",
  "subscriptionStatus":        "string (active/expired/not_yet_billed)",
  "monthlySubscriptionFee":    "number",
  "subscriptionDueDate":       "timestamp",
  "lastPaidAt":                "timestamp (optional)",
  "lastBankTransferReference": "string (optional — entered by admin on manual mark paid)",
  "rejectionReason":           "string (optional)",
  "blockedReason":             "string (optional)",
  "createdAt":                 "timestamp",
  "activatedAt":               "timestamp (optional)",
  "updatedAt":                 "timestamp"
}
```

### 5.10 `services` Collection

```json
{
  "serviceId":        "string",
  "providerId":       "string",
  "providerPhone":    "string (denormalized — auto-updated by onProviderPhoneUpdate trigger)",
  "categoryId":       "string",
  "name":             "string",
  "description":      "string",
  "price":            "number (informational — not charged via platform)",
  "durationEstimate": "string (optional)",
  "isAvailable":      "boolean",
  "createdAt":        "timestamp",
  "updatedAt":        "timestamp"
}
```

### 5.11 `riders` Collection

```json
{
  "riderId":              "string",
  "name":                 "string",
  "phone":                "string (verified)",
  "nationalId":           "string",
  "idPhotoUrl":           "string (required)",
  "licensePhotoUrl":      "string (optional)",
  "vehicleType":          "string (bike/motorbike)",
  "status":               "string (pending_activation/active/rejected/blocked)",
  "isOnline":             "boolean",
  "activeDeliveryCount":  "number",
  "lastAssignedAt":       "timestamp (initialised to Unix epoch on activation; set on successful assignment only — NOT updated on decline)",
  "currentMonthEarnings": "number",
  "pendingCODBalance":    "number",
  "codRechargeDueDate":   "timestamp",
  "riderAccountStatus":   "string (good_standing/locked)",
  "totalDeliveries":      "number",
  "rejectionReason":      "string (optional)",
  "blockedReason":        "string (optional)",
  "createdAt":            "timestamp",
  "activatedAt":          "timestamp (optional)",
  "updatedAt":            "timestamp"
}
```

### 5.12 `rider_wallet_transactions` Collection

```json
{
  "transactionId":  "string",
  "riderId":        "string",
  "type":           "string (cod_collection/recharge_payment/monthly_settlement)",
  "amount":         "number",
  "relatedOrderId": "string (optional — for cod_collection type)",
  "mpesaReference": "string (optional — for recharge_payment type)",
  "periodMonth":    "string (optional — e.g. '2026-06', for monthly_settlement type)",
  "balanceAfter":   "number",
  "createdAt":      "timestamp"
}
```

### 5.13 `payouts` Collection

```json
{
  "payoutId":                  "string",
  "payeeType":                 "string (rider/vendor)",
  "payeeId":                   "string",
  "payeeName":                 "string",
  "payeePhone":                "string",
  "periodMonth":               "string (e.g. 2026-06)",
  "totalDeliveries":           "number (riders only)",
  "totalSubOrders":            "number (vendors only)",
  "grossAmount":               "number",
  "commissionOrSharePercent":  "number",
  "pendingCODDeducted":        "number (riders only)",
  "netPayoutAmount":           "number",
  "payoutStatus":              "string (pending_manual/paid)",
  "mpesaManualReference":      "string (entered by admin)",
  "generatedAt":               "timestamp",
  "paidAt":                    "timestamp (optional)"
}
```

### 5.14 `platform_settings` Collection

```json
{
  "settingId": "platform_delivery_settings",
  "deliveryZones": [
    { "zone": "string", "maxDistance": "number (km)", "fee": "number (KES)" }
  ],
  "freeDeliveryThreshold":              "number (KES)",
  "baseFee":                            "number (KES)",
  "perKmRate":                          "number (KES per km)",
  "riderSharePercent":                  "number",
  "maxConcurrentDeliveries":            "number",
  "codSettings": {
    "rechargeDueDayOfWeek":    "string",
    "rechargeGracePeriodDays": "number"
  },
  "vendorResponseWindowMinutes":         "number (default 15)",
  "vendorResponseWarningMinutes":        "number (default 10)",
  "vendorAutoDeclinePollIntervalMinutes":"number (default 2)",
  "customerCancelWindowMinutes":         "number (default 5)",
  "bankDetails": {
    "accountName":  "string",
    "accountNumber":"string",
    "bankName":     "string",
    "billingEmail": "string"
  },
  "updatedAt": "timestamp"
}
```

---

## 6. Firestore Security Rules

Security rules check **both** the Firebase Auth custom claim (`role`) **and** the Firestore `status` field. A blocked vendor's claim remains valid for up to 1 hour after blocking — the Firestore status check is mandatory to close that window.

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // ── helpers ──────────────────────────────────────────────────
    function isAuth()    { return request.auth != null; }
    function role()      { return request.auth.token.role; }
    function isAdmin()   { return isAuth() && role() == 'admin'; }
    function isCustomer(){ return isAuth() && role() == 'customer'; }
    function isVendor()  { return isAuth() && role() == 'vendor'; }
    function isRider()   { return isAuth() && role() == 'rider'; }
    function isSP()      { return isAuth() && role() == 'service_provider'; }

    // Claim alone is insufficient after a block — always check Firestore status
    function vendorActive() {
      return isVendor() &&
             get(/databases/$(database)/documents/vendors/$(request.auth.uid))
               .data.status == 'active';
    }
    function riderActive() {
      return isRider() &&
             get(/databases/$(database)/documents/riders/$(request.auth.uid))
               .data.status == 'active';
    }
    function spActive() {
      return isSP() &&
             get(/databases/$(database)/documents/service_providers/$(request.auth.uid))
               .data.status == 'active';
    }

    // ── orders ───────────────────────────────────────────────────
    match /orders/{orderId} {
      allow read:   if isAdmin()
                      || (isCustomer() && resource.data.userId == request.auth.uid)
                      || (riderActive() && resource.data.riderId == request.auth.uid);
      allow create: if isCustomer() || isAuth();
      allow update: if isAdmin()
                      || (riderActive() && resource.data.riderId == request.auth.uid
                          && onlyUpdating(['orderStatus','deliveredAt','updatedAt']));

      match /subOrders/{vendorId} {
        allow read:   if isAdmin()
                        || (vendorActive() && vendorId == request.auth.uid)
                        || (riderActive() && get(/databases/$(database)/documents/orders/$(orderId))
                              .data.riderId == request.auth.uid);
        allow update: if vendorActive() && vendorId == request.auth.uid
                        && onlyUpdating(['subOrderStatus','declineReason','acceptedAt',
                                         'readyAt','updatedAt']);
      }
    }

    // ── products ─────────────────────────────────────────────────
    match /products/{productId} {
      allow read:  if true;
      allow write: if vendorActive()
                     && request.resource.data.vendorId == request.auth.uid;
    }

    // ── vendors ──────────────────────────────────────────────────
    match /vendors/{vendorId} {
      allow read:   if isAdmin() || (isAuth() && vendorId == request.auth.uid);
      allow create: if isAuth() && vendorId == request.auth.uid;
      allow update: if isAdmin()
                      || (vendorActive() && vendorId == request.auth.uid
                          && onlyUpdating(['location','locationConfigured','updatedAt']));
    }

    // ── riders ───────────────────────────────────────────────────
    match /riders/{riderId} {
      allow read:   if isAdmin() || (isAuth() && riderId == request.auth.uid);
      allow create: if isAuth() && riderId == request.auth.uid;
      allow update: if isAdmin()
                      || (riderActive() && riderId == request.auth.uid
                          && onlyUpdating(['isOnline','updatedAt']));
    }

    // ── services ─────────────────────────────────────────────────
    match /services/{serviceId} {
      allow read:  if true;
      allow write: if spActive()
                     && request.resource.data.providerId == request.auth.uid;
    }

    // ── service providers ────────────────────────────────────────
    match /service_providers/{providerId} {
      allow read:   if isAdmin() || (isAuth() && providerId == request.auth.uid);
      allow create: if isAuth() && providerId == request.auth.uid;
      allow update: if isAdmin();
    }

    // ── categories ───────────────────────────────────────────────
    match /categories/{catId} {
      allow read:  if true;
      allow write: if isAdmin();
    }

    // ── platform_settings ────────────────────────────────────────
    match /platform_settings/{docId} {
      allow read:  if true;
      allow write: if isAdmin();
    }

    // ── payouts ──────────────────────────────────────────────────
    match /payouts/{payoutId} {
      allow read:  if isAdmin()
                     || (isVendor() && resource.data.payeeId == request.auth.uid)
                     || (isRider() && resource.data.payeeId == request.auth.uid);
      allow write: if isAdmin();
    }

    // ── refunds ──────────────────────────────────────────────────
    match /refunds/{refundId} {
      allow read:  if isAdmin()
                     || (isCustomer() && resource.data.userId == request.auth.uid);
      allow write: if isAdmin();
    }

    // ── rider_wallet_transactions ─────────────────────────────────
    match /rider_wallet_transactions/{txId} {
      allow read:  if isAdmin()
                     || (isRider() && resource.data.riderId == request.auth.uid);
      allow write: if false; // Cloud Functions only
    }

    // ── pending_suborders (poll index) ───────────────────────────
    match /pending_suborders/{docId} {
      allow read, write: if false; // Cloud Functions only
    }

    // ── failed_notifications ─────────────────────────────────────
    match /failed_notifications/{docId} {
      allow read, write: if false; // Cloud Functions only
    }

    // ── payment_attempts ─────────────────────────────────────────
    match /payment_attempts/{docId} {
      allow read, write: if false; // Cloud Functions only
    }

    // ── upload_rejections ────────────────────────────────────────
    match /upload_rejections/{docId} {
      allow read:  if isAdmin();
      allow write: if false; // Cloud Functions only
    }

    function onlyUpdating(fields) {
      return request.resource.data.diff(resource.data).affectedKeys().hasOnly(fields);
    }
  }
}
```

---

## 7. Cloud Functions — Specifications

All Cloud Functions v2. Every `onCall` function begins with:

```javascript
if (!context.auth) throw new HttpsError('unauthenticated', 'Authentication required');
```

Credentials (Twilio auth token, Daraja consumer secret) are loaded from **Firebase Secret Manager** — never from environment config. All Cloud Functions use structured JSON logging:

```javascript
console.log(JSON.stringify({ severity: 'INFO', orderId, event: 'order_created' }));
console.error(JSON.stringify({ severity: 'ERROR', orderId, error: err.message }));
```

### 7.1 `onOrderCreate`

- Snapshot `riderSharePercent` from `platform_settings` onto order doc
- Validate vendor locations server-side — reject if any `vendorId` in the cart has no `lat`/`lng`; sanitize and length-limit all address fields before writing
- For each vendor: create sub-document in `orders/{orderId}/subOrders/{vendorId}` with `autoDeclineWarningSent = false`, `codConfirmedAt = null`
- Write shallow record to `pending_suborders`: `{ orderId, vendorId, createdAt, subOrderStatus: 'pending', warningSent: false }`
- Set `cancellationWindowExpiresAt = createdAt + customerCancelWindowMinutes`
- Set `cancellationWindowClosed = false`
- Notify each vendor separately via `sendWhatsApp()` helper (their items only)
- Send customer confirmation via `sendWhatsApp()`

### 7.2 M-Pesa Callback Authentication

- Callback URL includes a secret token in path: `/mpesa/callback/{SECRET_TOKEN}`
- Cloud Function validates the token against Secret Manager before processing any payload
- Validates source IP against Safaricom's published callback IP allowlist
- On validation failure: return HTTP 403, log the attempt, do not create or modify any order

### 7.3 `confirmCODCollection` — Idempotency

```javascript
exports.confirmCODCollection = functions.https.onCall(async (data, context) => {
  if (!context.auth) throw new HttpsError('unauthenticated');
  const { orderId } = data;

  await db.runTransaction(async (tx) => {
    const orderRef = db.collection('orders').doc(orderId);
    const order = (await tx.get(orderRef)).data();

    if (order.codConfirmedAt) {
      throw new HttpsError('already-exists', 'COD already confirmed for this order');
    }
    if (order.riderId !== context.auth.uid) {
      throw new HttpsError('permission-denied', 'Not the assigned rider');
    }

    const riderRef = db.collection('riders').doc(context.auth.uid);
    tx.update(orderRef, {
      codAmountCollected: order.codAmountDue,
      codSettlementStatus: 'awaiting_rider_recharge',
      codConfirmedAt: admin.firestore.FieldValue.serverTimestamp(),
    });
    tx.update(riderRef, {
      pendingCODBalance: admin.firestore.FieldValue.increment(order.codAmountDue),
    });
  });
  // Write rider_wallet_transactions record after transaction
});
```

### 7.4 `vendorAutoDeclinePoll` — Subcollection-Based

Required Firestore composite index: `Collection: pending_suborders | Fields: subOrderStatus ASC, createdAt ASC`.

```javascript
exports.vendorAutoDeclinePoll = functions.pubsub
  .schedule('every 2 minutes').onRun(async () => {
    const settings = await getPlatformSettings();
    const now = Date.now();
    const warnThreshold    = now - settings.vendorResponseWarningMinutes * 60000;
    const declineThreshold = now - settings.vendorResponseWindowMinutes * 60000;

    // Warning pass
    const warnSnap = await db.collection('pending_suborders')
      .where('subOrderStatus', '==', 'pending')
      .where('warningSent', '==', false)
      .where('createdAt', '<=', new Date(warnThreshold))
      .get();

    for (const doc of warnSnap.docs) {
      const { orderId, vendorId } = doc.data();
      const subRef = db.collection('orders').doc(orderId)
                       .collection('subOrders').doc(vendorId);
      await subRef.update({ autoDeclineWarningSent: true });
      await doc.ref.update({ warningSent: true });
      await sendWhatsApp(vendorPhone, warningMessage, { orderId, vendorId });
    }

    // Auto-decline pass
    const declineSnap = await db.collection('pending_suborders')
      .where('subOrderStatus', '==', 'pending')
      .where('createdAt', '<=', new Date(declineThreshold))
      .get();

    for (const doc of declineSnap.docs) {
      const { orderId, vendorId } = doc.data();
      const subRef = db.collection('orders').doc(orderId)
                       .collection('subOrders').doc(vendorId);
      await subRef.update({ subOrderStatus: 'declined', declineReason: 'auto_timeout' });
      await doc.ref.update({ subOrderStatus: 'declined' });
      // onSubOrderUpdate trigger handles refund + re-evaluation
    }
  });
```

### 7.5 `autoAssignRider` — Fresh Snapshot Inside Transaction

Sorted by: (1) lowest `activeDeliveryCount` (idle first), then (2) `lastAssignedAt` ascending (round-robin fairness — rider who has gone longest since last assignment goes first).

```javascript
async function autoAssignRider(orderId) {
  const settings = await getPlatformSettings();
  const riders = await getAvailableRidersSortedByIdleThenLastAssigned(
    settings.maxConcurrentDeliveries
  );

  for (const rider of riders) {
    const success = await db.runTransaction(async (tx) => {
      const orderRef = db.collection('orders').doc(orderId);
      const riderRef = db.collection('riders').doc(rider.riderId);

      // Re-read BOTH order and rider fresh inside the transaction
      const freshOrder = (await tx.get(orderRef)).data();
      const freshRider = (await tx.get(riderRef)).data();

      if (freshOrder.riderId) return false;  // already assigned
      if (freshRider.activeDeliveryCount >= settings.maxConcurrentDeliveries) return false;
      if (freshRider.riderAccountStatus === 'locked') return false;

      // Recompute codAmountDue from fresh sub-orders snapshot
      const subOrdersSnap = await tx.getAll(
        ...freshOrder.vendorSubOrderIds.map(vid =>
          db.collection('orders').doc(orderId).collection('subOrders').doc(vid)
        )
      );
      const accepted = subOrdersSnap
        .map(s => s.data())
        .filter(s => !['declined', 'cancelled'].includes(s.subOrderStatus));
      const codAmountDue = accepted.reduce((sum, s) => sum + s.subTotal, 0)
                         + freshOrder.deliveryFee;

      tx.update(orderRef, {
        riderId: rider.riderId,
        riderName: rider.name,
        assignedAt: admin.firestore.FieldValue.serverTimestamp(),
        orderStatus: 'all_accepted_awaiting_rider',
        codAmountDue,  // computed from fresh snapshot — uses subTotal NOT vendorNetAmount
      });
      tx.update(riderRef, {
        activeDeliveryCount: admin.firestore.FieldValue.increment(1),
        lastAssignedAt: admin.firestore.FieldValue.serverTimestamp(),
      });
      return true;
    });

    if (success) {
      // Build WhatsApp from fresh Firestore read after transaction
      const freshOrder = (await db.collection('orders').doc(orderId).get()).data();
      // notify rider, customer, vendors via sendWhatsApp()
      return;
    }
  }

  await db.collection('orders').doc(orderId).update({
    orderStatus: 'all_accepted_awaiting_rider',
    adminAlert: 'no_rider_available',
  });
}
```

### 7.6 `onSubOrderUpdate` — Race Fix + Early Exit + Post-Assign Decline Handling

```javascript
exports.onSubOrderUpdate = functions.firestore
  .document('orders/{orderId}/subOrders/{vendorId}')
  .onUpdate(async (change, context) => {
    const before = change.before.data();
    const after  = change.after.data();

    // Early exit — if status hasn't changed, skip heavy logic
    if (before.subOrderStatus === after.subOrderStatus) return;

    const orderId = context.params.orderId;
    const orderRef = db.collection('orders').doc(orderId);

    // Detect first vendor accept — atomically close cancellation window
    if (after.subOrderStatus === 'accepted') {
      await db.runTransaction(async (tx) => {
        const order = (await tx.get(orderRef)).data();
        if (order.cancellationWindowClosed) return; // already closed
        tx.update(orderRef, {
          cancellationWindowExpiresAt: admin.firestore.FieldValue.serverTimestamp(),
          cancellationWindowClosed: true,
        });
      });
    }

    // Re-read all sub-orders to evaluate overall state
    const subOrdersSnap = await db.collection('orders').doc(orderId)
                                  .collection('subOrders').get();
    const subOrders = subOrdersSnap.docs.map(d => d.data());
    const activeSubOrders = subOrders.filter(
      s => !['declined', 'cancelled'].includes(s.subOrderStatus)
    );

    if (activeSubOrders.length === 0) {
      const order = (await orderRef.get()).data();
      const refundAmount = order.totalAmountCharged - (order.refundedAmount || 0);
      if (order.paymentMethod === 'mpesa' && refundAmount > 0) {
        await createRefundRecord(orderId, order, refundAmount, 'all_vendors_declined');
      }
      await orderRef.update({ orderStatus: 'cancelled' });
      return;
    }

    const allAccepted = activeSubOrders.every(
      s => ['accepted', 'preparing', 'ready', 'picked_up'].includes(s.subOrderStatus)
    );
    const order = (await orderRef.get()).data();
    if (allAccepted && !order.riderId) {
      await autoAssignRider(orderId);
    }

    // Handle partial decline — including post-assignment decline
    if (after.subOrderStatus === 'declined' && before.subOrderStatus !== 'declined') {
      await createRefundRecord(orderId, order, after.subTotal, 'vendor_declined', context.params.vendorId);
      const refundUpdates = {
        refundedAmount: admin.firestore.FieldValue.increment(after.subTotal),
        paymentStatus: 'partially_refunded',
      };
      // Recompute codAmountDue if rider already assigned
      if (order.riderId) {
        const remainingAccepted = subOrders.filter(
          s => !['declined', 'cancelled'].includes(s.subOrderStatus)
        );
        const updatedCOD = remainingAccepted.reduce((sum, s) => sum + s.subTotal, 0)
                         + order.deliveryFee;
        refundUpdates.codAmountDue = updatedCOD;
        // Notify rider of updated pickup list and new codAmountDue via sendWhatsApp()
      }
      await orderRef.update(refundUpdates);
      // Notify customer of item removal via sendWhatsApp()
    }
  });
```

### 7.7 `respondToAssignment` — Rider Accept/Decline

```javascript
exports.respondToAssignment = functions.https.onCall(async (data, context) => {
  if (!context.auth) throw new HttpsError('unauthenticated');
  // Verify caller is the assigned rider
  // Accept → log acceptedAt, unlock full delivery detail
  // Decline → reset order.riderId to null, decrement activeDeliveryCount
  //   → DO NOT update rider.lastAssignedAt on decline
  //     (rider didn't take the delivery — stays in queue position)
  //   → retry autoAssignRider with fresh order read
  //   → if none available → adminAlert flag set
});
```

### 7.8 `initiateMpesaPayment` — STK Push Rate Limiting

```javascript
exports.initiateMpesaPayment = functions.https.onCall(async (data, context) => {
  if (!context.auth) throw new HttpsError('unauthenticated');
  const { orderId, phone } = data;

  const attemptRef = db.collection('payment_attempts')
    .doc(`${orderId}_${context.auth.uid}`);
  const attempt = await attemptRef.get();
  const attempts = attempt.exists ? attempt.data() : { count: 0, hourlyCount: 0 };

  if (attempts.count >= 5) {
    throw new HttpsError('resource-exhausted', 'Too many payment attempts for this order');
  }
  const hoursSinceFirst = attempt.exists
    ? (Date.now() - attempt.data().firstAttemptAt.toMillis()) / 3600000 : 0;
  if (hoursSinceFirst < 1 && attempts.hourlyCount >= 10) {
    throw new HttpsError('resource-exhausted', 'Too many attempts. Try again later');
  }

  await attemptRef.set({
    count: admin.firestore.FieldValue.increment(1),
    hourlyCount: hoursSinceFirst >= 1 ? 1 : admin.firestore.FieldValue.increment(1),
    lastAttemptAt: admin.firestore.FieldValue.serverTimestamp(),
    firstAttemptAt: attempt.exists
      ? attempt.data().firstAttemptAt
      : admin.firestore.FieldValue.serverTimestamp(),
  }, { merge: true });

  // Proceed with Daraja STK Push call
});
```

### 7.9 `calculateDeliveryFee` — Haversine, Farthest Vendor

```javascript
exports.calculateDeliveryFee = functions.https.onCall(async (data, context) => {
  if (!context.auth) throw new HttpsError('unauthenticated');
  const vendors = await Promise.all(data.vendorIds.map(id => getVendor(id)));
  let farthestDistanceKm = 0;

  for (const vendor of vendors) {
    if (!vendor.location || vendor.location.lat == null || vendor.location.lng == null) {
      // Server-side rejection if any cart vendor has no location
      throw new HttpsError('failed-precondition',
        `Vendor ${vendor.vendorId} has no configured location. Remove their items and retry.`);
    }
    const dist = haversineDistanceKm(
      vendor.location.lat, vendor.location.lng,
      data.customerLat, data.customerLng
    );
    if (dist > farthestDistanceKm) farthestDistanceKm = dist;
  }

  const settings = await getPlatformSettings();
  let fee = settings.baseFee + (settings.perKmRate * farthestDistanceKm);
  let zone = 'beyond_zone';

  for (const z of settings.deliveryZones) {
    if (farthestDistanceKm <= z.maxDistance) { fee = z.fee; zone = z.zone; break; }
  }

  const waived = data.subtotal >= settings.freeDeliveryThreshold;
  const deliveryFeeBaseAmount = fee; // always the pre-waiver fee

  return {
    fee: waived ? 0 : fee,
    deliveryFeeBaseAmount,  // always stored regardless of waiver
    zone,
    farthestVendorDistanceKm: farthestDistanceKm,
    waived,
  };
});

function haversineDistanceKm(lat1, lng1, lat2, lng2) {
  const R = 6371;
  const dLat = (lat2 - lat1) * Math.PI / 180;
  const dLng = (lng2 - lng1) * Math.PI / 180;
  const a = Math.sin(dLat/2)**2 +
            Math.cos(lat1*Math.PI/180) * Math.cos(lat2*Math.PI/180) * Math.sin(dLng/2)**2;
  return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
}
```

### 7.10 `cancelOrder` — Customer-Initiated

```javascript
exports.cancelOrder = functions.https.onCall(async (data, context) => {
  if (!context.auth) throw new HttpsError('unauthenticated');
  const order = await getOrder(data.orderId);
  const now = Date.now();

  if (now > order.cancellationWindowExpiresAt.toMillis()) {
    throw new HttpsError('failed-precondition',
      'Cancellation window has closed. Please contact support.');
  }

  // Cancel all sub-orders → orderStatus = 'cancelled'
  // If mpesa → full refund = totalAmountCharged − refundedAmount
  // If cod → no refund record needed
  // Notify customer + all vendors via sendWhatsApp()
});
```

### 7.11 `markPickup` — Rider Per-Vendor Pickup

```javascript
exports.markPickup = functions.https.onCall(async (data, context) => {
  if (!context.auth) throw new HttpsError('unauthenticated');
  // Verify caller is the assigned rider
  // Set subOrders/{vendorId}.subOrderStatus = 'picked_up', pickedUpAt = now
  // Check if ALL non-declined sub-orders are picked_up
  // → if yes → orderStatus = 'out_for_delivery'
  // → notify customer via sendWhatsApp()
});
```

### 7.12 `onDeliveryComplete`

```javascript
exports.onDeliveryComplete = functions.firestore
  .document('orders/{orderId}')
  .onUpdate(async (change, context) => {
    const after = change.after.data();
    if (after.orderStatus !== 'delivered') return;

    // Always use deliveryFeeBaseAmount — never deliveryFee
    const riderShare = after.deliveryFeeBaseAmount * after.riderSharePercent / 100;
    // Increment rider.currentMonthEarnings by riderShare
    // Decrement rider.activeDeliveryCount
    // Notify customer via sendWhatsApp()
  });
```

### 7.13 `riderWalletRecharge`

```javascript
exports.riderWalletRecharge = functions.https.onCall(async (data, context) => {
  if (!context.auth) throw new HttpsError('unauthenticated');
  // Trigger STK Push for data.amount (partial or full — both always allowed)
  // On successful Daraja callback:
  //   Decrement rider.pendingCODBalance by paid amount
  //   Write rider_wallet_transactions (type: 'recharge_payment')
  //   Mark COD orders codSettlementStatus = 'collected' FIFO up to amount recharged
  //   If pendingCODBalance = 0 → riderAccountStatus = 'good_standing'
});
```

### 7.14 `setRiderOnline` — Server-side Lock Enforcement

```javascript
exports.setRiderOnline = functions.https.onCall(async (data, context) => {
  if (!context.auth) throw new HttpsError('unauthenticated');
  const rider = await getRider(context.auth.uid);
  if (rider.status !== 'active') {
    throw new HttpsError('permission-denied', 'Account not active');
  }
  if (rider.riderAccountStatus === 'locked') {
    throw new HttpsError('failed-precondition',
      `Account locked. Recharge KES ${rider.pendingCODBalance} to go online.`);
  }
  await db.collection('riders').doc(context.auth.uid).update({ isOnline: true });
});
```

### 7.15 `setRiderOffline` — Active Delivery Guard

```javascript
exports.setRiderOffline = functions.https.onCall(async (data, context) => {
  if (!context.auth) throw new HttpsError('unauthenticated');
  const rider = await getRider(context.auth.uid);
  if (rider.activeDeliveryCount > 0) {
    throw new HttpsError('failed-precondition',
      `Complete your ${rider.activeDeliveryCount} active delivery/deliveries first.`);
  }
  await db.collection('riders').doc(context.auth.uid).update({ isOnline: false });
});
```

### 7.16 `validateUpload` — File Upload Quarantine

```javascript
exports.validateUpload = functions.storage.object().onFinalize(async (object) => {
  const filePath = object.name;
  const contentType = object.contentType;
  const size = parseInt(object.size, 10);
  const allowedTypes = ['image/jpeg', 'image/png', 'application/pdf'];
  const maxBytes = 5 * 1024 * 1024; // 5 MB

  if (!allowedTypes.includes(contentType) || size > maxBytes) {
    const bucket = admin.storage().bucket(object.bucket);
    const quarantinePath = filePath.replace('uploads/', 'quarantine/');
    await bucket.file(filePath).move(quarantinePath);
    await db.collection('upload_rejections').add({
      originalPath: filePath, quarantinePath, contentType, size,
      rejectedAt: admin.firestore.FieldValue.serverTimestamp(),
    });
  }
});
```

### 7.17 `resizeProductImage` — Image Resize on Upload

```javascript
exports.resizeProductImage = functions.storage.object().onFinalize(async (object) => {
  const filePath = object.name;
  if (!filePath.startsWith('products/')) return;
  if (!object.contentType?.startsWith('image/')) return;

  const sharp = require('sharp');
  const bucket = admin.storage().bucket(object.bucket);
  const tempPath = `/tmp/${path.basename(filePath)}`;
  await bucket.file(filePath).download({ destination: tempPath });

  const dir = path.dirname(filePath);
  const base = path.basename(filePath);

  const resizedPath = `/tmp/800_${base}`;
  const thumbPath   = `/tmp/thumb200_${base}`;
  await sharp(tempPath).resize({ width: 800,  withoutEnlargement: true }).toFile(resizedPath);
  await sharp(tempPath).resize({ width: 200,  withoutEnlargement: true }).toFile(thumbPath);

  await bucket.upload(resizedPath, { destination: `${dir}/800_${base}` });
  await bucket.upload(thumbPath,   { destination: `${dir}/thumb200_${base}` });
});
```

### 7.18 `cleanupGuestAccounts` — Monthly

```javascript
exports.cleanupGuestAccounts = functions.pubsub
  .schedule('0 3 1 * *').timeZone('Africa/Nairobi')
  .onRun(async () => {
    // List anonymous Firebase Auth users (no email, no phone provider)
    // For each: check if any order exists with userId
    // If zero orders AND account older than 30 days → admin.auth().deleteUser(uid)
    // Process in batches of 100 to avoid timeout
  });
```

### 7.19 WhatsApp `sendWhatsApp()` — Dead-Letter Wrapper

```javascript
async function sendWhatsApp(to, message, context) {
  try {
    await twilioClient.messages.create({ from: TWILIO_FROM, to, body: message });
  } catch (err) {
    await db.collection('failed_notifications').add({
      to, message, context: context || {},
      failedAt: admin.firestore.FieldValue.serverTimestamp(),
      retryCount: 0,
    });
  }
}
```

### 7.20 `retryFailedNotifications` — Every 15 Minutes

```javascript
exports.retryFailedNotifications = functions.pubsub
  .schedule('every 15 minutes').onRun(async () => {
    const snap = await db.collection('failed_notifications')
      .where('retryCount', '<', 3).limit(50).get();
    for (const doc of snap.docs) {
      const { to, message } = doc.data();
      try {
        await twilioClient.messages.create({ from: TWILIO_FROM, to, body: message });
        await doc.ref.delete();
      } catch {
        await doc.ref.update({ retryCount: admin.firestore.FieldValue.increment(1) });
      }
    }
  });
```

### 7.21 `onProviderPhoneUpdate` — providerPhone Fan-out

```javascript
exports.onProviderPhoneUpdate = functions.firestore
  .document('service_providers/{providerId}')
  .onUpdate(async (change, context) => {
    const before = change.before.data();
    const after  = change.after.data();
    if (before.phone === after.phone) return;

    const servicesSnap = await db.collection('services')
      .where('providerId', '==', context.params.providerId).get();
    const batch = db.batch();
    servicesSnap.docs.forEach(doc => batch.update(doc.ref, { providerPhone: after.phone }));
    await batch.commit();
  });
```

### 7.22 `checkGuestPhone` — Guest vs Registered Account

```javascript
exports.checkGuestPhone = functions.https.onCall(async (data, context) => {
  if (!context.auth) throw new HttpsError('unauthenticated');
  const usersSnap = await db.collection('users')
    .where('phone', '==', data.phone)
    .where('isGuestOrder', '==', false)
    .limit(1).get();
  return { existingAccountFound: !usersSnap.empty };
});
```

### 7.23 Admin Re-authentication — Financial Operations

```javascript
function requireRecentAuth(context) {
  const authTime = context.auth.token.auth_time * 1000;
  if (Date.now() - authTime > 5 * 60 * 1000) {
    throw new HttpsError('failed-precondition',
      'Please re-authenticate before performing financial operations.');
  }
}

exports.createPayoutRecord = functions.https.onCall(async (data, context) => {
  if (!context.auth) throw new HttpsError('unauthenticated');
  requireRecentAuth(context);
  // ... payout creation logic
  // Cloud Function trigger zeros currentMonthEarnings for rider after payout record created
});
```

---

## 8. Scheduled Functions

| Function | Schedule | Purpose |
|---|---|---|
| `vendorAutoDeclinePoll` | Every 2 minutes | Query `pending_suborders`; WhatsApp warning at ~10 min; auto-decline at ~15 min |
| `retryFailedNotifications` | Every 15 minutes | Retry up to 3× for failed Twilio calls; delete on success |
| `weeklyCODRechargeCheck` | Monday 8 AM EAT | Remind riders with pending COD balance; lock overdue accounts (no penalty fees) |
| `serviceProviderSubscriptionCheck` | Daily 7 AM EAT | Remind approaching due date; expire past-due; restore on admin mark-paid |
| `paymentReconciliation` | Daily 8 AM EAT | Verify M-Pesa STK Push transactions; flag mismatches for admin review |
| `dailySalesReport` | Daily 8 PM EAT | Generate daily summary to Firestore `reports` collection; WhatsApp to admin |
| `cleanupGuestAccounts` | 1st of month 3 AM EAT | Delete anonymous Auth accounts >30 days old with zero orders |

> Monthly rider and vendor payouts have **no cron job**. Admin manually reviews Reports tables, creates payout records via dashboard, pays via M-Pesa app. A Cloud Function trigger zeros `currentMonthEarnings` when the payout record is created — preventing the manual-forget bug.

---

## 9. Firebase Cost Optimisation

| Area | Problem | Fix |
|---|---|---|
| Firestore reads — order history | Stream on full order collection re-sends all docs on any change | Paginate (limit 20, `startAfter` cursor). Completed orders use one-time `get()`. All filters as Firestore query params — no client-side filtering. |
| Firestore reads — vendor orders | Full parent order doc returned | Subcollection: vendor stream reads only their own sub-order document |
| Firestore reads — admin reports | Full collection scan | Daily summary docs. Admin reads summary, not raw orders. Cursor-based server-side pagination for all tabular views. |
| Firestore reads — poll function | Array-of-maps cannot be indexed | `pending_suborders` index collection with composite index on `(subOrderStatus, createdAt)` |
| Cloud Function invocations | `onSubOrderUpdate` fires on every order doc write | Early-exit guard: compare before/after `subOrderStatus` — exit immediately if unchanged |
| Widget rebuilds — cart | Adding one Vendor B item rebuilds entire cart tree | Per-vendor `ChangeNotifier`; `context.select()` to scope rebuilds |
| Rider assignment contention | High volume simultaneous assignments contend on rider docs | Cloud Task queue with concurrency limit for `autoAssignRider` operations |

---

## 10. Customer Ordering Process

### 10.1 Onboarding

```
Customer visits web app → Sign up / Login → Profile setup
  → Phone number verified via Firebase Auth
  → Address collection (apartment, wing, floor)
  → Profile stored in Firestore
```

**Guest Checkout:**
```
Customer visits web app → [Continue as Guest]
→ Browse, add to cart exactly as a registered customer would
→ At checkout: enter name + phone number only
→ checkGuestPhone Cloud Function called first:
  ├── Existing registered account with same phone found → prompt to log in instead
  └── No account found → proceed with guest flow
→ Phone verified via OTP (Firebase Auth anonymous + phone verification)
→ Order placed under temporary guest session tied to that phone number
→ After delivery: "Save your details for faster checkout?" → optional
```

- Guests receive order updates via WhatsApp only
- Guests can cancel within the window during their active session (immediately after ordering)
- Once they close the tab, the session is gone — must contact support
- No WhatsApp cancel link needed

### 10.2 Vendor & Product Browsing

```
Home → Browse by Category (admin-defined) → See vendors offering that category
→ Select Vendor → Browse Vendor's Products → Add Items to Cart
→ Continue browsing → add items from other vendors to the SAME cart
```

- Categories created and managed only by admin
- Each vendor selects one or more admin-defined categories
- Customers can mix items from multiple vendors freely in one cart
- Vendors without `locationConfigured = true` are hidden from browse

### 10.3 Multi-Vendor Cart

```
Customer adds items from Vendor A → continues browsing → adds items from Vendor B
→ Cart now contains items from both vendors, grouped by vendor for clarity
→ One unified subtotal, one delivery fee, one checkout
```

- No single-vendor restriction — multi-vendor cart is the default
- Special instructions can be added per item
- Per-vendor ChangeNotifiers prevent full cart tree rebuilds on item changes

### 10.4 Delivery Fee Calculation (Haversine — Farthest Vendor)

```
Checkout → Delivery Location Step
  → Option A: Use Current Location (browser geolocation)
  → Option B: Enter Address Manually
  → Option C: Select on Map (Google Maps JS API pin picker)
  → Option D: Enter lat/lng manually (fallback if Maps API unavailable)

→ Customer lat/lng sent to calculateDeliveryFee Cloud Function
→ Server-side validation: if ANY vendorId in cart has no location → reject checkout
  (prevents client-side bypass)
→ Haversine distance computed for EACH vendor with valid location
→ FARTHEST distance across all vendors used for zone lookup:
  ├── distance ≤ Zone 1 maxDistance → fee = Zone 1 fee
  ├── distance ≤ Zone 2 maxDistance → fee = Zone 2 fee
  ├── ... continues through Zone 5
  └── distance > Zone 5 maxDistance → fee = baseFee + (perKmRate × distance)
→ If subtotal ≥ freeDeliveryThreshold (KES 500) → fee = 0 for customer
  deliveryFeeBaseAmount ALWAYS stored as the pre-waiver fee
  (rider payout always uses deliveryFeeBaseAmount, never deliveryFee)
→ Delivery fee LOCKED onto order — never changes regardless of what happens later
→ Fee displayed to customer before payment
```

**Why farthest vendor:** The rider must travel to ALL vendor locations before reaching the customer. The farthest vendor determines the maximum route length — using the nearest vendor would underprice orders requiring a long detour.

### 10.5 Order Confirmation Screen — Cancel Button & Countdown Timer

```
After order created (M-Pesa confirmed or COD submitted):
→ Customer lands on Order Confirmation screen
→ Screen shows: order summary, vendor list, total paid, "WhatsApp notification sent"
→ Shows '…' placeholder until cancellationWindowExpiresAt is fetched from Firestore
→ Cancel button displayed with live countdown timer:
  ⏱ "Cancel Order — 4:58 remaining"
→ Timer reads cancellationWindowExpiresAt from Firestore in real time
  (server-driven — refresh-safe; timer resumes correctly on page reload)
→ Button auto-hides when EITHER condition is met:
  ├── Timer reaches 0:00 (cancellationWindowExpiresAt passed)
  └── First vendor accepts (cancellationWindowExpiresAt atomically
      set to now(), cancellationWindowClosed = true — timer jumps to 0)
→ Customer taps [Cancel Order]:
  → cancelOrder Cloud Function checks cancellationWindowExpiresAt server-side
  → If still valid → order cancelled, refund created (mpesa) or nothing (cod)
  → If expired → error shown: "Cancellation window has closed"
→ After button hides: "Need to cancel? Contact support."
```

**Cancel button also visible in Order History** for registered users (subject to same server-side window check).

---

## 11. Payment Flow

### 11.1 M-Pesa STK Push

```
Checkout → Order total shown (subtotal + delivery fee)
→ Enter M-Pesa phone → Confirm
→ STK Push sent to phone (max 5 attempts per order, 10 per phone per hour)
→ User enters PIN on their phone
→ Daraja callback received at /mpesa/callback/{SECRET_TOKEN}
→ Cloud Function validates IP allowlist + secret token before processing
→ ResultCode = 0 → paymentStatus "confirmed"
  → Order created in Firestore
  → Sub-order records created in subOrders subcollection
  → Each vendor notified via Twilio WhatsApp (their items only)
  → Customer confirmation sent via Twilio WhatsApp
```

**M-Pesa Failure/Retry:**
```
STK Push fails (timeout / wrong PIN / insufficient funds / cancelled)
→ paymentStatus "failed" — order NOT created, cart preserved
→ User shown error message + [Retry Payment] button
→ User can retry (up to rate limit) — no forced COD suggestion
```

### 11.2 Cash on Delivery (COD)

```
Checkout → Select "Cash on Delivery" → Confirm
→ Order created: paymentMethod "cod", paymentStatus "pending_collection"
→ Sub-order records created per vendor
→ Each vendor notified via WhatsApp
→ codAmountDue computed at rider assignment time (not at checkout):
  codAmountDue = sum of ACCEPTED vendors' subTotals + deliveryFee
  (uses subTotal — customer-facing price — NOT vendorNetAmount which is post-commission)
→ Rider collects codAmountDue at door
→ Rider taps [Confirm Cash Collected: KES X] (idempotent — double-tap safe)
→ codAmountDue added to rider's pendingCODBalance
```

---

## 12. Notification Flow (WhatsApp)

All notifications sent via Twilio WhatsApp through the `sendWhatsApp()` wrapper. Failed messages written to `failed_notifications` and retried up to 3 times every 15 minutes.

### 12.1 Order Confirmation — to Customer
```
"Hi [Name], your order #ORD-1234 has been placed! Items from [Vendor A]
and [Vendor B]. Total: KES X. We'll keep you updated."
```

### 12.2 New Sub-order — to Each Vendor (Their Items Only)
```
"New order #ORD-1234 from [Customer Name]:
- 2x Fries KES 400
- 1x Chicken KES 650
Subtotal: KES 1,050 | Payment: M-Pesa (Paid) / COD
Reply or open app to Accept/Decline."
```

### 12.3 Sub-order Accepted — to Customer (Once Per Vendor)
```
"[Vendor Name] has accepted your items and is preparing them."
```

### 12.4 Auto-Decline Warning — to Vendor (~10 min mark)
```
"⚠ Order #ORD-1234 will be auto-declined in 5 minutes if not accepted.
Open the app now to Accept."
```

### 12.5 All Vendors Accepted, Rider Assigned — to Customer
```
"All vendors have confirmed! Rider [Name] is on the way to collect
your order from all locations."
```

### 12.6 New Assignment — to Rider (Built from Fresh Firestore Read)
```
"New delivery assigned: Order #ORD-1234
Pickups:
  1. [Vendor A Name] — [Address]
  2. [Vendor B Name] — [Address]
Drop-off: [Customer Address]
Payment: COD KES X / M-Pesa (already paid)
Open app to Accept/Decline."
```

### 12.7 Partial Cancellation — to Customer (One Vendor Declined)
```
"[Vendor Name] couldn't fulfill part of your order. Removed items: [list].
Refund of KES X processing. Your order from [other vendors] continues normally."
```

### 12.8 Updated Pickup + COD — to Rider (Post-Assignment Vendor Decline)
```
"Updated delivery for Order #ORD-1234. [Vendor Name] has been removed.
Updated pickups: [list]. New COD amount: KES [updatedCodAmountDue]."
```

### 12.9 Out for Delivery — to Customer
```
"Order #ORD-1234 is on its way! Rider: [Name], Phone: [number]."
```

### 12.10 Delivered — to Customer
```
"Order #ORD-1234 delivered! Thank you for choosing us."
(COD: "Cash of KES X collected.")
```

### 12.11 Refund Processed — to Customer
```
"Your refund of KES X for Order #ORD-1234 has been sent to [phone]."
```

### 12.12 Rider Assignment — to Each Active Vendor
```
"Rider [Name] has been assigned to collect your items for Order #ORD-1234.
Please have items ready."
```

---

## 13. Vendor Workflow

### 13.1 Self-Registration

```
Vendor opens Vendor Web → [Sign Up]
→ Enter business name, owner name, phone, business registration/ID
→ Upload business registration doc + owner ID photo (Firebase Storage)
  → validateUpload trigger: MIME check (jpeg/png/pdf only), max 5 MB,
    quarantine if invalid — admin reviews quarantine before making accessible
→ Select at least one admin-defined category
→ Enter single business location: address + map pin (lat/lng)
  → Manual lat/lng fallback available if Google Maps JS API is unavailable
→ Phone verification via Firebase Auth
→ Vendor profile created: status "pending_activation"
→ Admin notified via WhatsApp
```

### 13.2 Admin Activation & Commission Setup

```
Admin Web → Vendors → Pending registrations
→ Review details + uploaded docs
  ├── [Activate] → status "active" → set commission % → WhatsApp to vendor
  └── [Reject] → status "rejected" → WhatsApp with reason
→ Admin can [Block] active vendor at any time
→ Admin can update commission % (applies to new orders only — already-created
  orders use snapshotted vendorCommissionPercent)
```

### 13.3 Location & Product Management

```
Vendor Web → My Location → [Edit Location]
→ Enter/update location name, address, map pin lat/lng
  (manual lat/lng entry fields available as fallback)
→ Exactly one location per vendor — stored on vendor document directly
→ Until locationConfigured = true:
  → Vendor's products are hidden platform-wide
  → Status banner in Vendor Web: "Set your location to make products visible"

Vendor Web → Products → [Add Product]
→ Select admin-defined category
→ Enter name, description, price, unit, image
→ On image upload: resizeProductImage trigger creates 800px + thumb_200px variants
→ Toggle availability
→ Save → visible to customers immediately (if vendor active + locationConfigured)
```

### 13.4 Incoming Order Handling

```
Vendor Web → Orders → New sub-order arrives (sound + WhatsApp)
→ Shows ONLY this vendor's items:
  items, quantities, special instructions, payment method, customer contact
→ Vendor also sees: "This order also includes items from [N] other vendor(s)"
  (context only — they cannot see other vendors' items or details)

→ Vendor chooses:
  ├── [Accept] → subOrderStatus → "accepted"
  │   → Atomic transaction:
  │     - update subOrderStatus
  │     - if cancellationWindowClosed == false:
  │       set cancellationWindowExpiresAt = now(), cancellationWindowClosed = true
  │   → Customer notified via WhatsApp
  │   → System checks: have ALL vendors now accepted?
  │     ├── YES → autoAssignRider triggered
  │     └── NO → wait for remaining vendors
  │
  ├── [Decline] (with reason) → subOrderStatus → "declined"
  │   → Partial refund record created for their subTotal (if mpesa)
  │   → Customer notified of item removal
  │   → System re-evaluates remaining sub-orders
  │
  └── No response:
      → vendorAutoDeclinePoll fires:
        warning WhatsApp at ~10 min mark
        auto-decline at ~15 min mark
        declineReason = "auto_timeout"

→ After accepting → vendor updates status to "preparing" then "ready"
  ("ready" = items packaged, waiting for rider pickup)
```

### 13.5 Earnings & Payout

```
Vendor Web → Earnings
→ Shows: sub-orders fulfilled (their items only), gross sales,
  commission deducted, net owed, per period
→ Settlement status per sub-order:
  ├── M-Pesa → "Collected" immediately
  └── COD → "Awaiting Rider Recharge" until rider recharges, then "Collected"
→ Payout: admin manually reviews earnings table, calculates net payout,
  pays via M-Pesa app, creates payout record in dashboard, marks paid
→ WhatsApp to vendor with breakdown
```

---

## 14. Rider Workflow

### 14.1 Self-Registration & Activation

```
Rider Web (mobile browser) → [Sign Up]
→ Name, phone, national ID, vehicle type
→ Upload National ID photo (required); driving license / logbook photo (optional)
  → validateUpload trigger: MIME + size check, quarantine if invalid
→ Phone verification
→ Status: "pending_activation"
→ Admin reviews ID photo + docs → [Activate] or [Reject]
→ On activation:
  → status = "active"
  → lastAssignedAt = Unix epoch timestamp (Jan 1 1970)
    (ensures new riders sort correctly in ascending lastAssignedAt query)
  → WhatsApp sent to rider
```

### 14.2 Auto Assignment (System-Triggered)

```
Trigger: ALL vendors in order have subOrderStatus = "accepted"

→ Cloud Function fires automatically
→ Reads active, online, non-locked riders
→ Sorted by:
  1. Lowest activeDeliveryCount (idle first)
  2. Then lastAssignedAt ascending (round-robin fairness — rider who
     has gone longest since last assignment goes first)
     NOT random — guarantees fair rotation

→ For each candidate rider, atomic Firestore transaction:
  - Re-read order inside transaction (fresh codAmountDue)
  - Re-read rider inside transaction
  - If order.riderId already set → abort (duplicate trigger guard)
  - If rider.activeDeliveryCount >= maxConcurrentDeliveries → try next
  - If rider.riderAccountStatus == "locked" → try next
  - Else → set riderId, increment activeDeliveryCount,
    set lastAssignedAt = serverTimestamp()
    compute codAmountDue = sum of accepted subTotals + deliveryFee
    (uses subTotal NOT vendorNetAmount)

→ Build WhatsApp from FRESH Firestore read after transaction
  (always reflects current accepted vendor list — no stale snapshot)
→ Notify rider, customer, each active vendor
```

**Assignment Edge Cases:**
```
No riders available:
→ order stays in "all_accepted_awaiting_rider" + adminAlert = "no_rider_available"
→ Admin dashboard alerts; admin triggers reassignment manually when rider comes online

Rider declines assignment:
→ activeDeliveryCount decremented
→ lastAssignedAt NOT updated (they didn't take the delivery — stay in queue position)
→ autoAssignRider retried with fresh Firestore read
→ If no riders available → adminAlert set

All riders locked/busy → admin alerted
```

### 14.3 Delivery Flow (Multi-Vendor Pickup)

```
Rider → "My Deliveries" → new assignment notification
→ [Accept] → full order detail unlocked
→ [Decline] → system tries next available rider

Pickup stops (ordered by proximity, system-suggested):
  → [Mark Picked Up from Vendor A] → Vendor A notified, pickedUpAt set
  → [Mark Picked Up from Vendor B] → Vendor B notified, pickedUpAt set
  → Once ALL pickups marked → orderStatus → "out_for_delivery"
  → Customer notified: "Rider has all your items, on the way!"

At drop-off:
  ├── M-Pesa order → [Mark Delivered] directly
  └── COD order:
      → [Confirm Cash Collected: KES codAmountDue] (idempotent — double-tap safe)
      → [Mark Delivered]

→ deliveredAt timestamp set
→ Rider earnings: riderSharePercent % of deliveryFeeBaseAmount
  (always uses deliveryFeeBaseAmount, NEVER deliveryFee)
→ currentMonthEarnings incremented
→ activeDeliveryCount decremented
→ If COD → codAmountDue added to pendingCODBalance
→ Customer notified via WhatsApp
```

### 14.4 COD Wallet & Weekly Recharge

**Balance accrual:**
```
Rider delivers COD order → collects cash → [Confirm Cash Collected]
→ codAmountDue added to rider.pendingCODBalance
→ Accumulates across all COD deliveries during the week
```

**Weekly Recharge Cycle (every Monday 8 AM EAT):**
```
weeklyCODRechargeCheck runs:
→ For each rider with pendingCODBalance > 0:
  → WhatsApp: "Pending COD: KES X. Recharge by [due date]."
  → Rider Web banner: "Recharge Required: KES X due by [date]"
→ Rider → My Wallet → [Recharge Now] → M-Pesa STK Push
  (partial always allowed — no minimum)
→ On payment → pendingCODBalance reduced by paid amount
→ If pendingCODBalance = 0 → riderAccountStatus = "good_standing"
```

**Lock Enforcement (no penalty fees):**
```
Due date passes with pendingCODBalance > 0:
→ riderAccountStatus = "locked"
→ setRiderOnline Cloud Function rejects go-online attempts (server-side check)
→ Locked rider CAN complete deliveries already in progress
→ WhatsApp: "Account locked. Pending COD: KES X. Recharge to unlock."
→ pendingCODBalance = 0 → riderAccountStatus = "good_standing" immediately
```

### 14.5 Monthly Payout Settlement (Manual)

```
Admin Web → Reports → Rider Earnings table (filter by month)
→ Read currentMonthEarnings and pendingCODBalance per rider
→ netPayout = currentMonthEarnings − pendingCODBalance
→ Admin Web → Payouts → [Create Payout Record] per rider:
  Enter: riderId, periodMonth, grossAmount, pendingCODDeducted, netPayoutAmount
  ├── netPayout > 0 → admin pays via M-Pesa app → [Mark Paid] + M-Pesa ref
  │   → Cloud Function trigger zeros currentMonthEarnings automatically
  │   → pendingCODBalance reset to 0 via admin panel
  ├── netPayout = 0 → create KES 0 payout record, reset balance to 0
  └── netPayout < 0 → create KES 0 payout record,
      reduce pendingCODBalance by currentMonthEarnings (carry forward)
→ Write rider_wallet_transactions record (type: "monthly_settlement")
→ WhatsApp to rider with breakdown
```

**Going Offline Guard:**
```
Rider taps "Go Offline"
→ setRiderOffline Cloud Function checks activeDeliveryCount:
  > 0 → blocked: "Complete your [N] active delivery/deliveries first"
  = 0 → rider set offline
```

---

## 15. Service Provider Workflow

### 15.1 Self-Registration & Activation

```
Service Provider Web → [Sign Up]
→ Name, phone, ID/registration doc upload (validated by validateUpload trigger)
→ Select admin-defined category
→ Phone verification
→ Status: "pending_activation" → admin reviews → [Activate] or [Reject]
  → On activate: set monthly subscription fee
```

### 15.2 Service Listing

```
Service Provider Web → Services → [Add Service]
→ Admin-defined category, name, description, price (informational),
  duration estimate, availability toggle
→ Visible only when subscriptionStatus = "active"
```

### 15.3 Subscription Billing & Enforcement

```
Scheduled Function daily (7 AM EAT):
→ Approaching due date → WhatsApp reminder to provider

Provider payment flow:
→ Provider transfers subscription fee to platform bank account
→ Provider emails transaction proof/receipt to billing email
  (bank details + billing email shown in provider portal under Subscription → Payment Details)
→ Admin reviews proof → [Mark Subscription Paid] + bank transfer reference
→ subscriptionStatus = "active", subscriptionDueDate extended
→ Past due unpaid → subscriptionStatus = "expired" → all services hidden
→ Admin marks paid → subscriptionStatus = "active" → services reappear immediately
```

### 15.4 Customer Contact Flow

```
Customer → Browse Services → Select provider → View services
→ Each listing shows: name, description, price (informational),
  duration estimate, provider name
→ [Contact on WhatsApp] button → opens wa.me/[providerPhone]
  (providerPhone auto-updated on all service docs via onProviderPhoneUpdate trigger)
→ All scheduling, negotiation, and payment handled offline
→ Platform has no visibility into these interactions
```

---

## 16. Admin Workflow

### 16.1 Financial Operations — Re-authentication Required

All financial writes (create payout record, mark refund paid, mark payout paid) require the admin's Firebase ID token to have been issued within the last **5 minutes**. Stale sessions receive a re-authenticate prompt before proceeding.

### 16.2 Vendor Management

```
Admin Web → Vendors
→ Pending registrations tab:
  Review business docs + ID photo → [Activate] (set commission %) / [Reject]
  WhatsApp sent to vendor with outcome
→ Active vendors tab:
  Edit commission % (new orders only)
  [Block] vendor (status = "blocked"; security rules enforce via Firestore status check)
  View orders (read-only)
→ Payouts tab:
  Create payout record per vendor → pay via M-Pesa app → [Mark Paid] + reference
```

### 16.3 Rider Management

```
Admin Web → Riders
→ Pending tab: Review ID photo + docs → [Activate] / [Reject]
→ Active tab: [Block] (locks immediately via status check in rules)
  View earnings + COD balance
→ Settings: rider share %, max concurrent deliveries, recharge due-day window
```

### 16.4 Service Provider Management

```
Admin Web → Service Providers
→ Pending tab: [Activate] (set subscription fee) / [Reject]
→ Active tab: [Block]
→ Subscription tab:
  Table of providers with subscriptionDueDate + status
  [Mark Subscription Paid] + bank transfer reference
  → subscriptionStatus = "active", subscriptionDueDate extended
```

### 16.5 Category Management

```
Admin Web → Categories → Add / Edit / Delete
→ Applies to both vendors and service providers
→ Vendors and providers can only select from this list
```

### 16.6 Refund Management

```
Admin Web → Refunds → Table of all pending refunds
Columns: Order ID | Customer Phone | Amount | Reason | Type (full / partial)
→ Admin sends via M-Pesa app → [Mark Refund Paid] → enters M-Pesa reference
→ WhatsApp sent to customer confirming refund
→ Note on refund record if delivery was waived: "Delivery waived at checkout; no fee to refund"
→ Failed/disputed → admin marks "failed" with note, contacts customer
```

### 16.7 Order Management (Read-Only)

```
Admin Web → Orders → Filterable table of all orders across all vendors
→ Admin can cancel in exceptional cases + flag refund
→ Admin can manually trigger rider reassignment if auto-assign failed
  ("Order #ORD-1234 needs rider — none available" alert)
```

### 16.8 Delivery Zone & Fee Settings

```
Admin Web → Settings → Delivery Zones
→ Edit zone bands, fees, baseFee, perKmRate, freeDeliveryThreshold
→ Changes apply to new orders only
→ Also set: rider share %, max concurrent deliveries
```

### 16.9 Reporting (Tables Only — No Charts)

```
Admin Web → Reports
→ Sales by vendor: order counts, gross/commission/net — table
→ Payment method split (mpesa vs cod) — table
→ Rider payout summary + pending COD balances — table
→ Vendor commission revenue — table
→ Service provider subscription revenue — table
→ All filters as Firestore query parameters (server-side)
→ Cursor-based pagination — no client-side filtering
→ No chart components, no PDF export
```

---

## 17. Money Flow Scenarios

| Scenario | Customer pays | Customer refunded | Vendor receives | Rider receives | Platform keeps |
|---|---|---|---|---|---|
| All vendors accept — delivered | subtotal + deliveryFee | Nothing | subTotal − commission each | riderSharePercent % of deliveryFeeBaseAmount | All commissions + platform fee share |
| One vendor declines | subtotal + deliveryFee (locked) | Declined vendor's subTotal only | Accepting vendors: subTotal − commission each | riderSharePercent % of original deliveryFeeBaseAmount | Commissions from accepting vendors |
| All vendors decline | subtotal + deliveryFee | Full (totalAmountCharged − refundedAmount) | Nothing | Nothing (not assigned) | Nothing |
| Customer cancels within window | subtotal + deliveryFee | Full amount | Nothing (notified) | Nothing (not assigned) | Nothing |
| Post-window cancel attempt | subtotal + deliveryFee | Admin discretion (platform absorbs) | Vendor net still owed | Proceeds normally | Varies |

**Net check for Scenario 2 (partial decline):**
```
Customer paid:   full subtotal + deliveryFee
Customer refund: declined vendor's subTotal only
Customer net:    accepting vendors' subtotals + deliveryFee  ✓
Vendor payout:   each accepting vendor's subTotal − commission  ✓
Rider payout:    riderSharePercent % of ORIGINAL deliveryFeeBaseAmount  ✓
Platform keeps:  commissions from accepting vendors only  ✓
No stakeholder underpaid or overpaid  ✓
```

**Post-assign vendor decline:**
```
If vendor declines AFTER rider assigned:
→ codAmountDue recomputed = remaining accepted subTotals + deliveryFee
→ Updated codAmountDue written to order doc
→ Rider notified via WhatsApp with updated pickup list + new cash amount
→ Partial refund created for declined vendor's subTotal
```

**Free delivery edge case:**
If original subtotal ≥ `freeDeliveryThreshold` but post-decline subtotal falls below it, the delivery fee remains locked at 0 (intentional — fee locked at checkout). Admin refund record includes note: *"Delivery was waived at checkout; no delivery fee to refund."*

**All-decline with prior partial refund:**
```
fullCancelRefundAmount = totalAmountCharged − refundedAmount
(avoids double-counting any partial refunds already issued)
```

---

## 18. Order Cancellation Detail

### 18.1 Customer-Initiated (Within Window)

```
Order History / Confirmation Screen → [Cancel Order]
→ cancelOrder Cloud Function checks:
  ├── now < cancellationWindowExpiresAt → ALLOW
  └── now ≥ cancellationWindowExpiresAt → BLOCK ("Cancellation window has closed")

→ If allowed:
  → All sub-orders → "cancelled"
  → orderStatus → "cancelled"
  → If mpesa → full refund = totalAmountCharged − refundedAmount
  → If cod → no refund record needed
  → WhatsApp confirmation to customer
  → Each vendor notified: "Order #ORD-1234 cancelled by customer"
```

### 18.2 Vendor-Initiated (Decline)

```
Vendor Web → Orders → [Decline] (with reason)
→ subOrderStatus → "declined"
→ onSubOrderUpdate trigger fires:
  ├── If last accepting vendor → orderStatus → "cancelled"
  │   Full refund = totalAmountCharged − refundedAmount
  └── If other vendors still active:
      → Partial refund for declined vendor's subTotal
      → order.refundedAmount += declinedVendor.subTotal
      → Delivery fee NOT refunded (locked)
      → Customer notified via WhatsApp: "[Vendor] couldn't fulfill part of your order..."
      → Re-evaluate rider assignment:
        ├── If rider not yet assigned → check if remaining vendors all accepted
        │   → if yes → trigger autoAssignRider
        └── If rider already assigned → recompute codAmountDue, notify rider
```

### 18.3 Vendor No-Response — Auto-Decline

```
vendorAutoDeclinePoll runs every 2 minutes:
→ Warning pass: pending sub-orders older than vendorResponseWarningMinutes (~10 min)
  where warningSent == false → send WhatsApp warning → set warningSent = true
→ Decline pass: pending sub-orders older than vendorResponseWindowMinutes (~15 min)
  → subOrderStatus = "declined", declineReason = "auto_timeout"
  → onSubOrderUpdate trigger handles the rest (same as vendor-initiated decline)
→ No per-sub-order task to create, track, or cancel
  → Early accept/decline drops sub-order out of the "pending" query naturally
```

### 18.4 Post-Window Admin Cancel (Exceptional)

```
Customer contacts support → Admin reviews case-by-case
→ Admin can cancel via Admin Web → Orders → [Cancel + Flag Refund]
→ Vendor protected: vendor net amount still owed even if admin approves
  (platform absorbs cost, not the vendor)
```

---

## 19. UI Mockups

### 19.1 Customer Web — Order Confirmation Screen

```
┌─────────────────────────────────┐
│  ✅ Order Placed!               │
│  #ORD-1234                      │
├─────────────────────────────────┤
│  📦 Mama Njeri's Kitchen        │
│  📦 Fresh Produce Co.           │
│  Total: KES 1,320               │
│  Payment: M-Pesa ✓              │
│  ─────────────────────────────  │
│  📲 We'll update you on WhatsApp│
│  ─────────────────────────────  │
│  ⏱ Cancel within: 4:32         │
│  [Cancel Order]                 │
│                                 │
│  (button disappears when timer  │
│   hits 0 or vendor accepts)     │
└─────────────────────────────────┘

After timer expires or first vendor accepts:
┌─────────────────────────────────┐
│  ✅ Order Placed! #ORD-1234     │
│  📦 Mama Njeri's — ⏳ Pending   │
│  📦 Fresh Produce — ✅ Accepted  │
│  ─────────────────────────────  │
│  Need to cancel? Contact support│
└─────────────────────────────────┘
```

### 19.2 Customer Web — Multi-Vendor Cart Screen

```
┌─────────────────────────────────┐
│  ← Continue Shopping   🛒 Cart  │
├─────────────────────────────────┤
│  📦 Mama Njeri's Kitchen        │
│    2x Fries            KES 400  │
│    1x Chicken          KES 650  │
│    Subtotal:         KES 1,050  │
│  ─────────────────────────────  │
│  📦 Fresh Produce Co.           │
│    1kg Tomatoes        KES 120  │
│    Subtotal:           KES 120  │
│  ─────────────────────────────  │
│  Items Subtotal:     KES 1,170  │
│  Delivery Fee:         KES 150  │
│  Total:              KES 1,320  │
│                                 │
│  [Proceed to Checkout]          │
└─────────────────────────────────┘
```

### 19.3 Customer Web — Order History (Cancel Backup)

```
Active Order:
┌─────────────────────────────────┐
│  Order #ORD-1234     Pending    │
│  Placed: 2 min ago              │
│  ─────────────────────────────  │
│  📦 Mama Njeri's  — Pending     │
│  📦 Fresh Produce — Pending     │
│  ─────────────────────────────  │
│  ⏱ Cancel within: 3 min        │
│  [Cancel Order]                 │
└─────────────────────────────────┘

After first vendor accepts (window closes immediately):
┌─────────────────────────────────┐
│  Order #ORD-1234  Processing    │
│  ─────────────────────────────  │
│  📦 Mama Njeri's ✅ Accepted    │
│  📦 Fresh Produce ⏳ Pending    │
│  ─────────────────────────────  │
│  Need help? Contact support     │
│  (Cancel no longer available)   │
└─────────────────────────────────┘
```

### 19.4 Vendor Web — Incoming Order

```
┌─────────────────────────────────────────────────────────┐
│  New Order #ORD-1234                                     │
├─────────────────────────────────────────────────────────┤
│  Your Items:                                            │
│    2x Fries                               KES 400       │
│    1x Chicken                             KES 650       │
│  Your Subtotal:                         KES 1,050       │
│  Commission (12%):                       -KES 126       │
│  Your Net:                                KES 924       │
│                                                         │
│  Payment: M-Pesa (Paid) / COD                           │
│  Customer: John S. — 07XX-XXXXX                         │
│  ℹ This order also includes items from 1 other vendor   │
│                                                         │
│  [Accept]                         [Decline]             │
└─────────────────────────────────────────────────────────┘

After accepting:
┌─────────────────────────────────────────────────────────┐
│  Order #ORD-1234          ✅ Accepted                    │
├─────────────────────────────────────────────────────────┤
│  2x Fries | 1x Chicken                                  │
│                                                         │
│  [Mark as Preparing]                                    │
│                                                         │
│  Status: Preparing → [Mark Ready for Pickup]            │
└─────────────────────────────────────────────────────────┘
```

### 19.5 Rider Web — Active Delivery (Multi-Stop)

```
┌──────────────────────────────┐
│  Order #ORD-1234             │
│  COD: KES [codAmountDue]     │
├──────────────────────────────┤
│  PICKUP 1 of 2               │
│  📍 Mama Njeri's Kitchen     │
│     Westlands, Nairobi       │
│  Items:                      │
│    • 2x Fries                │
│    • 1x Chicken              │
│  [Mark Picked Up ✓]          │
├──────────────────────────────┤
│  PICKUP 2 of 2               │
│  📍 Fresh Produce Co.        │
│     Parklands, Nairobi       │
│  Items:                      │
│    • 1kg Tomatoes            │
│  [Mark Picked Up]            │
├──────────────────────────────┤
│  DROP-OFF                    │
│  🏠 Jane W. — Apt C-204      │
│  📞 07XX-XXXXX               │
│  (unlocks after all          │
│   pickups marked ✓)          │
└──────────────────────────────┘

COD drop-off:
┌──────────────────────────────┐
│  All items collected ✓       │
│                              │
│  [Confirm Cash Collected:    │
│   KES [codAmountDue]]        │
│                              │
│  [Mark Delivered]            │
│  (available after cash       │
│   confirmation)              │
└──────────────────────────────┘
```

### 19.6 Rider Web — My Wallet

```
┌──────────────────────────────┐
│  My Wallet                   │
├──────────────────────────────┤
│  Pending COD Balance:        │
│  KES 4,400                   │
│                              │
│  ⚠ Due by: Friday, Jul 4    │
│                              │
│  [Recharge Now]              │
│  (M-Pesa STK Push)           │
│                              │
│  Account Status: ✅ Good     │
├──────────────────────────────┤
│  Recent Transactions         │
│  COD Collected  +KES 1,200   │
│  COD Collected  +KES 980     │
│  Recharge Paid  -KES 2,000   │
└──────────────────────────────┘

When locked:
┌──────────────────────────────┐
│  ⛔ Account Locked           │
│                              │
│  Pending COD: KES 4,400      │
│  Recharge to go online       │
│                              │
│  [Recharge Now]              │
└──────────────────────────────┘
```

### 19.7 Admin Web — Payouts Screen

```
┌────────────────────────────────────────────────────────────┐
│  Payouts — June 2026                                        │
├────────────────────────────────────────────────────────────┤
│  Rider Payouts                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │ Name         │ Phone      │ Gross │ COD   │ Net     │    │
│  │ John Mwangi  │07XX-XXXXX  │ 8,400 │-4,400 │  4,000  │    │
│  │                                    [Mark Paid]      │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
│  Vendor Payouts                                             │
│  ┌────────────────────────────────────────────────────┐    │
│  │ Vendor          │ Phone     │ Orders │ Net          │    │
│  │ Mama Njeri's    │07XX-XXXXX │  142   │ KES 125,400  │    │
│  │                                       [Mark Paid]   │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
│  Refunds Pending                                            │
│  ┌────────────────────────────────────────────────────┐    │
│  │ Order     │ Phone     │ Amount  │ Type              │    │
│  │ #ORD-1234 │07XX-XXXXX │ KES 120 │ Partial (vendor)  │    │
│  │                                   [Mark Paid]       │    │
│  └────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────┘
```

### 19.8 Service Provider Web — Subscription Payment Details

```
┌──────────────────────────────────────────────────────────────┐
│  Subscription                                                 │
│  Status: Active  |  Due: 15 Jul 2026  |  Fee: KES 2,000/mo   │
├──────────────────────────────────────────────────────────────┤
│  Payment Details                                              │
│  Account Name:   Platform Payments Ltd                        │
│  Account Number: 1234567890                                   │
│  Bank:           Equity Bank Kenya                            │
│  ─────────────────────────────────────────────────────────   │
│  Transfer the subscription fee to the above account,         │
│  then email your transaction proof/receipt to:               │
│  billing@platform.co.ke                                       │
│                                                               │
│  Admin will verify and activate your subscription.            │
└──────────────────────────────────────────────────────────────┘

When expired:
┌──────────────────────────────────────────────────────────────┐
│  ⚠ Subscription Expired                                       │
│  Your services are currently hidden from customers.           │
│  Pay via bank transfer and email proof to restore access.     │
│                                                               │
│  [View Payment Details]                                       │
└──────────────────────────────────────────────────────────────┘
```

---

## 20. Implementation Phases

### Phase 1 — Foundation (Weeks 1–2)

- Firebase project setup (Firestore, Auth, Storage, Functions v2, Secret Manager)
- All 5 Flutter Web features scaffolded under `lib/features/`
- GoRouter with Firebase Auth custom claims guards (role-based routing)
- Firestore schema with subcollection structure and `vendorSubOrderIds` array on order doc
- Firestore security rules (Section 6) deployed and tested
- Admin app: Firestore offline persistence disabled

### Phase 2 — Vendor & Multi-Vendor Cart Core (Weeks 3–5)

- Admin category management
- Vendor self-registration + activation + commission setup
- `locationConfigured` flag; products hidden until location set
- Manual lat/lng fallback in Vendor Location UI (Google Maps API unavailable)
- `validateUpload` Storage trigger (MIME, size, quarantine)
- `resizeProductImage` trigger (800 px + thumb 200 px)
- Multi-vendor cart with per-vendor ChangeNotifiers; `context.select()` scoping
- `calculateDeliveryFee` Cloud Function (Haversine, server-side vendor location validation)
- `deliveryFeeBaseAmount` always stored regardless of waiver
- Zone-based fee calculation; fee locked at checkout
- M-Pesa STK Push + `initiateMpesaPayment` rate limiting (5/order, 10/hour)
- M-Pesa callback authentication (IP allowlist + secret token, Section 6.2)
- COD checkout flow
- `onOrderCreate` with sub-order subcollection creation + `pending_suborders` records
- `checkGuestPhone` function (guest vs registered account check)
- `cached_network_image` + `ListView.builder` lazy loading on browse screen

### Phase 3 — Order Flow & Rider Assignment (Weeks 6–7)

- Vendor Web: sub-order Accept/Decline (their items only; context note for other vendors)
- Accept: `cancellationWindowClosed` atomic flag transaction (Section 7.6)
- `vendorAutoDeclinePoll` with composite Firestore index on `pending_suborders`
  - Warning at ~10 min, auto-decline at ~15 min
  - No per-sub-order task scheduling
- Status updates: `preparing` → `ready`
- `autoAssignRider`: idle-first then `lastAssignedAt` ascending sort; fresh transaction snapshot; `codAmountDue` computed from `subTotal` not `vendorNetAmount`
- `respondToAssignment`: decline does NOT update `lastAssignedAt`
- `lastAssignedAt` initialised to Unix epoch on rider activation
- `setRiderOnline` server-side lock check
- Rider Web: multi-stop pickup UI; per-vendor [Mark Picked Up]
- `markPickup` Cloud Function + all-pickup → `out_for_delivery` trigger
- `onDeliveryComplete` using `deliveryFeeBaseAmount` for rider earnings

### Phase 4 — COD, Cancellations & Refunds (Weeks 8–9)

- `confirmCODCollection` with `codConfirmedAt` idempotency (Section 7.3)
- Order Confirmation countdown timer:
  - `'…'` placeholder until `cancellationWindowExpiresAt` fetched from Firestore
  - Server-driven; refresh-safe
  - Auto-hides on timer expiry or first vendor accept
- `cancelOrder` Cloud Function (server-side window check)
- `onSubOrderUpdate` partial decline handling + post-assignment `codAmountDue` recompute (Section 7.6)
- All-decline full refund = `totalAmountCharged − refundedAmount`
- Refund records with admin note for waived-delivery edge case
- M-Pesa retry UI: [Retry Payment] with rate-limit error messaging
- `riderWalletRecharge` (partial recharge always allowed)
- `setRiderOffline` active delivery guard
- `weeklyCODRechargeCheck` — lock enforcement, no penalty fees
- Admin manual payout UI: create records per rider/vendor; re-auth required
- `currentMonthEarnings` auto-zero trigger on payout record creation
- `rider_wallet_transactions` monthly_settlement record

### Phase 5 — Service Providers, Notifications & Reporting (Week 10)

- Service provider registration + subscription flow
- `platform_settings.bankDetails` — payment details shown in provider portal
- `serviceProviderSubscriptionCheck` daily scheduled function
- Service listing with [Contact on WhatsApp] button
- `onProviderPhoneUpdate` fan-out trigger for `providerPhone` denormalization
- `sendWhatsApp()` wrapper + `retryFailedNotifications` scheduler (Section 7.19–7.20)
- `failed_notifications` dead-letter collection
- Admin dashboard: server-side paginated tabular reporting (cursor-based)
- All report filters as Firestore query parameters
- Twilio credentials in Secret Manager confirmed

### Phase 6 — Security & Polish (Week 11)

- Security rules QA + penetration testing
- `requireRecentAuth()` on all admin financial `onCall` functions (Section 7.23)
- Firebase Hosting CDN for product images (public path)
- PWA manifest + service worker for Rider Web build target
- Structured JSON logging in all Cloud Functions (Section 7 preamble)
- `cleanupGuestAccounts` monthly scheduled function
- Admin app offline persistence disabled in build configuration
- `upload_rejections` admin review interface
- Cloud Task queue with concurrency limit for `autoAssignRider` at high volume

### Phase 7 — Testing & Deployment (Week 12)

- Unit tests: Haversine formula, `codAmountDue` computation, refund math
- Integration tests: all 5 money-flow scenarios end-to-end
- `codAmountDue` verified = sum of accepted `subTotal` values + `deliveryFee` (not `vendorNetAmount`)
- Rider rotation fairness: confirm `lastAssignedAt` ascending distributes ±15% over 7-day window
- `vendorAutoDeclinePoll` timing: confirm poll catches warning/decline within 2-minute margin
- Server-side vendor location validation: confirm checkout rejected if any vendor has no location
- Cancellation window race: concurrent accept test — confirm `cancellationWindowClosed` prevents double-write
- COD idempotency: double-tap test on `confirmCODCollection`
- Security rules QA: each role can only read/write their permitted collections
- M-Pesa callback spoofing: confirm 403 returned for invalid IP + wrong token
- Admin re-auth: confirm financial functions reject tokens older than 5 minutes
- M-Pesa STK Push sandbox → production cutover
- Firebase Hosting deployment (5 subdomains)
- Production monitoring + Stackdriver alerts on `severity: ERROR` log entries
- Launch

---

## 21. Success Metrics

| Metric | Target |
|---|---|
| Order Completion Rate | >95% |
| Payment Success Rate (M-Pesa) | >98% |
| Vendor Response Time | <15 minutes |
| Auto Rider Assignment Time | <2 minutes after all vendors accept |
| Customer Engagement | >70% active users |
| Order Accuracy | >99% |
| Delivery Time | <60 min (food), <24 hours (laundry/produce) |
| WhatsApp Notification Delivery Rate | >99% (including retry) |
| Rider Payout Accuracy | 100% |
| On-Time Monthly Payout Rate | 100% processed within first week of month |
| COD Recharge Compliance Rate | >95% before due date |
| Vendor Onboarding Time | <48 hours |
| Service Provider Subscription Renewal Rate | >90% |
| Zero financial disputes from incorrect calculations | 100% |
| Rider Assignment Fairness | Assignment count spread ±15% over rolling 7-day window
