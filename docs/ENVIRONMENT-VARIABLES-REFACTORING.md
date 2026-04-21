# Environment Variables Refactoring Guide

## Document Purpose

This document explains the current environment variable design issues in the QKD ETSI API C Wrapper and provides a refactoring plan to align with ETSI GS QKD-014 principles.

## Executive Summary

The current implementation uses `MASTER`/`SLAVE` terminology that conflates TLS roles with QKD roles, creating confusion and unnecessary constraints. This document proposes a clearer `MY`/`PEER` naming scheme that better reflects the actual relationship between nodes.

---

## 1. Current Environment Variables

### 1.1 ETSI 014 Variables (Current)

```bash
# KME Hostnames
QKD_MASTER_KME_HOSTNAME="https://kme-1.example.com"
QKD_SLAVE_KME_HOSTNAME="https://kme-2.example.com"

# SAE Identifiers
QKD_MASTER_SAE="sae-1"
QKD_SLAVE_SAE="sae-2"

# Certificates (Master)
QKD_MASTER_CERT_PATH="/path/to/master/cert.pem"
QKD_MASTER_KEY_PATH="/path/to/master/key.pem"
QKD_MASTER_CA_CERT_PATH="/path/to/master/ca.pem"

# Certificates (Slave)
QKD_SLAVE_CERT_PATH="/path/to/slave/cert.pem"
QKD_SLAVE_KEY_PATH="/path/to/slave/key.pem"
QKD_SLAVE_CA_CERT_PATH="/path/to/slave/ca.pem"
```

### 1.2 Problems with Current Design

#### Problem 1: Misleading Terminology

The terms `MASTER`/`SLAVE` suggest:
- ❌ Permanent hierarchical roles
- ❌ One node is "superior" to another
- ❌ Fixed topology-level distinction

Reality per ETSI GS QKD-014:
- ✅ Both nodes are peers (SAEs)
- ✅ Roles (initiator/responder) are transient and per-key
- ✅ No permanent "master" or "slave" exists

#### Problem 2: Unnecessary Duplication

Current design requires:
- Both `MASTER` and `SLAVE` KME hostnames
- Both `MASTER` and `SLAVE` SAE identifiers
- Both sets of certificates

Reality:
- Each node only needs **its own** information
- Peer information is only needed for API calls (peer SAE ID)
- Peer certificates are **not** needed for authentication

#### Problem 3: Confusion with TLS Roles

Users often incorrectly assume:
- `MASTER` = TLS Client
- `SLAVE` = TLS Server

This is **incorrect** because:
- TLS roles are about connection establishment
- QKD roles are about key acquisition
- These are independent concerns

---

## 2. Proposed Environment Variables

### 2.1 New Naming Scheme

```bash
# Self identification (always required)
QKD_MY_KME_HOSTNAME="https://my-kme.example.com"
QKD_MY_SAE_ID="my-sae-id"
QKD_MY_CERT_PATH="/path/to/my/cert.pem"
QKD_MY_KEY_PATH="/path/to/my/key.pem"
QKD_MY_CA_CERT_PATH="/path/to/my/ca.pem"

# Peer identification (for current key exchange)
QKD_PEER_SAE_ID="peer-sae-id"

# Optional: Future Key Stream support
QKD_KEY_STREAM_ID="stream-identifier"
```

### 2.2 Rationale

| Aspect | Old Design | New Design | Benefit |
|--------|-----------|------------|---------|
| Terminology | `MASTER`/`SLAVE` | `MY`/`PEER` | Clear, non-hierarchical |
| Scope | Both nodes' info | Only my info + peer ID | Simpler configuration |
| Certificates | Both sets | Only my set | Reduced complexity |
| Scalability | 1:1 only | Supports 1:N | Future-proof |

---

## 3. Implementation in qkd-etsi-api-c-wrapper

### 3.1 Current Certificate Loading

From `src/etsi014/backends/qkd_etsi014_backend.c`:

```c
static int init_cert_config(int is_master, etsi014_cert_config_t *config) {
    if (is_master) {
        config->cert_path = getenv("QKD_MASTER_CERT_PATH");
        config->key_path = getenv("QKD_MASTER_KEY_PATH");
        config->ca_cert_path = getenv("QKD_MASTER_CA_CERT_PATH");
    } else {
        config->cert_path = getenv("QKD_SLAVE_CERT_PATH");
        config->key_path = getenv("QKD_SLAVE_KEY_PATH");
        config->ca_cert_path = getenv("QKD_SLAVE_CA_CERT_PATH");
    }
    // ...
}
```

### 3.2 Proposed Certificate Loading

