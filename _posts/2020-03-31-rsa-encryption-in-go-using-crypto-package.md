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

Focus of this article is encryption. With a public key cryptosystem, private key is always kept secure by the owner and public key is publically accessible. Encryption is always done with a public key, this ensures that only the owner of private key can access the data unencrypted and will remain private between the encrytor and owner of private key.

## Cryptography in Golang
Golang's [crypto package](https://golang.org/pkg/crypto/) and subdirectories/sub packages provides implementation of various cryptography algorithms. In this article we will look at RSA encryption capabilities.  

## Implementation
Lets start using RSA in our code. We would need to import following crypto packages.
```go
import (
	"crypto/rand"
	"crypto/rsa"
	"crypto/sha256"
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

### Encryption
Encrypt method accepts plain text as string and RsaPublicKeyParameters serialized as json, encrypts string with key using OaepSHA256 padding and returns a base64 encoded encrypted string. I have used sha256 to make this sample comapatible with samples I have written in other programing langugages, sha512 would be more secure than sha256.  

We will start by deserializing public key json and converting it to `rsa.PublicKey`.  
```go
var rsaPublicKeyParameters RsaPublicKeyParameters
jsonBytes := []byte(publicKeyJson)
err := json.Unmarshal(jsonBytes, &rsaPublicKeyParameters)
publicKey, err := rsaPublicKeyParameters.toRsaPublicKey()
```
Next we will create an instane of `sha256`, convert plain text string to bytes and call `EncryptOAEP` method to encrypt data.  
```go
hash := sha256.New()
plainTextBytes := []byte(plainText)
ciphertext, err := rsa.EncryptOAEP(hash, rand.Reader, publicKey, plainTextBytes, nil)
```
We then convert encrypted data to base64 string and return to caller.  

Complete code for the method is below
```go
func (crypto RsaCrypto) Encrypt(plainText string, publicKeyJson string) (string, error) {
	// create a new aes cipher using key
	var rsaPublicKeyParameters RsaPublicKeyParameters
	jsonBytes := []byte(publicKeyJson)
	err := json.Unmarshal(jsonBytes, &rsaPublicKeyParameters)
	publicKey, err := rsaPublicKeyParameters.toRsaPublicKey()
	if err != nil {
		return "", err
	}

	hash := sha256.New()
	plainTextBytes := []byte(plainText)
	ciphertext, err := rsa.EncryptOAEP(hash, rand.Reader, publicKey, plainTextBytes, nil)
	if err != nil {
		return "", err
	}

	return base64.StdEncoding.EncodeToString(ciphertext), nil
}
```

### Decryption
Decrypt method works in conjunction with Encrypt method above, it accepts base64 encoded encrypted string and RsaPrivateKeyParameters serialized as json. It converts key to rsa.PrivateKey, performs decryption and returns plain text.  

We will start by deserialize private key json and convert it to go package's `rsa.PrivateKey`.  
```go
var rsaPrivateKeyParameters RsaPrivateKeyParameters
jsonBytes := []byte(privateKeyJson)
err = json.Unmarshal(jsonBytes, &rsaPrivateKeyParameters)
privateKey, err := rsaPrivateKeyParameters.toRsaPrivateKey()
```

Next we will convert base64 encrypted data to byte array.  
```go
data, err := base64.StdEncoding.DecodeString(cipherText)
```

Next we will create an instane of `sha256` and call `DecryptOAEP` to decrypt encrypted data.  
```go
hash := sha256.New()
plainText, err := rsa.DecryptOAEP(hash, rand.Reader, privateKey, data, nil)
```
And finally we convert decrypted data bytes to UTF8 string and return to caller.  

Complete code for the method is below  
```go
func (crypto RsaCrypto) Decrypt(cipherText string, privateKeyJson string, provider string) (string, error) {
	data, err := base64.StdEncoding.DecodeString(cipherText)
	if err != nil {
		return "", err
	}

	var rsaPrivateKeyParameters RsaPrivateKeyParameters
	jsonBytes := []byte(privateKeyJson)
	err = json.Unmarshal(jsonBytes, &rsaPrivateKeyParameters)
	privateKey, err := rsaPrivateKeyParameters.toRsaPrivateKey()
	if err != nil {
		return "", err
	}
	
	hash := sha256.New()
	plainText, err := rsa.DecryptOAEP(hash, rand.Reader, privateKey, data, nil)
	if err != nil {
		return "", err
	}

	return string(plainText), nil
}
```

## TL;DR
Complete code for the wrapper class that implements RSA encryption and decryption using Golang Crypto package can be found at [rsacrypto.go]https://github.com/kashifsoofi/crypto-sandbox/blob/master/go/rsacrypto/rsacrypto.go). Unit tests for the wrapper class can be found at [rsacrypto_test.go](https://github.com/kashifsoofi/crypto-sandbox/blob/master/go/rsacrypto/rsacrypto_test.go). Complete repository with other samples and languages can be found at [CryptoSandbox](https://github.com/kashifsoofi/crypto-sandbox).

## References
In no particular order  
[https://en.wikipedia.org/wiki/RSA_(cryptosystem)](https://en.wikipedia.org/wiki/RSA_(cryptosystem))  
[https://gobyexample.com/json](https://gobyexample.com/json)  
[Golang RSA encrypt and decrypt example](https://gist.github.com/miguelmota/3ea9286bd1d3c2a985b67cac4ba2130a)  
[Add new methods to an existing type in Go](https://stackoverflow.com/a/28800807/2524922)  
And many more
