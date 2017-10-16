# Overview

This document is broken into two sections:
1. A section regarding configuring DC/OS with certificates, including the following:
    a. Using OpenSSL, creating a self-signed CA certificate and key
    b. Using OpenSSL, creating an intermediate CA, signed by the above self-signed CA certificate
    c. Configuring DC/OS EE 1.10.0 to use the intermediate CA certificate generated above
    d. Using the DC/OS EE CA APIs to generate and sign certificates
2. A section regarding configuring DC/OS to trust certificates, including:
    a. Create a local Docker repo using a self signed certificate
    b. Configuring the Docker daemon within DC/OS to trust certificates
    c. Configuring the UCR fetcher to trust certificates
    d. Configuring the Docker daemon with Docker creds
    e. Configuring the UCR fetcher to use Docker creds

# DC/OS Certificates

## Using OpenSSL, create a self-signed CA certificate and key
1. Install openssl

```bash
sudo yum install -y openssl
```

