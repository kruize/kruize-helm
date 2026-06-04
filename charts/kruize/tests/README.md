# Kruize Helm Chart Tests

This directory contains unit tests for the Kruize Helm chart using the [helm-unittest](https://github.com/helm-unittest/helm-unittest) plugin.

## Prerequisites

Install the helm-unittest plugin:

```bash
helm plugin install https://github.com/helm-unittest/helm-unittest
```

## Running Tests

> **Note:** The default `helm unittest charts/kruize` command will not find tests in subdirectories. You must explicitly specify test file patterns using the `-f` flag.

### Run all tests
```bash
helm unittest -f 'tests/with-default-values/*.yaml' -f 'tests/with-openshift-values/*.yaml' -f 'tests/with-minikube-values/*.yaml' charts/kruize
```

Or from within the chart directory:
```bash
cd charts/kruize
helm unittest -f 'tests/with-default-values/*.yaml' -f 'tests/with-openshift-values/*.yaml' -f 'tests/with-minikube-values/*.yaml' .
```

### Run tests with default values only
```bash
cd charts/kruize
helm unittest -f 'tests/with-default-values/*.yaml' .
```

### Run tests with OpenShift values
```bash
cd charts/kruize
helm unittest -f 'tests/with-openshift-values/*.yaml' .
```

### Run tests with Minikube values
```bash
cd charts/kruize
helm unittest -f 'tests/with-minikube-values/*.yaml' .
```

### Run tests with verbose output
```bash
cd charts/kruize
helm unittest -d -f tests/with-default-values/kruize_ui_test.yaml .
```

### Generate JUnit Test Report
```bash
cd charts/kruize
helm unittest --output-type JUnit --output-file test-results.xml -f 'tests/with-default-values/*.yaml' -f 'tests/with-openshift-values/*.yaml' -f 'tests/with-minikube-values/*.yaml' .
```

## Directory Structure

```
tests/
├── with-default-values/          # Tests using default values.yaml (39 tests)
│   ├── configmap_test.yaml
│   ├── cronjobs_test.yaml
│   ├── kruize_db_deployment_test.yaml
│   ├── kruize_db_service_test.yaml
│   ├── kruize_service_test.yaml
│   ├── kruize_ui_test.yaml
│   ├── network_policy_test.yaml
│   ├── service_monitor_test.yaml
│   └── storage_test.yaml
│
├── with-openshift-values/        # Tests using values-openshift.yaml (20 tests)
│   ├── kruize_deployment_test.yaml
│   └── rbac_test.yaml
│
└── with-minikube-values/         # Tests using values-minikube.yaml (34 tests)
    ├── kruize_db_deployment_minikube_test.yaml
    ├── kruize_deployment_minikube_test.yaml
    ├── network_policy_minikube_test.yaml
    ├── rbac_minikube_test.yaml
    ├── service_monitor_minikube_test.yaml
    └── storage_minikube_test.yaml
```

> **Important:** All tests are **unit tests** that render and validate Helm templates. They do not require an actual Kubernetes cluster to run. The folder names indicate which values file configuration is being tested, not which cluster type is required.

## Values-based Test Organization

### Tests with Default Values (`tests/with-default-values/`)
Tests that use the default `values.yaml`. These validate core functionality with standard configurations.

### Tests with OpenShift Values (`tests/with-openshift-values/`)
Tests that use `values-openshift.yaml` to validate OpenShift-specific configurations:
- `serviceAccount.create: true` — creates a dedicated service account
- `rbac.create: true` — includes OpenShift-specific ClusterRoleBindings (cluster-monitoring-view, SCC anyuid)
- Resource requests/limits are set
- Full KAFKA_RESPONSE_FILTER_INCLUDE configuration

### Tests with Minikube Values (`tests/with-minikube-values/`)
Tests that use `values-minikube.yaml` to validate minikube-specific configurations:
- `serviceAccount.create: false` — uses the `default` service account
- `rbac.create: false` — skips OpenShift-specific ClusterRoleBindings
- `db.pgData: ""` — PGDATA env var is not set
- `db.volumeMountPath: /var/lib/postgresql/data` — minikube postgres path
- `db.pvc.storageSize: 1Gi` — smaller storage for minikube
- `db.pvc.accessModes: [ReadWriteOnce]` — minikube storage access mode
- `db.pvc.hostPath: /data/postgres` — minikube host path
- `db.pvc.reclaimPolicy: Retain` — retain policy for minikube
- `networkPolicy.enabled: true` — network policy is enabled on minikube
- `monitoring.enabled: true` — monitoring is enabled on minikube
- Empty resource requests/limits — no resource constraints on minikube
- `KAFKA_RESPONSE_FILTER_INCLUDE: summary` — simplified kafka filter for minikube

## Test File Structure

Each test file follows the below style:

```yaml
suite: test <template-name>
templates:
  - <template-file>

tests:
  - it: should create a <Resource> with the correct default settings
    asserts:
      - hasDocuments:
          count: 1
      - equal:
          path: kind
          value: <Kind>
      - equal:
          path: metadata.name
          value: RELEASE-NAME-<name>
      # ... all default assertions grouped together

  - it: should create a <Resource> with the correct settings overrides
    set:
      some.value: override
    asserts:
      - equal:
          path: spec.someField
          value: override

  - it: should create a <Resource> in the release namespace
    release:
      namespace: custom-namespace
    asserts:
      - equal:
          path: metadata.namespace
          value: custom-namespace
```

## Common Assertions

- `equal` - Checks if a value equals the expected value
- `notEqual` - Checks if a value does not equal the expected value
- `exists` - Checks if a path exists
- `notExists` - Checks if a path does not exist
- `contains` - Checks if an array contains a specific element
- `hasDocuments` - Checks the number of documents in the output
- `matchRegex` - Checks if a value matches a regex pattern


## Adding New Tests

When adding new templates or modifying existing ones:

1. **For tests with default values**:
   - Create or update test file in `charts/kruize/tests/with-default-values/`
   - Follow naming convention: `<template-name>_test.yaml`
   - Use default `values.yaml`

2. **For tests with OpenShift values**:
   - Create or update test file in `charts/kruize/tests/with-openshift-values/`
   - Add `values: - ../../values-openshift.yaml` to the test suite
   - Test OpenShift-specific features (RBAC, SCC, monitoring, etc.)

3. **For tests with Minikube values**:
   - Create or update test file in `charts/kruize/tests/with-minikube-values/`
   - Add `values: - ../../values-minikube.yaml` to the test suite
   - Test minikube-specific overrides (empty resources, default SA, etc.)

4. Include tests for:
   - Default values
   - Settings overrides (using `set:`)
   - Conditional resource creation (enabled/disabled flags)
   - Namespace propagation (using `release.namespace`)
   - Label and selector validation

## Updating Version Numbers in Tests

Version numbers in test files are hardcoded because helm-unittest does not support dynamic value substitution in test assertions.

when versions are updated (e.g., kruize from 0.9 to 0.10) update the values in the tests as well.

**Verify all tests pass:**
   ```bash
   cd charts/kruize
   helm unittest -f 'tests/with-default-values/*.yaml' -f 'tests/with-openshift-values/*.yaml' -f 'tests/with-minikube-values/*.yaml' .
   ```

## Troubleshooting

### Plugin Not Found

```bash
helm plugin list
helm plugin install https://github.com/helm-unittest/helm-unittest
```

### Test Failures

Run with verbose output to see detailed failure information:
```bash
helm unittest -d -f 'tests/with-default-values/*.yaml' -f 'tests/with-openshift-values/*.yaml' -f 'tests/with-minikube-values/*.yaml' charts/kruize
```

### Debugging Specific Tests

```bash
helm unittest -f 'tests/with-default-values/kruize_db_deployment_test.yaml' charts/kruize
helm unittest -f 'tests/with-default-values/kruize_ui_test.yaml' charts/kruize
helm unittest -f 'tests/with-openshift-values/kruize_deployment_test.yaml' charts/kruize
helm unittest -f 'tests/with-minikube-values/kruize_deployment_minikube_test.yaml' charts/kruize
```

## Resources

- [Helm Unittest Documentation](https://github.com/helm-unittest/helm-unittest/blob/main/DOCUMENT.md)
- [Helm Chart Testing Guide](https://helm.sh/docs/topics/chart_tests/)
