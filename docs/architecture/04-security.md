# Security Architecture

**Language:** English | [中文](04-security.zh-CN.md)

Comprehensive security documentation covering encryption, authentication, and data protection.

---

## Security Overview

NOFX implements defense-in-depth security:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Security Layers                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Layer 1: Transport Security                                           │ │
│  │  - HTTPS (TLS 1.3)                                                     │ │
│  │  - Optional end-to-end encryption (RSA + AES)                          │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Layer 2: Authentication                                               │ │
│  │  - JWT tokens with blacklisting                                        │ │
│  │  - Optional 2FA (TOTP)                                                 │ │
│  │  - Rate limiting                                                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Layer 3: Authorization                                                │ │
│  │  - Role-based access control                                           │ │
│  │  - Resource ownership validation                                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Layer 4: Data Protection                                              │ │
│  │  - AES-256-GCM encryption at rest                                      │ │
│  │  - Encrypted database fields                                           │ │
│  │  - Secure key management                                               │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Encryption

### Encryption at Rest (AES-256-GCM)

Sensitive data is encrypted before storage using AES-256-GCM.

```go
// crypto/crypto.go
type CryptoService struct {
    key []byte  // 32 bytes for AES-256
}

func (s *CryptoService) Encrypt(plaintext string) (string, error) {
    // 1. Create AES cipher
    block, _ := aes.NewCipher(s.key)

    // 2. Create GCM mode
    gcm, _ := cipher.NewGCM(block)

    // 3. Generate random nonce (12 bytes)
    nonce := make([]byte, gcm.NonceSize())
    io.ReadFull(rand.Reader, nonce)

    // 4. Encrypt with authentication
    ciphertext := gcm.Seal(nonce, nonce, []byte(plaintext), nil)

    // 5. Return base64 encoded
    return base64.StdEncoding.EncodeToString(ciphertext), nil
}

func (s *CryptoService) Decrypt(encrypted string) (string, error) {
    // 1. Decode base64
    ciphertext, _ := base64.StdEncoding.DecodeString(encrypted)

    // 2. Create cipher
    block, _ := aes.NewCipher(s.key)
    gcm, _ := cipher.NewGCM(block)

    // 3. Extract nonce
    nonce := ciphertext[:gcm.NonceSize()]
    ciphertext = ciphertext[gcm.NonceSize():]

    // 4. Decrypt and verify
    plaintext, err := gcm.Open(nil, nonce, ciphertext, nil)
    return string(plaintext), err
}
```

### Encrypted Database Fields

GORM custom type for automatic encryption:

```go
// store/encrypted_string.go
type EncryptedString string

func (e *EncryptedString) Scan(value interface{}) error {
    str := value.(string)
    decrypted, err := crypto.Decrypt(str)
    if err != nil {
        return err
    }
    *e = EncryptedString(decrypted)
    return nil
}

func (e EncryptedString) Value() (driver.Value, error) {
    encrypted, err := crypto.Encrypt(string(e))
    return encrypted, err
}

// Usage in models
type Trader struct {
    APIKey    EncryptedString  // Automatically encrypted
    APISecret EncryptedString
}
```

### Protected Fields

| Field | Model | Encryption |
|-------|-------|------------|
| API Key | Trader | AES-256-GCM |
| API Secret | Trader | AES-256-GCM |
| TOTP Secret | User | AES-256-GCM |
| Passphrase | Exchange | AES-256-GCM |

### Key Management

```bash
# Generate encryption key (32 bytes, base64 encoded)
openssl rand -base64 32

# Set in environment
DATA_ENCRYPTION_KEY=your-generated-key
```

---

## Transport Encryption (Optional)

For additional security, NOFX supports end-to-end encryption:

```
┌─────────────┐                                      ┌─────────────┐
│   Browser   │                                      │   Backend   │
├─────────────┤                                      ├─────────────┤
│             │                                      │             │
│  1. Get public key from server                     │             │
│             │───────────────────────────────────── │             │
│             │◄──────────────────────────────────── │  RSA-4096   │
│             │         RSA public key               │  key pair   │
│             │                                      │             │
│  2. Generate AES session key                       │             │
│             │                                      │             │
│  3. Encrypt session key with RSA                   │             │
│             │                                      │             │
│  4. Encrypt data with AES session key              │             │
│             │                                      │             │
│  5. Send encrypted request                         │             │
│             │───────────────────────────────────── │             │
│             │    {encryptedKey, encryptedData}     │             │
│             │                                      │             │
│             │                                      │  6. Decrypt │
│             │                                      │     key     │
│             │                                      │  7. Decrypt │
│             │                                      │     data    │
│             │                                      │             │
└─────────────┘                                      └─────────────┘
```

### Configuration

