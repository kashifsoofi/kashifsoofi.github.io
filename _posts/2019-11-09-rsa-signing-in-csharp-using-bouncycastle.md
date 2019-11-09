---
title:  "RSA Signing In C# using Microsoft BouncyCastle.Net"
date:   2019-11-09
categories:
  - cryptography
tags:
  - C#
  - rsa
  - crytpography
---
Sample class library implementing RSA signing using Bouncy Castle (1.8.5)

## Introduction
**RSA (Rivest–Shamir–Adleman)** is a public-key cryptosystem. In such a cryptosystem, a pair of keys is used often called private and public key pair.

Public key cryptosystems are used for 2 major use cases
1. Encryption
2. Verification

Focus of this article is signing/verification. With a public key cryptosystem, private key is always kept secure by the owner and public key is publically accessible. Process of verification involves signing of data with a private key that can be verified using associated public key. Signing is always done with a private key that is only accessible by owner. Verification is done using a public key accessible by any member of the public. Anybody can use it (public key) to verify a data signature, if successful meaning it is genuinely coming from the owner of the private key.

## Bouncy Castle
[Bouncy Castle Crypto APIs](http://www.bouncycastle.org/index.html) are lightweight crypto APIs for Java and C#. In this article we will look at using C# implementation of RSA.

## Implementation
Lets start using RSA in our code. We would need to include following namespaces.

```csharp
using Org.BouncyCastle.Crypto;
using Org.BouncyCastle.Crypto.Generators;
using Org.BouncyCastle.Security;
```

### Key Generation
GenerateKeyPair method creates a new instance of `RsaKeyPairGenerator`, sets desired key size using `KeyGenerationParameters` and then calls `GenerateKeyPair` to generate a public and private key pair. It then converts that pair to `RsaPrivateKeyParameters`/`RsaPublicKeyParameters` helper classes and export as json strings.

```
var random = new SecureRandom();
var keyGenerationParameters = new KeyGenerationParameters(random, keySize);
RsaKeyPairGenerator generator = new RsaKeyPairGenerator();
generator.Init(keyGenerationParameters);

var keyPair = generator.GenerateKeyPair();
var privateKeyParametersJson = JsonConvert.SerializeObject(keyPair.Private.ToPrivateKeyParameters());
var publicKeyParametersJson = JsonConvert.SerializeObject(keyPair.Public.ToPublicKeyParameters());
```

### Sign Data
SignData method accepts a string and `RsaPrivateKeyParameters` serialized as json, signs data with key using a hash and padding and finally returns a base64 encoded data signature.
We will start by deserializing private key json and converting it to Bouncy Castle's `RsaPrivateKeyParameters`.
```csharp
var signatureKey = JsonConvert.DeserializeObject<RsaPrivateKeyParameters>(privateKeyJson).ToRsaPrivateCrtKeyParameters();
```
We then encode data as bytes.
```chsarp
var dataToSign = Encoding.UTF8.GetBytes(data);
```
Then we get an instance of signer by calling `GetSigner` method on `SignerUtilities` helper class by passing signature algorithm name, in our case it is `SHA512WITHRSA` and initilises it with signing mode and signature key. Then we call `BlockUpdate` method on `signer` to set the data we want to generate signature for.
```csharp
var signer = SignerUtilities.GetSigner(SignatureAlgorithm);
signer.Init(true, signatureKey);
signer.BlockUpdate(dataToSign, 0, dataToSign.Length);
```
Next step is to call `GenerateSignature` metho on `signer` to generate cryptographic signature for the data.
```csharp
var signature = signer.GenerateSignature();
```
We then convert signature to base64 string and return to caller.

Complete code for the method is below
```csharp
public string SignData(string data, string privateKeyJson)
{
    var signatureKey = JsonConvert.DeserializeObject<RsaPrivateKeyParameters>(privateKeyJson).ToRsaPrivateCrtKeyParameters();

    var dataToSign = Encoding.UTF8.GetBytes(data);

    var signer = SignerUtilities.GetSigner(SignatureAlgorithm);
    signer.Init(true, signatureKey);
    signer.BlockUpdate(dataToSign, 0, dataToSign.Length);

    var signature = signer.GenerateSignature();
    return Convert.ToBase64String(signature);
}
```

### Verify Signature
VerifySignature method works in conjunction with SignData method above, it accepts data, base64 encoded signature and `RsaPublicKeyParameters` serialized as json. It imports key, performs verification and returns a boolean result.

We will start by deserializing public key json and converting it to Bouncy Castle's `RsaKeyParameters`.
```csharp
var signatureKey = JsonConvert.DeserializeObject<RsaPublicKeyParameters>(publicKeyJson).ToRsaKeyParameters();
```
We then UTF8 encode data we want to verify as bytes and convert signature from base64 string to bytes.
```chsarp
var dataToVerify = Encoding.UTF8.GetBytes(data);
var binarySignature = Convert.FromBase64String(signature);
```
Then we get an instance of signer by calling `GetSigner` method on `SignerUtilities` helper class by passing signature algorithm name, in our case it is `SHA512WITHRSA` and initilises it with verification mode and signature key. Then we call `BlockUpdate` method on `signer` to set the data we want to verify signature for.
```csharp
var signer = SignerUtilities.GetSigner(SignatureAlgorithm);
signer.Init(false, signatureKey);
signer.BlockUpdate(dataToVerify, 0, dataToVerify.Length);
```
We then call `VerifySignature` method on `signer` by passing the signature bytes.
```csharp
signer.VerifySignature(binarySignature);
```
And return the result of the above method call back to caller.

Complete code for the method is below
```csharp
public bool VerifySignature(string data, string signature, string publicKeyJson)
{
    var signatureKey = JsonConvert.DeserializeObject<RsaPublicKeyParameters>(publicKeyJson).ToRsaKeyParameters();

    var dataToVerify = Encoding.UTF8.GetBytes(data);
    var binarySignature = Convert.FromBase64String(signature);

    var signer = SignerUtilities.GetSigner(SignatureAlgorithm);
    signer.Init(false, signatureKey);
    signer.BlockUpdate(dataToVerify, 0, dataToVerify.Length);

    return signer.VerifySignature(binarySignature);
}
```

Complete code for the wrapper class that implements signing and its verification using RSA can be found at [RsaBcCrypto.cs](https://github.com/kashifsoofi/crypto-sandbox/blob/master/dotnet/src/Sandbox.Crypto/RsaBcCrypto.cs). Unit tests for the wrapper class can be found at [RsaBcCryptoTests.cs](https://github.com/kashifsoofi/crypto-sandbox/blob/master/dotnet/test/Sandbox.Crypto.Tests/RsaBcCryptoTests.cs). Complete project as class library along with tests is at [CryptoSandbox](https://github.com/kashifsoofi/crypto-sandbox/tree/master/dotnet).

[RsaCryptoTests.cs](https://github.com/kashifsoofi/crypto-sandbox/blob/master/dotnet/test/Sandbox.Crypto.Tests/RsaCryptoTests.cs) is also updated to include the tests cases for interoperability between Microsoft RSA cryptography and BouncyCastle.


## References
[https://en.wikipedia.org/wiki/RSA_(cryptosystem)](https://en.wikipedia.org/wiki/RSA_(cryptosystem))
[http://www.bouncycastle.org/csharp/](http://www.bouncycastle.org/csharp/)
[https://src-bin.com/en/q/e7ddf](https://src-bin.com/en/q/e7ddf)
And many more