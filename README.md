# supply-chain-pipelines

The following pipeline shows security features applied into a CI/CD pipelines:
- Clone code from a Git repository
- Scan application code using CodeReady Dependency Analytics (CRDA): https://github.com/fabric8-analytics/cli-tools/blob/main/docs/cli_README.md
- Store a signed image in Red Hat Quay
- Run code analysis with Sonarqube and run tests
- Release application binary and store it in Nexus
- Generate an SBOM file using syft: https://github.com/anchore/syft
- Build and sign a container image using Tekton Chains and Cosign
- Sign all TaskRuns using Tekton Chains
- Provide provenance attestations and store them in Rekor: https://github.com/sigstore/rekor
- Scan container images using Red Hat Advanced Security for Kubernetes (RHACS) and use results as a gate in the pipeline: https://www.redhat.com/en/technologies/cloud-computing/openshift/advanced-cluster-security-kubernetes
- Scan OpenShift resources (deployment, etc) using RHACS (kubelinter) for best-practices, such as non-privileged users: https://github.com/stackrox/kube-linter
- Deploy application to multiple clusters using OpenShift GitOps (based on argocd) and using Pull-Request workflow to promote application through different environments (from DEV to Staging)

<img width="1395" alt="image" src="https://user-images.githubusercontent.com/19349382/182812739-cb6b4da3-4f79-4b34-9ba5-f3650d9c2128.png">

Below are some high-level highlights of the security features implemented:

# Scan Application code using CodeReady Dependency Analytics

## Pipeline task: crda-scan
The CLI tool allows to scan application code and provide feedback about used libraries and dependencies, and provides a report similar to this:



