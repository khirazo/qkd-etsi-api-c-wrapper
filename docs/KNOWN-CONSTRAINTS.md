# Known Constraints and Design Limitations

## Document Purpose

This document records known constraints and design limitations in the current QKD ETSI API C Wrapper implementation. These constraints are inherent to the current architecture and should be considered when using or extending this library.

---

## 1. Environment Variable Design

### 1.1 Current Variable Naming

The implementation uses `MASTER`/`SLAVE` terminology for environment variables:

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

### 1.2 Terminology Constraints

**Constraint:** The terms `MASTER` and `SLAVE` are used to distinguish between two nodes, but these terms:
- Do not align with ETSI GS QKD-014 official terminology (which only defines SAE and KME)
- May suggest permanent hierarchical roles, which is not accurate per ETSI specification
- Are often confused with TLS client/server roles, which are independent concerns

**Impact:**
- Users must understand that `MASTER`/`SLAVE` are arbitrary labels for configuration purposes
- The same node may need different variable sets depending on its role in a specific connection
- Configuration becomes more complex in multi-peer scenarios

### 1.3 Certificate Configuration Constraints

**Constraint:** The current implementation requires both `MASTER` and `SLAVE` certificate sets to be configured, even though:
- Each node only uses its own certificates for KME authentication
- Peer certificates are not needed for the QKD API operations
- This creates redundant configuration requirements

**Impact:**
- Configuration files must include certificate paths for both roles
- Deployment scripts must handle certificate distribution for both sets
- Certificate rotation requires updating multiple variable sets

---

## 2. Role Assignment and TLS Integration

### 2.1 Role Determination

**Constraint:** The implementation determines which KME and SAE identifiers to use based on the `is_initiator` flag, which is typically derived from TLS roles:

```c
// Conceptual logic (not actual code)
if (is_tls_client) {
    use_master_variables();
} else {
    use_slave_variables();
}
```

**Impact:**
- TLS client always uses `MASTER` variables
- TLS server always uses `SLAVE` variables
- This coupling makes it difficult to support scenarios where TLS and QKD roles differ

### 2.2 ETSI 014 Role Semantics

**Constraint:** Per ETSI GS QKD-014:
- **Initiator**: The SAE that calls `GET_KEY()` first for a specific key
- **Responder**: The SAE that calls `GET_KEY_WITH_IDS()` to retrieve the same key

These roles are:
- Transient (per-key exchange)
- Independent of TLS client/server roles
- Not permanent node characteristics

**Current Implementation:** The implementation assumes a fixed mapping between TLS roles and QKD initiator/responder roles, which limits flexibility.

---

## 3. Topology Constraints

### 3.1 Point-to-Point Limitation

**Constraint:** The current design assumes a 1:1 topology:

```
Node A (MASTER) ←→ Node B (SLAVE)
```

**Impact:**
- Each node can only communicate with one peer using a single set of QKD variables
- Multi-peer scenarios require multiple SAE instances or complex variable management
- Key Stream concept (ETSI 014 feature for 1:N topologies) is not supported

### 3.2 Configuration Scalability

**Constraint:** For N nodes in a mesh topology, each node would need:
- N-1 sets of peer variables
- Complex logic to select the correct variable set per connection
- No built-in mechanism for dynamic peer selection

---

## 4. Certificate Management

### 4.1 Dual Certificate Sets

**Constraint:** The implementation requires separate certificate configurations for `MASTER` and `SLAVE` roles:

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
}
```

**Impact:**
- Certificate rotation requires updating both sets
- Deployment complexity increases
- In practice, both sets often point to the same certificates

### 4.2 KME Authentication

**Constraint:** Certificates are used for:
- Authenticating the SAE to its local KME (mTLS)
- Not for peer-to-peer authentication (handled by TLS layer)

**Current Design:** The dual certificate requirement suggests peer authentication, which is misleading.

---

## 5. API Call Patterns

### 5.1 KME and SAE Selection

**Constraint:** API calls must select the correct KME hostname and peer SAE ID based on role:

```c
// Initiator (master)
uint32_t ret = GET_STATUS(master_kme, slave_sae, &status);

// Responder (slave)
uint32_t ret = GET_KEY_WITH_IDS(slave_kme, master_sae, &key_ids, &container);
```

**Impact:**
- Each node connects to its own KME but uses different variable names depending on role
- The `dest_uri` (peer's KME) is loaded but never used in ETSI 014 API calls
- Variable naming does not reflect actual usage patterns

---

## 6. Backward Compatibility Considerations

### 6.1 Variable Name Changes

**Constraint:** Any future changes to environment variable names must maintain backward compatibility with existing deployments.

**Considerations:**
- Existing scripts and configurations use current variable names
- Deprecation warnings would be needed for smooth migration
- Fallback logic would increase code complexity

### 6.2 API Stability

**Constraint:** The C API signatures are used by downstream projects (e.g., qkd-kem-provider, strongSwan integration).

**Impact:**
- Changes to function signatures require coordinated updates across projects
- Internal refactoring must preserve external API compatibility
- New features should be additive rather than breaking

---

## 7. Testing and Validation

### 7.1 Backend-Specific Constraints

**Constraint:** Different backends have different requirements:

- **Simulated backend**: No external dependencies, works with any variable configuration
- **Cerberis XGR / QuKayDee backends**: Require valid certificates and network connectivity
- **Python client backend**: Requires Python runtime and specific module installation

**Impact:**
- Test environments must be configured differently per backend
- CI/CD pipelines need backend-specific test paths
- Documentation must cover multiple configuration scenarios

### 7.2 Full Integration Testing

**Constraint:** Full end-to-end testing requires:
- Two SAE instances with proper certificate configuration
- Access to KME endpoints (real or simulated)
- Coordinated key exchange operations

**Current Limitation:** Unit tests can only validate individual API calls, not complete key exchange flows.

---

## 8. Documentation and Usability

### 8.1 Configuration Complexity

**Constraint:** Users must understand:
- The distinction between `MASTER` and `SLAVE` variables
- How these map to their specific deployment topology
- Which variables to use for TLS client vs. server roles
- Certificate requirements for KME authentication

**Impact:**
- Steep learning curve for new users
- Common misconfiguration scenarios
- Need for detailed deployment guides

### 8.2 Error Messages

**Constraint:** Current error messages may not clearly indicate:
- Which variable is missing or misconfigured
- Whether the issue is with `MASTER` or `SLAVE` configuration
- The relationship between TLS roles and QKD variable selection

---

## 9. Future Considerations

### 9.1 Potential Improvements

While this document focuses on current constraints, potential areas for future improvement include:

1. **Simplified variable naming** that reflects actual usage (e.g., `MY_*` and `PEER_*`)
2. **Role-independent certificate configuration** (single set per node)
3. **Key Stream support** for 1:N topologies
4. **Dynamic peer selection** without environment variable changes
5. **Improved error diagnostics** with context-aware messages

### 9.2 Compatibility Requirements

Any future improvements must:
- Maintain backward compatibility with existing deployments
- Provide clear migration paths
- Include comprehensive testing across all backends
- Update documentation and examples

---

## 10. References

- **ETSI GS QKD 004 V2.1.1 (2020-08)**: Application Interface
- **ETSI GS QKD 014 V1.1.1 (2019-02)**: Protocol and data format of REST-based key delivery API
- **Project Repository**: https://github.com/qursa-uc3m/qkd-etsi-api-c-wrapper