```c
static int init_cert_config(etsi014_cert_config_t *config) {
    // Always load "my" certificates
    config->cert_path = getenv("QKD_MY_CERT_PATH");
    config->key_path = getenv("QKD_MY_KEY_PATH");
    config->ca_cert_path = getenv("QKD_MY_CA_CERT_PATH");
    
    // Backward compatibility (with deprecation warning)
    if (!config->cert_path) {
        config->cert_path = getenv("QKD_MASTER_CERT_PATH");
        if (!config->cert_path) {
            config->cert_path = getenv("QKD_SLAVE_CERT_PATH");
        }
        if (config->cert_path) {
            fprintf(stderr, "WARNING: QKD_MASTER/SLAVE_CERT_PATH is deprecated. "
                            "Use QKD_MY_CERT_PATH instead.\n");
        }
    }
    
    // Validate
    if (!config->cert_path || !config->key_path || !config->ca_cert_path) {
        return QKD_STATUS_BAD_REQUEST;
    }
    
    return QKD_STATUS_OK;
}
```

### 3.3 API Call Updates

#### Current Implementation

```c
// Initiator (master)
uint32_t ret = GET_STATUS(ctx->master_kme, ctx->slave_sae, &ctx->status);

// Responder (slave)
uint32_t ret = GET_KEY_WITH_IDS(ctx->slave_kme, ctx->master_sae, &key_ids, &container);
```

#### Proposed Implementation

```c
// Initiator
uint32_t ret = GET_STATUS(ctx->my_kme, ctx->peer_sae, &ctx->status);

// Responder
uint32_t ret = GET_KEY_WITH_IDS(ctx->my_kme, ctx->peer_sae, &key_ids, &container);
```

**Key Insight:** Both initiator and responder always use:
- `my_kme`: The KME this node connects to
- `peer_sae`: The peer's SAE identifier

---

## 4. Migration Strategy

### 4.1 Phase 1: Add Support for New Variables

```c
// Helper function to get variable with fallback
static const char* get_env_with_fallback(const char *new_var, 
                                         const char *old_var1,
                                         const char *old_var2,
                                         bool *used_deprecated) {
    const char *value = getenv(new_var);
    if (value) {
        return value;
    }
    
    // Try old variables
    value = getenv(old_var1);
    if (value) {
        *used_deprecated = true;
        return value;
    }
    
    value = getenv(old_var2);
    if (value) {
        *used_deprecated = true;
        return value;
    }
    
    return NULL;
}

// Usage
bool deprecated = false;
const char *my_kme = get_env_with_fallback(
    "QKD_MY_KME_HOSTNAME",
    "QKD_MASTER_KME_HOSTNAME",
    "QKD_SLAVE_KME_HOSTNAME",
    &deprecated
);

if (deprecated) {
    fprintf(stderr, "WARNING: QKD_MASTER/SLAVE_* variables are deprecated. "
                    "Please migrate to QKD_MY_* variables.\n");
}
```

### 4.2 Phase 2: Update Documentation

Update `README.md` to show new variables:

```markdown
## Environment Variables (ETSI 014)

### Required Variables

```bash
# Your node's information
export QKD_MY_KME_HOSTNAME="https://my-kme.example.com"
export QKD_MY_SAE_ID="my-sae-identifier"
export QKD_MY_CERT_PATH="/path/to/my/cert.pem"
export QKD_MY_KEY_PATH="/path/to/my/key.pem"
export QKD_MY_CA_CERT_PATH="/path/to/my/ca.pem"

# Peer's information
export QKD_PEER_SAE_ID="peer-sae-identifier"
```

### Deprecated Variables (Backward Compatible)

The following variables are deprecated but still supported:
- `QKD_MASTER_KME_HOSTNAME` → Use `QKD_MY_KME_HOSTNAME`
- `QKD_SLAVE_KME_HOSTNAME` → Use `QKD_MY_KME_HOSTNAME`
- `QKD_MASTER_SAE` → Use `QKD_MY_SAE_ID` or `QKD_PEER_SAE_ID`
- `QKD_SLAVE_SAE` → Use `QKD_MY_SAE_ID` or `QKD_PEER_SAE_ID`
- `QKD_MASTER_CERT_PATH` → Use `QKD_MY_CERT_PATH`
- `QKD_SLAVE_CERT_PATH` → Use `QKD_MY_CERT_PATH`
```

### 4.3 Phase 3: Update Tests

Update test scripts to use new variables:

```bash
# Old (deprecated)
export QKD_MASTER_KME_HOSTNAME="https://kme-1.example.com"
export QKD_SLAVE_KME_HOSTNAME="https://kme-2.example.com"
export QKD_MASTER_SAE="sae-1"
export QKD_SLAVE_SAE="sae-2"

# New (recommended)
# Node A configuration
export QKD_MY_KME_HOSTNAME="https://kme-1.example.com"
export QKD_MY_SAE_ID="sae-1"
export QKD_PEER_SAE_ID="sae-2"

# Node B configuration
export QKD_MY_KME_HOSTNAME="https://kme-2.example.com"
export QKD_MY_SAE_ID="sae-2"
export QKD_PEER_SAE_ID="sae-1"
```

### 4.4 Phase 4: Remove Old Variables

After sufficient deprecation period (e.g., 6 months):
- Remove fallback code
- Update all documentation
- Remove old variable support from tests

---

