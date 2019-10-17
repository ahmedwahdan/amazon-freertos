# Developer-Mode Key Provisioning #
## Introduction ##
The purpose of the Amazon FreeRTOS developer-mode key provisioning demo is to provide embedded developers with options for getting a trusted X.509 client certificate onto an IoT device for lab testing. Depending on the capabilities of the device, various provisioning-related operations may or may not be supported, including onboard ECDSA key generation, private key import, and X.509 certificate enrollment. In addition, different use cases call for different levels of key protection, ranging from onboard flash storage to the use of dedicated crypto hardware. This demo provides logic for working within the cryptographic capabilities of your device.

## Option #1: Private Key Import from AWS IoT ##
For lab testing purposes, if your device allows the import of private keys, you can follow the instructions in [Configuring the Amazon FreeRTOS Demos](https://docs.aws.amazon.com/freertos/latest/userguide/freertos-configure.html). 

## Option #2: Onboard Private Key Generation ##
If your device does not allow the import of private keys, or if your solution calls for the use of [Just-in-Time Provisioning](https://docs.aws.amazon.com/iot/latest/developerguide/jit-provisioning.html) (JITP), you can follow the instructions below. In summary, the sequence of operations is:

1. Public Key Infrastructure Setup
1. Create a certificate authority (CA) certificate
1. Register your CA certificate with AWS IoT
1. Create a Device Certificate, issued by your registered CA
1. Import the CA certificate and device certificate into your Amazon FreeRTOS device
1. Update the AWS IoT Registry in order to mark your certificate as Active
1. Create an AWS IoT Thing, attach a policy to the Thing, and attach a device certificate to the Thing 
1. Run the Amazon FreeRTOS "Hello World" demo on your device

### Initial Configuration ###
First, perform the steps in [Configuring the Amazon FreeRTOS Demos](https://docs.aws.amazon.com/freertos/latest/userguide/freertos-configure.html), but skip the last step (that is, don't do *To format your AWS IoT credentials*). The net result should be that the *demos/include/aws_clientcredential.h* file has been updated with your settings, but the *demos/include/aws_clientcredential_keys.h* file has not. 

### Demo Project Configuration ###
Open the Hello World MQTT demo as described in the [Getting Started guide](https://docs.aws.amazon.com/freertos/latest/userguide/getting-started-guides.html) for your board. In the project, open the file *aws_dev_mode_key_provisioning.c* and find the definition keyprovisioningFORCE_GENERATE_NEW_KEY_PAIR, which is set to zero by default. Set it to one:

```
#define keyprovisioningFORCE_GENERATE_NEW_KEY_PAIR 1
```

Double-check that your device network connection is working (i.e. the [Configuring](https://docs.aws.amazon.com/freertos/latest/userguide/freertos-configure.html) link, above). Then build and run the demo project and continue to the next section. 

### Public Key Extraction ###
Since the device has not yet been provisioned with a private key and client certificate, the demo will fail to authenticate to AWS IoT. However, the Hello World MQTT demo starts by running developer-mode key provisioning, resulting in the creation of a private key if one was not already present. You should see something like the following near the beginning of the serial console output:

```
7 910 [IP-task] Device public key, 91 bytes:
3059 3013 0607 2a86 48ce 3d02 0106 082a
8648 ce3d 0301 0703 4200 04cd 6569 ceb8
1bb9 1e72 339f e8cf 60ef 0f9f b473 33ac
6f19 1813 6999 3fa0 c293 5fae 08f1 1ad0
41b7 345c e746 1046 228e 5a5f d787 d571
dcb2 4e8d 75b3 2586 e2cc 0c
```

Copy the six lines of key bytes into a file called *DevicePublicKeyAsciiHex.txt*. Then use the command-line tool xxd to parse the hex bytes into binary:

```
xxd -r -ps DevicePublicKeyAsciiHex.txt DevicePublicKeyDer.bin
```

Use openssl to format the binary encoded (DER) device public key as PEM:

```
openssl ec -inform der -in DevicePublicKeyDer.bin -pubin -pubout -outform pem -out DevicePublicKey.pem
```

Don't forget to disable the temporary key generation setting you enabled above. Otherwise, the device will create yet another key pair, and you will have to repeat the previous steps:

```
#define keyprovisioningFORCE_GENERATE_NEW_KEY_PAIR 0
```

### Public Key Infrastructure Setup ###
Follow the instructions in [Registering Your CA Certificate](https://docs.aws.amazon.com/iot/latest/developerguide/device-certs-your-own.html#register-CA-cert) to create a certificate hierarchy for your device lab certificate. Stop before executing the sequence described in the section Creating a Device Certificate Using Your CA Certificate. 

In this case, the device will not be signing the certificate request (i.e. Certificate Service Request, or CSR). That's because the X.509 encoding logic required for creating and signing a CSR has been excluded from the Amazon FreeRTOS demo projects in the interest of reducing ROM size. Instead, for lab testing purposes, create a private key on your workstation for the purpose of signing the CSR. 

```
openssl genrsa -out tempCsrSigner.key 2048
openssl req -new -key tempCsrSigner.key -out deviceCert.csr
```

Once your Certificate Authority has been created and registered with AWS IoT, use the following command to issue a client certificate based on the device CSR that was signed in the previous step:

```
openssl x509 -req -in deviceCert.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out deviceCert.pem -days 500 -sha256 -force_pubkey DevicePublicKey.pem
```

Even though the CSR was signed with a temporary private key, the issued certificate will only be usable in tandem with the real device private key. The same mechanism can be used in production; however, in that case, store the CSR signer key in separate hardware, and configure your certificate authority to only issue certificates for requests that have been signed by that specific key (e.g. which should remain in the possession of a designated administrator).

### Certificate Import ###
With the certificate issued, the next step is to import it into your device. You will also need to import your Certificate Authority (CA) certificate, since it is required in order to ensure that the latest certificate chain is read from device storage and transmitted to AWS IoT during TLS negotiation. In the *aws_clientcredential_keys.h* file in your project, set the `keyCLIENT_CERTIFICATE_PEM` macro to be the contents of *deviceCert.pem* and set the `keyJITR_DEVICE_CERTIFICATE_AUTHORITY_PEM` macro to be the contents of *rootCA.pem*. 

### Device Authorization ###
Import deviceCert.pem into the AWS IoT registry as described in [Use Your Own Certificate](https://docs.aws.amazon.com/iot/latest/developerguide/device-certs-your-own.html). You must create a new Thing, attach the PENDING certificate and a policy to your Thing, and then mark the certificate as ACTIVE. All of those steps can be performed manually in the AWS IoT console. 

Once the new client certificate is ACTIVE and associated with a Thing and policy, run the MQTT Hello World demo again. This time, the connection to the AWS IoT MQTT broker will succeed.
