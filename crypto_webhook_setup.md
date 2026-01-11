# Crypto Payment (NOWPayments) Webhook Setup Guide

## Overview

Electron Hub uses NOWPayments IPN (Instant Payment Notifications) webhooks to receive real-time payment status updates for cryptocurrency transactions. This ensures reliable payment processing and automatic credit allocation.

## Webhook Configuration

### Production Webhook URL

```
https://api.electronhub.ai/webhook/nowpayments
```

### Development/Testing Webhook URL

```
http://127.0.0.1:7000/webhook/nowpayments
```

## Setting Up Webhooks in NOWPayments Dashboard

### Step 1: Access NOWPayments Dashboard

1. **Log in to NOWPayments Dashboard**
   - Go to https://nowpayments.io/
   - Sign in with your account

2. **Navigate to Settings**
   - Go to your account settings
   - Find the "IPN Settings" or "Webhook Settings" section

### Step 2: Configure IPN URL

1. **Enter Webhook URL**
   - **Production**: `https://api.electronhub.ai/webhook/nowpayments`
   - **Development**: `http://127.0.0.1:7000/webhook/nowpayments` (for local testing)

2. **Set IPN Secret**
   - Generate a secure secret key (or use the one in your configuration)
   - This secret is used for HMAC-SHA512 signature verification
   - Add it to `values/secrets.json`:
   ```json
   {
     "nowpayments": {
       "api_key": "YOUR_API_KEY",
       "ipn_secret": "YOUR_IPN_SECRET",
       "public_key": "YOUR_PUBLIC_KEY"
     }
   }
   ```

### Step 3: Enable IPN Notifications

1. **Enable IPN** in your NOWPayments dashboard
2. **Select Events** (if applicable):
   - Payment status changes
   - Payment confirmations
   - Payment failures

## Webhook Security

### Signature Verification

All webhook requests are verified using HMAC-SHA512 signatures:

```python
# The system automatically verifies webhook signatures
# Using the IPN secret from your configuration
```

<Warning>
  <strong>Security:</strong> Always verify webhook signatures to prevent fraud and replay attacks. The system uses constant-time comparison to prevent timing attacks.
</Warning>

## Webhook Events

NOWPayments sends webhook notifications for the following payment statuses:

- `waiting` - Payment is waiting to be sent
- `confirming` - Payment is being confirmed on the blockchain
- `confirmed` - Payment has been confirmed
- `sending` - Payment is being sent
- `partially_paid` - Payment is partially paid
- `finished` - Payment completed successfully
- `failed` - Payment failed
- `expired` - Payment expired
- `cancelled` - Payment was cancelled
- `wrong_asset_confirmed` - Wrong cryptocurrency was sent

## Testing Webhooks

### Using NOWPayments Test Mode

1. Enable test mode in NOWPayments dashboard
2. Create a test payment
3. Check server logs for webhook events

### Local Testing with ngrok

For local development, use ngrok to expose your local server:

```bash
# Install ngrok
# Then run:
ngrok http 7000

# Use the ngrok URL in NOWPayments webhook configuration:
# https://your-ngrok-url.ngrok.io/webhook/nowpayments
```

### Manual Testing

You can test webhook endpoints using curl:

```bash
# Example webhook payload (for testing)
curl -X POST http://127.0.0.1:7000/webhook/nowpayments \
  -H "Content-Type: application/json" \
  -H "x-nowpayments-sig: YOUR_SIGNATURE" \
  -d '{
    "payment_id": "test_payment_123",
    "payment_status": "finished",
    "order_id": "test_order",
    "price_amount": 100.00,
    "price_currency": "usd",
    "pay_currency": "btc",
    "actually_paid": 0.0025
  }'
```

## Important Notes

- **Webhook URL must be HTTPS in production**
  - NOWPayments requires HTTPS for production webhooks
  - Local development can use HTTP

- **IPN Secret is critical**
  - Always keep your IPN secret secure
  - Never commit secrets to version control
  - Use environment variables or secure configuration files

- **Webhook verification is automatic**
  - The system automatically verifies all incoming webhooks
  - Invalid signatures are rejected and logged

- **Automatic Reconciliation**
  - The system runs reconciliation checks every 5 minutes
  - Stuck orders are automatically checked and updated
  - Orders older than 24 hours without confirmation are automatically deleted

## Troubleshooting

### Webhook not receiving events?

1. **Check webhook URL accessibility**
   - Ensure the URL is publicly accessible (not behind firewall)
   - Verify HTTPS certificate is valid (for production)

2. **Verify IPN is enabled**
   - Check NOWPayments dashboard to ensure IPN is enabled
   - Verify the webhook URL is correctly configured

3. **Check server logs**
   - Look for webhook verification errors
   - Check for network connectivity issues

4. **Verify IPN secret**
   - Ensure `ipn_secret` in `values/secrets.json` matches NOWPayments dashboard
   - Check for typos or encoding issues

### Webhook verification failing?

1. **Verify IPN secret matches**
   - Check that the secret in your config matches NOWPayments dashboard
   - Ensure there are no extra spaces or characters

2. **Check signature format**
   - NOWPayments uses HMAC-SHA512 signatures
   - Verify the signature header name is correct

3. **Review webhook payload**
   - Check that the payload format matches NOWPayments documentation
   - Verify JSON encoding is correct

### Payment not confirming?

1. **Check blockchain status**
   - Verify the transaction on the blockchain explorer
   - Check if the transaction has enough confirmations

2. **Review order status**
   - Check the order status in the dashboard
   - Look for any error messages or warnings

3. **Check reconciliation logs**
   - The system automatically reconciles stuck orders every 5 minutes
   - Review logs for reconciliation attempts

### Order expired?

1. **Orders expire after 24 hours** if payment is not confirmed
2. **Create a new order** if your payment didn't confirm in time
3. **Check payment status** on the blockchain before creating a new order

## Monitoring

### Discord Webhook Logging

All webhook requests are logged to a Discord webhook for monitoring:

- Payment status changes
- Webhook verification results
- Error conditions

This helps track payment processing in real-time.

### Server Logs

Check server logs for:
- Webhook reception and processing
- Payment status updates
- Reconciliation activities
- Error conditions

## Best Practices

1. **Always verify webhook signatures** - Never trust unsigned webhooks
2. **Use HTTPS in production** - Required for security
3. **Monitor webhook logs** - Track payment processing
4. **Set up alerts** - Get notified of payment issues
5. **Test thoroughly** - Use test mode before going live
6. **Keep secrets secure** - Never expose IPN secrets

## Support

For issues with:
- **NOWPayments API**: Contact NOWPayments support
- **Electron Hub Integration**: Check server logs and webhook configuration
- **Payment Processing**: Review order status and blockchain confirmations