## 5. Benefits of Refactoring

### 5.1 Conceptual Clarity

```
Before:
  "Am I master or slave?" → Confusing, implies hierarchy
  
After:
  "What is my KME and who is my peer?" → Clear, peer-to-peer
```

### 5.2 Configuration Simplicity

```
Before (each node needs to know both sides):
  Node A: MASTER_* (self) + SLAVE_* (peer)
  Node B: SLAVE_* (self) + MASTER_* (peer)
  
After (each node only knows itself):
  Node A: MY_* (self) + PEER_SAE_ID
  Node B: MY_* (self) + PEER_SAE_ID
```

### 5.3 Reduced Errors

Common mistakes with old design:
- ❌ Using wrong certificate set
- ❌ Confusing master/slave with TLS client/server
- ❌ Thinking roles are permanent

New design prevents these:
- ✅ Only one certificate set to configure
- ✅ Clear separation from TLS roles
- ✅ Obvious that roles are transient

---

## 6. Implementation Checklist

### 6.1 Code Changes

- [ ] Add `get_env_with_fallback()` helper function
- [ ] Update `init_cert_config()` to use new variables
- [ ] Add deprecation warnings for old variables
- [ ] Update all API call sites
- [ ] Add backward compatibility tests

### 6.2 Documentation

- [ ] Update README.md with new variables
- [ ] Add migration guide section
- [ ] Update example scripts
- [ ] Document deprecation timeline
- [ ] Add FAQ about the change

### 6.3 Testing

- [ ] Test with new variables only
- [ ] Test with old variables (verify warnings)
- [ ] Test mixed old/new variables
- [ ] Update CI/CD pipelines
- [ ] Verify backward compatibility

---

## 7. Example Configurations

### 7.1 Simple 1:1 Setup (New Style)

**Node A:**
```bash
export QKD_MY_KME_HOSTNAME="https://kme-a.example.com"
export QKD_MY_SAE_ID="sae-a"
export QKD_MY_CERT_PATH="/certs/node-a.pem"
export QKD_MY_KEY_PATH="/certs/node-a-key.pem"
export QKD_MY_CA_CERT_PATH="/certs/ca.pem"
export QKD_PEER_SAE_ID="sae-b"
```

**Node B:**
```bash
export QKD_MY_KME_HOSTNAME="https://kme-b.example.com"
export QKD_MY_SAE_ID="sae-b"
export QKD_MY_CERT_PATH="/certs/node-b.pem"
export QKD_MY_KEY_PATH="/certs/node-b-key.pem"
export QKD_MY_CA_CERT_PATH="/certs/ca.pem"
export QKD_PEER_SAE_ID="sae-a"
```

### 7.2 QuKayDee Example (New Style)

```bash
# Node configuration
export QKD_MY_KME_HOSTNAME="https://kme-1.acct-2509.etsi-qkd-api.qukaydee.com"
export QKD_MY_SAE_ID="sae-1"
export QKD_MY_CERT_PATH="/certs/account-2509-sae-1-cert.pem"
export QKD_MY_KEY_PATH="/certs/account-2509-sae-1-key.pem"
export QKD_MY_CA_CERT_PATH="/certs/account-2509-server-ca-qukaydee-com.crt"
export QKD_PEER_SAE_ID="sae-2"
```

---

## 8. Relationship with qkd-kem-provider

This refactoring should be coordinated with the corresponding changes in `qkd-kem-provider`. See:
- `qkd-kem-provider/docs/ARCHITECTURE-REFACTORING.md`

Both projects should:
1. Use the same new variable names
2. Follow the same deprecation timeline
3. Maintain backward compatibility during transition
4. Update documentation consistently

---

## 9. Future Enhancements

### 9.1 Key Stream Support

Enable 1:N topology:

```bash
# Node A connecting to multiple peers
export QKD_MY_KME_HOSTNAME="https://kme-a.example.com"
export QKD_MY_SAE_ID="sae-a"

# Different Key Streams for different peers
export QKD_PEER_SAE_ID="sae-b"
export QKD_KEY_STREAM_ID="stream-a-to-b"

# Or for another peer
export QKD_PEER_SAE_ID="sae-c"
export QKD_KEY_STREAM_ID="stream-a-to-c"
```

### 9.2 Dynamic Configuration

Support runtime configuration instead of environment variables:

```c
qkd_config_t config = {
    .my_kme_hostname = "https://my-kme.example.com",
    .my_sae_id = "my-sae",
    .peer_sae_id = "peer-sae",
    .cert_path = "/path/to/cert.pem",
    // ...
};

qkd_init(&config);
```

---

## 10. References

- ETSI GS QKD 014 V1.1.1 (2019-02): Protocol and data format of REST-based key delivery API
- QURSA Project: https://github.com/qursa-uc3m
- Related: `qkd-kem-provider/docs/ARCHITECTURE-REFACTORING.md`

---

**Document Version:** 1.0  
**Last Updated:** 2026-04-21  
**Status:** Proposed for Implementation