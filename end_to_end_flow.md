
# K-of-N Authentication System

## ✅ 1. Onboarding Flow (App + User Setup)

### 🧾 Step-by-step Breakdown:

#### 1. Register Application

**UI** → `POST /api/app/register`  
Payload:
```json
{
  "appName": "string",
  "k": int,
  "n": int,
  "users": [{ "userId": "UUID", "email": "string" }]
}
```

**Backend:**
- Call `POST /kms/key/generate`
  → Returns `masterKey`, `DEK + EDEK`
- Call `POST /kms/key/split` with `masterKey`, `n`
  → Returns `n shares`

**DB Writes:**
- `Application` Table ← App metadata + EDEK
- `AppShare` Table ← N shares encrypted with DEK
- `AppNameUserMapping` Table ← Map appId ↔ userId (for later lookup)

#### 2. User Onboarding (Repeated for Each of the N Users)

**UI** → `POST /api/user/onboard`  
Payload:
```json
{
  "appId": "UUID",
  "userId": "UUID",
  "password": "string"
}
```

**Backend:**
- Lookup `AppShare` for (`appId`, `userId`)
- Call `POST /kms/key/decrypt` with EDEK
  → Returns DEK
- Use password to derive `passwordKey`
- Decrypt DEK-encrypted share
- Re-encrypt with `passwordKey`

**DB Writes:**
- `UserShare` Table ← Share encrypted with `passwordKey`
- Mark user as onboarded

---

## 🔐 2. Authentication Flow (K users logging in)

### 🧾 Step-by-step Breakdown:

#### 1. Start Session

**UI** → `POST /api/session/start`  
Payload:
```json
{
  "appId": "UUID",
  "initiatedBy": "UUID"
}
```

**Backend:**
- Store new `Session` row (status: "PENDING")
- Lookup `EDEK` from `Application` table
- Call `POST /kms/key/decrypt` with EDEK
  → Returns `DEK`
- Cache DEK temporarily

**DB Writes:**
- `Session` Table ← session metadata (`sessionId`, `appId`, `status`)

#### 2. Submit User Share (Repeated for each user)

**UI** → `POST /api/session/submit`  
Payload:
```json
{
  "sessionId": "UUID",
  "userId": "UUID",
  "password": "string"
}
```

**Backend:**
- Fetch encrypted share from `UserShare` (passwordKey-encrypted)
- Derive `passwordKey`
- Decrypt with passwordKey → Get user's share
- Re-encrypt user's share with cached DEK

**DB Writes:**
- `TempSessionShare` ← Re-encrypted share (DEK encrypted) + userId

#### 3. Once K shares received:

**Backend:**
- Fetch K re-encrypted shares from `TempSessionShare`
- Decrypt each using `DEK` → Get plaintext shares
- Call `POST /kms/key/reconstruct`  
  Payload:
  ```json
  {
    "shares": ["plaintext_share1", "plaintext_share2", "plaintext_share3"]
  }
  ```
  → Returns reconstructed Master Key

- Use MK to decrypt secret (e.g., admin password)

**DB Updates:**
- `Session` Table ← status: COMPLETED
- `TempSessionShare` ← optional cleanup

---

## 🧩 Database Tables

### Application Table
| Column       | Type    | Description                  |
|--------------|---------|------------------------------|
| appId        | UUID    | Unique ID for the application |
| appName      | String  | Human-readable name           |
| k            | Int     | Minimum threshold             |
| n            | Int     | Total number of users         |
| edek         | Blob    | Encrypted DEK from KMS        |

### AppShare Table
| Column       | Type    | Description                     |
|--------------|---------|---------------------------------|
| appId        | UUID    | FK to Application               |
| userId       | UUID    | FK to User                      |
| encryptedShare | Blob  | Share encrypted with DEK        |

### AppNameUserMapping Table
| Column       | Type    | Description                    |
|--------------|---------|--------------------------------|
| appId        | UUID    | FK to Application              |
| userId       | UUID    | FK to User                     |

### UserShare Table
| Column       | Type    | Description                        |
|--------------|---------|------------------------------------|
| appId        | UUID    | FK to Application                  |
| userId       | UUID    | FK to User                         |
| encryptedShare | Blob  | Share encrypted with passwordKey   |
| onboarded    | Bool    | Whether user has completed setup   |

### Session Table
| Column       | Type    | Description                     |
|--------------|---------|---------------------------------|
| sessionId    | UUID    | Unique session ID               |
| appId        | UUID    | FK to Application               |
| status       | String  | PENDING, COMPLETED              |
| initiatedBy  | UUID    | userId of the session initiator |

### TempSessionShare Table
| Column       | Type    | Description                           |
|--------------|---------|---------------------------------------|
| sessionId    | UUID    | FK to Session                         |
| userId       | UUID    | FK to User                            |
| encryptedShare | Blob  | Share encrypted with session's DEK    |
