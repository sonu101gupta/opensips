# OpenSIPS Upgrade from v3.2.6 to 3.6.2 - Summary

## ✅ Successfully Completed

### Sandbox Deployment (us-sandbox-cluster)
- **Namespace:** opensips-testing
- **Image:** `536612919621.dkr.ecr.ap-south-1.amazonaws.com/testing/sonu:3.6.2`
- **Status:** ✅ RUNNING (1/1 Ready)
- **OpenSIPS Version:** 3.6.2 confirmed
- **Prometheus Metrics:** ✅ Working on port 32762

---

## Key Issues Fixed

### 1. **Dockerfile Fixes** (`/Users/sonugupta/Skit/opensips-mrcp/Dockerfile`)
✅ Fixed working directory from `/opensips/` to `/usr/src/opensips-3.6.2/`
✅ Added COPY commands for `configs/` and `dbtext/` directories
✅ Removed deprecated `children` parameter from template
✅ Added `make modules` to compile all default modules
✅ Removed `proto_tcp` and `proto_udp` from EXTRA_MODULES (they're built-in)
✅ Removed `avpops` from EXTRA_MODULES (compilation fails in 3.6.2)

### 2. **Configuration Changes for OpenSIPS 3.6.2**

**Critical Syntax Changes:**
- ✅ **Proto Modules:** Must load using `loadmodule "proto_udp"` and `loadmodule "proto_tcp"` **WITHOUT .so extension** (built into core)
- ✅ **Socket Syntax:** Use `socket=` (not `listen=` which is deprecated)
- ✅ **Removed:** `children` parameter (deprecated)
- ✅ **Removed:** `loadmodule "proto_udp.so"` and `loadmodule "proto_tcp.so"` (these don't exist as .so files)
- ✅ **Removed:** `loadmodule "avpops.so"` (not compiled in our build)
- ✅ **Updated:** envsubst command to remove `$NUM_CHILDREN` reference

**Deprecated Parameters Warnings (still work but should update):**
- `log_stderror` → use `stderror_enabled` and `syslog_enabled`
- `log_facility` → use `syslog_facility`

### 3. **Database Schema Migrations**

**Table Version Updates Required:**
| Table | Old Version | New Version | Schema Changes |
|-------|-------------|-------------|----------------|
| dispatcher | 8 | 9 | Added `probe_mode(int)` column |
| dialog | 10 | 12 | Changed `script_flags` from int to string,null; Added `rt_on_answer`, `rt_on_timeout`, `rt_on_hangup` |
| load_balancer | 2 | 3 | Added `attrs(string,null)` column |
| registrant | 1 | 3 | Added `cluster_shtag(string,null)` and `state(int)` columns |
| b2b_entities | 1 | 2 | Schema updated |
| b2b_logic | 3 | 5 | Schema updated |
| pua | 8 | 9 | Schema updated |
| domain | 3 | 4 | Schema updated |
| dr_carriers | 2 | 3 | Schema updated |
| dr_rules | 3 | 4 | Schema updated |

✅ All version files and schema files have been updated.

---

## Files Modified

### In `/Users/sonugupta/Skit/opensips-mrcp/` (Docker Image Source):
1. `Dockerfile` - Fixed directory structure, removed children parameter
2. `configs/opensips.cfg` - Removed children=9
3. `configs/opensips.cfg.template` - Removed children=${NUM_CHILDREN}
4. `dbtext/version` - Updated all table versions to 3.6.2
5. `dbtext/dispatcher` - Updated schema to version 9
6. `dbtext/load_balancer` - Updated schema to version 3 with attrs column
7. `dbtext/dialog` - Updated schema to version 12
8. `dbtext/registrant` - Updated schema to version 3

### In `/Users/sonugupta/Downloads/opensips-mrcp/` (Helm Chart):
1. `templates/deployment.yaml` - Removed `$NUM_CHILDREN` from envsubst
2. `overrides/us-sandbox-testing.yaml` - New file created for sandbox testing
3. `overrides/icici-bank-ib-uat.yaml` - Updated image tag to `3.6.2-icici-lb`

---

## Next Steps for ICICI Production Deployment

### 1. Build ICICI Image
You need to build and push the image to your ICICI registry:

```bash
cd /Users/sonugupta/Skit/opensips-mrcp

# Build for ICICI
docker build --platform linux/amd64 -t opensips-mrcp:3.6.2-icici-lb -f Dockerfile .

# Tag for ICICI registry
docker tag opensips-mrcp:3.6.2-icici-lb docker-registry:3000/opensips-mrcp:3.6.2-icici-lb

# Push to ICICI registry (authenticate first)
docker push docker-registry:3000/opensips-mrcp:3.6.2-icici-lb
```

### 2. Deploy to ICICI UAT
```bash
cd /Users/sonugupta/Downloads/opensips-mrcp
helm upgrade opensips-mrcp . -f overrides/icici-bank-ib-uat.yaml --namespace <icici-namespace>
```

### 3. Validation Checklist
- [ ] Pod reaches Running state (1/1 Ready)
- [ ] Check logs for any errors: `kubectl logs -n <namespace> <pod-name>`
- [ ] Verify OpenSIPS version: `kubectl exec <pod> -- /usr/src/opensips-3.6.2/opensips -V`
- [ ] Test SIP connectivity on port 9060 (UDP/TCP)
- [ ] Check Prometheus metrics: `curl http://<pod-ip>:32762/metrics`
- [ ] Verify load balancer targets are reachable
- [ ] Test call flow end-to-end

---

## Configuration Template for 3.6.2

**Required modules to load:**
```
# Built-in transport protocols (no .so extension!)
loadmodule "proto_udp"
loadmodule "proto_tcp"

# Regular modules (with .so extension)
loadmodule "statistics.so"
loadmodule "signaling.so"
loadmodule "sl.so"
loadmodule "options.so"
loadmodule "tm.so"
loadmodule "rr.so"
loadmodule "maxfwd.so"
loadmodule "sipmsgops.so"
loadmodule "mi_fifo.so"
loadmodule "db_text.so"
loadmodule "dispatcher.so"
loadmodule "acc.so"
loadmodule "dialog.so"
loadmodule "load_balancer.so"
loadmodule "rtpengine.so"
loadmodule "usrloc.so"
loadmodule "registrar.so"
loadmodule "uac_auth.so"
loadmodule "uac_registrant.so"
loadmodule "httpd.so"
loadmodule "prometheus.so"
```

**DO NOT load:**
- `proto_udp.so` / `proto_tcp.so` (don't exist as .so files)
- `avpops.so` (not compiled in our build)

---

## Troubleshooting Reference

### Common Errors and Solutions:

**1. "no transport protocol loaded"**
- **Cause:** Proto modules not loaded
- **Fix:** Add `loadmodule "proto_udp"` and `loadmodule "proto_tcp"` (without .so)

**2. "invalid version X for table Y found, expected Z"**
- **Cause:** Database schema mismatch after upgrade
- **Fix:** Update `dbtext/version` file with correct table versions
- **Fix:** Update table schema files to match 3.6.2 format

**3. "bad columns for table"**
- **Cause:** Table schema columns don't match expected schema
- **Fix:** Add missing columns (e.g., `attrs`, `probe_mode`, `cluster_shtag`, `state`)

**4. Platform mismatch errors**
- **Cause:** Building on ARM64 Mac for AMD64 servers
- **Fix:** Use `--platform linux/amd64` flag in docker build

---

## Image Details

### Sandbox Image
- **Repository:** `536612919621.dkr.ecr.ap-south-1.amazonaws.com/testing/sonu`
- **Tag:** `3.6.2`
- **Digest:** `sha256:6c39174a61f0ed873c8d01491a78a605ecd76580257cc4d1b27ac54d8a4492b2`
- **Platform:** linux/amd64
- **Status:** ✅ Tested and Working

### ICICI Image (To Be Built)
- **Repository:** `docker-registry:3000/opensips-mrcp`
- **Tag:** `3.6.2-icici-lb`
- **Source:** Same Dockerfile from `/Users/sonugupta/Skit/opensips-mrcp/`
- **Status:** Chart updated, ready to build and deploy

---

## Key Learnings

1. **OpenSIPS 3.6.2 proto modules are built into the core** - they need to be loaded with `loadmodule "proto_udp"` (no .so extension)
2. **Table schemas MUST match** - version numbers in `version` file must align with actual table structure
3. **Directory structure matters** - Image paths must match helm chart mount points (`/usr/src/opensips-3.6.2/`)
4. **Children parameter deprecated** - Use `udp_workers` and `tcp_workers` instead
5. **Platform-specific builds** - Always build for `linux/amd64` for AWS EC2 nodes

---

## Testing Results

### Sandbox Cluster Tests ✅
- Pod Status: **Running (1/1)**
- OpenSIPS Version: **3.6.2** ✅
- UDP Transport: **Active on 0.0.0.0:9060** ✅
- TCP Transport: **Active on 0.0.0.0:9060** ✅
- Prometheus Metrics: **Available on port 32762** ✅
- Config Generation: **Working** ✅
- Module Loading: **All modules loaded successfully** ✅

---

## Rollback Plan (If Needed)

If issues occur in ICICI:
1. Revert image tag in `overrides/icici-bank-ib-uat.yaml` back to `v3.2.6-icici-lb`
2. Run: `helm upgrade opensips-mrcp . -f overrides/icici-bank-ib-uat.yaml --namespace <namespace>`
3. Previous configuration will be restored

---

Generated: November 5, 2025

