Frontend (User) → /api/app/register
  → Backend → /kms/key/generate → /kms/key/split
  → Store: Application, AppShare, EDEK

Each User → /api/user/onboard
  → Backend: Fetch AppShare → /kms/key/decrypt
  → Derive passwordKey → Decrypt + Re-encrypt → Store in UserShare

User → /api/session/start
  → Backend: Store session → /kms/key/decrypt (EDEK) → Cache DEK

Each User → /api/session/submit
  → Backend: Fetch UserShare → Derive passwordKey → Decrypt share
             → Re-encrypt with DEK → Store in TempSessionShare

Once K shares:
  → Fetch all TempSessionShare entries
  → Decrypt with DEK → /kms/key/reconstruct
  → Use MasterKey to decrypt protected secret
  → Mark session COMPLETE
