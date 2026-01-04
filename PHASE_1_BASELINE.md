# Phase 1 Baseline: Current State Documentation

**Created:** 2026-01-04
**Purpose:** Document current state before Go 1.25 migration
**Branch:** claude/list-project-cves-11izT

---

## Go Module Baseline

### Module: dataset-operator
- **Location:** `src/dataset-operator/`
- **Current Go Version:** 1.22.0
- **Toolchain:** go1.22.2
- **Backup:** `.migration-backup/phase1-baseline/dataset-operator_go.mod`
- **Key Dependencies:**
  - k8s.io/api v0.30.0
  - k8s.io/client-go v0.30.0
  - sigs.k8s.io/controller-runtime v0.18.0
  - golang.org/x/net v0.23.0 ⚠️ CVE-2024-45338
- **Status:** Ready for migration to Go 1.25

### Module: csi-s3
- **Location:** `src/csi-s3/`
- **Current Go Version:** 1.15
- **Backup:** `.migration-backup/phase1-baseline/csi-s3_go.mod`
- **Key Dependencies:**
  - golang.org/x/net v0.23.0 ⚠️ CVE-2024-45338
  - google.golang.org/grpc v1.56.3
  - github.com/minio/minio-go v0.0.0-20190430232750-10b3660b8f09
- **Status:** **HIGH RISK** - 10 version jump (1.15 → 1.25)

### Module: csi-driver-nfs
- **Location:** `src/csi-driver-nfs/`
- **Current Go Version:** 1.12
- **Backup:** `.migration-backup/phase1-baseline/csi-driver-nfs_go.mod`
- **Key Dependencies:**
  - golang.org/x/net v0.23.0 ⚠️ CVE-2024-45338
  - google.golang.org/grpc v1.56.3
  - k8s.io/kubernetes v1.14.1 ⚠️ **EXTREMELY OUTDATED**
- **Status:** **CRITICAL RISK** - 13 version jump (1.12 → 1.25)

### Module: apiclient
- **Location:** `src/apiclient/`
- **Current Go Version:** 1.16
- **Backup:** `.migration-backup/phase1-baseline/apiclient_go.mod`
- **Key Dependencies:**
  - k8s.io/apimachinery v0.24.3
  - k8s.io/client-go v0.24.3
- **Status:** **HIGH RISK** - 9 version jump (1.16 → 1.25)

