# Quick Start


## Prepare your working directoy
```bash
# The name is only for demo, please use your own azure resources instead
mkdir -p notation-akv-demo && cd notation-akv-demo
AKV_NAME=your_akv_name
ACR_NAME=your_acr_name
REGISTRY=${ACR_NAME}.azurecr.io
REPO=your_repo_name
TAG=your_tag_name
IMAGE=${REGISTRY}/${REPO}:${TAG}
KEY_NAME=your_key_name
```

## Create the CA
Create a config file for openssl to create a valid CA
```bash
cat <<EOF > ./ca_ext.cnf
[ v3_ca ]
basicConstraints = CA:TRUE
keyUsage = critical,keyCertSign
extendedKeyUsage = codeSigning
EOF
```
Sign the CA
```bash
# create a signing request to sign your root CA
openssl req -new -newkey rsa:2048 -nodes -out ca.csr -keyout ca.key -extensions v3_ca

# sign the root CA with ca_ext.cnf
openssl x509 -signkey ca.key -days 365 -req -in ca.csr -set_serial 01 -out ca.crt -extensions v3_ca -extfile ./ca_ext.cnf
```

## Create the leaf certificate from Azure KeyVault
Create a certificate policy, which will be used by keyvault to create the leaf certificate
```bash
cat <<EOF > ./leaf_policy.json
{
  "issuerParameters": {
    "certificateTransparency": null,
    "name": "Unknown"
  },
  "keyProperties": {
    "curve": null,
    "exportable": false,
    "keySize": 2048,
    "keyType": "RSA",
    "reuseKey": true
  },
  "secretProperties": {
    "contentType": "application/x-pem-file"
  },
  "x509CertificateProperties": {
     "ekus": [
        "1.3.6.1.5.5.7.3.3"
    ],
    "keyUsage": [
      "digitalSignature"
    ],
    "subject": "CN=Test Signer",
    "validityInMonths": 12
  }
}
EOF
```
Create your certificate
```bash
az keyvault certificate create -n ${KEY_NAME} --vault-name ${AKV_NAME} -p @leaf_policy.json
```

## Sign the leaf certificate
Download the certificate signing request(CSR)
```bash
CSR=$(az keyvault certificate pending show --vault-name ${AKV_NAME} --name ${KEY_NAME} --query 'csr' -o tsv)
CSR_PATH=${KEY_NAME}.csr
printf -- "-----BEGIN CERTIFICATE REQUEST-----\n%s\n-----END CERTIFICATE REQUEST-----\n" $CSR > ${CSR_PATH}
```

Create config file for openssl to sign the certificate
```bash
cat <<EOF > ./ext.cnf
[ v3_ca ]
keyUsage = critical,digitalSignature
extendedKeyUsage = codeSigning
EOF
```

Sign and merge the certificate chain:

```bash
# sign your certificate by using the CA previously created
SIGNED_CERT_PATH=${KEY_NAME}.crt
openssl x509 -CA ca.crt -CAkey ca.key -days 365 -req -in ${CSR_PATH} -set_serial 02 -out ${SIGNED_CERT_PATH} -extensions v3_ca -extfile ./ext.cnf

# Merge the certificate chain
CERTCHAIN_PATH=${KEY_NAME}-chain.crt
cat ${SIGNED_CERT_PATH} ca.crt > ${CERTCHAIN_PATH}
```

## Upload the leaf certificate to Azure KeyVault
```bash
az keyvault certificate pending merge --vault-name ${AKV_NAME} --name ${KEY_NAME} --file ${CERTCHAIN_PATH}
```

## Sign the container image:

Log in to the ACR:

```bash
export NOTATION_PASSWORD=$(az acr login --name ${ACR_NAME} --expose-token --output tsv --query accessToken)
```

Get the Key ID for the certificate and add the Key ID to the keys and certs:

```bash
KEY_ID=$(az keyvault certificate show -n ${KEY_NAME} --vault-name ${AKV_NAME} --query 'kid' -o tsv)
notation key add --name ${KEY_NAME} --plugin azure-kv --id ${KEY_ID}
notation key ls
```

Sign the image
```bash
notation sign --key ${KEY_NAME} ${IMAGE}
```

List the signatures:

```bash
notation ls ${IMAGE}
```

## Verify the container image

Download the certificate:

```bash
CERT_ID=$(az keyvault certificate show -n ${KEY_NAME} --vault-name ${AKV_NAME} --query 'sid' -o tsv)
CERT_PATH=${KEY_NAME}-cert.crt
az keyvault secret download --file ${CERT_PATH} --id ${CERT_ID}
```

Generate a CA certificate:

```bash
notation cert add --name ${KEY_NAME} ${CERT_PATH}
notation cert ls
```

Verify the container image:

```bash
notation verify --cert ${KEY_NAME} ${IMAGE}
```

You can use the notation verify command to ensure the container image hasn't been tampered with since build time.
