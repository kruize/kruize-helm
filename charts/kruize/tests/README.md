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
helm unittest -f 'tests/common-tests/*.yaml' -f 'tests/openshift/*.yaml' -f 'tests/minikube/*.yaml' charts/kruize
```

Or from within the chart directory:
```bash
cd charts/kruize
helm unittest -f 'tests/common-tests/*.yaml' -f 'tests/openshift/*.yaml' -f 'tests/minikube/*.yaml' .
```

### Run common tests only
```bash
cd charts/kruize
helm unittest -f 'tests/common-tests/*.yaml' .
```

### Run OpenShift-specific tests
```bash
cd charts/kruize
helm unittest -f 'tests/openshift/*.yaml' .
```

### Run Minikube-specific tests
```bash
cd charts/kruize
helm unittest -f 'tests/minikube/*.yaml' .
```

### Run tests with verbose output
```bash
cd charts/kruize
helm unittest -v -f 'tests/common-tests/*.yaml' -f 'tests/openshift/*.yaml' -f 'tests/minikube/*.yaml' .
```

### Generate JUnit Test Report
```bash
cd charts/kruize
helm unittest --output-type JUnit --output-file test-results.xml -f 'tests/common-tests/*.yaml' -f 'tests/openshift/*.yaml' -f 'tests/minikube/*.yaml' .
```

## Directory Structure

```
tests/
├── common-tests/          # Tests that work across all platforms (39 tests)
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
├── openshift/             # OpenShift-specific tests (20 tests)
│   ├── kruize_deployment_test.yaml
│   └── rbac_test.yaml
│
└── minikube/              # Minikube-specific tests (34 tests)
    ├── kruize_db_deployment_minikube_test.yaml
    ├── kruize_deployment_minikube_test.yaml
    ├── network_policy_minikube_test.yaml
    ├── rbac_minikube_test.yaml
    ├── service_monitor_minikube_test.yaml
    └── storage_minikube_test.yaml
```


## Environment-specific Tests

### Common Tests (`tests/common-tests/`)
Tests that work across all platforms using default `values.yaml`. These validate core functionality that is consistent regardless of deployment platform.

### OpenShift Tests (`tests/openshift/`)
Tests that use `values-openshift.yaml` to validate OpenShift-specific configurations:
- `serviceAccount.create: true` — creates a dedicated service account
- `rbac.create: true` — includes OpenShift-specific ClusterRoleBindings (cluster-monitoring-view, SCC anyuid)
- Resource requests/limits are set
- Full KAFKA_RESPONSE_FILTER_INCLUDE configuration

### Minikube Tests (`tests/minikube/`)
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

1. **For common functionality** (works across all platforms):
   - Create or update test file in `charts/kruize/tests/common-tests/`
   - Follow naming convention: `<template-name>_test.yaml`
   - Use default `values.yaml`

2. **For OpenShift-specific behavior**:
   - Create or update test file in `charts/kruize/tests/openshift/`
   - Add `values: - ../../values-openshift.yaml` to the test suite
   - Test OpenShift-specific features (RBAC, SCC, monitoring, etc.)

3. **For Minikube-specific behavior**:
   - Create or update test file in `charts/kruize/tests/minikube/`
   - Add `values: - ../../values-minikube.yaml` to the test suite
   - Test minikube-specific overrides (empty resources, default SA, etc.)

4. Include tests for:
   - Default values
   - Settings overrides (using `set:`)
   - Conditional resource creation (enabled/disabled flags)
   - Namespace propagation (using `release.namespace`)
   - Label and selector validation

## Troubleshooting

### Plugin Not Found

```bash
helm plugin list
helm plugin install https://github.com/helm-unittest/helm-unittest
```

### Test Failures

Run with verbose output to see detailed failure information:
```bash
helm unittest -v -f 'tests/common-tests/*.yaml' -f 'tests/openshift/*.yaml' -f 'tests/minikube/*.yaml' charts/kruize
```

### Debugging Specific Tests

```bash
helm unittest -f 'tests/common-tests/kruize_db_deployment_test.yaml' -v charts/kruize
helm unittest -f 'tests/openshift/kruize_deployment_test.yaml' -v charts/kruize
helm unittest -f 'tests/minikube/kruize_deployment_minikube_test.yaml' -v charts/kruize
```

## Resources

- [Helm Unittest Documentation](https://github.com/helm-unittest/helm-unittest/blob/main/DOCUMENT.md)
- [Helm Chart Testing Guide](https://helm.sh/docs/topics/chart_tests/)
