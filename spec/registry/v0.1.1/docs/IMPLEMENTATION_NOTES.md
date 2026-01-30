# MCP Registry Spec - Implementation Notes

**Version**: 0.1.1
**Last Updated**: 2026-01-30

This document provides implementation guidance for the MCP Registry Spec. For normative requirements, see [verifier-rulebook.v0.1.md](./verifier-rulebook.v0.1.md).

---

## Overview

The MCP Registry system uses content-addressed identifiers (CIDs) to provide tamper-evident, verifiable tool distribution. The key components are:

1. **Pointer** - Registry-signed reference to a tool version
2. **Descriptor** - Tool metadata (name, version, security policy)
3. **Manifest** - File listing with blob CIDs
4. **Attestation** - Third-party integrity claims

---

## 9-Step Verification Pipeline

### Step 1: Resolve Pointer

```typescript
// Fetch and parse pointer document
const pointer = await fetch(pointerUrl).then(r => r.json());

// Verify registry signature
const sig = pointer.signatures[0];
const preimage = buildSigningPreimage(pointer, sig.signed_fields);
if (!verifyEd25519(registryKey, preimage, sig.sig)) {
  throw new Error('POINTER_SIGNATURE_INVALID');
}

// Check for legacy channel
if (pointer.pointers[0].channel.startsWith('legacy-') && !policy.allow_legacy) {
  throw new Error('LEGACY_NOT_ALLOWED');
}
```

### Step 2: Fetch Descriptor and Manifest

```typescript
const descriptor = await fetchByDescriptorCid(pointer.pointers[0].descriptor_cid);
const manifest = await fetchByRootCid(pointer.pointers[0].root_cid);
```

### Step 3: Parse and Canonicalize

All JSON parsing MUST:
- Reject duplicate keys
- Reject invalid UTF-8
- Reject non-finite numbers (NaN, Infinity)

```typescript
function parseJson(input: string): unknown {
  detectDuplicateKeys(input);  // Throws JSON_CANONICALIZATION_ERROR
  return JSON.parse(input);    // Throws JSON_PARSE_ERROR on syntax errors
}
```

### Step 4: Compute CIDs and Compare

**Important**: Manifest CID uses the preimage rule (Rulebook §4.2.3).

```typescript
// 4a: CID profile match
if (!cidProfilesEqual(pointer.cid_profile, descriptor.artifact.cid_profile)) {
  throw new Error('CID_PROFILE_MISMATCH');
}

// 4b: Root CID match
if (pointer.root_cid !== descriptor.artifact.root_cid) {
  throw new Error('ROOT_CID_MISMATCH');
}

// 4c: Descriptor CID verification
const computedDescriptorCid = computeDocumentCid(descriptor);
if (computedDescriptorCid !== pointer.descriptor_cid) {
  throw new Error('DESCRIPTOR_CID_MISMATCH');
}

// 4d: Manifest CID verification (PREIMAGE RULE)
// Only include {schema_version, cid_profile, entries} in the CID computation
const manifestPreimage = {
  schema_version: manifest.schema_version,
  cid_profile: manifest.cid_profile,
  entries: manifest.entries,
};
const computedManifestCid = computeDocumentCid(manifestPreimage);
if (computedManifestCid !== manifest.root_cid) {
  throw new Error('MANIFEST_CID_MISMATCH');
}

// 4e: Manifest-descriptor link
if (manifest.descriptor_cid !== pointer.descriptor_cid) {
  throw new Error('MANIFEST_DESCRIPTOR_LINK_MISMATCH');
}
```

### Step 5: Validate Manifest

```typescript
// Entries must be sorted by path
const paths = manifest.entries.map(e => e.path);
const sorted = [...paths].sort();
if (!paths.every((p, i) => p === sorted[i])) {
  throw new Error('MANIFEST_ENTRY_ORDER_INVALID');
}

// No backslashes in paths
for (const entry of manifest.entries) {
  if (entry.path.includes('\\')) {
    throw new Error('MANIFEST_PATH_INVALID');
  }
}
```

### Step 6: Gather Attestations

```typescript
const validAttestations = [];

for (const attestation of attestations) {
  // Verify signature
  const sig = attestation.signatures[0];
  const preimage = buildSigningPreimage(attestation, sig.signed_fields);
  if (!keyRegistry.verify(sig.key_id, preimage, sig.sig)) continue;

  // Check subject match
  if (attestation.subject.root_cid !== pointer.root_cid) continue;

  // Find integrity claims
  for (const claim of attestation.claims) {
    if (claim.type !== 'mcp.claim.integrity') continue;

    // Check expiry
    if (claim.expires_at_utc && new Date(claim.expires_at_utc) < now) {
      throw new Error('ATTESTATION_EXPIRED');
    }

    // Verify claim matches
    if (claim.payload.verified_root_cid === pointer.root_cid) {
      validAttestations.push({ keyId: claim.issuer.key_id, role: claim.issuer.role });
    }
  }
}

if (validAttestations.length === 0) {
  throw new Error('NO_VALID_ATTESTATIONS');
}
```

