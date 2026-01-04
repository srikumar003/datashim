# Go 1.25 Migration Plan for Datashim Project

**Created:** 2026-01-04
**Target Go Version:** 1.25.5 (latest stable as of December 2025)
**Status:** Planning Phase

---

## Executive Summary

This document outlines the comprehensive plan to upgrade all Go modules in the Datashim project from their current versions (ranging from Go 1.12 to Go 1.22) to Go 1.25.5. This is a **major upgrade** that will address multiple CVE vulnerabilities and bring the project to the latest stable Go release.

### Current State

| Module | Current Go Version | Version Gap | Risk Level |
|--------|-------------------|-------------|------------|
| ceph-cache-plugin | 1.13 | 12 versions | **CRITICAL** |
| csi-driver-nfs | 1.12 | 13 versions | **CRITICAL** |
| csi-s3 | 1.15 | 10 versions | **HIGH** |
| apiclient | 1.16 | 9 versions | **HIGH** |
| dataset-operator | 1.22.0 | 3 versions | **MEDIUM** |

### Benefits of Migration

1. **Security:** Fixes CVE-2024-24791, CVE-2024-24790, CVE-2024-24787, CVE-2023-45288, and other Go runtime vulnerabilities
2. **Performance:** Significant compiler and runtime improvements across 13 versions
3. **Features:** Access to generics, improved error handling, better tooling
4. **Support:** Older Go versions are EOL and no longer receive security updates
5. **Dependencies:** Enables updating to newer versions of dependencies that require modern Go

---

## Migration Strategy

### Phase 1: Preparation and Analysis (Estimated: 1-2 days)

**Objective:** Prepare the environment and understand the full scope of changes needed.

#### Tasks:

1. **Create a dedicated branch**
   ```bash
   git checkout -b feature/go-1.25-migration
   ```

2. **Set up Go 1.25.5 development environment**
   - Install Go 1.25.5 locally
   - Update PATH and GOROOT
   - Verify installation: `go version`

3. **Backup and document current state**
   - Create snapshot of all go.mod and go.sum files
   - Document current build process
   - Run full test suite to establish baseline

4. **Identify all files requiring updates**
   - [x] 5 go.mod files
   - [x] 5+ Dockerfiles
   - [x] GitHub Actions workflows
   - [x] GitLab CI configuration
   - [x] Build scripts
   - [ ] Documentation (README, contributing guides)

### Phase 2: Module-by-Module Migration (Estimated: 3-5 days)

**Strategy:** Migrate in order from newest to oldest, starting with dataset-operator (1.22) and ending with csi-driver-nfs (1.12).

#### 2.1 dataset-operator (Go 1.22 → 1.25) **[LOW RISK]**

**Files to Update:**
- `src/dataset-operator/go.mod`
- `src/dataset-operator/Dockerfile`
- `src/dataset-operator/build/Dockerfile`
- `src/dataset-operator/build_multiarch_dataset_operator.sh`

**Breaking Changes to Address:**
- Go 1.22: For-loop variable scoping changes (already using 1.22)
- Go 1.23: Minor stdlib changes
- Go 1.24: Generic type aliases
- Go 1.25: Compiler nil check fixes

**Steps:**
1. Update `go.mod`: Change `go 1.22.0` to `go 1.25`
2. Update toolchain: Add `toolchain go1.25.5`
3. Update Dockerfile: `FROM golang:1.22` → `FROM golang:1.25`
4. Run `go mod tidy`
5. Fix any deprecation warnings
6. Update dependencies (especially golang.org/x/net to v0.33.0+)
7. Run tests: `go test ./...`
8. Build and verify: `make build`

**Expected Issues:**
- Minimal - only 3 version jump
- Possible linter warnings from stricter checks
- May need to update controller-runtime to compatible version

#### 2.2 apiclient (Go 1.16 → 1.25) **[HIGH RISK]**

