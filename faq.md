# Electron Hub Billing & Credits FAQ

Welcome to the Electron Hub Billing FAQ! This guide will help you understand everything about credits, payments, subscriptions, and how our billing system works.

## Table of Contents

1. [Getting Started - Basic Concepts](#getting-started---basic-concepts)
2. [Understanding Credits](#understanding-credits)
3. [Payment Methods](#payment-methods)
4. [Subscription Tiers](#subscription-tiers)
5. [Model Types & Access](#model-types--access)
6. [Credit Management](#credit-management)
7. [Advanced Topics](#advanced-topics)
8. [Troubleshooting](#troubleshooting)
9. [Privacy & Security](#privacy--security)
10. [Helpful Links](#helpful-links)

---

## Getting Started - Basic Concepts

### What is Electron Hub?

Electron Hub is an AI API platform that provides access to various AI models (like GPT-4, Claude, and many others) through a unified API interface. You can use these models for chat completions, image generation, audio processing, and more.

### What are Credits?

**Credits** are the currency used in Electron Hub to pay for AI model usage. Think of them like tokens in an arcade - you need credits to use the AI services. Different models and services cost different amounts of credits.

### What are Tokens?

**Tokens** are the units that AI models use to process text. When you send a message to an AI model:
- **Input tokens** = the text you send (your prompt/question)
- **Output tokens** = the text the AI generates (the response)

For example, if you ask "What is the weather?" and the AI responds "The weather is sunny today", both your question and the response are counted as tokens. The cost of using a model depends on how many tokens you use.

**Key Points:**
- Tokens are priced per million tokens (MTok)
- Different models have different token prices
- Input tokens and output tokens are priced separately (output is usually more expensive)
- Some models support **prompt caching**, which can reduce costs for repeated content

---

## Understanding Credits

### What are Daily Credits?

**Daily Credits** are credits that are automatically added to your account every day at **21:00 UTC** (9:00 PM UTC). The amount you receive depends on your subscription tier:

| Tier | Daily Credits |
|------|---------------|
| Free | 0.25 credits |
| Starter | 2 credits |
| Plus | 5 credits |
| Core | 10 credits |
| Pro | 15 credits |
| Business | 30 credits |
| Enterprise | 69+ credits (scales with custom payment) |

**Important Notes:**
- Daily credits reset at **21:00 UTC** every day
- Unused daily credits do **not** carry over to the next day
- Your balance is automatically set to your daily credits amount at reset time
- Daily credits are separate from permanent credits (see below)

### What are Permanent Credits?

**Permanent Credits** are credits you purchase with real money. Unlike daily credits, these credits:
- **Never expire** - they stay in your account forever
- Can be purchased through PayPal
- Can be used alongside daily credits
- Have volume discounts (the more you spend, the more bonus credits you get)

**Pricing:**
- Base rate: **$1 USD = 3 credits**
- Volume discount: Every $500 spent gives you a 10% bonus
- Maximum discount: 30% bonus (reached at $1,500+ total spending)

**Example:**
- First purchase: $100 = 300 credits (no bonus yet)
- After $500 total: $100 purchase = 330 credits (10% bonus)
- After $1,500 total: $100 purchase = 390 credits (30% bonus, maximum)

### What are Flex Credits?

**Flex Credits** (also called "Flex Wallet") are a savings feature that lets you save unused daily credits for later use.

**How it works:**
- You can save up to **20% of your daily credits** per day to your flex wallet
- The total cap is **100% of your daily credits** (e.g., if you get 15 credits/day, your flex wallet can hold up to 15 credits)
- You can only save once every 24 hours
- Flex credits are used automatically when your regular balance runs out (if flex credits are enabled)

**Example:**
- You have a Pro subscription (15 credits/day)
- Daily save limit: 3 credits (20% of 15)
- Maximum flex wallet: 15 credits (100% of daily credits)
- If you save 3 credits every day for 5 days, you'll have 15 credits saved (the maximum)

**When to use Flex Credits:**
- Enable flex credits if you want to build up a buffer for heavy usage days
- Useful if you don't use all your daily credits every day
- Helps prevent losing unused credits at daily reset

### What are Neutrinos?

**Neutrinos** are special free usage requests for **free models** (models that end with `:free` suffix). 

**Key Points:**
- All users get **10 Neutrinos per day** (resets at 21:00 UTC)
- Neutrinos are used instead of credits for free models
- You can earn more Neutrinos by watching ads (if available)
- **Important**: Neutrinos are charged for **all requests** (failed, cancelled, and successful) - unlike regular credits which are only charged for successful/cancelled requests

---

## Payment Methods

### What payment methods do you accept?

Currently, Electron Hub accepts:
- **PayPal** - for purchasing permanent credits

**Payment Limits:**
- Maximum 20 orders per day
- Maximum $5,000 USD per month in purchases
- Minimum purchase amount may apply (check the dashboard)

### How do I purchase permanent credits?

1. Go to the [Electron Hub Dashboard](https://playground.electronhub.ai/console)
2. Navigate to the credits/payment section
3. Select the amount you want to purchase
4. Complete the payment through PayPal
5. Credits are added to your account immediately after payment

### Are there any purchase limits?

Yes, there are limits to prevent abuse:
- **Daily limit**: 20 orders per day
- **Monthly limit**: $5,000 USD per month
- These limits reset automatically

### Can I get a refund?

**Refund Policy:**

- **Permanent Credits Purchases**: Refunds are available within 30 days of purchase if you haven't used the credits. Once credits have been used, refunds are not available except in cases of technical errors or service issues.
- **Subscription Refunds**: Subscriptions are managed through Patreon. Refund requests for subscriptions should be made through Patreon's refund system.
- **Failed Requests**: Credits are **not charged** for failed requests (only successful and cancelled requests are charged). If you believe you were incorrectly charged for a failed request, contact support with your request ID.
- **Technical Issues**: If you experience technical issues that prevent you from using the service, contact support for assistance. Refunds may be provided on a case-by-case basis.

To request a refund, contact support through the dashboard or email with:
- Your order ID or transaction ID
- Reason for refund request
- Date of purchase

For complete refund terms, please review our [Terms of Service](https://tos.electronhub.ai).

---

## Subscription Tiers

### What subscription tiers are available?

Electron Hub offers several subscription tiers through Patreon:

| Tier | Name | Daily Credits | Max Bots | Studio Credits |
|------|------|---------------|----------|----------------|
| 0 | Free | 0.25 | 5 | 5 |
| 1 | Starter | 2 | 20 | 20 |
| 5 | Plus | 5 | 30 | 50 |
| 6 | Core | 10 | 50 | 75 |
| 2 | Pro | 15 | 100 | 150 |
| 3 | Business | 30 | 150 | 300 |
| 4 | Enterprise | 69+ (custom) | 200 | 500 |

**Benefits of Higher Tiers:**
- More daily credits
- Access to premium/ultimate models
- Higher rate limits
- More bot creation slots
- More studio credits (for AI bot creation)

### Enterprise Tier (Custom Plan)

The **Enterprise tier (Tier 4)** is a custom plan that scales with your Patreon payment amount:

**Base Benefits:**
- Base daily credits: **69 credits**
- Base monthly cost: $79.20 (old plan) or $199.20 (new plan, 80% of $249)
- 200 max bots
- 500 studio credits

**Custom Scaling:**
If you pay more than the base cost on Patreon, your daily credits scale automatically:
- **Extra credits formula**: For every dollar above the base cost, you get **0.5 additional credits**
- **Example**: If you pay $300/month instead of $249:
  - Extra paid: $300 - $249 = $51
  - Extra credits: $51 × 0.5 = 25.5 credits
  - **Total daily credits**: 69 + 25.5 = **94.5 credits/day**

**How it works:**
- Your Patreon payment amount is automatically detected
- Daily credits are calculated based on your actual payment
- Credits update automatically when your payment changes
- Works for both monthly and yearly subscriptions (yearly payments are divided by 12 for monthly calculation)

This allows Enterprise users to customize their plan based on their needs and budget.

### How do I subscribe?

Subscriptions are managed through **Patreon**. To subscribe:
1. **First, purchase your subscription on Patreon** - Visit the Electron Hub Patreon page and select your desired tier
2. Go to the [Electron Hub Dashboard](https://playground.electronhub.ai/console) → **Billing** section
3. Link your Patreon account to your Electron Hub account
4. Your subscription benefits will be applied automatically

**Important**: You must purchase the subscription on Patreon first before linking your account. If you link your account without an active Patreon subscription, you won't receive any benefits.

### When do subscriptions renew?

Subscriptions renew based on your Patreon subscription cycle (usually monthly). Your subscription expiry date is shown in your dashboard.

### What happens if my subscription expires?

If your subscription expires:
- You'll be downgraded to the Free tier
- You'll receive 0.25 daily credits
- You'll lose access to premium/ultimate models
- Your permanent credits remain (they never expire)
- Your flex credits remain (if you have any)

---

## Model Types & Access

### What are the different model types?

Electron Hub has three main model categories:

#### 1. Free Models (`:free` suffix)
- Models that end with `:free` (e.g., `gpt-3.5-turbo:free`)
- Use **Neutrinos** instead of credits
- No subscription required
- **Uptime**: We don't guarantee 100% uptime at peak times (we host these ourselves and need to ensure fair usage)
- Best for: Testing, light usage, learning

#### 2. Freemium Models
- Models that don't require a subscription
- **Do deduct credits** from your balance
- Available to all users (Free tier and above)
- Better uptime than free models
- Best for: Regular usage without subscription commitment

#### 3. Premium/Ultimate Models
- Models that **require a subscription** (Pro tier or higher)
- Examples: `gpt-4.5-preview`, `o1`, `o3-pro`, `gpt-4`
- **Best uptime** - routed to multiple providers for reliability
- Also deduct credits from your balance
- Best for: Production applications, critical workloads

**Premium Model Examples:**
- `gpt-4.5-preview`
- `gpt-4.5-preview-2025-02-27`
- `gpt-4`
- `o1`, `o1-low`, `o1-high`
- `o3-pro`, `o3-pro-low`, `o3-pro-high`
- `o3-deep-research`

### Why don't free models guarantee 100% uptime?

Free models are hosted by Electron Hub directly, and during peak usage times, we need to ensure fair distribution of resources among all users. For guaranteed uptime and better performance, we recommend:
- Using freemium models (deduct credits but better uptime)
- Subscribing to a premium tier to access premium models (routed to multiple providers)

### How do I know which models I can access?

- **Free tier**: Free models (with Neutrinos) and freemium models (with credits)
- **Starter/Plus/Core tiers**: All freemium models + some premium models
- **Pro/Business/Enterprise tiers**: All models including premium/ultimate models

Check the model list in the dashboard or API documentation to see which models require which tier.

---

## Credit Management

### How are credits deducted?

Credits are deducted in this priority order:
1. **Permanent credits** (if enabled and within monthly limit)
2. **Regular balance** (daily credits)
3. **Flex credits** (if enabled)

**Important Charging Rules:**
- **Regular Credits**: Only charged for **successful** and **cancelled** requests. **Failed requests are NOT charged.**
- **Neutrinos**: Charged for **all requests** (failed, cancelled, and successful) - this is different from regular credits.

**Example:**
- You have 5 permanent credits, 10 regular credits, and 3 flex credits
- You make a request costing 15 credits
- System uses: 5 permanent + 10 regular + 3 flex = 18 credits total (you have enough!)

**Failed Request Example:**
- You make a request that fails (e.g., model unavailable, timeout)
- **Regular credits**: NOT charged (you keep your credits)
- **Neutrinos**: Still charged (if using a `:free` model)

### When do credits reset?

- **Daily credits**: Reset every day at **21:00 UTC** (9:00 PM UTC)
- **Neutrinos**: Reset every day at **21:00 UTC**
- **Permanent credits**: Never reset (they're permanent!)
- **Flex credits**: Never reset (but you can only save once per 24 hours)
- **Monthly spending limits**: Reset on the 1st of each month (for permanent credits monthly limits)

### What happens to unused daily credits?

Unused daily credits are **lost** at the daily reset (21:00 UTC). This is why Flex Credits are useful - you can save up to 20% of your daily credits to your flex wallet to use later.

### Can I transfer credits to another account?

No, credits cannot be transferred between accounts. Each account's credits are tied to that specific account.

### How do I check my credit balance?

You can check your balance:
- In the [Electron Hub Dashboard](https://playground.electronhub.ai/console)
- Through the API by checking your user information
- Balance includes: regular credits, flex credits (if enabled), and permanent credits (if enabled)

---

## Advanced Topics

### Prompt Caching & Token Pricing

Some models support **prompt caching**, which can reduce costs when you reuse the same content.

**How it works:**
- You can mark parts of your prompt to be cached
- Cached content is stored for 5 minutes (or 1 hour with paid option)
- When you reuse cached content, you pay less (cache reads are cheaper than regular input)

**Token Pricing Structure:**
- **Base input tokens**: Regular price per million tokens
- **Cache write (5 min)**: 1.25x base input price (25% more expensive)
- **Cache write (1 hour)**: 2x base input price (100% more expensive)
- **Cache read**: 0.1x base input price (90% cheaper!)
- **Output tokens**: Usually more expensive than input tokens

**When to use caching:**
- When you have large, repeated context (like a long document)
- When you're asking multiple questions about the same content
- When you want to reduce costs for repeated prompts

### Permanent Credits Monthly Limits

You can set a **monthly spending limit** for permanent credits to control costs:
- **0 = Unlimited** (no limit)
- **Any positive number** = Maximum amount you can spend per month
- **Maximum limit**: $100,000 USD per month
- Limits reset on the 1st of each month

**Use cases:**
- Budget control for teams
- Preventing accidental overspending
- Managing costs for production applications

### Rate Limit Scaling

Your rate limits can scale based on your **total purchased credits** (permanent credits spending). The scaling applies a multiplier to your base tier rate limit:

| Total Spent | Scale Tier | Multiplier | Example (Pro Tier) |
|-------------|------------|------------|-------------------|
| $0-$99 | Base | 1x (no scaling) | 30 requests/min |
| $100+ | Tier 1 | 2x | 60 requests/min (30 × 2) |
| $250+ | Tier 2 | 3x | 90 requests/min (30 × 3) |
| $500+ | Tier 3 | 5x | 150 requests/min (30 × 5) |
| $1,000+ | Tier 4 | 7x | 210 requests/min (30 × 7) |
| $2,500+ | Tier 5 | 10x | 300 requests/min (30 × 10) |

**How it works:**
- Your base rate limit depends on your subscription tier (e.g., Pro tier = 30 requests/min)
- The multiplier is applied to your base limit based on your total permanent credits spending
- **Example**: If you're on Pro tier (30 req/min) and have spent $500+ on permanent credits, your rate limit becomes 150 requests/min (30 × 5)

**Base Rate Limits by Tier:**
- Free: 5 requests/min
- Starter: 10 requests/min
- Plus: 15 requests/min
- Core: 20 requests/min
- Pro: 30 requests/min
- Business: 60 requests/min
- Enterprise: 100 requests/min

**Note:** Rate limit scaling applies to all tiers based on your total permanent credits spending. The multiplier is cumulative - you get the highest multiplier you qualify for.

### Low Balance Alerts

You can enable **low balance alerts** for permanent credits:
- Set a threshold (e.g., 10 credits)
- Receive email alerts when your permanent credits balance drops below the threshold
- Alerts are sent at most once every 24 hours (to prevent spam)

---

## Troubleshooting

### I ran out of credits. What can I do?

1. **Wait for daily reset** (21:00 UTC) to get your daily credits
2. **Enable flex credits** if you have any saved
3. **Enable permanent credits** if you have purchased credits
4. **Purchase more permanent credits** through PayPal
5. **Upgrade your subscription** for more daily credits

### Why can't I use a premium model?

Premium models require a **Pro tier or higher** subscription. Check your subscription tier in the dashboard. If you're on a lower tier, you'll need to upgrade through Patreon.

### My credits didn't reset at 21:00 UTC

Credits reset automatically at 21:00 UTC. If you don't see your credits:
1. Refresh your dashboard
2. Check that it's actually past 21:00 UTC
3. Contact support if the issue persists

### I purchased credits but they're not showing up

1. Check your PayPal payment status
2. Refresh your dashboard
3. Check your permanent credits balance (separate from regular balance)
4. Make sure permanent credits are enabled in your settings
5. Contact support with your order ID if still not showing

### How do I enable/disable flex credits or permanent credits?

You can manage these settings in the [Electron Hub Dashboard](https://playground.electronhub.ai/console):
- Navigate to account settings
- Toggle flex credits on/off
- Toggle permanent credits on/off
- Set permanent credits monthly limits

### Why are free models sometimes unavailable?

Free models are hosted by Electron Hub and may experience downtime during peak usage to ensure fair resource distribution. For better reliability:
- Use freemium models (they deduct credits but have better uptime)
- Subscribe to access premium models (routed to multiple providers for maximum uptime)

### How do I calculate how many credits I need?

Credits are calculated based on:
- **Token-based models**: Price per million tokens × tokens used
- **Fixed-price models**: Fixed cost per request (like image generation)

Check the model pricing in the dashboard or API documentation for specific costs.

**Example:**
- Model costs $3 per million input tokens and $15 per million output tokens
- You use 1,000 input tokens and 500 output tokens
- Cost = (1,000 / 1,000,000 × $3) + (500 / 1,000,000 × $15) = $0.003 + $0.0075 = $0.0105 credits

---

## Privacy & Security

### No-Log Policy for Prompts

**We do not log your prompt content.** Your prompts and AI responses are not stored in our logs. We only log:
- Request metadata (model used, tokens, credits, timestamp, status)
- Technical information (IP address, API key used, route accessed)
- Usage statistics (for billing and analytics)

**What this means:**
- Your conversations and prompts remain private
- We cannot see what you're asking the AI
- We cannot reconstruct your conversations from our logs
- Your data privacy is protected

### Content Moderation

To comply with our AI provider content policies and legal requirements, we apply moderation to detect and flag requests involving:
- **CSAM (Child Sexual Abuse Material)**: Any content related to child exploitation
- **Illegal Activities**: Content about making drugs, bombs, weapons, or other illegal activities
- **Harmful Content**: Content that could cause real-world harm

**How it works:**
- Requests are automatically scanned before being sent to AI providers
- Flagged requests are blocked and not processed
- No credits are charged for blocked requests
- This helps us maintain compliance with provider policies and legal requirements

**Your privacy is still protected:**
- Moderation checks are automated and do not involve human review of your content
- We do not store the content of blocked requests
- This is a necessary measure to ensure service availability and legal compliance

## Helpful Links

- **Documentation**: [https://docs.electronhub.ai](https://docs.electronhub.ai)
- **Dashboard**: [https://playground.electronhub.ai/console](https://playground.electronhub.ai/console)
- **Status Page**: [https://status.electronhub.ai](https://status.electronhub.ai) - Check uptime status for popular models
- **Terms of Service**: [https://tos.electronhub.ai](https://tos.electronhub.ai)
- **Privacy Policy**: [https://privacy.electronhub.ai](https://privacy.electronhub.ai)

---

## Quick Reference

### Daily Reset Time
- **21:00 UTC** (9:00 PM UTC) every day

### Credit Types
- **Daily Credits**: Reset daily, amount depends on tier
- **Permanent Credits**: Never expire, purchased with money
- **Flex Credits**: Saved daily credits, up to 100% of daily amount
- **Neutrinos**: Free requests for `:free` models (charged for all requests: failed/cancelled/successful)

### Credit Charging
- **Regular Credits**: Only charged for successful/cancelled requests (NOT failed requests)
- **Neutrinos**: Charged for all requests (failed, cancelled, and successful)

### Model Access
- **Free Models** (`:free`): Use Neutrinos, no subscription needed
- **Freemium Models**: Use credits, no subscription needed
- **Premium Models**: Require Pro+ subscription + credits

### Payment
- **Method**: PayPal
- **Base Rate**: $1 = 3 credits
- **Volume Discount**: 10% per $500, max 30% at $1,500+

### Support
For additional help, contact support through the dashboard or visit the documentation.

---

*Last updated: 2025*

