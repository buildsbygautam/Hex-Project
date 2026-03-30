# Instasend Keys Usage Guide

## 🔑 Two Types of Keys

Instasend provides **two different keys** for different purposes:

### 1. Publishable Key (Frontend) - ✅ Currently Using
- **Format**: `ISPK_live_...` (live) or `ISPK_test_...` (sandbox)
- **Environment Variable**: `VITE_INSTASEND_PUBLISHABLE_KEY`
- **Usage**: Frontend payment button initialization
- **Security**: Safe to expose in frontend code
- **Current Status**: ✅ Implemented correctly

### 2. Secret Key (Backend) - ❌ Missing Implementation
- **Format**: `ISK_live_...` (live) or `ISK_test_...` (sandbox)
- **Environment Variable**: `INSTASEND_SECRET_KEY`
- **Usage**: Server-to-server API calls, webhook validation
- **Security**: Must be kept secret, never expose in frontend
- **Current Status**: ❌ Not implemented yet

## 📋 What We're Currently Using

### Frontend (Working ✅)
```javascript
// In Billing.tsx - Frontend payment button
new window.IntaSend({
  publicAPIKey: import.meta.env.VITE_INSTASEND_PUBLISHABLE_KEY, // ISPK_live_...
  live: import.meta.env.VITE_INSTASEND_LIVE === 'true'
})
```

### Backend (Missing ❌)
```javascript
// For server-to-server API calls (not implemented yet)
const response = await fetch('https://api.intasend.com/v1/collections', {
  headers: {
    'Authorization': `Bearer ${process.env.INSTASEND_SECRET_KEY}`, // ISK_live_...
    'Content-Type': 'application/json'
  }
})
```

## 🚀 What You Need to Do

### 1. Get Both Keys from Instasend Dashboard
- **Publishable Key**: `ISPK_live_...` (for frontend)
- **Secret Key**: `ISK_live_...` (for backend)

### 2. Set Environment Variables
```bash
# Frontend (already set)
VITE_INSTASEND_PUBLISHABLE_KEY=ISPK_live_...
VITE_INSTASEND_LIVE=true

# Backend (need to add)
INSTASEND_SECRET_KEY=ISK_live_...
INSTASEND_WEBHOOK_SECRET=your_webhook_secret
```

### 3. Webhook Configuration
- **Endpoint**: `https://hexai.website/api/webhooks/instasend`
- **Challenge**: Use `INSTASEND_WEBHOOK_SECRET` value
- **Events**: Collection event + Send money event

## 🔧 Current Implementation Status

| Component | Publishable Key | Secret Key | Status |
|-----------|----------------|------------|---------|
| Frontend Payment Button | ✅ Using | ❌ N/A | ✅ Working |
| Webhook Validation | ❌ N/A | ✅ Need | ❌ Missing |
| Server API Calls | ❌ N/A | ✅ Need | ❌ Missing |

## 📚 Next Steps

1. **Add Secret Key** to environment variables
2. **Implement webhook endpoint** with secret key validation
3. **Add server-to-server API calls** if needed
4. **Test webhook integration** end-to-end

The frontend payment flow is working correctly with the publishable key. We just need to add the backend implementation with the secret key for webhooks and server-to-server communication.