### Module: ceph-cache-plugin
- **Location:** `plugins/ceph-cache-plugin/`
- **Current Go Version:** 1.13
- **Backup:** `.migration-backup/phase1-baseline/ceph-cache-plugin_go.mod`
- **Key Dependencies:**
  - k8s.io/* packages pinned to v1.16.2
  - github.com/operator-framework/operator-sdk v0.16.0 ⚠️ **OUTDATED**
  - github.com/rook/rook v1.3.3
- **Status:** **CRITICAL RISK** - 12 version jump (1.13 → 1.25)

---

## Dockerfile Baseline

### dataset-operator Dockerfile
- **File:** `src/dataset-operator/Dockerfile`
- **Build Base:** `golang:1.22`
- **Runtime Base:** `gcr.io/distroless/static:nonroot`
- **Backup:** Committed to git

### csi-s3 Dockerfile
- **File:** `src/csi-s3/cmd/s3driver/Dockerfile`
- **Build Base:** `golang:1.23-alpine3.20`
- **Runtime Base:** `alpine:3.20`
- **Additional Components:** goofys
- **Backup:** Committed to git

### csi-driver-nfs Dockerfile
- **File:** `src/csi-driver-nfs/Dockerfile`
- **Build Base:** `golang:1.22-bookworm`
- **Runtime Base:** `mirror.gcr.io/debian:bookworm-slim`
- **Backup:** Committed to git

---

## Build Process Documentation

### Component Build Process
**Script:** `build-tools/build_components.sh`
**Components:**
- dataset-operator
- generate-keys

**Process:**
1. Detects Docker/Podman
2. Creates buildx context (if needed)
3. Builds multi-arch images (amd64, arm64, ppc64le)
4. Optionally pushes to registry

**Current Working:** ✅ Yes

### CSI Plugin Build Process
**Script:** `build-tools/build_csi_plugins.sh`
**Plugins:**
- csi-s3
- csi-driver-nfs

**Process:**
1. Detects Docker/Podman
2. Creates buildx context (if needed)
3. Builds multi-arch images (amd64, arm64, ppc64le)
4. Optionally pushes to registry

**Current Working:** ✅ Yes

---

## CI/CD Baseline

### GitHub Actions
- **Workflows:** 5 workflows updated in Phase 4
- **Action Versions:** All modernized (v3-v4)
- **Status:** ✅ Ready for Go 1.25

### GitLab CI
- **File:** `src/csi-s3/.gitlab-ci.yml`
- **Test Image:** ctrox/csi-s3:test
- **Status:** ✅ Documented

---

## Dependency Summary

### Critical CVEs Requiring Fix
1. **CVE-2024-45338** - golang.org/x/net v0.23.0
   - Affects: dataset-operator, csi-s3, csi-driver-nfs
   - Fix: Update to v0.33.0+

2. **CVE-2024-24791** - Go runtime (net/http)
   - Affects: All modules using Go < 1.22.5
   - Fix: Update to Go 1.25

3. **CVE-2024-24790** - Go runtime (net/netip)
   - Affects: All modules
   - Fix: Update to Go 1.25

### Outdated Dependencies
1. **k8s.io/kubernetes v1.14.1** (csi-driver-nfs)
   - Released: 2019
   - Target: v1.30.0+

2. **operator-sdk v0.16.0** (ceph-cache-plugin)
   - Current: v1.36+
   - Requires: Major code rewrite

---

## Test Baseline

### Unit Tests Status
```bash
# To establish baseline before migration
cd src/dataset-operator && go test ./...
cd src/csi-s3 && go test ./...
cd src/csi-driver-nfs && go test ./...
cd plugins/ceph-cache-plugin && go test ./...
```

**Status:** To be run in Phase 1 testing phase

### Integration Tests
**Workflow:** `.github/workflows/testing.yml`
**Tests:**
- MinIO backend
- Dataset creation
- Read/write operations
- KinD cluster deployment

**Status:** Baseline to be established

---

## File Inventory

### Files Requiring Updates (Phase 2-3)

**Go Modules (5):**
- ✅ `src/dataset-operator/go.mod`
- ✅ `src/csi-s3/go.mod`
- ✅ `src/csi-driver-nfs/go.mod`
- ✅ `src/apiclient/go.mod`
- ✅ `plugins/ceph-cache-plugin/go.mod`

**Dockerfiles (3+):**
- ✅ `src/dataset-operator/Dockerfile`
- ✅ `src/csi-s3/cmd/s3driver/Dockerfile`
- ✅ `src/csi-driver-nfs/Dockerfile`
- ✅ `src/generate-keys/Dockerfile`
- ✅ `src/cos-uploader/Dockerfile`

**GitHub Actions (5):**
- ✅ Already updated in Phase 4

**Build Scripts (7+):**
- ✅ `build-tools/build_components.sh` (updated in Phase 4)
- ✅ `build-tools/build_csi_plugins.sh` (updated in Phase 4)
- ✅ All multiarch build scripts

**Documentation (pending):**
- [ ] README.md
- [ ] CONTRIBUTING.md
- [ ] docs/

---

## Risk Assessment

### Low Risk (Ready for Migration)
- ✅ **dataset-operator** - 3 version jump, modern K8s deps
- ✅ **CI/CD** - Already updated and tested

### High Risk (Requires Careful Planning)
- ⚠️ **csi-s3** - 10 version jump, minio-go dependency
- ⚠️ **apiclient** - 9 version jump, K8s version alignment

### Critical Risk (Requires Extensive Testing)
- 🔴 **csi-driver-nfs** - 13 version jump, K8s 1.14 → 1.30
- 🔴 **ceph-cache-plugin** - 12 version jump, operator-sdk rewrite

---

## Migration Order (Recommended)

Based on risk assessment and dependencies:

1. **dataset-operator** (Go 1.22 → 1.25)
   - Lowest risk
   - Establishes pattern for others

2. **apiclient** (Go 1.16 → 1.25)
   - Depends on dataset-operator for K8s version alignment

3. **csi-s3** (Go 1.15 → 1.25)
   - Medium complexity
   - Critical CVE fix needed

4. **csi-driver-nfs** (Go 1.12 → 1.25)
   - Highest complexity
   - Consider incremental upgrade

5. **ceph-cache-plugin** (Go 1.13 → 1.25)
   - Major operator-sdk rewrite
   - May require significant time

---

## Backup Locations

All current state backed up to:
- **Directory:** `.migration-backup/phase1-baseline/`
- **Go Modules:** 5 go.mod files + 4 go.sum files
- **Size:** ~520KB
- **Git Commit:** f671068 (current HEAD)

---

## Environment

- **Go Version (Current):** 1.24.7
- **Go Version (Target):** 1.25.5
- **Platform:** linux/amd64
- **Branch:** claude/list-project-cves-11izT
- **Date:** 2026-01-04

---

## Next Steps (Phase 2)

1. Start with dataset-operator migration
2. Update go.mod to Go 1.25
3. Update Dockerfile to golang:1.25
4. Run go mod tidy
5. Fix any breaking changes
6. Test thoroughly
7. Proceed to next module

---

**Baseline Established**
**Ready for Phase 2 Migration**
