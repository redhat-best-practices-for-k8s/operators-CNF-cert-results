# AGENTS.md

This file provides guidance to AI coding agents when working with code in this repository.

## Repository Overview

This repository contains automation scripts and archived results for running the CNF (Cloud Native Function) certification test suite against certified and Red Hat operators. The primary purpose is to systematically test operators from the Red Hat Catalog against best practices defined in the `cnf-certification-test` suite.

The repository serves two main functions:
1. **Automation Scripts**: Shell scripts to automate the process of installing operators, running certification tests, and collecting results
2. **Historical Results**: Archived test results (claim.json, JUnit XML, logs) for operators tested in previous certification runs

## Key Components

### Bundle Lists
- `bundlelist-certified-new.txt`: List of certified operators with their bundle image references from `registry.connect.redhat.com`
- `bundlelist-redhat-new.txt`: List of Red Hat operators with their bundle image references from `registry.redhat.io`

Format: `<package-name>, <bundle-image-reference>`

### Scripts

**`run-cnf-operator.sh`** - Main automation script that:
1. Reads operator bundle information from bundle list files
2. Creates a test namespace for each operator
3. Installs the operator using `operator-sdk run bundle`
4. Labels the CSV and pod for test discovery
5. Runs the CNF certification tests via `run-tnf-container.sh`
6. Cleans up and removes the operator
7. Merges results to a CSV file

**`get-latest-bundle.sh`** - Retrieves the latest bundle image for a given package:
- Uses gRPC to query the operator registry API
- Requires a running catalog registry (port 50051)
- Returns package name and bundle path for the latest version

**`run-tnf-container.sh`** - Wrapper script for running the TNF container:
- Configures kubeconfig and docker config binding
- Sets up output directories
- Supports custom test labels (-l flag)
- Handles test execution in a containerized environment

**`run-cnf-suites.sh`** - Direct test execution script:
- Runs the CNF certification test binary
- Supports label filtering (common, extended, faredge, telco)
- Generates JUnit reports and claim files

**`combine.sh`** - Merges claim.json files from multiple operator test runs into a single JSON file

**`jsontocsv.py`** - Converts merged JSON results to CSV format for analysis

### Test Results Structure

Results are organized in directories per operator:
```
reports-<catalog>-<month>/
  <operator-name>/
    claim.json                     # Full test claim file
    cnf-certification-tests_junit.xml  # JUnit test report
    tnf-execution.log              # Test execution logs
    <timestamp>-cnf-test-results.tar.gz  # Archived results
```

Consolidated results:
- `results-certified-september.csv`: All certified operator results
- `results-redhat-september.csv`: All Red Hat operator results
- `october-CNF-reports.tar.gz`: Archived October test reports

### Configuration

**`tnf_config.yml`** - Template TNF configuration file:
```yaml
targetNameSpaces:
  - name: $ns
podsUnderTestLabels:
  - "test-network-function.com/generic: target"
operatorsUnderTestLabels:
  - "test-network-function.com/operator: target"
```
The `$ns` placeholder is replaced dynamically during test execution.

## Prerequisites

To run the automation scripts, you need:

1. **OpenShift Client (oc)** - installed and configured
2. **Operator SDK** - for operator installation/removal
3. **Kubeconfig** - valid access to an OpenShift cluster
4. **Registry Credentials** - for `registry.connect.redhat.com` and `registry.redhat.io`
5. **Docker/Podman** - for running the TNF container
6. **gRPCurl** (optional) - for `get-latest-bundle.sh` to query operator catalogs

## Usage

### Running Tests for All Operators in a Catalog

1. Get operator packages from a catalog:
```bash
oc get packagemanifest | grep "Red Hat Operators" > redhat-operators.txt
```

2. Get bundle images for each package:
```bash
while read i; do ./get-latest-bundle.sh $i; done < redhat-operators.txt > redhat-bundlelist.txt
```

3. Create results directory:
```bash
mkdir -p report-redhat-operators
```

4. Modify `run-cnf-operator.sh` with appropriate paths and run:
```bash
./run-cnf-operator.sh
```

### Running Tests for a Single Operator

Use `run-tnf-container.sh` directly:
```bash
./run-tnf-container.sh -k ~/.kube/config -t ./tnf-config -o ./output -l all
```

Options:
- `-k`: Path to kubeconfig file(s)
- `-t`: Path to TNF config directory
- `-o`: Output location for results
- `-l`: Test labels (all, access-control, networking, lifecycle, etc.)
- `-c`: Path to docker config for registry authentication

### Analyzing Results

Merge claim files:
```bash
./combine.sh
```

Convert to CSV:
```bash
python jsontocsv.py
```

## Git Submodule

This repository includes the `cnf-certification-test` project as a git submodule:
```
[submodule "cnf-certification-test"]
    path = cnf-certification-test
    url = https://github.com/test-network-function/cnf-certification-test.git
```

Initialize the submodule after cloning:
```bash
git submodule update --init
```

## Contributing Guidelines

### Adding New Test Results

1. Create a directory under `reports-<catalog>-<month>/<operator-name>/`
2. Include: `claim.json`, `cnf-certification-tests_junit.xml`, `tnf-execution.log`
3. Optionally add compressed archives

### Updating Bundle Lists

1. Update `bundlelist-certified-new.txt` or `bundlelist-redhat-new.txt`
2. Format: `<package-name>, <registry-path>@<sha256-digest>`
3. Ensure bundle images are from appropriate registries

### Script Modifications

- Test changes locally before committing
- Update paths in scripts as needed (many use hardcoded paths)
- Ensure proper cleanup of test namespaces and operators

## Related Projects

- **cnf-certification-test**: The main certification test suite (submodule)
- **certsuite**: Current name for the certification test suite in the main monorepo
- **oct**: Offline Catalog Tool for retrieving certified artifacts
