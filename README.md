# Helm Charts

Somewhere I store some helm stuff I'm playing about with.

### Openshift binary build helm chart

[Openshift Binary Build Helm Chart](./openshift-binary-build)

- External Registry
- External Nexus
- Builds happen in the openshift environment

Using the binary build style, say we have a fat spring boot jar, we can then run a build after after deploying from the chart.

Typically an admin/elevated style user would setup the initial bits (and know the registry username/password) - and would perform the following:

```
# deploy openshift/kube objects via chart
helm install myapp ./openshift-binary-build -f some-values.yaml

# ensure registry cretificate setup
openssl s_client -connect ${REGISTRY_HOST}:${REGISTRY_PORT} 2>/dev/null </dev/null |  sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > registry-cert.pem
oc create configmap registry-cert -n openshift-config --from-file=${REGISTRY_HOST}..${REGISTRY_PORT}=registry-cert.pem
# patch cluster config with registry certificate
oc patch image.config.openshift.io/cluster --patch '{"spec":{"additionalTrustedCA":{"name":"registry-cert"}}}' --type=merge

# ensure openshift builder account has access to the registry secret created in the helm install
oc secrets link builder myapp-binary-build-registry
```

And then users can:

```
# can actually perform a binary build!
oc start-build myapp-binary-build --from-file=http://some-maven-repo/repository/maven-releases/com/codergists/spring-boot-camel/1.0/spring-boot-camel-1.0.jar
```

All of this is explained in the [NOTES.txt](./openshift-binary-build/templates/NOTES.txt) post-deploy.

Ultimately the following items make up where the image will be pushed to:
```
app_name: binary-build
app_version: "1.0"
registry:
  url: someregistry:1234
  repo: somerepo
```
i.e. `someregistry:1234/somerepo/binary-build:1.0`

For rolling out new changes, update the `app_version`, upgrade the chart and build with a different binary. Note we re-use the values from the initial setup e.g.:

```
helm upgrade --reuse-values --set app_version=1.1 myapp charts/binary-build
oc start-build myapp-binary-build --from-file=http://some-maven-repo/repository/maven-releases/com/codergists/spring-boot-camel/1.1/spring-boot-camel-1.1.jar
```

This should then result in a new tagged version of the image being available in the external registry.