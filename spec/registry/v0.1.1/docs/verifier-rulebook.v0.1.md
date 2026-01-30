# MCP Registry Spec - Verifier Rulebook v0.1.1

**Status**: Normative
**Version**: 0.1.1
**Last Updated**: 2026-01-30

This document defines the normative rules for verifying MCP tool artifacts. All verifier implementations MUST follow these rules exactly.

---

## 1. Canonical JSON Serialization

All JSON documents in the MCP registry system MUST be serialized using Canonical JSON before hashing or signing.

### 1.1 Encoding

- **Character encoding**: UTF-8, no BOM
- **Line endings**: Not applicable (minified)

### 1.2 Object Key Ordering

Object keys MUST be sorted lexicographically by Unicode code point order.

```
Example: {"Z":1,"a":2,"Å":3} (code points: Z=90, a=97, Å=197)
```

### 1.3 Duplicate Keys

Duplicate keys MUST be rejected with error code `JSON_CANONICALIZATION_ERROR`.

### 1.4 Whitespace

Canonical JSON MUST be minified with no spaces or newlines between tokens.

### 1.5 Numbers

- Shortest round-trip representation
- No leading `+` sign
- No leading zeros (except `0.x`)
- No trailing zeros after decimal point
- Non-finite values (`NaN`, `Infinity`, `-Infinity`) MUST be rejected with error code `JSON_PARSE_ERROR`

### 1.6 Strings

