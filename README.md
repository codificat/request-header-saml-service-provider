# README #

This Docker image is used for SAML authentication.

# OpenShift Instructions #
The deployment of this pod involves loading a template and using it to create a
new application.  This running pod will mount in secrets for all custom
configuration.

## SAML Metadata
Create the secret for the httpd saml configuration files (saml-sp.cert,
saml-sp.key, saml-sp.xml, sp-idp-metadata.xml).  Asuming an EntityID of
`https://sp.example.org/mellon` and a mellon service running at the same
location the first three files can be generated automatically using the
following command:

```
mkdir ./httpd-saml-config

mellon_create_metadata.sh https://sp.example.org/mellon https://sp.example.org/mellon

# Note, Secrets cannot have key names with an 'underscore' in them, so when
# creating metadata files with `mellon_create_metadata.sh` the resulting files
# must be renamed appropriately.
mv saml_sp.cert saml-sp.cert
mv saml_sp.key saml-sp.key
mv saml_sp.xml saml-sp.xml
```

The sp-idp-metadata.xml must be supplied by your [Identity Provider](keycloak/testing_with_keycloak.md#creating-the-saml-metadata).


```sh
oc secrets new httpd-saml-config-secret ./httpd-saml-config
```


## Authentication certificate
This certifcate is used by the saml service provider pod to make a secure
request to the Master.  Using all the defaults a suitable file can be created
as follows:

```
oadm create-api-client-config   --certificate-authority='/etc/origin/master/ca.crt' \
                                --client-dir='/etc/origin/master/' \
                                --signer-cert='/etc/origin/master/ca.crt' \
                                --signer-key='/etc/origin/master/ca.key' \
                                --signer-serial='/etc/origin/master/ca.serial.txt' \
                                --user='system:proxy'

mkdir ./httpd-ose-certs
cat /etc/origin/master/system\:proxy.crt /etc/origin/master/system\:proxy.key > ./httpd-ose-certs/authproxy.pem
cp /etc/origin/master/ca.crt httpd-ose-certs/ca.crt
```

Now create the secret:

```sh
oc secrets new httpd-ose-certs-secret ./httpd-ose-certs
```

The saml service provider pod will itself expose a TLS endpoint.

We will use OpenShift's service serving certificates service to automatically
generate these, and the OpenShift Router will use TLS reencrypt to terminate
external TLS connections using a route-provided certificate.

Optional: Create a secret for a custom CA (secret and cert names must be unique)
```sh
oc secrets new my-ca-cert-secret ./my-ca.crt
```

### Making changes to secrets
It's likely you will need to update the value of some secrets.  To do this
simply delete the secret and recreate it.  Then trigger a new deployment.

```
oc delete secret <secret name>
oc secrets new <secret name> <path>
oc rollout latest saml-auth
```

### Deploying the image
Add saml-auth template to OSE - (required parameters: APPLICATION_DOMAIN, PROXY_PATH, PROXY_DESTINATION)
```sh
oc create -f ./saml-auth.template -n openshift
```


Create a new application (test with '-o json', remove when satisfied with the result)
```sh
oc new-app saml-auth \
    -p APPLICATION_DOMAIN=sp.example.org -p PROXY_PATH=/oauth/ -p PROXY_DESTINATION=https://ose.example.com:8443/oauth/ -o json
```


### Mounting the secrets

Mount the secret for the SAML configuration (saml-sp.cert,saml-sp.key,saml-sp.xml,sp-idp-metadata.xml)
```sh
oc volume deploymentconfigs/saml-auth \
     --add --overwrite --name=httpd-saml-config --mount-path=/etc/httpd/conf/saml \
     --type=secret --secret-name=httpd-saml-config-secret
```

Mount the secret for OSE certs (authproxy.pem,ca.crt)
```sh
oc volume deploymentconfigs/saml-auth \
     --add --overwrite --name=httpd-ose-certs --mount-path=/etc/httpd/conf/ose_certs \
     --type=secret --secret-name=httpd-ose-certs-secret
```

Optional: Mount the secret for a custom CA cert (duplicate as required)
```sh
oc volume deploymentconfigs/saml-auth \
     --add --overwrite --name=my-ca-cert --mount-path=/etc/pki/ca-trust/source/anchors/my-ca.crt \
     --type=secret --secret-name=my-ca-cert-secret
```

The template defines replicas as 0.  This pod can be scaled to multiple
replicas for high availability.
```sh
oc scale --replicas=1 dc saml-auth
```

After that command runs you will likely see several deployments for each of the
volumes that are mounted.

### Master configuration changes.

Update /etc/origin/master/master-config.yaml:
```
oauthConfig:
  assetPublicURL: https://ose.example.com:8443/console/
  grantConfig:
    method: auto
  identityProviders:
  - name: saml
    challenge: false
    login: true
    mappingMethod: add
    provider:
      apiVersion: v1
      kind: RequestHeaderIdentityProvider
      loginURL: "https://sp.example.org/oauth/authorize?${query}"
      clientCA: /etc/origin/master/ca.crt
      headers:
      - Remote-User
  masterCA: ca-bundle.crt
...

```

```
assetConfig:
  logoutURL: "https://sp.example.org/mellon/logout?ReturnTo=https://sp.example.org/logged_out.html"
```

Restart the master(s) at this point for the configuration to take effect.

### Making local modifications

#### ImageStream preparation
If building the image locally or pulling from another location it's helpful to
create an ImageStream to simplify ongoing deployments.  As the cluster-admin
this can be accomplished as follows:

```
# create a project for hosting the images
oc new-project openshift3

# allow all authenticated users to pull this image
oadm policy add-cluster-role-to-group system:image-puller system:authenticated -n openshift3

```

At this point you can either manually build the image or pull it from another location.

### Manually building the docker image
Create the docker image
```sh
docker build --tag=saml-service-provider .
```

#### Pushing the image to the internal docker registry

Since the builder service account has access to create ImageStreams in the
`openshift3` project we can use its token.

```
docker login -u unused -e unused -p `oc sa get-token builder -n openshift3` 172.30.36.214:5000

# Find the internal registry IP or use DNS. In this example 172.30.36.214 is
# the internal registry.
oc get services | grep docker-registry
docker tag <your.local.image/saml-service-provider> 172.30.36.214:5000/openshift3/saml-service-provider
docker push 172.30.36.214:5000/openshift3/saml-service-provider

# If this is your first time deploying the saml pod you will need to manually scale up
oc scale --replicas=1 dc saml-auth

```

