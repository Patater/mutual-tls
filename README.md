[![Go Report Card](https://goreportcard.com/badge/github.com/patater/mutual-tls)](https://goreportcard.com/report/github.com/patater/mutual-tls)

# Mutually authenticated TLS in Go

Apache 2.0 licensed demo for doing mutually authenticated TLS for Go,
comprising a server that authenticates clients and a client that authenticates
servers.

## Running the demo

### Server

```cmd
$ go run server.go
```

### Client

The client demo requires a valid client certificate in order to function. A
sample certificate is provided as `Client.crt`. If you'd like to use your own
certificate, place your `Client.crt` as created as part of running the
provisioning steps in the same folder as this README.

```cmd
$ go run client.go
```

## Managing certificates

### This example requires a certificate authority

For the sake of this example, we'll assume you have three certificate
authorities (CAs): a Root CA, a Server Sub-CA, and a Client Sub-CA. We'll
assume you want to use openssl to run this test CA. We assume the following
file names for the relevant files that comprise your CA, and that the file
contents are PEM encoded.

*Root CA*
The Root CA is used to create subordinate CAs. Its certificate is used by both
servers and clients in a certificate chain.
- Private key in `RootCA.key`
- Root certificate in `RootCA.crt`

*Server Sub-CA*
The Server Sub-CA is used to sign a server's CSR to produce a server
certificate.
- Private key in `ServerSub.key`
- Server Sub-CA certificate in `ServerSub.crt`

*Client Sub-CA*
- Private key in `ClientSub.key`
- Client Sub-CA certificate in `ClientSub.crt`

We also assume the presence of the following certificate chains:

*Server certificate chain*
- The Root CA certificate followed by the Server Sub-CA certificate in
  `ServerChain.pem`
- The server certificate chain is used by clients to authenticate servers they
  connect to.

*Device certificate chain*
- The Root CA certificate followed by the Device Sub-CA certificate in
  `ClientChain.pem`
- The client certificate chain is used by servers to authenticate clients,
  enabling mutually authenticated TLS connections.

### Creating a test Root CA

1. Generate keys and certificates for the Root CA
    ```
    openssl req -x509 \
    -newkey ec -pkeyopt ec_paramgen_curve:secp384r1 -sha256 \
    -keyout RootCA.key -nodes -out RootCA.crt \
    -days 30 \
    -subj "/CN=RootCA/C=UK/O=Patater/OU=RootCA" \
    -extensions v3_ca
    ```
1. View the Root CA certificate
    ```
    openssl x509 -in RootCA.crt -text | less
    ```

### Creating a test Server Sub-CA

1. Run the following openssl command to create a Server Sub-CA key in
   `ServerSub.key`
    ```
    openssl ecparam -name secp384r1 -genkey -noout -out ServerSub.key
    ```
1. Run the following openssl command to create a new CSR
   in `ServerSub.csr`:
    ```
    openssl req -new -sha256 \
    -key ServerSub.key -out ServerSub.csr \
    -subj "/CN=ServerSub/C=UK/O=Patater/OU=CA" \
    -addext "basicConstraints = critical, CA:TRUE, pathlen:1" \
    -addext "keyUsage = critical, keyCertSign, cRLSign"
    ```
1. View your generated CSR with this openssl command:
    ```
    openssl req -noout -text -in ServerSub.csr | less
    ```
1. Run the following openssl command to make your Root CA generate a
   certificate:
    ```
    openssl x509 -req -sha256 -CA RootCA.crt -CAkey RootCA.key \
    -in ServerSub.csr -out ServerSub.crt \
    -CAcreateserial -days 30 -extensions v3_ca
    ```
1. View your generated certificate with this openssl command:
    ```
    openssl x509 -in ServerSub.crt -text | less
    ```
1. Verify your generated certificate with this openssl command:
    ```
    openssl verify -CAfile RootCA.crt ServerSub.crt
    ```
1. Create a certificate chain, useful for verifying certificates signed by the
   CA.
   ```
   cat RootCA.crt ServerSub.crt > ServerChain.pem
   ```

### Creating a test Client Sub-CA

1. Run the following openssl command to create a Client Sub-CA key in
   `ClientSub.key`
    ```
    openssl ecparam -name secp384r1 -genkey -noout -out ClientSub.key
    ```
1. Run the following openssl command to create a new CSR
   in `ClientSub.csr`:
    ```
    openssl req -new -sha256 \
    -key ClientSub.key -out ClientSub.csr \
    -subj "/CN=ClientSub/C=UK/O=Patater/OU=CA" \
    -addext "basicConstraints = critical, CA:TRUE, pathlen:1" \
    -addext "keyUsage = critical, keyCertSign, cRLSign"
    ```
1. View your generated CSR with this openssl command:
    ```
    openssl req -noout -text -in ClientSub.csr | less
    ```
1. Run the following openssl command to make your Root CA generate a
   certificate:
    ```
    openssl x509 -req -sha256 -CA RootCA.crt -CAkey RootCA.key \
    -in ClientSub.csr -out ClientSub.crt \
    -CAcreateserial -days 30 -extensions v3_ca
    ```
1. View your generated certificate with this openssl command:
    ```
    openssl x509 -in ClientSub.crt -text | less
    ```
1. Verify your generated certificate with this openssl command:
    ```
    openssl verify -CAfile RootCA.crt ClientSub.crt
    ```
1. Create a certificate chain, useful for verifying certificates signed by the
   CA.
   ```
   cat RootCA.crt ClientSub.crt > ClientChain.pem
   ```

### Creating server certificates

Let's create a server certificate signing request (CSR) which can be used for
by TLS clients to verify the servier. The CSR is used by your certificate
authority (CA) to produce a client certificate.

The Server Sub-CA will consume the client-generated certificate signing request
and produce a certificate for the server which you can use for running the
server side of the TLS with client authentication demo (`server.go`).

1. Run the following openssl command to create a server key in `Server.key`
    ```
    openssl ecparam -name secp384r1 -genkey -noout -out Server.key
    ```
1. Run the following openssl command to create a new CSR
   in `Server.csr`:
    ```
    openssl req -new -sha256 \
    -key Server.key -out Server.csr \
    -subj "/CN=Server/O=Patater/OU=server/C=UK" \
    -addext "nsCertType = server" \
    -addext "keyUsage = digitalSignature" \
    -addext "extendedKeyUsage = serverAuth"
    ```
1. View your generated CSR with this openssl command:
    ```
    openssl req -noout -text -in Server.csr | less
    ```
1. Run the following openssl command to make your CA generate a certificate:
    ```
    openssl x509 -req -sha256 -CA ServerSub.crt -CAkey ServerSub.key \
    -in Server.csr -out Server.crt \
    -CAcreateserial -days 30 -extensions v3
    ```
1. View your generated certificate with this openssl command:
    ```
    openssl x509 -in Server.crt -text | less
    ```
1. Verify your generated certificate with this openssl command:
    ```
    openssl verify -CAfile ServerChain.pem Server.crt
    ```
1. Copy the `Server.crt` file to the `mutual-tls` example folder for use with
   making a mutually-authenticated TLS connection.

### Creating client certificates

Let's create a client certificate signing request (CSR) which can be used for
TLS client authentication. The CSR is used by your certificate authority (CA)
to produce a client certificate.

The Client Sub-CA will consume the client-generated certificate signing request
and produce a certificate for the client which you can use for running the
client side of the TLS with client authentication demo (`client.go`).

1. Run the following openssl command to create a client key in `Client.key`
    ```
    openssl ecparam -name secp384r1 -genkey -noout -out Client.key
    ```
1. Run the following openssl command to create a new CSR
   in `Client.csr`:
    ```
    openssl req -new -sha256 \
    -key Client.key -out Client.csr \
    -subj "/CN=Client/O=Patater/OU=client/C=UK" \
    -addext "nsCertType = client" \
    -addext "keyUsage = digitalSignature" \
    -addext "extendedKeyUsage = clientAuth"
    ```
1. View your generated CSR with this openssl command:
    ```
    openssl req -noout -text -in Client.csr | less
    ```
1. Run the following openssl command to make your CA generate a certificate:
    ```
    openssl x509 -req -sha256 -CA ClientSub.crt -CAkey ClientSub.key \
    -in Client.csr -out Client.crt \
    -CAcreateserial -days 30 -extensions v3
    ```
1. View your generated certificate with this openssl command:
    ```
    openssl x509 -in Client.crt -text | less
    ```
1. Verify your generated certificate with this openssl command:
    ```
    openssl verify -CAfile ClientChain.pem Client.crt
    ```
1. Copy the `Client.crt` file to the `mutual-tls` example folder for use with
   making a mutually-authenticated TLS connection.
