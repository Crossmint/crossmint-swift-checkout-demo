# Crossmint Swift Checkout Example

Example iOS app demonstrating the `CrossmintEmbeddedCheckout` component from the [CrossmintCheckout](https://github.com/Crossmint/crossmint-checkout-swift) Swift package.

## Installation

This example uses [`crossmint-checkout-swift`](https://github.com/Crossmint/crossmint-checkout-swift) — a standalone, lightweight Swift Package with zero external dependencies.

Add it to your project via SPM:

```swift
dependencies: [
    .package(url: "https://github.com/Crossmint/crossmint-checkout-swift", from: "1.0.0")
]
```

Or in Xcode: **File > Add Package Dependencies** → paste `https://github.com/Crossmint/crossmint-checkout-swift`

## Integration Flow

### 1. Create Order (Server-side)

First, create an order on your server using the Crossmint API:

**Important:** Order creation must be done server-side to keep your API key secure.

```bash
# Production
curl --location 'https://www.crossmint.com/api/2022-06-09/orders' \
--header 'x-api-key: YOUR_API_KEY' \
--header 'Content-Type: application/json' \
--data-raw '{
    "recipient": {
        "walletAddress": "WALLET_ADDRESS"
    },
    "payment": {
        "receiptEmail": "user@example.com",
        "method": "card"
    },
    "lineItems": {
        "tokenLocator": "chain:token",
        "executionParameters": {
            "mode": "exact-in",
            "amount": "1"
        }
    }
}'

# Staging
curl --location 'https://staging.crossmint.com/api/2022-06-09/orders' \
--header 'x-api-key: YOUR_API_KEY' \
--header 'Content-Type: application/json' \
--data-raw '{...}'
```

See full documentation: [Create Order API](https://docs.crossmint.com/api-reference/headless/create-order) and [Payment Methods](https://docs.crossmint.com/payments/introduction)

The response will include:

- `orderId` - Unique identifier for the order
- `clientSecret` - Token scoped to this order for client-side operations

### 2. Use the Component (Client-side)

Pass the `orderId`, `clientSecret`, and optional configuration to the component:

```swift
import SwiftUI
import CrossmintCheckout

CrossmintEmbeddedCheckout(
    orderId: "your-order-id",
    clientSecret: "your-client-secret",
    payment: CheckoutPayment(
        crypto: CheckoutCryptoPayment(enabled: false),
        fiat: CheckoutFiatPayment(
            enabled: true,
            allowedMethods: CheckoutAllowedMethods(
                googlePay: false,
                applePay: true,
                card: false
            )
        )
    ),
    appearance: CheckoutAppearance(
        rules: CheckoutAppearanceRules(
            destinationInput: CheckoutDestinationInputRule(display: "hidden"),
            receiptEmailInput: CheckoutReceiptEmailInputRule(display: "hidden")
        )
    ),
    environment: .production  // or .staging
)
```

### 3. Track Order Status (Server-side)

Monitor the order as it progresses through payment and delivery. Use webhooks for real-time updates or polling as a fallback.

#### Option A: Webhooks (Recommended)

Set up webhooks to receive real-time updates as the order progresses through payment and delivery.

**Setup:**

1. Create a `POST` endpoint on your server (e.g., `/webhooks/crossmint`)
2. Configure webhook in [Crossmint Console](https://www.crossmint.com/console/webhooks)
3. Save the signing secret for verification

**Your endpoint will receive:**

```json
{
  "type": "orders.payment.succeeded",
  "payload": {
    "orderId": "...",
    "payment": {
      "status": "completed",
      "received": {
        "amount": "100.00",
        "currency": "usd"
      }
    }
  }
}
```

**Key Events:**

- `orders.quote.created` - Order created
- `orders.payment.succeeded` - Payment confirmed
- `orders.delivery.completed` - Tokens delivered (includes `txId`)
- `orders.payment.failed` - Payment failed

**Important:** Always respond with HTTP 200 status to acknowledge receipt.

See full documentation: [Webhooks Guide](https://docs.crossmint.com/introduction/platform/webhooks/overview)

#### Option B: Polling (Fallback)

Poll the order status if webhooks aren't feasible. Be mindful of rate limits.

```bash
# Production
curl --location 'https://www.crossmint.com/api/2022-06-09/orders/{orderId}' \
--header 'x-api-key: YOUR_API_KEY'

# Staging
curl --location 'https://staging.crossmint.com/api/2022-06-09/orders/{orderId}' \
--header 'x-api-key: YOUR_API_KEY'
```

**Response includes order phase:**

- `quote` - Order created, awaiting payment
- `payment` - Processing payment
- `delivery` - Payment complete, delivering tokens
- `completed` - Tokens delivered successfully

**Polling Guidelines:**

- Check `order.phase === "completed"` for success
- Check `order.payment.failureReason` for payment errors
- Transaction ID available at `order.lineItems[0].delivery.txId` when completed

See full documentation: [Get Order API](https://docs.crossmint.com/api-reference/headless/get-order)

## Available Properties

- `orderId` - Order identifier from create order API
- `clientSecret` - Client secret from create order API
- `payment` - Payment method configuration (crypto, fiat, allowed methods)
- `appearance` - UI customization (variables, rules for inputs/buttons/tabs)
- `environment` - `.staging` or `.production`

### Not Yet Implemented

- `lineItems` - Client-side line items configuration
- `recipient` - Recipient information

## Example

See `ContentView.swift` for a complete working example.

## Resources

- [CrossmintCheckout Swift Package](https://github.com/Crossmint/crossmint-checkout-swift)
- [Crossmint Documentation](https://docs.crossmint.com)
- [Create Order API](https://docs.crossmint.com/api-reference/headless/create-order)
- [Get Order API](https://docs.crossmint.com/api-reference/headless/get-order)
