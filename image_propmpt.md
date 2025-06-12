Create text-based or image diagrams for a secure K-of-N password-based authentication system.

1. Diagram 1: Onboarding Flow
   - A user registers a new application with K and N values and user list.
   - KMS generates a master key, splits it into N parts.
   - DEK is generated and used to encrypt each share.
   - Each user logs in separately, derives a password-based key, and re-encrypts their share.

2. Diagram 2: Authentication Flow
   - Session starts for a registered app.
   - K users log in within 15 mins.
   - Their encrypted shares are decrypted and sent to KMS.
   - KMS reconstructs the master key and returns it.
   - Master key is used to decrypt a privileged secret.

Include data flows between:
- Browser/Frontend
- Backend
- KMS
- DB

Ensure the diagram shows:
- DB lookups
- API calls (including /kms/key/reconstruct, /session/start etc.)
- User input
- KMS responsibilities and boundaries

Make the diagrams readable, with clear arrows and naming conventions.
