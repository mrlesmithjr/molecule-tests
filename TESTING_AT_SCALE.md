# Testing Ansible at Scale with Molecule

## Overview

This document outlines strategies and lessons learned from implementing Molecule testing across multiple platforms and at scale.

## Scale Considerations

### What "Scale" Means in Molecule Testing

1. **Platform Coverage** - Testing across 15+ OS distributions simultaneously
2. **Test Complexity** - Comprehensive verification beyond basic assertions
3. **CI/CD Integration** - Parallel execution with matrix strategies
4. **Shared Pattern Architecture** - DRY principles to maintain large test suites

## Architecture Patterns for Scale

### 1. Shared Playbook Pattern

**Problem**: Duplicating test logic across 15+ scenarios becomes unmaintainable

**Solution**: Centralized playbooks in `molecule/shared/`
- Single source of truth for test logic
- Scenarios reference shared playbooks via relative paths
- Changes propagate to all scenarios automatically

**Benefits**:
- Reduces maintenance burden from O(n) to O(1)
- Ensures consistent testing across all platforms
- Easier to add new scenarios (just reference shared playbooks)

### 2. Multi-Driver Support

**Docker Driver** (Primary):
- Fast spin-up/teardown (seconds vs minutes)
- Suitable for most systemd-based testing
- Resource efficient for parallel execution
- Requires privileged mode and cgroup mounting for systemd support

**Vagrant Driver** (Special Cases):
- Full VM isolation for specific scenarios
- Better for testing that requires kernel modules or hardware simulation
- Slower but more realistic environment

**When to Choose**:
- Use Docker for 90% of scenarios (speed and efficiency)
- Use Vagrant when Docker limitations are hit (kernel modules, nested virtualization)

### 3. Platform-Specific Handling

**Challenge**: Different distros have different init systems, package managers, paths

**Solution**: Conditional task execution
```yaml
when: ansible_os_family == "Debian"
when: ansible_service_mgr == "systemd"
when: ansible_distribution_major_version | int >= 8
```

**Prepare Phase** handles platform differences:
- APT cache updates for Debian/Ubuntu
- YUM cache for RHEL 7 and below
- DNF cache for RHEL 8+
- APK cache for Alpine
- Python dependency installation per platform

### 4. Comprehensive Verification

**Beyond Basic Assertions**:
- Package installation verification (idempotency checks)
- User/group existence and configuration
- File permissions, ownership, and content validation
- Service state and enablement status
- Command availability and PATH verification

**Verification Strategy**:
- Use `check_mode` to verify idempotency
- Gather facts before assertions
- Test actual state, not just absence of errors
- Provide clear fail/success messages for debugging

## CI/CD Integration at Scale

### Parallel Execution Strategy

```yaml
strategy:
  fail-fast: false
  max-parallel: 4
  matrix:
    molecule_distro: [15+ scenarios]
```

**Key Decisions**:
- `fail-fast: false` - See all failures, not just first
- `max-parallel: 4` - Balance between speed and resource constraints
- Matrix expansion - Single workflow definition, multiple executions

### Optimization Techniques

1. **Caching**:
   - Pip cache for Python dependencies
   - Reduces installation time by 60-70%

2. **Pre-commit Integration**:
   - Run linters before heavy Molecule tests
   - Fail fast on syntax/style issues
   - Skip certain hooks in CI (`no-commit-to-branch`)

3. **Resource Management**:
   - Docker containers share host kernel (memory efficient)
   - Systemd cgroup mounting for service testing
   - Privileged mode only when necessary

## Practical Lessons Learned

### 1. The Return on Investment Question

**Initial Setup Cost**: High
- 16 scenarios × configuration time
- Shared playbook architecture design
- CI/CD pipeline configuration

**Ongoing Maintenance**: Low
- Shared playbooks mean single point of update
- Adding new scenario: < 10 minutes
- Platform updates handled in prepare phase

**Value Delivered**:
- Confidence in multi-platform compatibility
- Early detection of distro-specific issues
- Automated regression testing
- Documentation through tests

### 2. Common Pitfalls

**Issue**: Tests pass locally, fail in CI
- **Cause**: Docker daemon differences, networking, caching
- **Solution**: Reproduce with exact CI conditions locally

**Issue**: Systemd services fail in containers
- **Cause**: Missing cgroup mounts or wrong cgroupns_mode
- **Solution**: Use `cgroupns_mode: host` and mount cgroups

**Issue**: Slow test execution
- **Cause**: Serial execution, no caching
- **Solution**: Parallel matrix, pip caching, Docker layer caching

### 3. When Molecule Testing Doesn't Make Sense

**Don't Use Molecule For**:
- Simple, single-platform roles
- Roles that don't interact with system services
- One-off scripts or playbooks
- Roles with < 50 lines of tasks

**Do Use Molecule For**:
- Multi-platform role development
- Roles managing system services
- Complex configuration management
- Roles used across multiple projects
- Infrastructure as Code with high reliability needs

## Scaling Strategies

### From 1 to 10 Scenarios
- Start with primary target platform
- Add shared playbooks early
- Set up CI/CD from the beginning

### From 10 to 50+ Scenarios
- Consider test segmentation (fast vs slow)
- Implement parallel execution
- Use Docker for most scenarios
- Monitor CI resource consumption

### Managing Test Suites at Enterprise Scale
- Separate repos vs monorepo (depends on team structure)
- Shared molecule configurations as templates
- Central Docker image repository
- Automated scenario generation from templates

## Metrics That Matter

1. **Test Execution Time**
   - Target: < 5 minutes per scenario
   - Total suite: < 20 minutes (with parallelization)

2. **Coverage**
   - OS Family coverage: All major families (Debian, RedHat, Alpine)
   - Version coverage: N-1 versions still supported

3. **Reliability**
   - Flaky test rate: < 1%
   - False positive rate: < 2%

## Future Enhancements

- **Idempotency Testing**: Run converge twice, ensure no changes on second run
- **Performance Testing**: Measure task execution times
- **Destructive Testing**: Test failure scenarios and rollback
- **Integration Testing**: Multi-host scenarios with dependencies
- **Security Testing**: Verify security configurations (file permissions, SELinux, etc.)

## Resources and References

- Molecule Documentation: https://ansible.readthedocs.io/projects/molecule/
- Best Practices: Test infrastructure as rigorously as application code
- Community: ansible-community on Matrix/IRC