**Files to Update:**
- `src/apiclient/go.mod`

**Breaking Changes to Address:**
- Go 1.17: `go get` deprecation for installing binaries
- Go 1.18: Generics support (backward compatible)
- Go 1.20-1.25: Same as dataset-operator

**Steps:**
1. Update `go.mod`: Change `go 1.16` to `go 1.25`
2. Add `toolchain go1.25.5`
3. Run `go mod tidy` - expect significant dependency updates
4. Update k8s.io dependencies to versions compatible with Go 1.25
5. Fix any build errors related to deprecated APIs
6. Test integration with dataset-operator
7. Verify API compatibility

**Expected Issues:**
- Dependency conflicts with k8s.io/client-go and k8s.io/apimachinery
- May need to update from v0.24.3 to v0.30.0+ to match dataset-operator
- Replace directives may need adjustment

#### 2.3 csi-s3 (Go 1.15 → 1.25) **[HIGH RISK]**

**Files to Update:**
- `src/csi-s3/go.mod`
- `src/csi-s3/cmd/s3driver/Dockerfile`
- `src/csi-s3/test/Dockerfile`
- `src/csi-s3/build_and_push_multiarch_csis3.sh`
- `src/csi-s3/.gitlab-ci.yml`

**Breaking Changes to Address:**
- Go 1.15: Context API changes (WithValue with nil parent panics)
- Go 1.16: GO111MODULE=on by default, `go get`/`go install` separation
- Go 1.17-1.25: All subsequent breaking changes

**Steps:**
1. Update `go.mod`: Change `go 1.15` to `go 1.25`
2. Add `toolchain go1.25.5`
3. Update Dockerfile: `FROM golang:1.23-alpine3.20` → `FROM golang:1.25-alpine3.21`
4. Run `go mod tidy` - expect major dependency updates
5. Update golang.org/x/net from v0.23.0 to v0.33.0+ (CVE fix)
6. Update google.golang.org/grpc to latest
7. Update minio-go to latest version
8. Review and update CSI spec dependency
9. Fix any deprecated API usage
10. Update goofys build in Dockerfile if needed
11. Run full test suite
12. Test with actual S3 backend

**Expected Issues:**
- **CRITICAL:** golang.org/x/net v0.23.0 has CVE-2024-45338
- minio-go may have breaking API changes
- CSI spec compatibility issues
- Context usage may need review (nil parent checks)
- Build process for goofys may need updates

#### 2.4 csi-driver-nfs (Go 1.12 → 1.25) **[CRITICAL RISK]**

**Files to Update:**
- `src/csi-driver-nfs/go.mod`
- `src/csi-driver-nfs/Dockerfile`
- `src/csi-driver-nfs/build_and_push_multiarch_csinfs.sh`

**Breaking Changes to Address:**
- Go 1.13: Module mode changes, TLS 1.3 enabled
- Go 1.14-1.25: **ALL** breaking changes from 13 versions
- This is the most challenging migration

**Steps:**
1. **Pre-migration audit:**
   - Review all code for Go 1.12-specific patterns
   - Document custom build tags or special configurations
   - Check for vendor directory (if exists, remove and use modules)

2. **Initial update:**
   - Update `go.mod`: Change `go 1.12` to `go 1.25`
   - Add `toolchain go1.25.5`
   - Update Dockerfile: `FROM golang:1.22-bookworm` → `FROM golang:1.25-bookworm`

3. **Dependency resolution:**
   - Run `go mod tidy` - expect many errors
   - Update golang.org/x/net from v0.23.0 to v0.33.0+ (CVE fix)
   - Update google.golang.org/grpc to latest
   - Update k8s.io/kubernetes dependency (currently 1.14.1 - **extremely outdated**)
   - May need to replace k8s.io/kubernetes with specific k8s.io/* modules

4. **Code fixes:**
   - Fix context API usage (Go 1.15 breaking change)
   - Update any deprecated stdlib calls
   - Fix for-loop variable scoping (Go 1.22 change)
   - Review TLS configuration (multiple changes across versions)

5. **Testing:**
   - Unit tests: `go test ./...`
   - Integration tests with NFS backend
   - Test CSI driver operations
   - Verify Kubernetes compatibility

**Expected Issues:**
- **CRITICAL:** k8s.io/kubernetes v1.14.1 is from 2019 - massive API changes
- **CRITICAL:** golang.org/x/net v0.23.0 has CVE-2024-45338
- Context usage throughout codebase may need updates
- CSI library compatibility issues
- Build system may need significant changes
- Vendor directory conflicts (if present)
- May need to rewrite portions of code due to deprecated K8s APIs

#### 2.5 ceph-cache-plugin (Go 1.13 → 1.25) **[CRITICAL RISK]**

**Files to Update:**
- `plugins/ceph-cache-plugin/go.mod`
- `plugins/ceph-cache-plugin/Dockerfile`
- `plugins/ceph-cache-plugin/build/Dockerfile`

**Breaking Changes to Address:**
- Go 1.14-1.25: 12 versions of breaking changes
- Pinned to Kubernetes 1.16.2 - needs major update

**Steps:**
1. **Pre-migration audit:**
   - Review extensive replace directives (pinned to k8s 1.16.2)
   - Document operator-framework/operator-sdk v0.16.0 compatibility
   - Check rook/rook v1.3.3 compatibility

2. **Update process:**
   - Update `go.mod`: Change `go 1.13` to `go 1.25`
   - Add `toolchain go1.25.5`
   - Update Dockerfiles

3. **Dependency overhaul:**
   - Remove outdated replace directives
   - Update k8s.io/* packages from 1.16.2 to 1.30.0+
   - Update operator-sdk to v1.x (current is v0.16.0)
   - Update rook/rook to compatible version
   - Update controller-runtime to v0.18.0+

4. **Code modernization:**
   - Update to new operator-sdk patterns (significant changes from v0.16 to v1.x)
   - Fix deprecated Kubernetes API usage
   - Update logging (go-logr v0.1.0 → v1.4.1+)

5. **Testing:**
   - Unit tests
   - Integration with Ceph backend
   - Operator deployment tests

**Expected Issues:**
- **CRITICAL:** operator-sdk v0.16.0 → v1.x is a major rewrite
- **CRITICAL:** Kubernetes 1.16.2 → 1.30.0 has massive API changes
- rook/rook compatibility issues
- All replace directives need reconsideration
- May require significant code rewrites

### Phase 3: Container Base Images Update (Estimated: 1 day)

**Objective:** Update all Dockerfile base images to use Go 1.25 and latest OS versions.

#### Updates Needed:

1. **Alpine-based images:**
   - Current: `golang:1.23-alpine3.20`
   - Target: `golang:1.25-alpine3.21`
   - Runtime: `alpine:3.20` → `alpine:3.21`

2. **Debian-based images:**
   - Current: `golang:1.22-bookworm`
   - Target: `golang:1.25-bookworm`
   - Runtime: `debian:bookworm-slim` (already latest)

3. **Distroless images:**
   - Current: `gcr.io/distroless/static:nonroot`
   - Target: `gcr.io/distroless/static:nonroot` (no change, already minimal)

**Security Benefits:**
- Fixes Alpine CVE-2024-39689, CVE-2024-12797, CVE-2025-58098, etc.
- Updates to latest Debian Bookworm patches (CVE-2024-12084 fixed)

### Phase 4: CI/CD Pipeline Updates (Estimated: 1 day)

**Files to Update:**

1. **GitHub Actions:**
   - `.github/workflows/testing.yml` - Update build scripts
   - `.github/workflows/push-updated-csi-plugins.yml`
   - `.github/workflows/push-updated-images.yml`
   - `.github/workflows/release.yml`

2. **GitLab CI:**
   - `src/csi-s3/.gitlab-ci.yml` - Update test image

3. **Build Scripts:**
   - `build-tools/build_components.sh` - Verify Go version requirements
   - `build-tools/build_csi_plugins.sh` - Verify Go version requirements
   - All `build_multiarch_*.sh` scripts

**Changes Required:**
- No direct Go version references in CI (uses Dockerfiles)
- Verify docker buildx compatibility
- Update any Go-specific caching strategies
- Update test environments

### Phase 5: Testing and Validation (Estimated: 2-3 days)

**Objective:** Comprehensive testing to ensure no regressions.

#### Test Levels:

1. **Unit Tests:**
   ```bash
   cd src/dataset-operator && go test ./...
   cd src/apiclient && go test ./...
   cd src/csi-s3 && go test ./...
   cd src/csi-driver-nfs && go test ./...
   cd plugins/ceph-cache-plugin && go test ./...
   ```

2. **Build Tests:**
   ```bash
   cd build-tools
   ./build_components.sh
   ./build_csi_plugins.sh
   ```

3. **Integration Tests:**
   - Run GitHub Actions testing workflow
   - Test on KinD cluster (as per .github/workflows/testing.yml)
   - Create Dataset and verify read/write operations

4. **Manual Testing:**
   - Deploy to test Kubernetes cluster
   - Test S3 backend with csi-s3
   - Test NFS backend with csi-driver-nfs
   - Test Ceph backend with ceph-cache-plugin
   - Verify dataset-operator webhook functionality
   - Test all Dataset CRD operations

5. **Performance Testing:**
   - Compare build times (Go 1.25 should be similar to 1.22)
   - Benchmark runtime performance
   - Check binary sizes

6. **Security Validation:**
   - Run `govulncheck` on all modules
   - Verify CVE-2024-45338 is resolved (golang.org/x/net)
   - Verify no new vulnerabilities introduced
   - Scan container images with Trivy/Grype

### Phase 6: Documentation and Rollout (Estimated: 1 day)

**Documentation Updates:**

1. **README.md:**
   - Update Go version requirements
   - Update build instructions
   - Update development setup guide

2. **CONTRIBUTING.md:**
   - Update Go installation instructions
   - Update local development setup

3. **Release Notes:**
   - Document Go version upgrade
   - List fixed CVEs
   - Note any breaking changes for users
   - Migration guide for developers

4. **Dockerfiles:**
   - Add comments about Go version requirements

**Rollout Strategy:**

1. **Merge to feature branch:** Complete all modules
2. **Create PR:** Comprehensive description with testing results
3. **Review:** Team review focusing on:
   - Breaking changes
   - Test coverage
   - Security improvements
4. **Merge to main:** After approval
5. **Tag release:** New version with Go 1.25
6. **Announce:** Communicate upgrade to users/contributors

---

## Risk Mitigation

### High-Risk Areas

1. **csi-driver-nfs and ceph-cache-plugin:**
   - **Risk:** 12-13 version jump with major K8s API changes
   - **Mitigation:**
     - Allocate extra time for these modules
     - Consider incremental upgrade (1.12 → 1.18 → 1.25)
     - Extensive testing with actual backends
     - Keep original branch for rollback

2. **Kubernetes API Compatibility:**
   - **Risk:** K8s 1.14/1.16 → 1.30 has removed/changed many APIs
   - **Mitigation:**
     - Review k8s 1.14-1.30 API deprecation guides
     - Update CRD definitions if needed
     - Test with multiple K8s versions (1.28, 1.29, 1.30)

3. **Dependency Conflicts:**
   - **Risk:** Transitive dependencies may conflict
   - **Mitigation:**
     - Use `go mod graph` to visualize dependencies
     - Resolve conflicts with replace directives if needed
     - Consider using `go mod vendor` for reproducibility

4. **Operator SDK Migration:**
   - **Risk:** ceph-cache-plugin uses v0.16.0, current is v1.36+
   - **Mitigation:**
     - Follow operator-sdk migration guides
     - May need significant code rewrites
     - Consider if plugin is still needed/used

### Rollback Plan

1. **Branch strategy:** Keep feature branch separate until fully tested
2. **Git tags:** Tag before starting migration
3. **Docker images:** Keep old images with previous tags
4. **Documentation:** Maintain both old and new setup instructions during transition

---

## Timeline and Resource Allocation

### Estimated Total Time: 8-12 days

| Phase | Duration | Dependencies |
|-------|----------|--------------|
| 1. Preparation | 1-2 days | None |
| 2. Module Migration | 3-5 days | Phase 1 |
| 3. Container Updates | 1 day | Phase 2 |
| 4. CI/CD Updates | 1 day | Phase 2, 3 |
| 5. Testing | 2-3 days | All previous |
| 6. Documentation | 1 day | All previous |

### Resource Requirements

- **Developer Time:** 1-2 developers full-time
- **Infrastructure:** Test Kubernetes cluster with S3, NFS, Ceph backends
- **Tools:** Go 1.25.5, Docker, kubectl, kind

---

## Success Criteria

1. ✅ All modules build successfully with Go 1.25.5
2. ✅ All unit tests pass
3. ✅ Integration tests pass on KinD cluster
4. ✅ No new security vulnerabilities (govulncheck clean)
5. ✅ CVE-2024-45338 resolved (golang.org/x/net updated)
6. ✅ All Dockerfiles use Go 1.25 base images
7. ✅ CI/CD pipelines pass
8. ✅ Documentation updated
9. ✅ Manual testing confirms functionality with all backends
10. ✅ Performance is equal or better than current version

---

## References

- [Go 1.25 Release Notes](https://go.dev/doc/go1.25)
- [Go 1.24 Release Notes](https://go.dev/doc/go1.24)
- [Go Release History](https://go.dev/doc/devel/release)
- [Go Toolchains Documentation](https://go.dev/doc/toolchain)
- [GODEBUG for Backwards Compatibility](https://go.dev/doc/godebug)
- [Kubernetes API Deprecation Guide](https://kubernetes.io/docs/reference/using-api/deprecation-guide/)
- [CVE-2024-45338 Details](https://pkg.go.dev/vuln/GO-2024-3333)

---

## Next Steps

1. **Review this plan** with the team
2. **Allocate resources** and set timeline
3. **Create feature branch:** `feature/go-1.25-migration`
4. **Begin Phase 1:** Preparation and environment setup
5. **Daily standups** during migration to track progress
6. **Continuous testing** throughout the process

---

## Notes and Considerations

### Incremental vs. Direct Upgrade

For modules with large version gaps (csi-driver-nfs, ceph-cache-plugin), consider incremental upgrades:
- **Option A (Direct):** Go 1.12 → Go 1.25 in one step
  - Faster but higher risk
  - Many breaking changes at once

- **Option B (Incremental):** Go 1.12 → 1.16 → 1.20 → 1.25
  - Slower but safer
  - Easier to identify which change caused issues
  - Recommended for csi-driver-nfs and ceph-cache-plugin

### GODEBUG Settings

If certain breaking changes cause issues, temporary GODEBUG settings can maintain old behavior:
```go
GODEBUG=http2client=0  // Disable HTTP/2
GODEBUG=tls13=0        // Disable TLS 1.3
```

However, these should only be temporary workarounds.

### Vendor Directory

If any module uses a vendor directory:
1. Remove it: `rm -rf vendor/`
2. Clean go.mod: `go mod tidy`
3. Rely on module cache instead

This ensures reproducible builds and clearer dependency management.

---

**Document Version:** 1.0
**Last Updated:** 2026-01-04
**Author:** Claude (AI Assistant)
**Status:** Ready for Team Review