- Valid Unicode only
- Escape only characters required by JSON (`"`, `\`, control characters)
- Use `\uXXXX` for control characters

### 1.7 Error Codes

| Code | Condition |
|------|-----------|
| `JSON_PARSE_ERROR` | Invalid JSON syntax, invalid literals (NaN/Infinity), invalid UTF-8 |
| `JSON_CANONICALIZATION_ERROR` | Duplicate keys, structural violations after successful parse |

---

## 2. JSON Pointer Evaluation

JSON Pointers (RFC 6901) are used in `signed_fields` arrays to specify which document fields are included in signing preimages.

### 2.1 Syntax

- Empty string `""` refers to the entire document
- Pointers start with `/` and use `/` as separator
- `~0` escapes `~`, `~1` escapes `/`

### 2.2 Evaluation

Given a pointer and a JSON value:
1. If pointer is `""`, return the entire value
2. Split pointer by `/` (after the leading `/`)
3. For each token, descend into the value:
   - For objects: look up by key
   - For arrays: parse token as integer index
4. Return the resolved value or error if path invalid

---

## 3. Signing Preimage Construction

Signatures in MCP registry documents use a deterministic preimage constructed from specified fields.

### 3.1 Preimage Format

```
preimage = canonical_json(field_1) || 0x00 || canonical_json(field_2) || 0x00 || ...
```

Where:
- `||` denotes byte concatenation
- `0x00` is a single null byte separator
- Fields are processed in the order specified in `signed_fields`
- Each field value is serialized as Canonical JSON (UTF-8 bytes)

### 3.2 Signature Algorithm

- **Algorithm**: Ed25519 (RFC 8032)
- **Key encoding**: Base64url (no padding) for `sig` and `public_key` fields
- **Key ID format**: `did:key:z6Mk...` (did:key with multibase-encoded Ed25519 public key)

### 3.3 Signature Object

```json
{
  "key_id": "did:key:z6Mk...",
  "alg": "ed25519",
  "sig": "<base64url-encoded-signature>",
  "signed_fields": ["/field1", "/field2"]
}
```

---

## 4. CID Computation

Content Identifiers (CIDs) provide tamper-evident addressing for all registry artifacts.

### 4.1 CID Profile

The default CID profile is `mcp.cidprofile.default.v1`:

```json
{
  "name": "mcp.cidprofile.default",
  "version": "1",
  "hash": "sha2-256",
  "codec": "dag-cbor",
  "base": "base32"
}
```

### 4.2 Document CID Computation

#### 4.2.1 General Procedure

To compute a CID for a JSON document:

1. Parse JSON document
2. Serialize as Canonical JSON (§1)
3. Parse Canonical JSON back to native value
4. Encode as deterministic CBOR (RFC 8949, with DAG-CBOR conventions)
5. Hash CBOR bytes with SHA2-256
6. Construct CID v1 with codec `dag-cbor` (0x71) and multihash
7. Encode as base32 (lowercase, no padding)

Result format: `bafyrei...` (CID v1, dag-cbor, sha2-256, base32)

#### 4.2.2 Blob CID Computation

For raw binary blobs:

1. Hash blob bytes with SHA2-256
2. Construct CID v1 with codec `raw` (0x55) and multihash
3. Encode as base32

Result format: `bafkrei...` (CID v1, raw, sha2-256, base32)

#### 4.2.3 toolbundle.manifest CID Preimage (normative)

To compute `toolbundle.manifest.root_cid` under `mcp.cidprofile.default.v1`, verifiers and packers MUST compute the CID over a derived "manifest preimage" value as follows:

1. Parse the `toolbundle.manifest` JSON document to a JSON value.

2. Construct `manifest_preimage` by including ONLY the following top-level fields:
   - `schema_version`
   - `cid_profile`
   - `entries`

3. All other top-level fields MUST be excluded from the preimage, including (but not limited to):
   - `root_cid`
   - `descriptor_cid`
   - `bundle_size_bytes`
   - `created_at_utc`

4. Serialize `manifest_preimage` as Canonical JSON (§1).

5. Convert Canonical JSON to deterministic CBOR (§4.2.1).

6. Compute the CID of the CBOR bytes using codec `dag-cbor` and multihash `sha2-256`.

**Verification requirement:**

A manifest is valid only if `manifest.root_cid` equals the computed CID of `manifest_preimage` produced above.

**Rationale (non-normative):**

This prevents self-reference (`root_cid`) and mutual reference (`descriptor_cid`) from creating circular CID dependencies. The manifest's identity is defined by its content (entries), not by its linkage metadata.

---

## 5. Pointer Log Rules

*(Reserved for v0.2 - transparency log verification)*

---

## 6. Install Acceptance Criteria

The 9-step verification pipeline for accepting a tool installation.

### Step 1: Resolve Pointer

- Fetch `registry.pointer` document
- Verify pointer signature against trusted registry keys
- Extract target `tool`, `channel`, `root_cid`, `descriptor_cid`

### Step 2: Fetch Descriptor and Manifest

- Fetch `tool.descriptor` at `descriptor_cid`
- Fetch `toolbundle.manifest` at `root_cid`

### Step 3: Parse and Canonicalize

- Parse all documents as JSON
- Validate against schemas
- Reject duplicate keys, invalid UTF-8

### Step 4: Compute CIDs and Compare

1. Verify CID profile match between pointer and descriptor
2. Verify `pointer.root_cid == descriptor.artifact.root_cid`
3. Compute descriptor CID, verify `== pointer.descriptor_cid`
4. Compute manifest CID (using preimage rule §4.2.3), verify `== manifest.root_cid`
5. Verify `manifest.descriptor_cid == pointer.descriptor_cid`

### Step 5: Validate Manifest

- Verify entries are sorted by path (lexicographic)
- Verify no backslashes in paths
- Verify path separator is `/`

### Step 6: Gather Attestations

For each attestation:
1. Verify signature against known keys
2. Check `subject.root_cid` matches
3. Find `mcp.claim.integrity` claims
4. Check expiry (`expires_at_utc` if present)
5. Verify `payload.verified_root_cid` matches

### Step 7: Enforce Constraints

Check pointer constraints:
- `require_signers`: At least one attestation from listed keys
- `require_verifier_attestation`: At least one attestation with `role: "verifier"`
- `min_attestations`: Minimum valid attestation count (default: 1)

### Step 8: Policy Gate

Compare descriptor security policy against installer policy:
- `network`: Tool requests `allow`, installer denies → REJECT
- `filesystem`: Tool requests `read_write`, installer only allows `read_only` → REJECT
- `exec`: Tool requests `allow`, installer denies → REJECT

### Step 9: Accept or Reject

If all checks pass: `ACCEPT` with provenance record.
Otherwise: `REJECT` with specific error code.

---

## 7. Error Codes

| Code | Step | Description |
|------|------|-------------|
| `POINTER_SIGNATURE_INVALID` | 1 | Pointer signature verification failed |
| `LEGACY_NOT_ALLOWED` | 1 | Legacy channel without `--allow-legacy` |
| `CID_PROFILE_MISMATCH` | 4 | Pointer and descriptor CID profiles differ |
| `ROOT_CID_MISMATCH` | 4 | Pointer root_cid != descriptor.artifact.root_cid |
| `DESCRIPTOR_CID_MISMATCH` | 4 | Computed descriptor CID != pointer.descriptor_cid |
| `MANIFEST_CID_MISMATCH` | 4 | Computed manifest CID != manifest.root_cid |
| `MANIFEST_DESCRIPTOR_LINK_MISMATCH` | 4 | manifest.descriptor_cid != pointer.descriptor_cid |
| `MANIFEST_ENTRY_ORDER_INVALID` | 5 | Entries not sorted by path |
| `MANIFEST_PATH_INVALID` | 5 | Path contains backslash |
| `ATTESTATION_EXPIRED` | 6 | Integrity claim past expires_at_utc |
| `NO_VALID_ATTESTATIONS` | 6 | No valid attestations found |
| `REQUIRED_SIGNER_MISSING` | 7 | No attestation from required signers |
| `VERIFIER_ATTESTATION_REQUIRED` | 7 | No verifier-role attestation |
| `INSUFFICIENT_ATTESTATIONS` | 7 | Below min_attestations threshold |
| `POLICY_BLOCKED_NETWORK` | 8 | Tool requires network, policy denies |
| `POLICY_BLOCKED_FILESYSTEM` | 8 | Tool requires write, policy denies |
| `POLICY_BLOCKED_EXEC` | 8 | Tool requires exec, policy denies |

---

## Appendix A: Changelog

### v0.1.1 (2026-01-30)

- Added §4.2.3: Manifest CID preimage rule to break circular CID dependency
- Clarified error taxonomy: `JSON_PARSE_ERROR` for invalid JSON tokens (NaN/Infinity)

### v0.1.0 (2026-01-30)

- Initial release
