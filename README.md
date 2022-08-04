# supply-chain-pipelines

The following pipeline shows security features applied into a CI/CD pipelines:
- Clone code from a Git repository
- Scan application code using CodeReady Dependency Analytics (CRDA): https://github.com/fabric8-analytics/cli-tools/blob/main/docs/cli_README.md
- Store a signed image in Red Hat Quay: https://www.redhat.com/en/technologies/cloud-computing/quay
- Run code analysis with Sonarqube and run tests
- Release application binary and store it in Nexus
- Generate an SBOM file using syft: https://github.com/anchore/syft
- Build and sign a container image using Tekton Chains and Cosign: https://next.redhat.com/project/tekton-chains/
- Sign all TaskRuns using Tekton Chains
- Provide provenance attestations and store them in Rekor: https://github.com/sigstore/rekor
- Scan container images using Red Hat Advanced Security for Kubernetes (RHACS) and use results as a gate in the pipeline: https://www.redhat.com/en/technologies/cloud-computing/openshift/advanced-cluster-security-kubernetes
- Scan OpenShift resources (deployment, etc) using RHACS (kubelinter) for best-practices, such as non-privileged users: https://github.com/stackrox/kube-linter
- Deploy application to multiple clusters using OpenShift GitOps (based on argocd) and using Pull-Request workflow to promote application through different environments (from DEV to Staging)

<img width="1395" alt="image" src="https://user-images.githubusercontent.com/19349382/182812739-cb6b4da3-4f79-4b34-9ba5-f3650d9c2128.png">

Below are some high-level highlights of the security features implemented:

## Scan Application code using CodeReady Dependency Analytics
## Pipeline task: crda-scan

The CLI tool allows to scan application code and provide feedback about used libraries and dependencies, and provides a report similar to this:

<img width="1666" alt="image" src="https://user-images.githubusercontent.com/19349382/182817459-46bb0c56-76e6-4282-8169-f1d28694ff4c.png">

The report is generated by a Red Hat hosted service, and a sample report can be accessed here: https://recommender.api.openshift.io/api/v2/stack-report/cb61da376e2b417a85d011818b6a45df

## Generate an SBOM file using syft: https://github.com/anchore/syft
## Pipeline task: syft-sbom

This task uses the syft cli to generate a SBOM based on the application code. Syft can also generate an SBOM from a container image.

## Sign all TaskRuns using TektonChains with the OpenShift Pipelines Operator
The OpenShift Pipelines Operator allows you to easily add Tekton Chains in your cluster through a simple configuration, and thus sign all TaskRuns and container images that are produced within the pipeline.
Here is a sample of a signed TaskRun that is adding the signed payload as an annotation to the Tekton TaskRun:

