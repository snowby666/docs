# PayPal Webhook Setup Guide

## Getting the Webhook ID

### Method 1: PayPal Developer Dashboard (Recommended)

1. **Log in to PayPal Developer Dashboard**
   - Go to https://developer.paypal.com/
   - Sign in with your PayPal business account

2. **Navigate to Your App**
   - Click on "My Apps & Credentials"
   - Select your app (or create a new one)

3. **Create a Webhook**
   - Scroll down to the "Webhooks" section
   - Click "Add Webhook"
   - Enter your webhook URL:
     - **Production**: `https://api.electronhub.ai/auth/paypal/webhook`
     - **Sandbox**: `http://127.0.0.1:7000/auth/paypal/webhook` (for local testing)
   - Select the following events:
     - `PAYMENT.CAPTURE.COMPLETED` ✅ (Required)
     - `PAYMENT.CAPTURE.DENIED` ✅ (Required)
     - `PAYMENT.CAPTURE.REFUNDED` (Optional)
     - `PAYMENT.CAPTURE.REVERSED` (Optional)

4. **Copy the Webhook ID**
   - After creating the webhook, you'll see a "Webhook ID" field
   - Copy this ID (it looks like: `1J234567890ABCDEFGHIJ`)

5. **Add to Configuration**
   - Add the webhook_id to `values/secrets.json`:
   ```json
   {
     "paypal": {
       "client_id": "your_paypal_client_id",
       "client_secret": "your_paypal_client_secret",
       "webhook_id": "YOUR_WEBHOOK_ID_HERE"
     }
   }
   ```

### Method 2: List Webhooks via API

You can also retrieve webhook IDs programmatically using the PayPal API:

```python
import httpx
import base64

# Get access token
auth_string = f"{PAYPAL_CLIENT_ID}:{PAYPAL_CLIENT_SECRET}"
auth_b64 = base64.b64encode(auth_string.encode('ascii')).decode('ascii')

async with httpx.AsyncClient() as client:
    # Get access token
    token_response = await client.post(
        f"{PAYPAL_BASE_URL}/v1/oauth2/token",
        headers={
            "Authorization": f"Basic {auth_b64}",
            "Content-Type": "application/x-www-form-urlencoded"
        },
        data={"grant_type": "client_credentials"}
    )
    access_token = token_response.json()["access_token"]
    
    # List webhooks
    webhooks_response = await client.get(
        f"{PAYPAL_BASE_URL}/v1/notifications/webhooks",
        headers={
            "Authorization": f"Bearer {access_token}",
            "Content-Type": "application/json"
        }
    )
    webhooks = webhooks_response.json()
    print(webhooks)  # Find your webhook_id in the response
```

## Testing Webhooks

### Using PayPal Webhook Simulator

1. Go to your app in PayPal Developer Dashboard
2. Click on "Webhooks" section
3. Click on your webhook
4. Use the "Send test event" button to test your webhook endpoint

### Local Testing with ngrok

For local development, use ngrok to expose your local server:

```bash
# Install ngrok
# Then run:
ngrok http 7000

# Use the ngrok URL in PayPal webhook configuration:
# https://your-ngrok-url.ngrok.io/auth/paypal/webhook
```

## Important Notes

- **Webhook ID is different for Sandbox and Production**
  - You need separate webhook IDs for sandbox and production environments
  - Update your config based on which environment you're using

- **Webhook URL must be HTTPS in production**
  - PayPal requires HTTPS for production webhooks
  - Local development can use HTTP

- **Webhook verification is critical**
  - Always verify webhook signatures to prevent fraud
  - The system will log warnings if webhook_id is not configured

## Troubleshooting

### Webhook not receiving events?
1. Check that the webhook URL is accessible (not behind firewall)
2. Verify the webhook is active in PayPal dashboard
3. Check server logs for webhook verification errors
4. Ensure webhook_id is correctly set in secrets.json

### Webhook verification failing?
1. Verify webhook_id matches the one in PayPal dashboard
2. Check that PayPal credentials (client_id, client_secret) are correct
3. Ensure you're using the correct PayPal base URL (sandbox vs production)