```bash
# Enable transport encryption
TRANSPORT_ENCRYPTION=true

# RSA private key (PEM format)
RSA_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----"
```

### Frontend Implementation

```typescript
// web/src/lib/crypto.ts
async function encryptRequest(data: object): Promise<EncryptedPayload> {
    // 1. Generate random AES key
    const aesKey = await crypto.subtle.generateKey(
        { name: 'AES-GCM', length: 256 },
        true,
        ['encrypt']
    );

    // 2. Encrypt data with AES
    const iv = crypto.getRandomValues(new Uint8Array(12));
    const encrypted = await crypto.subtle.encrypt(
        { name: 'AES-GCM', iv },
        aesKey,
        new TextEncoder().encode(JSON.stringify(data))
    );

    // 3. Encrypt AES key with server's RSA public key
    const exportedKey = await crypto.subtle.exportKey('raw', aesKey);
    const encryptedKey = await crypto.subtle.encrypt(
        { name: 'RSA-OAEP' },
        serverPublicKey,
        exportedKey
    );

    return {
        key: base64(encryptedKey),
        iv: base64(iv),
        data: base64(encrypted)
    };
}
```

---

## Authentication

### JWT Authentication

```go
// auth/auth.go
type Claims struct {
    UserID   uint   `json:"user_id"`
    Username string `json:"username"`
    IsAdmin  bool   `json:"is_admin"`
    jwt.RegisteredClaims
}

func GenerateToken(user *User) (string, error) {
    claims := Claims{
        UserID:   user.ID,
        Username: user.Username,
        IsAdmin:  user.IsAdmin,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte(jwtSecret))
}
```

### Token Blacklisting

Tokens can be invalidated before expiration:

```go
// auth/blacklist.go
type TokenBlacklist struct {
    tokens sync.Map  // token -> expiration time
}

func (b *TokenBlacklist) Add(token string, expiration time.Time) {
    b.tokens.Store(token, expiration)
}

func (b *TokenBlacklist) IsBlacklisted(token string) bool {
    _, exists := b.tokens.Load(token)
    return exists
}

// Cleanup expired entries periodically
func (b *TokenBlacklist) Cleanup() {
    now := time.Now()
    b.tokens.Range(func(key, value interface{}) bool {
        if value.(time.Time).Before(now) {
            b.tokens.Delete(key)
        }
        return true
    })
}
```

### Two-Factor Authentication (2FA)

TOTP-based 2FA using RFC 6238:

```go
// auth/totp.go
func GenerateTOTPSecret(username string) (*otp.Key, error) {
    return totp.Generate(totp.GenerateOpts{
        Issuer:      "NOFX",
        AccountName: username,
        SecretSize:  32,
    })
}

func ValidateTOTP(secret, code string) bool {
    return totp.Validate(code, secret)
}
```

### 2FA Flow

```
┌─────────────┐                                      ┌─────────────┐
│   Browser   │                                      │   Backend   │
├─────────────┤                                      ├─────────────┤
│             │                                      │             │
│  Setup 2FA:                                        │             │
│  1. Request QR code                                │             │
│             │───────────────────────────────────── │             │
│             │◄──────────────────────────────────── │ Generate    │
│             │         QR code + secret             │ TOTP secret │
│             │                                      │             │
│  2. Scan QR with authenticator app                 │             │
│             │                                      │             │
│  3. Enter verification code                        │             │
│             │───────────────────────────────────── │             │
│             │◄──────────────────────────────────── │ Validate    │
│             │         2FA enabled                  │ and save    │
│             │                                      │             │
│  Login with 2FA:                                   │             │
│  1. Submit username + password                     │             │
│             │───────────────────────────────────── │             │
│             │◄──────────────────────────────────── │ Verify      │
│             │       Requires 2FA code              │ credentials │
│             │                                      │             │
│  2. Submit TOTP code                               │             │
│             │───────────────────────────────────── │             │
│             │◄──────────────────────────────────── │ Validate    │
│             │         JWT token                    │ TOTP        │
│             │                                      │             │
└─────────────┘                                      └─────────────┘
```

---

## Authorization

### Middleware

```go
// api/middleware.go
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        // 1. Extract token from header
        token := c.GetHeader("Authorization")
        if token == "" {
            c.AbortWithStatusJSON(401, gin.H{"error": "unauthorized"})
            return
        }

        // 2. Remove "Bearer " prefix
        token = strings.TrimPrefix(token, "Bearer ")

        // 3. Check blacklist
        if blacklist.IsBlacklisted(token) {
            c.AbortWithStatusJSON(401, gin.H{"error": "token revoked"})
            return
        }

        // 4. Validate and parse token
        claims, err := auth.ValidateToken(token)
        if err != nil {
            c.AbortWithStatusJSON(401, gin.H{"error": "invalid token"})
            return
        }

        // 5. Set user context
        c.Set("user_id", claims.UserID)
        c.Set("username", claims.Username)
        c.Set("is_admin", claims.IsAdmin)

        c.Next()
    }
}
```

