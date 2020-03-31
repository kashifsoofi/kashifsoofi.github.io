---
title:  "RSA In Go using Crypto Package"
date:   2020-03-31
categories:
  - cryptography
tags:
  - Go
  - rsa
  - crytpography
---
Sample wrapper package implementing RSA text encrytion/decrytion using Golang Crypto Package

## Introduction
**RSA (Rivest–Shamir–Adleman)** is a public-key cryptosystem. In such a cryptosystem, a pair of keys is used often called private and public key pair.

Public key cryptosystems are used for 2 major use cases
1. Encryption
2. Verification

Focus of this article is signing/verification. With a public key cryptosystem, private key is always kept secure by the owner and public key is publically accessible. Process of verification involves signing of data with a private key that can be verified using associated public key. Signing is always done with a private key that is only accessible by owner. Verification is done using a public key accessible by any member of the public. Anybody can use it (public key) to verify a data signature, if successful meaning it is genuinely coming from the owner of the private key.  

## Cryptography in Golang
Golang's [crypto package](https://golang.org/pkg/crypto/) and subdirectories/sub packages provides implementation of various cryptography algorithms. In this article we will look at RSA encryption capabilities.  

## Implementation
Lets start using RSA in our code. We would need to import following crypto packages.
```go
import (
	"crypto/rand"
	"crypto/rsa"
	"crypto/sha256"
	"crypto/sha512"
)
```

We would also need following pakcages to complete the implementation.
```go
import (
	base64 "encoding/base64"
	"encoding/binary"
	"encoding/json"
	"math/big"
)
```

### Key Generation
GenerateKeyPair method uses `GenerateKey` method of `crypto/rsa` package by passing an instance of rand reader and key size between 1024 and 4096. It then converts that pair to RsaPrivateKeyParameters/RsaPublicKeyParameters helper classes and export as json strings.  

Full source of the method as follows  
```go
privateKey, err := rsa.GenerateKey(rand.Reader, keySize)
if err != nil {
  return "", "", err
}
  
var rsaPrivateKey RsaPrivateKey = RsaPrivateKey(*privateKey)
rsaPrivateKeyParameters := rsaPrivateKey.toRsaPrivateKeyParameters()

var rsaPublicKey RsaPublicKey = RsaPublicKey(privateKey.PublicKey)
rsaPublicKeyParameters, err := rsaPublicKey.toRsaPublicKeyParameters()
if err != nil {
  return "", "", err
}

return rsaPrivateKeyParameters.toJson(), rsaPublicKeyParameters.toJson(), nil
```

### Sign Data
SignData method accepts plain text as string and RsaPrivateKeyParameters serialized as json, signs data with key using a hash and padding and finally returns a base64 encoded data signature.  

We will start by deserializing private key json and converting it to `rsa.PrivateKey`.  
```go
var rsaPrivateKeyParameters RsaPrivateKeyParameters
jsonBytes := []byte(privateKeyJson)
err := json.Unmarshal(jsonBytes, &rsaPrivateKeyParameters)
signatureKey, err := rsaPrivateKeyParameters.toRsaPrivateKey()
```
Next we will generate hash of data to be signed and call `SignPKCS1v15` method to generate signature.  
```go
dataToSign := []byte(data)
hashed := sha512.Sum512(dataToSign)
signature, err := rsa.SignPKCS1v15(rand.Reader, signatureKey, crypto.SHA512, hashed[:])
```
We then convert encrypted data to base64 string and return to caller.  

Complete code for the method is below
```go
func (rsaCrypto RsaCrypto) SignData(data string, privateKeyJson string) (string, error) {
	var rsaPrivateKeyParameters RsaPrivateKeyParameters
	jsonBytes := []byte(privateKeyJson)
	err := json.Unmarshal(jsonBytes, &rsaPrivateKeyParameters)
	signatureKey, err := rsaPrivateKeyParameters.toRsaPrivateKey()
	if err != nil {
		return "", err
	}

	dataToSign := []byte(data)
	hashed := sha512.Sum512(dataToSign)
	signature, err := rsa.SignPKCS1v15(rand.Reader, signatureKey, crypto.SHA512, hashed[:])
	if err != nil {
		return "", err
	}

	return base64.StdEncoding.EncodeToString(signature), nil
}
```

### Verify Signature
VerifySignature method works in conjunction with SignData method above, it accepts data, base64 encoded signature and RsaPublicKeyParameters serialized as json. It converts key to rsa.PublicKey, performs verification and returns a boolean result.  

We will start by deserialize private key json and convert it to go package's `rsa.PublicKey`.  
```go
var rsaPublicKeyParameters RsaPublicKeyParameters
jsonBytes := []byte(publicKeyJson)
err := json.Unmarshal(jsonBytes, &rsaPublicKeyParameters)
signatureKey, err := rsaPublicKeyParameters.toRsaPublicKey()
```
Next we will generate hash for data to be verified and convert base64 encoded signature to byte array.  
```go
dataToVerify := []byte(data)
hashed := sha512.Sum512(dataToVerify)
binarySignature, _ := base64.StdEncoding.DecodeString(signature)
```
Next we will call `VerifyPKCS1v15` method to verify sigature.  
```go
verifyErr := rsa.VerifyPKCS1v15(signatureKey, crypto.SHA512, hashed[:], binarySignature)
```
And finally we return result of verification back to caller.  

Complete code for the method is below  
```go
func (rsaCrypto RsaCrypto) VerifySignature(data string, signature string, publicKeyJson string) (bool, error) {
	var rsaPublicKeyParameters RsaPublicKeyParameters
	jsonBytes := []byte(publicKeyJson)
	err := json.Unmarshal(jsonBytes, &rsaPublicKeyParameters)
	signatureKey, err := rsaPublicKeyParameters.toRsaPublicKey()
	if err != nil {
		return false, err
	}

	dataToVerify := []byte(data)
	hashed := sha512.Sum512(dataToVerify)
	binarySignature, _ := base64.StdEncoding.DecodeString(signature)

	verifyErr := rsa.VerifyPKCS1v15(signatureKey, crypto.SHA512, hashed[:], binarySignature)
	if verifyErr != nil {
		return false, verifyErr
	}

	return true, nil
}
```

## TL;DR
Complete code for the wrapper class that implements RSA signing and verification using Golang Crypto package can be found at [rsacrypto.go]https://github.com/kashifsoofi/crypto-sandbox/blob/master/go/rsacrypto/rsacrypto.go). Unit tests for the wrapper class can be found at [rsacrypto_test.go](https://github.com/kashifsoofi/crypto-sandbox/blob/master/go/rsacrypto/rsacrypto_test.go). Complete repository with other samples and languages can be found at [CryptoSandbox](https://github.com/kashifsoofi/crypto-sandbox).

## References
In no particular order  
[https://en.wikipedia.org/wiki/RSA_(cryptosystem)](https://en.wikipedia.org/wiki/RSA_(cryptosystem))  
[https://gobyexample.com/json](https://gobyexample.com/json)  
[Package rsa](https://golang.org/pkg/crypto/rsa)  
[Go Lang RSA Cryptography](https://8gwifi.org/docs/go-rsa.jsp)
[Add new methods to an existing type in Go](https://stackoverflow.com/a/28800807/2524922)  
And many more
