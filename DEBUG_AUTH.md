# 🔐 FuseConnects Authentication Debug Report

## 📋 Current Auth Flow

```
LOGIN → Generate Tokens → Set Cookies → Store Refresh Hash → Response
                                              ↓
                                        (user.refreshTokens[])
                                              
API Request → axios + credentials:true → cookies auto-sent
                                              ↓
                                        auth middleware checks
                                              ↓
                                        JWT verification
                                              ↓
                                        Database user lookup
```

---

## ✅ What's Working

### 1. **Login Route** (`routes/login.js`)
- ✅ Generates access token (15min expiry) + refresh token (90day expiry)
- ✅ Stores refresh token hash in database
- ✅ Sets cookies with correct options:
  - `httpOnly: true` (XSS protection)
  - `secure: isProd` (HTTPS only in production)
  - `sameSite: isProd ? "None" : "Lax"` (CSRF protection)
  - `maxAge` correctly set for both tokens

### 2. **Auth Middleware** (`middleware/auth.js`)
- ✅ Checks for accessToken cookie
- ✅ Verifies JWT signature against JWT_SECRET
- ✅ Handles TokenExpiredError separately with `code: "TOKEN_EXPIRED"`
- ✅ Fetches fresh user data from DB (allows role updates without re-login)

### 3. **Refresh Route** (`routes/refresh.js`)
- ✅ Token rotation (removes old, creates new)
- ✅ Reuse detection (clears all tokens if reused)
- ✅ Proper refresh token validation

### 4. **Frontend Axios** (`api/axiosInterceptor.js`)
- ✅ `withCredentials: true` (sends cookies)
- ✅ 401 response triggers `/refresh` call
- ✅ Queues failed requests while refreshing
- ✅ Redirects to `/login` if refresh fails

---

## 🔴 POTENTIAL ISSUES

### 1. **Missing Environment Variables** ⚠️
```javascript
// Both login.js and refresh.js use:
process.env.JWT_SECRET        // ✅ Must be set
process.env.REFRESH_TOKEN_SECRET  // ✅ Must be set (different from JWT_SECRET!)
process.env.NODE_ENV          // ✅ Dev vs Prod mode
```

**Check your `.env`:**
```bash
JWT_SECRET=your_jwt_secret_here
REFRESH_TOKEN_SECRET=your_refresh_secret_here
NODE_ENV=development  # or production
```

---

### 2. **Cookie Not Sent Between Different Origins** ⚠️

**Frontend**: `http://localhost:3000`  
**Backend**: `http://localhost:5000`

**CORS must have:**
```javascript
credentials: true  // ✅ server.js line 50 - looks correct
```

**Frontend axios must have:**
```javascript
withCredentials: true  // ✅ axiosInterceptor.js - looks correct
```

---

### 3. **Secure Flag Issue in Development** 🚨

If running on `http://localhost` (not HTTPS):
```javascript
const isProd = process.env.NODE_ENV === "production";
secure: isProd,  // ✅ Correct! Not secure in dev
```

✅ This is correct - cookies work on `http://localhost`

---

### 4. **SameSite=None Requires Secure** 🚨

```javascript
sameSite: isProd ? "None" : "Lax"
secure: isProd
```

✅ This is correct - only uses `SameSite=None` when `secure: true`

---

## 🧪 Testing Checklist

### Test 1: Login & Check Cookies
```bash
curl -X POST http://localhost:5000/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"test"}' \
  -v  # Shows response headers with Set-Cookie
```

Expected:
```
Set-Cookie: accessToken=jwt_token_here; Max-Age=900; HttpOnly; Path=/; SameSite=Lax
Set-Cookie: refreshToken=jwt_token_here; Max-Age=7776000; HttpOnly; Path=/; SameSite=Lax
```

---

### Test 2: Use Token in Request
```bash
curl -X GET http://localhost:5000/home \
  -H "Cookie: accessToken=YOUR_TOKEN_HERE" \
  -v
```

Expected: 200 OK with user data

---

### Test 3: Verify Token Expiry
```bash
# Wait for token to expire OR test with expired token
curl -X GET http://localhost:5000/home \
  -H "Cookie: accessToken=expired_token" \
  -v
```

Expected: 401 with `code: "TOKEN_EXPIRED"`

---

### Test 4: Test Refresh
```bash
curl -X POST http://localhost:5000/refresh \
  -H "Cookie: refreshToken=YOUR_REFRESH_TOKEN_HERE" \
  -v
```

Expected:
- 200 OK
- New `Set-Cookie` headers for both accessToken and refreshToken
- Old refresh token removed from DB

---

## 🔧 Optional Improvements

### 1. **Add Debug Logging**

In `middleware/auth.js`, add after line 6:
```javascript
console.log('🔑 Auth check:', {
  hasAccessToken: !!accessToken,
  tokenPreview: accessToken ? accessToken.substring(0, 20) + '...' : 'none',
  timestamp: new Date().toISOString()
});
```

### 2. **Explicit Token Type Check**

In `middleware/auth.js`, add validation:
```javascript
if (decoded.type !== "access") {
  return res.status(401).json({ message: "Invalid token type" });
}
```

### 3. **Token Not Found in DB**

In `middleware/auth.js`, optionally verify refresh token is in DB:
```javascript
const user = await User.findById(decoded.id);
const hasValidRefresh = user?.refreshTokens?.some(t => 
  t.tokenHash === hashToken(refreshToken)  // Optional extra validation
);
```

---

## 🚀 Debug Command (Backend Console)

Add this to `server.js` for debugging:
```javascript
app.use((req, res, next) => {
  console.log(`[${new Date().toISOString()}] ${req.method} ${req.path}`, {
    cookies: req.cookies,
    auth: req.user || 'none'
  });
  next();
});
```

---

## 📝 Summary

| Component | Status | Issue |
|-----------|--------|-------|
| Login token generation | ✅ | None |
| Cookie set/secure options | ✅ | None |
| Auth middleware | ✅ | Could add token type check |
| Refresh token rotation | ✅ | None |
| Frontend axios config | ✅ | None |
| CORS credentials | ✅ | None |
| Frontend interceptor | ✅ | None |
| **Environment vars** | ❓ | **VERIFY THESE ARE SET** |

**Most likely issue:** Missing or incorrect `JWT_SECRET` or `REFRESH_TOKEN_SECRET` in `.env`