### Step 7: Enforce Constraints

```typescript
const constraints = pointer.pointers[0].constraints ?? {};

// require_signers
if (constraints.require_signers?.length > 0) {
  const hasRequired = validAttestations.some(a =>
    constraints.require_signers.includes(a.keyId)
  );
  if (!hasRequired) throw new Error('REQUIRED_SIGNER_MISSING');
}

// require_verifier_attestation
if (constraints.require_verifier_attestation) {
  const hasVerifier = validAttestations.some(a => a.role === 'verifier');
  if (!hasVerifier) throw new Error('VERIFIER_ATTESTATION_REQUIRED');
}

// min_attestations
const minRequired = constraints.min_attestations ?? 1;
if (validAttestations.length < minRequired) {
  throw new Error('INSUFFICIENT_ATTESTATIONS');
}
```

### Step 8: Policy Gate

```typescript
const toolPolicy = descriptor.security.policy;

if (toolPolicy.network === 'allow' && installerPolicy.network === 'deny') {
  throw new Error('POLICY_BLOCKED_NETWORK');
}

if (toolPolicy.filesystem === 'read_write' && installerPolicy.filesystem !== 'allow') {
  throw new Error('POLICY_BLOCKED_FILESYSTEM');
}

if (toolPolicy.exec === 'allow' && installerPolicy.exec === 'deny') {
  throw new Error('POLICY_BLOCKED_EXEC');
}
```

### Step 9: Accept

```typescript
return {
  result: 'ACCEPT',
  provenance: {
    tool: pointer.pointers[0].tool,
    channel: pointer.pointers[0].channel,
    root_cid: pointer.pointers[0].root_cid,
    descriptor_cid: pointer.pointers[0].descriptor_cid,
    attestations_used: validAttestations.map(a => a.keyId),
  }
};
```

---

## CID Computation

### Document CID

```typescript
import { encode as encodeCbor } from '@ipld/dag-cbor';
import { sha256 } from '@noble/hashes/sha256';
import { CID } from 'multiformats/cid';
import * as dagCbor from '@ipld/dag-cbor';

function computeDocumentCid(doc: unknown): string {
  // 1. Canonical JSON
  const canonical = canonicalStringify(doc);

  // 2. Parse back to value (ensures clean types)
  const value = JSON.parse(canonical);

  // 3. Deterministic CBOR
  const cbor = encodeCbor(value);

  // 4. SHA2-256 hash
  const hash = sha256(cbor);

  // 5. Construct CID v1, dag-cbor codec
  const cid = CID.createV1(dagCbor.code, { code: 0x12, digest: hash });

  // 6. Base32 encoding
  return cid.toString();  // bafyrei...
}
```

### Blob CID

```typescript
function computeBlobCid(blob: Uint8Array): string {
  const hash = sha256(blob);
  const cid = CID.createV1(0x55, { code: 0x12, digest: hash }); // raw codec
  return cid.toString();  // bafkrei...
}
```

---

## Signing Preimage

```typescript
function buildSigningPreimage(doc: unknown, signedFields: string[]): Uint8Array {
  const parts: Uint8Array[] = [];

  for (const pointer of signedFields) {
    const value = evaluateJsonPointer(doc, pointer);
    const canonical = canonicalStringify(value);
    parts.push(new TextEncoder().encode(canonical));
  }

  // Join with 0x00 separator
  const totalLength = parts.reduce((sum, p) => sum + p.length, 0) + parts.length - 1;
  const result = new Uint8Array(totalLength);
  let offset = 0;

  for (let i = 0; i < parts.length; i++) {
    result.set(parts[i], offset);
    offset += parts[i].length;
    if (i < parts.length - 1) {
      result[offset++] = 0x00;
    }
  }

  return result;
}
```

---

## Test Vectors

The spec includes 24 test vectors across 4 categories:

| Range | Category | Description |
|-------|----------|-------------|
| 001-008 | canonical-json | Canonical JSON serialization and error handling |
| 009-012 | cid-computation | CID computation for documents and blobs |
| 013-016 | signing-preimage | Signing preimage construction |
| 017-024 | acceptance | Full verification pipeline scenarios |

Run the test suite:

```bash
cd verifier
npm test
```

---

## Changelog

### v0.1.1 (2026-01-30)

- Updated Step 4 to use manifest CID preimage rule
- Clarified that NaN/Infinity are parse errors, not canonicalization errors

### v0.1.0 (2026-01-30)

- Initial release
