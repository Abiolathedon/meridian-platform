# meridian-platform

Infrastructure-as-code for the Meridian Retail estate. Every change to the estate is
delivered from this repository — versioned, tested in non-production, reviewed, and
promoted to production under change control.

Built to the Meridian Target-State Architecture (HLD) and Standard Build Specification.

## Golden rules
- Non-production before production, always.
- No secret is ever committed — secrets live in Vault.
- Every change: branch -> PR -> review -> non-prod test -> change approval -> promote.