### Resource Ownership

```go
// api/handlers.go
func GetTrader(c *gin.Context) {
    userID := c.GetUint("user_id")
    traderID := c.Param("id")

    trader, err := store.GetTrader(traderID)
    if err != nil {
        c.JSON(404, gin.H{"error": "not found"})
        return
    }

    // Verify ownership
    if trader.UserID != userID {
        c.JSON(403, gin.H{"error": "forbidden"})
        return
    }

    c.JSON(200, trader)
}
```

---

## Rate Limiting

```go
// api/ratelimit.go
type RateLimiter struct {
    visitors map[string]*visitor
    mu       sync.Mutex
    rate     int           // requests per window
    window   time.Duration
}

type visitor struct {
    count    int
    lastSeen time.Time
}

func (rl *RateLimiter) Allow(ip string) bool {
    rl.mu.Lock()
    defer rl.mu.Unlock()

    v, exists := rl.visitors[ip]
    if !exists || time.Since(v.lastSeen) > rl.window {
        rl.visitors[ip] = &visitor{count: 1, lastSeen: time.Now()}
        return true
    }

    if v.count >= rl.rate {
        return false
    }

    v.count++
    return true
}
```

### Default Limits

| Endpoint | Limit | Window |
|----------|-------|--------|
| `/api/auth/login` | 5 | 1 minute |
| `/api/auth/register` | 3 | 1 hour |
| `/api/*` (general) | 100 | 1 minute |

---

## Password Security

```go
// auth/password.go
func HashPassword(password string) (string, error) {
    // bcrypt with cost 12
    hash, err := bcrypt.GenerateFromPassword([]byte(password), 12)
    return string(hash), err
}

func VerifyPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}
```

### Password Requirements

| Requirement | Minimum |
|-------------|---------|
| Length | 8 characters |
| Uppercase | 1 character |
| Lowercase | 1 character |
| Number | 1 digit |
| Special | 1 character (optional) |

---

## Sensitive Data Handling

### What Gets Encrypted

| Data | Storage | Encryption |
|------|---------|------------|
| Exchange API keys | Database | AES-256-GCM |
| Exchange API secrets | Database | AES-256-GCM |
| User TOTP secrets | Database | AES-256-GCM |
| User passwords | Database | bcrypt (hashed, not encrypted) |

### What Never Gets Stored

| Data | Handling |
|------|----------|
| Raw passwords | Hashed immediately, never stored |
| Session tokens | In-memory only |
| Decrypted API keys | Memory only, never logged |

### Logging Safety

```go
// logger/safe.go
func SafeLog(data interface{}) interface{} {
    // Redact sensitive fields
    switch v := data.(type) {
    case map[string]interface{}:
        for key := range v {
            if isSensitive(key) {
                v[key] = "[REDACTED]"
            }
        }
    }
    return data
}

func isSensitive(key string) bool {
    sensitive := []string{
        "password", "api_key", "api_secret",
        "secret", "token", "passphrase",
    }
    for _, s := range sensitive {
        if strings.Contains(strings.ToLower(key), s) {
            return true
        }
    }
    return false
}
```

---

## Security Checklist

### Deployment

- [ ] Set strong `JWT_SECRET` (32+ characters)
- [ ] Set `DATA_ENCRYPTION_KEY` for sensitive data
- [ ] Enable HTTPS in production
- [ ] Configure rate limiting
- [ ] Use PostgreSQL in production (not SQLite)
- [ ] Enable 2FA for all users
- [ ] Regular security updates

### Development

- [ ] Never log sensitive data
- [ ] Always validate user input
- [ ] Use parameterized queries (GORM handles this)
- [ ] Validate resource ownership
- [ ] Handle errors without leaking internals

---

## Incident Response

### Token Compromise

1. Blacklist the compromised token
2. Revoke all user sessions
3. Force password reset
4. Review access logs

### API Key Compromise

1. Disable the trader immediately
2. Rotate API keys on exchange
3. Update encrypted keys in NOFX
4. Review trading history

### Database Compromise

1. If encryption key is safe, data remains protected
2. Rotate all API keys on exchanges
3. Force all users to reset passwords
4. Review and audit all access

---

## Next Steps

- [System Overview](01-system-overview.md) - High-level architecture
- [Trading Core Flow](02-trading-core.md) - How trading decisions are made
- [Tech Stack & Deployment](03-tech-stack.md) - Technologies and deployment

---

[← Back to Architecture](README.md)
