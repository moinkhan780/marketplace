# Multi-Vendor Delivery & Services Marketplace

**Project Specification — Version 6**
**Flutter Web · Firebase · M-Pesa · Twilio WhatsApp**

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [System Components](#2-system-components)
3. [Technology Stack](#3-technology-stack)
4. [Architecture — Folder Structure](#4-architecture--folder-structure)
5. [Database Schema (Firestore)](#5-database-schema-firestore)
6. [Cloud Functions — Specifications](#6-cloud-functions--specifications)
7. [Scheduled Functions](#7-scheduled-functions)
8. [Reducing Firebase Costs](#8-reducing-firebase-costs)
9. [Customer Ordering Process](#9-customer-ordering-process)
10. [Payment Flow](#10-payment-flow)
11. [Notification Flow (WhatsApp)](#11-notification-flow-whatsapp)
12. [Vendor Workflow](#12-vendor-workflow)
13. [Rider Workflow](#13-rider-workflow)
14. [Service Provider Workflow](#14-service-provider-workflow)
15. [Admin Workflow](#15-admin-workflow)
16. [Sub-Admin System](#16-sub-admin-system)
17. [In-App Chat System](#17-in-app-chat-system)
18. [Vendor Self-Delivery](#18-vendor-self-delivery)
19. [Money Flow Scenarios](#19-money-flow-scenarios)
20. [Order Cancellation Detail](#20-order-cancellation-detail)
21. [UI Mockups](#21-ui-mockups)
22. [Implementation Phases](#22-implementation-phases)

---

## 1. Project Overview

A Flutter Web marketplace platform connecting customers with multiple independent vendors (laundry, fresh produce, quick bites, etc.) and independent service providers (e.g. home services), supported by a dedicated rider network for deliveries.

Customers can add items from multiple vendors into a single cart — one order, one rider, one delivery fee, one payment. Vendors manage their own products and a single business location, and receive their portion of incoming orders directly. Service providers list bookable services that customers can request — customers can chat with providers directly via in-app messaging to arrange bookings; no payment is collected through the platform for service bookings.

Riders pick up from all vendors in the order then deliver to the customer. Vendors may optionally opt into **self-delivery**, using their own in-house rider instead of the platform rider network. Admin manually pays out riders and vendors via M-Pesa app and marks payments settled in the dashboard. A **sub-admin** role allows the platform owner to delegate limited operational duties without granting full financial access.

---

## 2. System Components

### 2.1 Customer Web
Browse vendors/products and services, add items from multiple vendors into one cart, place orders, pay (M-Pesa or COD), track orders, view order history, and chat with service providers directly from service listings. Accessible via web browser (responsive design); guest checkout supported.

### 2.2 Admin Web
Approve and manage vendors, service providers, and riders; set commission and subscription rates; manage admin-defined categories; view platform-wide reports; manually mark rider/vendor payouts as paid; process refunds manually; manage sub-admin accounts and permissions. Requires secure admin login with re-authentication for all financial write operations.

### 2.3 Sub-Admin Web
Performs a restricted subset of admin operations as defined per sub-admin account. No access to financial operations (payouts, refunds, commission editing) unless explicitly granted. Permissions are enforced server-side.

### 2.4 Vendor Web
Vendor self-registration, managing own products/pricing (within admin-defined categories), managing a single business location, receiving and actioning their portion of incoming orders, viewing commission deductions and payout status, and opting into self-delivery with their own riders. Requires secure vendor login, gated by admin activation.

Vendors using platform delivery do not assign riders — a rider is auto-assigned by the system once all vendors in the order accept. Vendors opted into self-delivery manage their own rider assignment.

### 2.5 Service Provider Web
Service provider self-registration, listing services and pricing (within admin-defined categories), and viewing subscription status. Customers contact providers via in-app chat directly from the service listing. Requires secure service provider login, gated by admin activation.

### 2.6 Rider Web
Rider self-registration, viewing assigned orders, updating delivery status per pickup and drop-off, collecting COD cash, managing wallet/recharge, and viewing earnings and payout history. Accessible via mobile browser; login gated by admin activation (or vendor activation for vendor riders). Installable as a PWA for home-screen access.

---

## 3. Technology Stack

This platform is built almost entirely on Firebase, keeping the stack simple and easy to maintain.

### 3.1 Frontend
- **Framework:** Flutter Web — single codebase, single repository, with role-based routing
- **State Management:** Provider package, scoped per feature to avoid unnecessary rebuilds
- **Responsive:** Mobile-first design across all roles

### 3.2 Firebase Services
- **Firebase Authentication** — phone-based login and role management for all user types
- **Cloud Firestore** — primary database for all app data, used in real time for orders, chat, and live status updates
- **Firebase Cloud Messaging (FCM)** — push notifications for order updates and chat alerts
- **Firebase Hosting** — serves the Flutter Web app and public assets (e.g. product images) via CDN
- **Firebase Storage** — stores uploaded files such as ID documents and registration paperwork
- **Cloud Functions** — backend logic, validation, and integrations (M-Pesa, WhatsApp)
- **Remote Config** — used where needed to toggle features or adjust settings without a redeploy

### 3.3 Third-Party Integrations
- **WhatsApp Notifications:** Twilio API
- **Payment:** M-Pesa Daraja API (STK Push)
- **Map Picker:** Google Maps JavaScript API, with manual latitude/longitude entry as a fallback
- **Distance Calculation:** calculated directly in Cloud Functions — no external API needed

---

## 4. Architecture — Folder Structure

Single Flutter project. All user roles are organised as top-level features under `lib/features/`. Shared domain logic, models, and services live in `lib/core/`. Routing is role-based, guarded by Firebase Auth custom claims.

```
lib/
├── core/
│   ├── models/           order.dart, vendor.dart, rider.dart, chat_room.dart, ...
│   ├── services/         firestore_service.dart, auth_service.dart,
│   │                     twilio_service.dart, mpesa_service.dart, chat_service.dart
│   ├── providers/        auth_provider.dart  (shared role detection)
│   ├── routing/          app_router.dart  (role guards)
│   └── widgets/          shared UI components
├── features/
│   ├── customer/         browse/ cart/ checkout/ orders/ services/ chat/
│   ├── vendor/            products/ orders/ location/ earnings/ self_delivery/ vendor_riders/
│   ├── rider/             deliveries/ wallet/ earnings/
│   ├── service_provider/  services/ subscription/ chat/
│   ├── admin/             vendors/ riders/ orders/ payouts/ refunds/ settings/ sub_admins/
│   └── sub_admin/         dashboard/ vendors/ riders/ orders/
└── main.dart
```

---

## 5. Database Schema (Firestore)

### 5.1 Collection Overview

| Collection / Subcollection | Key Fields | Notes |
|---|---|---|
| `users` | userId, name, phone, role, isActive | Customer, vendor, rider, service_provider, admin, sub_admin |
| `categories` | categoryId, name, isActive | Admin-managed only |
| `vendors` | vendorId, status, commissionPercent, location{}, locationConfigured, selfDeliveryEnabled | Single location embedded |
| `vendor_riders` | vendorRiderId, vendorId, name, phone, isActive | Riders owned by a self-delivery vendor |
| `products` | productId, vendorId, categoryId, price, isAvailable | Hidden if vendor locationConfigured=false |
| `orders` | orderId, userId, paymentMethod, orderStatus, codAmountDue, deliveryFeeBaseAmount, cancellationWindowExpiresAt, cancellationWindowClosed, refundedAmount, deliveryMode | vendorSubOrders stored as subcollection |
| `orders/{id}/subOrders/{vendorId}` | subOrderStatus, subTotal, vendorNetAmount, autoDeclineWarningSent, codConfirmedAt | One per vendor in the order |
| `pending_suborders` | orderId, vendorId, createdAt, subOrderStatus, warningSent | Lightweight index for poll queries |
| `riders` | riderId, isOnline, activeDeliveryCount, lastAssignedAt, pendingCODBalance, riderAccountStatus, riderType (platform/vendor), vendorId (if vendor rider) | lastAssignedAt initialised to epoch on activation |
| `rider_wallet_transactions` | type, amount, balanceAfter | Written only by Cloud Functions |
| `payouts` | payeeType, grossAmount, netPayoutAmount, payoutStatus | Created manually by admin |
| `refunds` | refundType, refundStatus, declinedVendorId (optional) | Managed by admin |
| `service_providers` | providerId, subscriptionStatus, subscriptionDueDate | |
| `services` | serviceId, providerId, categoryId, price, isAvailable | |
| `chat_rooms` | roomId, participants[], participantRoles{}, lastMessage, lastMessageAt, roomType | customer↔provider or vendor↔platform rider |
| `chat_rooms/{roomId}/messages` | messageId, senderId, text, createdAt, readBy[] | Subcollection |
| `sub_admins` | subAdminId, name, phone, email, permissions[], createdBy, isActive | |
| `failed_notifications` | orderId, recipient, message, failedAt, retryCount | Dead-letter for Twilio failures |
| `payment_attempts` | orderId+uid composite key, count, hourlyCount | STK Push rate limiting |
| `upload_rejections` | originalPath, quarantinePath, contentType, size | Admin review of rejected uploads |
| `platform_settings` | deliveryZones[], riderSharePercent, selfDeliveryVendorSharePercent, customerCancelWindowMinutes, bankDetails{} | |

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
  "role":      "string (customer/admin/sub_admin/vendor/service_provider/rider)",
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
  "vendorId":             "string (Firebase Auth UID)",
  "businessName":         "string",
  "ownerName":            "string",
  "phone":                "string (verified)",
  "businessRegDocUrl":    "string (Firebase Storage URL)",
  "ownerIdPhotoUrl":      "string (Firebase Storage URL)",
  "categoryIds":          "array of strings",
  "status":               "string (pending_activation/active/rejected/blocked)",
  "commissionPercent":    "number",
  "location": {
    "name":    "string",
    "address": "string",
    "lat":     "number",
    "lng":     "number"
  },
  "locationConfigured":   "boolean (false until vendor sets lat/lng)",
  "selfDeliveryEnabled":  "boolean (default false — set by vendor; admin can disable)",
  "rejectionReason":      "string (optional)",
  "blockedReason":        "string (optional)",
  "createdAt":            "timestamp",
  "activatedAt":          "timestamp (optional)",
  "updatedAt":            "timestamp"
}
```

### 5.5 `vendor_riders` Collection

Riders belonging to a vendor who has opted into self-delivery. These riders are not part of the platform rider pool and do not go through admin activation.

```json
{
  "vendorRiderId":   "string (Firebase Auth UID — separate phone login)",
  "vendorId":        "string",
  "name":            "string",
  "phone":           "string (verified via Firebase Auth)",
  "nationalId":      "string",
  "idPhotoUrl":      "string",
  "vehicleType":     "string (bike/motorbike)",
  "isActive":        "boolean (toggled by vendor)",
  "isOnline":        "boolean",
  "activeDeliveryCount": "number",
  "createdAt":       "timestamp",
  "updatedAt":       "timestamp"
}
```

> Vendor riders use the same Rider Web PWA, but their `riderType = 'vendor'` claim restricts their view to orders from their owning vendor only. They have no COD wallet or monthly payout — compensation between vendor and rider is handled off-platform.

### 5.6 `products` Collection

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

### 5.7 `orders/{orderId}/subOrders/{vendorId}` Subcollection

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
  "deliveryMode":            "string (platform|self_delivery — snapshotted from vendor at order creation)",
  "selfDeliveryRiderId":     "string (optional — vendorRiderId assigned by vendor)",
  "selfDeliveryRiderName":   "string (optional)",
  "selfDeliveryVendorShare": "number (optional — vendor's share of delivery fee for self-delivery sub-orders)",
  "acceptedAt":              "timestamp (optional)",
  "readyAt":                 "timestamp (optional)",
  "pickedUpAt":              "timestamp (optional)",
  "codConfirmedAt":          "timestamp (optional)"
}
```

> `vendorCommissionPercent` and `deliveryMode` are both **snapshotted at order creation and never mutated** — a vendor toggling self-delivery mid-order does not affect in-progress orders.

### 5.8 `orders` Document

```json
{
  "orderId":                     "string",
  "userId":                      "string (Firebase Auth UID or anonymous)",
  "isGuestOrder":                "boolean",
  "userName":                    "string",
  "userPhone":                   "string",
  "vendorSubOrderIds":           "array of vendorIds (for transaction getAll)",
  "overallSubtotal":             "number",
  "deliveryFee":                 "number (0 if waived; locked at checkout)",
  "deliveryFeeWaived":           "boolean",
  "deliveryFeeBaseAmount":       "number (pre-waiver fee — always stored; used for rider/vendor payout)",
  "riderSharePercent":           "number (snapshotted at order creation — platform sub-orders)",
  "selfDeliveryVendorSharePercent": "number (snapshotted at order creation — self-delivery sub-orders)",
  "farthestVendorDistanceKm":    "number",
  "deliveryZone":                "string",
  "totalAmountCharged":          "number",
  "refundedAmount":              "number (default 0)",
  "codAmountDue":                "number",
  "codAmountCollected":          "number (optional)",
  "codSettlementStatus":         "string (n/a | awaiting_rider_recharge | collected)",
  "paymentMethod":               "string (mpesa | cod)",
  "paymentStatus":               "string (pending/pending_collection/confirmed/failed/partially_refunded/fully_refunded)",
  "paymentReference":            "string (optional)",
  "orderStatus":                 "string (pending/all_accepted_awaiting_rider/partially_declined/out_for_delivery/delivered/cancelled)",
  "hasSelfDeliverySubOrders":    "boolean (true if any subOrder has deliveryMode=self_delivery)",
  "hasPlatformSubOrders":        "boolean (true if any subOrder has deliveryMode=platform)",
  "cancellationWindowExpiresAt": "timestamp",
  "cancellationWindowClosed":    "boolean",
  "cancelledAt":                 "timestamp (optional)",
  "cancellationReason":          "string (optional)",
  "deliveryLocation": {
    "method":      "string",
    "lat":         "number",
    "lng":         "number",
    "apartment":   "string (optional)",
    "wing":        "string (optional)",
    "floor":       "string (optional)",
    "addressText": "string (optional)"
  },
  "riderId":      "string (null until auto-assigned — platform rider for platform sub-orders)",
  "riderName":    "string (optional)",
  "assignedAt":   "timestamp (optional)",
  "adminAlert":   "string (optional)",
  "createdAt":    "timestamp",
  "updatedAt":    "timestamp",
  "deliveredAt":  "timestamp (optional)"
}
```

### 5.9 `chat_rooms` Collection

```json
{
  "roomId":          "string (auto-generated)",
  "roomType":        "string (customer_provider | vendor_platform_rider)",
  "participants":    "array of userIds (exactly 2)",
  "participantRoles": {
    "<userId1>": "string (customer | service_provider | vendor | rider)",
    "<userId2>": "string"
  },
  "participantNames": {
    "<userId1>": "string",
    "<userId2>": "string"
  },
  "relatedEntityId": "string (serviceId for customer_provider; orderId for vendor_platform_rider)",
  "lastMessage":     "string (text preview — max 100 chars)",
  "lastMessageAt":   "timestamp",
  "lastMessageBy":   "string (userId)",
  "unreadCount": {
    "<userId1>": "number",
    "<userId2>": "number"
  },
  "isActive":        "boolean (false if either party deleted/blocked)",
  "createdAt":       "timestamp",
  "updatedAt":       "timestamp"
}
```

### 5.10 `chat_rooms/{roomId}/messages` Subcollection

```json
{
  "messageId":  "string",
  "senderId":   "string (userId)",
  "text":       "string (max 2000 chars)",
  "createdAt":  "timestamp (server timestamp)",
  "readBy":     "array of userIds",
  "isDeleted":  "boolean (default false — sender can delete own message within 5 min)"
}
```

### 5.11 `sub_admins` Collection

```json
{
  "subAdminId":   "string (Firebase Auth UID)",
  "name":         "string",
  "phone":        "string (verified)",
  "email":        "string (optional)",
  "permissions":  "array of strings — see Section 16 for permission keys",
  "isActive":     "boolean",
  "createdBy":    "string (adminUserId)",
  "createdAt":    "timestamp",
  "updatedAt":    "timestamp"
}
```

### 5.12 `refunds` Collection

```json
{
  "refundId":             "string",
  "orderId":              "string",
  "userId":               "string",
  "userPhone":            "string",
  "amount":               "number",
  "refundType":           "string (full / partial_vendor_decline / partial_item_adjust)",
  "reason":               "string",
  "initiatedBy":          "string (customer/vendor/admin/sub_admin/system)",
  "declinedVendorId":     "string (optional)",
  "paymentReference":     "string",
  "refundStatus":         "string (pending_manual/completed/failed)",
  "mpesaManualReference": "string",
  "adminNote":            "string (optional)",
  "createdAt":            "timestamp",
  "completedAt":          "timestamp (optional)"
}
```

### 5.13 `service_providers` Collection

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
  "lastBankTransferReference": "string (optional)",
  "rejectionReason":           "string (optional)",
  "blockedReason":             "string (optional)",
  "createdAt":                 "timestamp",
  "activatedAt":               "timestamp (optional)",
  "updatedAt":                 "timestamp"
}
```

### 5.14 `services` Collection

```json
{
  "serviceId":        "string",
  "providerId":       "string",
  "providerPhone":    "string (denormalized)",
  "categoryId":       "string",
  "name":             "string",
  "description":      "string",
  "price":            "number (informational)",
  "durationEstimate": "string (optional)",
  "isAvailable":      "boolean",
  "createdAt":        "timestamp",
  "updatedAt":        "timestamp"
}
```

### 5.15 `riders` Collection

```json
{
  "riderId":              "string",
  "name":                 "string",
  "phone":                "string (verified)",
  "nationalId":           "string",
  "idPhotoUrl":           "string",
  "licensePhotoUrl":      "string (optional)",
  "vehicleType":          "string",
  "riderType":            "string (platform | vendor)",
  "vendorId":             "string (only set for riderType=vendor)",
  "status":               "string (pending_activation/active/rejected/blocked)",
  "isOnline":             "boolean",
  "activeDeliveryCount":  "number",
  "lastAssignedAt":       "timestamp",
  "currentMonthEarnings": "number (platform riders only)",
  "pendingCODBalance":    "number (platform riders only)",
  "codRechargeDueDate":   "timestamp (platform riders only)",
  "riderAccountStatus":   "string (good_standing/locked — platform riders only)",
  "totalDeliveries":      "number",
  "rejectionReason":      "string (optional)",
  "blockedReason":        "string (optional)",
  "createdAt":            "timestamp",
  "activatedAt":          "timestamp (optional)",
  "updatedAt":            "timestamp"
}
```

### 5.16 Other Collections

`rider_wallet_transactions`, `payouts`, and `platform_settings` follow the structures referenced in Section 5.1 above.

---

## 6. Cloud Functions — Specifications

All Cloud Functions are written using Firebase Cloud Functions v2. Every callable function begins with an authentication check, and all credentials are loaded from Firebase Secret Manager rather than environment config. Logging is structured JSON throughout for easier debugging.

### 6.1 `onOrderCreate`
- Snapshots `riderSharePercent` and `selfDeliveryVendorSharePercent` from `platform_settings` onto the order doc
- For each vendor in the cart, snapshots `selfDeliveryEnabled` onto the sub-order as `deliveryMode`
- Sets `hasSelfDeliverySubOrders` and `hasPlatformSubOrders` flags on the order doc
- Validates vendor locations server-side — rejects the order if any vendor has no `lat`/`lng` set
- Creates sub-order documents in `orders/{orderId}/subOrders/{vendorId}`
- Writes lightweight records to `pending_suborders` for the auto-decline poll
- Sets `cancellationWindowExpiresAt` and `cancellationWindowClosed = false`
- Notifies each vendor and the customer via WhatsApp

### 6.2 M-Pesa Payment Handling
Covers STK Push initiation and the M-Pesa callback. The callback endpoint is protected with an IP allowlist and a secret token check before any payment status is updated.

### 6.3 COD Confirmation
Rider confirms cash collection on delivery; the function is idempotent so a duplicate call cannot double-count a payment.

### 6.4 `vendorAutoDeclinePoll`
Runs on a schedule (see Section 7) to warn, then auto-decline, vendors who haven't responded to a sub-order in time.

### 6.5 `autoAssignRider` — Platform Riders Only
Assigns a platform rider only to sub-orders with `deliveryMode = 'platform'`. If an order contains only self-delivery sub-orders, this function is skipped entirely — those vendors assign their own riders independently. For mixed orders, a platform rider is assigned to cover the platform sub-orders only.

The function loops through available riders (sorted by idle time, then last assignment) and assigns the first rider who passes all checks inside a transaction: no rider already assigned, rider not at their concurrent-delivery limit, rider account not locked, and rider type is `platform`. The order's `codAmountDue` is calculated using only the accepted platform sub-orders plus the delivery fee. If no rider is available, the order is flagged with `adminAlert: 'no_rider_available'`.

### 6.6 `assignSelfDeliveryRider` — Vendor-Triggered
Lets a vendor assign one of their own riders to their sub-order after accepting it. Validates that the sub-order belongs to the calling vendor, that its `deliveryMode` is `self_delivery`, that it's in an assignable status (`accepted`, `preparing`, or `ready`), and that the chosen rider belongs to the vendor and is active. On success, the sub-order is updated with the rider's ID and name, and the vendor's own rider is notified via WhatsApp.

### 6.7 `onSubOrderUpdate`
Fires whenever a sub-order's status changes and handles several outcomes:

- When a sub-order is **accepted**, closes the customer's cancellation window (once, via transaction) if it isn't closed already.
- Recalculates whether **all active sub-orders are now accepted** (or further along). If so, and the order has platform sub-orders with no rider yet, triggers `autoAssignRider`. Self-delivery sub-orders never block platform rider assignment.
- If **every** sub-order ends up declined or cancelled, the whole order is cancelled and a full refund is created automatically.
- If a **single vendor declines**, a partial refund is created for that vendor's sub-total, and — if a platform rider is already assigned — `codAmountDue` is recalculated using the remaining accepted platform sub-orders.

### 6.8 Self-Delivery Fee Split

For self-delivery sub-orders, the vendor receives a share of the delivery fee instead of a platform rider — calculated the same way the rider's share is calculated for platform deliveries, just paid to the vendor rather than a rider.

```javascript
// Platform delivery:
riderPayout = deliveryFeeBaseAmount * (riderSharePercent / 100);     // e.g. 80%
platformKeeps = deliveryFeeBaseAmount * ((100 - riderSharePercent) / 100); // e.g. 20%

// Self-delivery (vendor replaces the rider role):
vendorDeliveryShare = deliveryFeeBaseAmount * (selfDeliveryVendorSharePercent / 100); // 80%
platformKeeps = deliveryFeeBaseAmount * ((100 - selfDeliveryVendorSharePercent) / 100); // 20%
```

`selfDeliveryVendorSharePercent` is a platform setting (default **80%**, platform keeps **20%**), snapshotted onto the order at creation just like `riderSharePercent`. The calculated `vendorDeliveryShare` is stored on the relevant sub-order as `selfDeliveryVendorShare` once the sub-order is delivered, and is included alongside the vendor's item commission payout — both paid out manually by admin via M-Pesa, same as a normal vendor payout.

For mixed-cart orders, this split only applies to the self-delivery vendor's own portion of the delivery fee; the platform sub-orders' rider share is calculated and paid separately as usual.

Example: Order with KES 200 delivery fee (self-delivery vendor):
→ Vendor gets: KES 160 (80%)
→ Platform keeps: KES 40 (20%)
→ Vendor pays rider: off-platform (e.g., KES 150)
→ Vendor net from delivery: KES 10

### 6.9 In-App Chat Functions

- **`createChatRoom`** — creates (or returns an existing) chat room between two participants. Validates room type rules server-side:
  - `customer_provider`: caller must be a customer, target must be an active, subscribed service provider.
  - `vendor_platform_rider`: caller must be an active vendor with a sub-order on the given order, the target must be the **platform** rider currently assigned to that order (`order.riderId == targetUserId`), and vendor riders (`riderType = 'vendor'`) are always rejected as a target.
  - Rate-limited to 10 new rooms per user per day.
- **`sendMessage`** — validates the sender is a participant and the room is active, writes the message, and updates the room's `lastMessage`, `lastMessageAt`, and the other participant's unread count in a single transaction.
- **`markMessagesRead`** — clears the calling user's unread count on a room.
- **`deleteMessage`** — lets a sender soft-delete their own message within 5 minutes of sending.

### 6.10 Sub-Admin Management Functions

- **`createSubAdmin`** (admin only) — validates the requested permission keys against the allowed list (Section 16), creates a Firebase Auth account with the `sub_admin` role claim, and writes the `sub_admins` and `users` records.
- **`updateSubAdminPermissions`** (admin only, requires recent re-authentication) — updates a sub-admin's permissions or active status.

### 6.11 `registerVendorRider` — Vendor Registers Own Rider
Callable only by vendors with `selfDeliveryEnabled = true`. Creates a Firebase Auth account for the new rider with `riderType: 'vendor'` and `vendorId` claims, then writes matching `vendor_riders` and `riders` documents — no admin approval is required since the vendor is directly accountable for their own riders.

### 6.12 Other Functions
The platform also includes standard supporting functions for: responding to a rider assignment, calculating delivery fees, cancelling orders, marking pickup, completing delivery, rider wallet recharge, setting a rider online/offline (vendor riders skip the COD lock check), validating and resizing uploaded images, cleaning up abandoned guest accounts, the WhatsApp sending wrapper, retrying failed notifications, and an admin re-authentication helper used to gate financial actions.

---

## 7. Scheduled Functions

| Function | Schedule | Purpose |
|---|---|---|
| `vendorAutoDeclinePoll` | Every 2 minutes | Warns vendors who haven't responded to a sub-order, then auto-declines if still unanswered |
| `retryFailedNotifications` | Every 15 minutes | Retries failed WhatsApp messages, up to 3 attempts |
| `weeklyCODRechargeCheck` | Weekly | Reminds platform riders with pending COD balances; locks overdue accounts (vendor riders excluded) |
| `serviceProviderSubscriptionCheck` | Daily | Reminds providers nearing their renewal date; expires past-due subscriptions; restores on admin mark-paid |
| `paymentReconciliation` | Daily | Cross-checks M-Pesa transactions against order records and flags mismatches |
| `dailySalesReport` | Daily | Sends a summary report to admin via WhatsApp and saves it to Firestore |
| `cleanupGuestAccounts` | Monthly | Removes anonymous accounts older than 30 days with no orders |
| `cleanupInactiveChats` | Weekly | Archives chat rooms with no activity in the last 90 days |

---

## 8. Reducing Firebase Costs

Firebase costs scale with how much data is read, written, and streamed — so the simplest way to keep costs down is to be deliberate about when and how much data the app touches. A few general principles guide this platform:

- **Read only what you need.** Use pagination instead of loading full collections, and prefer a one-time `get()` for data that's already finished (like a completed order) instead of a live stream.
- **Split large documents into subcollections.** Storing each vendor's portion of an order as its own document means a vendor only ever reads their own data, not the entire order.
- **Avoid unnecessary live listeners.** Real-time streams are great for things that genuinely change often (active orders, chat messages) but are wasteful for static or rarely-changing data.
- **Be careful with Cloud Functions triggers.** A function that fires on every database write should exit early if nothing relevant actually changed, so it doesn't do extra work (and cost) for no reason.
- **Paginate chat and message history.** Load a reasonable batch of recent messages first, and only fetch older ones if the user scrolls back, rather than loading an entire conversation at once.
- **Batch related updates together.** When multiple fields need to change at the same time (like a message and its parent room's "last message" preview), update them in a single transaction instead of several separate writes.
- **Keep widget rebuilds scoped.** On the frontend, structure state management so that only the part of the screen that actually changed re-renders — this reduces wasted reads tied to UI rebuilds.

Following these habits consistently is more effective than optimizing for any single number — small inefficiencies in reads and writes add up quickly at scale.

---

## 9. Customer Ordering Process

The customer browses vendors and products, builds a cart that may include items from multiple vendors, and checks out with a single payment and single delivery fee.

### 9.1 Mixed-Delivery Cart

When a customer's cart contains items from vendors with different delivery modes:

```
Cart has:
  Vendor A — platform delivery
  Vendor B — self-delivery

→ Checkout proceeds normally — one payment, one delivery fee
→ At order creation:
  → Vendor B's sub-order.deliveryMode = 'self_delivery'
  → Vendor A's sub-order.deliveryMode = 'platform'
→ Platform rider assigned for Vendor A's items only
→ Vendor B assigns their own rider separately
→ Customer sees a separate "Vendor B: [Rider Name] is on the way" notification
```

The delivery fee is still calculated using the farthest vendor regardless of delivery mode. Fee attribution (who gets the delivery-fee share) is split per sub-order's delivery mode — see Section 6.8.

---

## 10. Payment Flow

Customers pay via M-Pesa (STK Push) at checkout, or choose Cash on Delivery, where the rider (or self-delivery vendor's rider) collects payment on drop-off and confirms collection in the app.

---

## 11. Notification Flow (WhatsApp)

WhatsApp (via Twilio) is the platform's primary outbound notification channel, used to keep customers, vendors, riders, and service providers informed at each step of an order or booking. Key notification moments include:

- Order placed → vendors notified to accept/decline
- Vendor accepts/declines → customer notified
- Rider assigned (platform delivery) → customer and vendor notified
- Self-delivery rider assigned → customer notified with the rider's name and phone number, scoped to that vendor's items
- Order out for delivery / delivered → customer notified
- Refund issued → customer notified
- Rider COD recharge due / service provider subscription due → reminders sent

**Chat messages are not sent over WhatsApp.** In-app chat (Section 17) is delivered only through Firestore's real-time stream, with an in-app unread badge — this keeps notification costs down and avoids duplicate alerts.

---

## 12. Vendor Workflow

### 12.1 Registration & Activation
Vendor self-registers with business details and required documents, and is gated by admin activation before they can go live.

### 12.2 Managing Products & Location
Vendor manages their own product catalog and pricing within admin-defined categories, and sets a single business location (map pin or manual lat/lng). Products stay hidden from customers until the location is configured.

### 12.3 Handling Incoming Orders
Vendor receives their portion of each order as a sub-order, and accepts, declines, marks preparing/ready as appropriate.

### 12.4 Earnings & Payout
Vendor views their commission deductions and payout status; admin pays out manually via M-Pesa and marks the payout as settled in the dashboard.

### 12.5 Self-Delivery Setup

```
Vendor Web → Settings → Delivery Mode
→ Toggle [Enable Self-Delivery]
  → Sets vendors.selfDeliveryEnabled = true
  → Affects new orders only (existing orders keep their snapshotted deliveryMode)

Vendor Web → My Riders → [Add Rider]
→ Enter rider name, phone, national ID, vehicle type, ID photo
→ registerVendorRider Cloud Function:
  → Creates a Firebase Auth account for the rider (phone login)
  → Sets riderType='vendor' + vendorId claims
  → Creates vendor_riders + riders docs (active immediately — no admin approval needed)
→ Rider can log into Rider Web right away

Vendor Web → My Riders
→ Table of own riders: name, phone, status, active deliveries
→ [Deactivate] / [Reactivate] — vendor controls their own rider access
→ Cannot see other vendors' riders
```

### 12.6 Self-Delivery Order Flow

```
Incoming order with deliveryMode=self_delivery for this vendor:
→ [Accept] order as normal
→ New button appears: [Assign Rider]
→ Dropdown: list of own active riders
→ Select rider → assignSelfDeliveryRider Cloud Function
→ Rider sees the new assignment in their Rider Web (scoped to this vendor's orders)
→ Customer notified via WhatsApp with self-delivery rider details
→ Rider marks pickup + delivery as normal
```

### 12.7 Self-Delivery Earnings

Self-delivery vendors don't pay a rider share to the platform on their own deliveries — instead, they receive a share of the delivery fee directly, calculated the same way a rider's share is calculated (see Section 6.8). Commission on order items still applies as normal regardless of delivery mode.

---

## 13. Rider Workflow

### 13.1 Platform Riders
Self-register, get activated by admin, go online to receive auto-assigned deliveries, update pickup/delivery status, collect COD where applicable, manage their wallet and recharge, and view earnings and payout history.

### 13.2 Vendor Riders

```
Vendor Rider logs into Rider Web (same PWA, restricted view)
→ Only sees orders assigned to them by their own vendor
→ No platform assignment queue visible
→ No COD wallet or pending balance (no platform money handling)
→ Delivery flow same: mark pickup → mark delivered
→ COD orders: rider collects cash and hands it to the vendor — off-platform
→ No wallet recharge, no monthly payout from the platform
→ Account managed entirely by the vendor, who can deactivate it at any time
```

---

## 14. Service Provider Workflow

### 14.1 Registration & Listings
Service provider self-registers, lists services with pricing within admin-defined categories, and is gated by admin activation. Subscription billing and renewal reminders follow the schedule in Section 7.

### 14.2 Customer Contact Flow — In-App Chat

```
Customer → Browse Services → Select provider → View services
→ Each listing shows: name, description, price, duration, provider name
→ [Chat with Provider] button → createChatRoom Cloud Function called
  (roomType = 'customer_provider', relatedEntityId = serviceId)
  → If a room already exists → opens it
  → If new → creates the room; both parties can send messages

Chat Room:
→ Customer types message → sendMessage Cloud Function
→ Provider sees a new message badge on their Service Provider Web
→ Provider replies in the same room
→ Scheduling, negotiation, and payment are all handled directly in chat (off-platform)
→ The platform has no visibility into message content (no moderation currently)

[Contact on WhatsApp] button still available as a secondary option for customers
who prefer WhatsApp.
```

---

## 15. Admin Workflow

### 15.1 Financial Operations
All financial writes (creating a payout record, marking a refund or payout as paid, updating commission) require the admin's login to have been refreshed within the last 5 minutes. Sub-admins cannot access financial operations regardless of token age.

### 15.2 Vendor, Rider & Service Provider Management
Admin reviews and approves or rejects new registrations, and can block an active account if needed.

### 15.3 Reports & Payouts
Admin reviews platform-wide order and earnings reports, and manually pays out vendors and riders via the M-Pesa app, marking each payout as settled once paid.

### 15.4 Refunds
Admin manually processes refunds and records the M-Pesa reference for each.

### 15.5 Sub-Admin Management

```
Admin Web → Sub-Admins → [Create Sub-Admin]
→ Enter name, phone, optional email
→ Select permissions from the permission list (Section 16)
→ createSubAdmin Cloud Function:
  → Firebase Auth account created
  → role=sub_admin claim set
  → sub_admins doc created

Admin Web → Sub-Admins → [Edit Sub-Admin]
→ Update permissions or toggle isActive
→ updateSubAdminPermissions Cloud Function (requires recent re-authentication)
→ Changes take effect immediately

Admin Web → Sub-Admins → [Deactivate]
→ isActive = false → sub-admin can no longer log in or call restricted functions
```

### 15.6 Vendor Self-Delivery Oversight

```
Admin Web → Vendors → [Vendor Name] → Self-Delivery
→ See: selfDeliveryEnabled toggle, list of vendor's own riders
→ Admin can disable self-delivery for a vendor (e.g. compliance issue)
  → Sets vendor.selfDeliveryEnabled = false
  → Affects new orders only
→ Admin cannot manage individual vendor riders — that's the vendor's responsibility
→ Self-delivery orders are flagged separately in reports
```

### 15.7 Chat Oversight

```
Admin Web → Chat (read-only)
→ Can view the list of active chat rooms and message counts
→ Cannot read message content (privacy)
→ [Deactivate Room] → sets chat_rooms.isActive = false
  (use case: reported abuse, spam)
→ No per-message moderation in the current version
```

---

## 16. Sub-Admin System

### 16.1 Permission Model
Permissions are additive — a sub-admin has no access to anything not explicitly listed on their account. Financial operations (payouts, refunds, commission editing) are never grantable to sub-admins; those remain admin-only, regardless of permissions assigned.

### 16.2 Permission Keys

```
vendors.read              — View vendor list, registrations, details
vendors.update             — Activate, reject, block vendors (not commission editing)
vendors.approve            — Activate/reject pending registrations

riders.read                — View rider list, details, earnings summary (read-only)
riders.update               — Activate, reject, block platform riders

service_providers.read      — View provider list, subscription status
service_providers.update    — Activate, reject, block providers (not fee editing)

orders.read                 — View all orders, statuses, sub-orders (read-only)
orders.update                — Cancel orders in exceptional cases

categories.write            — Add, edit, deactivate categories

refunds.read                 — View pending/completed refunds (read-only)

uploads.review                — View and quarantine rejected uploads

chat.read                      — View chat room list and message counts (not content)
chat.moderate                   — Deactivate chat rooms
```

### 16.3 Sub-Admin UI Behaviour
The Sub-Admin Web dashboard only shows tabs and widgets relevant to the account's permissions. There's no payout tab, no refund actions, and no commission fields. Financial figures are masked if the account has no financial-read permission, and re-authentication is never prompted since no financial actions are available to a sub-admin.

### 16.4 Server-Side Enforcement
Every sub-admin-callable function checks the caller's permissions against their live `sub_admins` document before executing — a stale cached login claim cannot be used to bypass this, since the permissions list is always re-checked from the database.

---

## 17. In-App Chat System

### 17.1 Room Types

| Room Type | Participants | Trigger | Related Entity | Restriction |
|---|---|---|---|---|
| `customer_provider` | Customer ↔ Service Provider | Customer taps [Chat with Provider] on a service listing | `serviceId` | Target must be an active, subscribed service provider |
| `vendor_platform_rider` | Vendor ↔ Platform Rider | Vendor opens an order and taps [Chat with Rider] | `orderId` | Target must be the platform rider currently assigned to the order, and must be `riderType == 'platform'` |

> **Vendor-to-vendor-rider chat is not supported.** Vendors manage their own riders directly through the Vendor Riders UI and off-platform contact (WhatsApp, phone). The `vendor_platform_rider` room type only appears when a platform (system) rider is assigned — vendors whose orders are self-delivery only will never see the [Chat with Rider] button.

### 17.2 Chat Room Lifecycle

```
vendor_platform_rider room:
  → Created: when vendor taps [Chat with Rider] (after a rider is assigned)
  → Active: while the order is in progress
  → Auto-archived: 90 days after the last message
  → [Chat with Rider] button only shows when a platform rider is assigned
    AND the vendor has a non-declined sub-order in the order
  → Hidden when no platform rider is assigned, or the vendor is self-delivery only

customer_provider room:
  → Created: when customer taps [Chat with Provider]
  → Active: indefinitely while both parties exist and the provider's subscription is active
  → Auto-archived: 90 days idle
  → Customer can still read archived room history (read-only)
```

### 17.3 Chat Rules
- One room per participant pair per related entity — duplicate creation just returns the existing room.
- Maximum message length: 2,000 characters.
- Sender can soft-delete their own message within 5 minutes (shown as "Message deleted").
- Text only — no file or image sharing in the current version.
- No automated content moderation; admin can only deactivate a whole room.
- Maximum 10 new room creations per user per day.
- Unread badge resets when a room is opened.
- Messages load 30 at a time, with older messages loading on scroll.
- Vendor riders can never be a participant in any chat room — this is enforced both when a room is created and at the database level.

### 17.4 Chat UI Flow

**Vendor ↔ Platform Rider** — from an order with a platform rider assigned, the vendor taps [Chat with Rider] to open (or create) the room and message the rider directly — useful for pickup coordination or delivery issues. This button is hidden entirely for self-delivery orders.

**Customer ↔ Service Provider** — from any service listing, the customer taps [Chat with Provider] to start a conversation that persists across sessions. A [Contact on WhatsApp] button remains available as a backup option.

**Platform Rider Side** — from their assigned deliveries, a platform rider can open [Chat with Vendor] for any vendor with a platform sub-order on that order. Each vendor–rider–order combination gets its own separate room.

---

## 18. Vendor Self-Delivery

### 18.1 Overview
Vendors can opt out of the platform rider pool and use their own riders for delivery — useful for vendors with existing logistics, such as a laundry service with its own van or a produce vendor with a driver.

### 18.2 Eligibility
Available to any active vendor. Enabled via a Settings toggle; admin can override and disable it. No extra fee or subscription required.

### 18.3 Delivery Flow Comparison

| Step | Platform Delivery | Self-Delivery |
|---|---|---|
| Rider pool | Platform riders (admin-activated) | Vendor's own riders (vendor-activated) |
| Assignment trigger | Automatic, once all vendors accept | Manual, by the vendor after accepting |
| COD handling | Platform manages, weekly recharge | Vendor manages directly (off-platform) |
| Rider payout | Platform admin, monthly | Vendor pays their rider directly |
| Rider accountability | Platform admin | Vendor |
| Commission on items | Normal | Normal (no change) |
| Delivery fee to customer | Normal | Normal |
| Delivery fee share | Rider gets 80%, platform keeps 20% | Vendor gets 80%, platform keeps 20% — same split, vendor replaces the rider role |
| In-app chat with delivery person | Yes (`vendor_platform_rider` room) | No — coordinated off-platform |

### 18.4 Mixed-Cart Orders

```
Cart: Vendor A (platform) + Vendor B (self-delivery)
→ Platform rider assigned for Vendor A's items only
→ Vendor B assigns their own rider independently
→ Customer may receive two separate partial deliveries
→ Order marked 'out_for_delivery' once all platform pickups are confirmed
→ Order marked 'delivered' once all sub-orders are picked up and the platform rider
  confirms delivery
```

### 18.5 Vendor Rider Management

```
Vendor Web → My Riders → [Add Rider]
→ Enter name, phone, national ID, vehicle type, ID photo
→ registerVendorRider Cloud Function creates the Firebase Auth account + vendor_riders doc
→ Rider logs into Rider Web (scoped to this vendor's orders only)

Vendor Web → My Riders
→ Table: name, phone, status, active deliveries
→ [Deactivate] / [Reactivate]
→ No admin approval required
```

### 18.6 Limitations
- Vendor riders cannot be shared between vendors.
- Vendor riders are not visible in admin rider management.
- No COD settlement with the platform for vendor riders.
- No platform live location tracking for vendor riders.
- No in-app chat between a vendor and their own riders — use WhatsApp or direct contact instead.
- If a vendor disables self-delivery, orders already in progress continue unchanged.

---

## 19. Money Flow Scenarios

| Scenario | Customer pays | Customer refunded | Vendor receives | Rider receives | Platform keeps |
|---|---|---|---|---|---|
| All vendors accept — platform delivery — delivered | subtotal + deliveryFee | Nothing | subTotal − commission each | 80% of deliveryFeeBaseAmount | Commissions + 20% of deliveryFeeBaseAmount |
| All vendors accept — self-delivery — delivered | subtotal + deliveryFee | Nothing | subTotal − commission, plus 80% of deliveryFeeBaseAmount | Paid by vendor, off-platform | Commissions + 20% of deliveryFeeBaseAmount |
| Mixed delivery — delivered | subtotal + deliveryFee | Nothing | Each vendor: subTotal − commission (self-delivery vendor also gets their 80% share) | Platform rider: 80% of deliveryFeeBaseAmount for platform sub-orders | Commissions + 20% share from each delivery-fee split |
| One vendor declines | subtotal + deliveryFee | Declined vendor's subTotal | Accepting vendors: subTotal − commission | 80% of the original deliveryFeeBaseAmount (platform delivery) | Commissions from accepting vendors |
| All vendors decline | subtotal + deliveryFee | Full (totalAmountCharged − refundedAmount) | Nothing | Nothing | Nothing |
| Customer cancels within window | subtotal + deliveryFee | Full amount | Nothing | Nothing | Nothing |

---

## 20. Order Cancellation Detail

### 20.1 Customer-Initiated Cancellation
Customers can cancel within a short window after placing the order (before vendors start accepting); a full refund is created automatically.

### 20.2 Vendor Decline
If a vendor declines their sub-order, a partial refund for that vendor's sub-total is created, and the remaining sub-orders proceed normally.

### 20.3 All Vendors Decline
If every sub-order in the order is declined or cancelled, the entire order is cancelled and a full refund is issued.

### 20.4 Cancellation Window Closing
Once any vendor accepts their sub-order, the cancellation window closes for the whole order — the customer can no longer self-cancel from that point on.

### 20.5 Self-Delivery Vendor Decline Post-Assignment
If a self-delivery vendor declines after their own rider was already assigned:
- The assigned rider's details are cleared from the sub-order.
- A partial refund is created for the declined sub-total.
- The vendor's rider is notified via WhatsApp that the delivery is cancelled.
- The platform does **not** auto-assign a platform rider to cover a declined self-delivery sub-order.

---

## 21. UI Mockups

### 21.1 Vendor Web — Self-Delivery Rider Assignment

```
┌─────────────────────────────────────────────────┐
│ Order #ORD-1234           ✅ Accepted            │
│ Delivery Mode: Self-Delivery                     │
├─────────────────────────────────────────────────┤
│ Your Items:                                      │
│   1x Fries    KES 200                            │
│   2x Chicken  KES 1,300                          │
│                                                  │
│ Assign Rider:                                    │
│ ┌──────────────────┐                             │
│ │ James M.    ▼   │                             │
│ └──────────────────┘                             │
│ [Assign Rider]                                   │
│                                                  │
│ Rider Assigned: James M.                         │
│ (Coordinate directly with your rider)            │
│ Your delivery fee share: KES 240 (80%)           │
└─────────────────────────────────────────────────┘
```

### 21.2 Vendor Web — Chat with Platform Rider

```
Order #ORD-5678 (Platform delivery, rider assigned)
┌─────────────────────────────────────────────────┐
│ Order #ORD-5678           ✅ Ready for Pickup    │
│ Delivery Mode: Platform                          │
│ Rider: John M. — 07XX-XXXX                       │
├─────────────────────────────────────────────────┤
│ Your Items: 2x Burger  KES 800                   │
│                                                  │
│ [Mark Ready]   [Chat with Rider]                 │
└─────────────────────────────────────────────────┘

→ Tapping [Chat with Rider]:
┌──────────────────────────────────────┐
│ ← Back    John M. (Rider)            │
├──────────────────────────────────────┤
│                                      │
│  Order ready, I'm at the counter     │
│  — just ask for Mama Njeri's  10:22  │
│                                      │
│     On my way, 5 min away    10:23   │
│                                      │
├──────────────────────────────────────┤
│  [Type a message...]      [Send]     │
└──────────────────────────────────────┘

Self-delivery order — NO [Chat with Rider] button:
┌─────────────────────────────────────────────────┐
│ Order #ORD-9999           ✅ Ready for Pickup    │
│ Delivery Mode: Self-Delivery                     │
│ Rider: James M. (your rider)                     │
├─────────────────────────────────────────────────┤
│ Your Items: 1x Salad  KES 350                    │
│                                                  │
│ [Mark Ready]                                     │
│ (Contact your rider directly)                    │
└─────────────────────────────────────────────────┘
```

### 21.3 Customer Web — Chat with Service Provider

```
┌────────────────────────────────────┐
│ ← Back    CleanPro Laundry         │
├────────────────────────────────────┤
│  Hi, turnaround time for 2kg?10:12 │
│     Same day if before 9am   10:14 │
│  Do you service Westlands?   10:15 │
│     Yes, Tue & Fri pickup    10:16 │
├────────────────────────────────────┤
│  [Type a message...]      [Send]   │
└────────────────────────────────────┘
```

### 21.4 Service Provider Web — Chat Inbox

```
┌──────────────────────────────────────┐
│ Messages                             │
├──────────────────────────────────────┤
│ Jane W.          "Do you service..." │
│ Deep Clean Svc   2 new  ·  10:16am  │
├──────────────────────────────────────┤
│ John M.          "What's the price?" │
│ Ironing          ·  Yesterday        │
└──────────────────────────────────────┘
```

### 21.5 Admin Web — Sub-Admin Management

```
┌────────────────────────────────────────────────────────┐
│ Sub-Admins                          [Create Sub-Admin]  │
├────────────────────────────────────────────────────────┤
│ Name       Phone       Permissions          Status      │
│ Alice K.   07XX-XXXX   vendors, orders      ✅ Active   │
│                         [Edit]  [Deactivate]            │
│ Brian O.   07XX-XXXX   riders, uploads      ✅ Active   │
│                         [Edit]  [Deactivate]            │
└────────────────────────────────────────────────────────┘
```

---

## 22. Implementation Phases

### Phase 1 — Foundation
Project setup, Firebase configuration, core auth/role system, base Firestore collections including `sub_admins`, `chat_rooms` + `messages`, `vendor_riders`, and the `riderType` field on `riders`.

### Phase 2 — Vendor & Cart Core
Vendor registration and product management, multi-vendor cart, `selfDeliveryEnabled` on the vendor doc, `deliveryMode` snapshotted onto sub-orders, and the `hasSelfDeliverySubOrders` / `hasPlatformSubOrders` flags on the order doc.

### Phase 3 — Order Flow & Rider Assignment
Order creation and sub-order lifecycle, `autoAssignRider` (skipping self-delivery sub-orders), `assignSelfDeliveryRider`, vendor rider registration and management UI, and the self-delivery Settings toggle.

### Phase 4 — COD, Cancellations & Refunds
Cash-on-delivery flow, cancellation windows, refund creation for declines and cancellations, and the self-delivery post-decline edge case.

### Phase 5 — Chat, Sub-Admin, Service Providers & Notifications
- In-app chat: Firestore setup, `createChatRoom` (with full `vendor_platform_rider` validation), `sendMessage`, `markMessagesRead`, `deleteMessage`, customer and service provider chat UI, vendor [Chat with Rider] button logic, rider [Chat with Vendor] UI, and the weekly inactive-chat cleanup job.
- Sub-admin: `createSubAdmin`, `updateSubAdminPermissions`, and permission-scoped Admin/Sub-Admin Web UI.
- Service provider listings flow and the WhatsApp notification wrapper with retry support.

### Phase 6 — Polish & Hardening
Chat room creation rate limiting, confirming vendor riders can never be targeted in a `vendor_platform_rider` chat room, and confirming the chat tab is hidden for vendor riders.

### Phase 7 — Testing & Deployment
End-to-end testing across all roles and flows, including: chat room validation and deduplication, the self-delivery fee split calculation, sub-admin permission enforcement across all permission keys, and a final review of the money flow scenarios in Section 19 before go-live.