[source,yaml]
----
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  annotations:
    chains.tekton.dev/cert-taskrun-3ca800d5-3ad2-41ab-ad7d-96acc81e6edf: ''
    pipeline.tekton.dev/release: 6b5710c
    chains.tekton.dev/chain-taskrun-3ca800d5-3ad2-41ab-ad7d-96acc81e6edf: ''
    pipeline.openshift.io/started-by: admin
    chains.tekton.dev/transparency: 'https://rekor.sigstore.dev/api/v1/log/entries?logIndex=2842400'
    chains.tekton.dev/signed: 'true'
    kubectl.kubernetes.io/last-applied-configuration: >
      {"apiVersion":"tekton.dev/v1beta1","kind":"Pipeline","metadata":{"annotations":{},"name":"petclinic-build-dev-v2","namespace":"cicd"},"spec":{"params":[{"default":"http://gogs-cicd.apps.ocp1.openshift.fr/gogs/spring-petclinic","description":"The
      application git
      repository","name":"APP_SOURCE_GIT","type":"string"},{"default":"master","description":"The
      application git
      revision","name":"APP_SOURCE_REVISION","type":"string"},{"default":"http://gogs-cicd.apps.ocp1.openshift.fr/gogs/spring-petclinic-config","description":"The
      application manifests git
      repository","name":"APP_MANIFESTS_GIT","type":"string"},{"default":"latest","description":"The
      application image tag to
      build","name":"APP_IMAGE_TAG","type":"string"},{"default":"devsecops-dev","description":"The
      namespace for Stage
      environments","name":"DEV_NAMESPACE","type":"string"},{"default":"https://github.com/rcarrata/spring-petclinic-gatling","description":"The
      application test cases git
      repository","name":"APP_TESTS_GIT","type":"string"}],"tasks":[{"name":"source-clone","params":[{"name":"url","value":"$(params.APP_SOURCE_GIT)"},{"name":"revision","value":"$(params.APP_SOURCE_REVISION)"},{"name":"depth","value":"0"},{"name":"subdirectory","value":"spring-petclinic"},{"name":"deleteExisting","value":"true"}],"taskRef":{"kind":"ClusterTask","name":"git-clone"},"workspaces":[{"name":"output","workspace":"workspace"}]},{"name":"unit-tests","params":[{"name":"GOALS","value":["package","-f","spring-petclinic"]}],"runAfter":["source-clone"],"taskRef":{"kind":"Task","name":"maven"},"workspaces":[{"name":"source","workspace":"workspace"},{"name":"maven-settings","workspace":"maven-settings"}]},{"name":"code-analysis","params":[{"name":"GOALS","value":["install","sonar:sonar","-f","spring-petclinic","-Dsonar.host.url=http://sonarqube:9000","-Dsonar.userHome=/tmp/sonar","-DskipTests=true"]}],"runAfter":["source-clone"],"taskRef":{"kind":"Task","name":"maven"},"workspaces":[{"name":"source","workspace":"workspace"},{"name":"maven-settings","workspace":"maven-settings"}]},{"name":"dependency-report","params":[{"name":"SOURCE_DIR","value":"spring-petclinic"}],"runAfter":["source-clone"],"taskRef":{"kind":"Task","name":"dependency-report"},"workspaces":[{"name":"source","workspace":"workspace"},{"name":"maven-settings","workspace":"maven-settings"}]},{"name":"release-app","params":[{"name":"GOALS","value":["deploy","-f","spring-petclinic","-DskipTests=true","-DaltDeploymentRepository=nexus::default::http://nexus:8081/repository/maven-releases/","-DaltSnapshotDeploymentRepository=nexus::default::http://nexus:8081/repository/maven-snapshots/"]}],"runAfter":["code-analysis","unit-tests","dependency-report"],"taskRef":{"kind":"Task","name":"maven"},"workspaces":[{"name":"source","workspace":"workspace"},{"name":"maven-settings","workspace":"maven-settings"}]},{"name":"build-image","params":[{"name":"TLSVERIFY","value":"false"},{"name":"MAVEN_MIRROR_URL","value":"http://nexus:8081/repository/maven-public/"},{"name":"PATH_CONTEXT","value":"spring-petclinic/target"},{"name":"IMAGE_NAME","value":"image-registry.openshift-image-registry.svc:5000/$(context.pipelineRun.namespace)/spring-petclinic"},{"name":"IMAGE_TAG","value":"$(params.APP_IMAGE_TAG)"}],"runAfter":["release-app"],"taskRef":{"kind":"Task","name":"s2i-java-11"},"workspaces":[{"name":"source","workspace":"workspace"}]},{"name":"image-scan","params":[{"name":"image","value":"image-registry.openshift-image-registry.svc:5000/$(context.pipelineRun.namespace)/spring-petclinic"},{"name":"rox_api_token","value":"roxsecrets"},{"name":"rox_central_endpoint","value":"roxsecrets"},{"name":"output_format","value":"table"},{"name":"image_digest","value":"$(tasks.build-image.results.IMAGE_DIGEST)"}],"runAfter":["build-image"],"taskRef":{"kind":"ClusterTask","name":"rox-image-scan"}},{"name":"image-check","params":[{"name":"image","value":"image-registry.openshift-image-registry.svc:5000/$(context.pipelineRun.namespace)/spring-petclinic"},{"name":"rox_api_token","value":"roxsecrets"},{"name":"rox_central_endpoint","value":"roxsecrets"},{"name":"image_digest","value":"$(tasks.build-image.results.IMAGE_DIGEST)"}],"runAfter":["build-image"],"taskRef":{"kind":"ClusterTask","name":"rox-image-check"}},{"name":"deploy-check","params":[{"name":"GIT_REPOSITORY","value":"$(params.APP_MANIFESTS_GIT)"},{"name":"rox_api_token","value":"roxsecrets"},{"name":"rox_central_endpoint","value":"roxsecrets"},{"name":"file","value":"deployment.yaml"},{"name":"deployment_files_path","value":"app"}],"runAfter":["build-image"],"taskRef":{"kind":"ClusterTask","name":"rox-deployment-check"},"workspaces":[{"name":"workspace","workspace":"workspace"}]},{"name":"update-deployment","params":[{"name":"GIT_REPOSITORY","value":"$(params.APP_MANIFESTS_GIT)"},{"name":"GIT_USERNAME","value":"gogs"},{"name":"GIT_PASSWORD","value":"gogs"},{"name":"CURRENT_IMAGE","value":"quay.io/siamaksade/spring-petclinic:latest"},{"name":"NEW_IMAGE","value":"image-registry.openshift-image-registry.svc:5000/$(context.pipelineRun.namespace)/spring-petclinic"},{"name":"NEW_DIGEST","value":"$(tasks.build-image.results.IMAGE_DIGEST)"},{"name":"KUSTOMIZATION_PATH","value":"environments/dev"}],"runAfter":["image-scan","image-check","deploy-check"],"taskRef":{"kind":"Task","name":"git-update-deployment"},"workspaces":[{"name":"workspace","workspace":"workspace"}]},{"name":"wait-application","params":[{"name":"application-name","value":"dev-spring-petclinic"}],"runAfter":["update-deployment"],"taskRef":{"kind":"Task","name":"argocd-task-sync-and-wait"}},{"name":"perf-tests-clone","params":[{"name":"url","value":"$(params.APP_TESTS_GIT)"},{"name":"subdirectory","value":"spring-petclinic-gatling"},{"name":"deleteExisting","value":"true"}],"runAfter":["wait-application"],"taskRef":{"kind":"ClusterTask","name":"git-clone"},"workspaces":[{"name":"output","workspace":"workspace"}]},{"name":"pentesting-test","params":[{"name":"APP_URL","value":"http://spring-petclinic.$(params.DEV_NAMESPACE).svc.cluster.local:8080"}],"runAfter":["perf-tests-clone"],"taskRef":{"kind":"Task","name":"zap-proxy"},"workspaces":[{"name":"workspace","workspace":"workspace"}]},{"name":"performance-test","params":[{"name":"APP_URL","value":"http://spring-petclinic.$(params.DEV_NAMESPACE).svc.cluster.local:8080"}],"runAfter":["perf-tests-clone"],"taskRef":{"kind":"Task","name":"gatling"},"workspaces":[{"name":"simulations","subPath":"spring-petclinic-gatling","workspace":"workspace"}]}],"workspaces":[{"name":"workspace"},{"name":"maven-settings"}]}}
    chains.tekton.dev/signature-taskrun-3ca800d5-3ad2-41ab-ad7d-96acc81e6edf: >-
      MEYCIQDtIcuk1RW6tzQ1xNY4o3ZOOzrGLVvy9GLg+ZEtxBkGyAIhAJN/8ej8U2V7iGpombTZ/9wlRWfZsnEJSBIb5sIZjoAU
    chains.tekton.dev/payload-taskrun-3ca800d5-3ad2-41ab-ad7d-96acc81e6edf: >-
      eyJjb25kaXRpb25zIjpbeyJ0eXBlIjoiU3VjY2VlZGVkIiwic3RhdHVzIjoiVHJ1ZSIsImxhc3RUcmFuc2l0aW9uVGltZSI6IjIwMjItMDctMDVUMDg6NTI6MzdaIiwicmVhc29uIjoiU3VjY2VlZGVkIiwibWVzc2FnZSI6IkFsbCBTdGVwcyBoYXZlIGNvbXBsZXRlZCBleGVjdXRpbmcifV0sInBvZE5hbWUiOiJwZXRjbGluaWMtYnVpbGQtZGV2LXYyLXNlb3hhbi1pbWFnZS1zY2FuLXBvZCIsInN0YXJ0VGltZSI6IjIwMjItMDctMDVUMDg6NTI6MTZaIiwiY29tcGxldGlvblRpbWUiOiIyMDIyLTA3LTA1VDA4OjUyOjM3WiIsInN0ZXBzIjpbeyJ0ZXJtaW5hdGVkIjp7ImV4aXRDb2RlIjowLCJyZWFzb24iOiJDb21wbGV0ZWQiLCJzdGFydGVkQXQiOiIyMDIyLTA3LTA1VDA4OjUyOjI4WiIsImZpbmlzaGVkQXQiOiIyMDIyLTA3LTA1VDA4OjUyOjM2WiIsImNvbnRhaW5lcklEIjoiY3JpLW86Ly8wMjFiYTc4ZGJlMjU3NTExM2U3MDdiYTQ3YWY1ZTA3ZDc5YjJmYWNiMGNlMDBkNmYxZmQ0NzNhNzJkM2Q0NDhiIn0sIm5hbWUiOiJyb3gtaW1hZ2Utc2NhbiIsImNvbnRhaW5lciI6InN0ZXAtcm94LWltYWdlLXNjYW4iLCJpbWFnZUlEIjoicmVnaXN0cnkuYWNjZXNzLnJlZGhhdC5jb20vdWJpOC91YmktbWluaW1hbEBzaGEyNTY6MmM4ZTA5MWIyNmNjNWE3M2NjN2M2MWY0YmFlZTcxODAyMWNmZTViZDJmYmM1NTZkMTQxMTQ5OWM5YTk5Y2NkYiJ9XSwidGFza1NwZWMiOnsicGFyYW1zIjpbeyJuYW1lIjoicm94X2NlbnRyYWxfZW5kcG9pbnQiLCJ0eXBlIjoic3RyaW5nIiwiZGVzY3JpcHRpb24iOiJTZWNyZXQgY29udGFpbmluZyB0aGUgYWRkcmVzczpwb3J0IHR1cGxlIGZvciBTdGFja1JveCBDZW50cmFsIChleGFtcGxlIC0gcm94LnN0YWNrcm94LmlvOjQ0MykifSx7Im5hbWUiOiJyb3hfYXBpX3Rva2VuIiwidHlwZSI6InN0cmluZyIsImRlc2NyaXB0aW9uIjoiU2VjcmV0IGNvbnRhaW5pbmcgdGhlIFN0YWNrUm94IEFQSSB0b2tlbiB3aXRoIENJIHBlcm1pc3Npb25zIn0seyJuYW1lIjoiaW1hZ2UiLCJ0eXBlIjoic3RyaW5nIiwiZGVzY3JpcHRpb24iOiJGdWxsIG5hbWUgb2YgaW1hZ2UgdG8gc2NhbiAoZXhhbXBsZSAtLSBnY3IuaW8vcm94L3NhbXBsZTo1LjAtcmMxKSJ9LHsibmFtZSI6Im91dHB1dF9mb3JtYXQiLCJ0eXBlIjoic3RyaW5nIiwiZGVzY3JpcHRpb24iOiJPdXRwdXQgZm9ybWF0IChqc29uIHwgY3N2IHwgdGFibGUpIiwiZGVmYXVsdCI6Impzb24ifSx7Im5hbWUiOiJpbWFnZV9kaWdlc3QiLCJ0eXBlIjoic3RyaW5nIiwiZGVzY3JpcHRpb24iOiJEaWdlc3QgaW4gc2hhMjU2IGhhc2ggZm9ybWF0IG9mIHRoZSBpbWFnZSB0byBzY2FuIn1dLCJzdGVwcyI6W3sibmFtZSI6InJveC1pbWFnZS1zY2FuIiwiaW1hZ2UiOiJyZWdpc3RyeS5hY2Nlc3MucmVkaGF0LmNvbS91Ymk4L3ViaS1taW5pbWFsOmxhdGVzdCIsImVudiI6W3sibmFtZSI6IlJPWF9BUElfVE9LRU4iLCJ2YWx1ZUZyb20iOnsic2VjcmV0S2V5UmVmIjp7Im5hbWUiOiIkKHBhcmFtcy5yb3hfYXBpX3Rva2VuKSIsImtleSI6InJveF9hcGlfdG9rZW4ifX19LHsibmFtZSI6IlJPWF9DRU5UUkFMX0VORFBPSU5UIiwidmFsdWVGcm9tIjp7InNlY3JldEtleVJlZiI6eyJuYW1lIjoiJChwYXJhbXMucm94X2NlbnRyYWxfZW5kcG9pbnQpIiwia2V5Ijoicm94X2NlbnRyYWxfZW5kcG9pbnQifX19XSwicmVzb3VyY2VzIjp7fSwic2NyaXB0IjoiIyEvdXNyL2Jpbi9lbnYgYmFzaFxuc2V0ICt4XG5leHBvcnQgTk9fQ09MT1I9XCJUcnVlXCJcbmN1cmwgLWsgLUwgLUggXCJBdXRob3JpemF0aW9uOiBCZWFyZXIgJFJPWF9BUElfVE9LRU5cIiBodHRwczovLyRST1hfQ0VOVFJBTF9FTkRQT0lOVC9hcGkvY2xpL2Rvd25sb2FkL3JveGN0bC1saW51eCAtLW91dHB1dCAuL3JveGN0bCAgXHUwMDNlIC9kZXYvbnVsbDsgZWNobyBcIkdldHRpbmcgcm94Y3RsXCJcbmNobW9kICt4IC4vcm94Y3RsIFx1MDAzZSAvZGV2L251bGxcbmVjaG8gXCIjIyBTY2FubmluZyBpbWFnZSAkKHBhcmFtcy5pbWFnZSlAJChwYXJhbXMuaW1hZ2VfZGlnZXN0KVwiXG4uL3JveGN0bCBpbWFnZSBzY2FuIC0taW5zZWN1cmUtc2tpcC10bHMtdmVyaWZ5IC1lICRST1hfQ0VOVFJBTF9FTkRQT0lOVCAtLWltYWdlICQocGFyYW1zLmltYWdlKUAkKHBhcmFtcy5pbWFnZV9kaWdlc3QpIC0tb3V0cHV0ICQocGFyYW1zLm91dHB1dF9mb3JtYXQpXG5lY2hvIFwiIyMgR28gdG8gaHR0cHM6Ly8kUk9YX0NFTlRSQUxfRU5EUE9JTlQvbWFpbi92dWxuZXJhYmlsaXR5LW1hbmFnZW1lbnQvaW1hZ2UvJChwYXJhbXMuaW1hZ2VfZGlnZXN0KSB0byBjaGVjayBtb3JlIGluZm9cIlxuIn1dfX0=
  resourceVersion: '611888875'
  name: petclinic-build-dev-v2-seoxan-image-scan
  uid: 3ca800d5-3ad2-41ab-ad7d-96acc81e6edf
  creationTimestamp: '2022-07-05T08:52:16Z'
---


