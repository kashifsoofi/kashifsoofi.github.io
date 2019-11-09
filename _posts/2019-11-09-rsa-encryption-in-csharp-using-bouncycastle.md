---
title:  "RSA Encryption In C# using BouncyCastle.Net"
date:   2019-11-09
categories:
  - cryptography
tags:
  - C#
  - rsa
  - crytpography
  - bouncy castle
---
Sample class library implementing RSA encryption using Bouncy Castle (1.8.5)

## Introduction
**RSA (Rivest–Shamir–Adleman)** is a public-key cryptosystem. In such a cryptosystem, a pair of keys is used often called private and public key pair.

Public key cryptosystems are used for 2 major use cases
1. Encryption
2. Verification

Focus of this article is encryption. With a public key cryptosystem, private key is always kept secure by the owner and public key is publically accessible. Encryption is always done with a public key, this ensures that only the owner of private key can access the data unencrypted and will remain private between the encrytor and owner of private key.

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

### Encryption
Encrypt method accepts a string and `RsaPublicKeyParameters` serialized as json, encrypts string with key using `OaepSHA256` padding and returns a base64 encoded encrypted string.  

We will start by deserializing public key json and converting it to Bouncy Castle's `RsaKeyParameters`.
```csharp
var encryptionKey = JsonConvert.DeserializeObject<RsaPublicKeyParameters>(publicKeyJson).ToRsaKeyParameters();
```
Then we get an instance of rsa engine by calling `GetCipher` method on `CipherUtilities` helper class by passing transformation name, in our case it is `RSA/ECB/OAEPWithSHA256AndMGF1Padding` and initilises it with encryption mode and encryption key parameters.
```csharp
var cipher = CipherUtilities.GetCipher(Algorithm);
cipher.Init(true, encryptionKey);
```
We then convert plain text string to bytes and then call `ApplyCipher` helper method to apply crytpo transformation, in the context of encrypt method it would be encrypt plain text bytes using rsa public key.
```csharp
var dataToEncrypt = Encoding.UTF8.GetBytes(plainText);
var encryptedData = ApplyCipher(dataToEncrypt, cipher, DefaultRsaBlockSize);
```
We then convert encrypted data to base64 string and return to caller.

Complete code for the method is below
```csharp
public string Encrypt(string plainText, string publicKeyJson)
{
    var encryptionKey = JsonConvert.DeserializeObject<RsaPublicKeyParameters>(publicKeyJson).ToRsaKeyParameters();

    var cipher = CipherUtilities.GetCipher(Algorithm);
    cipher.Init(true, encryptionKey);

    var dataToEncrypt = Encoding.UTF8.GetBytes(plainText);
    var encryptedData = ApplyCipher(dataToEncrypt, cipher, DefaultRsaBlockSize);
    return Convert.ToBase64String(encryptedData);
}
```

### Decryption
Decrypt method works in conjunction with Encrypt method above, it accepts base64 encoded string and `RsaPrivateKeyParameters` serialized as json. It imports key, performs decryption and returns plain text.

We will start by deserializing private key json and converting it to Bouncy Castle's `RsaPrivateKeyParameters`.
```csharp
var decryptionKey = JsonConvert.DeserializeObject<RsaPrivateKeyParameters>(privateKeyJson).ToRsaPrivateCrtKeyParameters();
```
Then we get an instance of rsa engine by calling `GetCipher` method on `CipherUtilities` helper class by passing transformation name, in our case it is `RSA/ECB/OAEPWithSHA256AndMGF1Padding` and initilises it with decryption mode and decryption key parameters.
```csharp
var cipher = CipherUtilities.GetCipher(Algorithm);
cipher.Init(false, decryptionKey);
```
We will start by creating an instance of RSA and importing key.
```csharp
var rsa = RSA.Create();
rsa.ImportParameters(rsaParameters);
```
We then convert base64 encrypted string to bytes and then call `ApplyCipher` helper method to apply crytpo transformation, in the context of decrypt method it would be decrypt encrypted bytes using rsa private key.
```csharp
var dataToDecrypt = Convert.FromBase64String(encryptedData);
var decryptedData = ApplyCipher(dataToDecrypt, cipher, blockSize);
```
We then convert decrypted data to string and return to caller.

Complete code for the method is below
```csharp
public string Decrypt(string encryptedData, string privateKeyJson)
{
    var decryptionKey = JsonConvert.DeserializeObject<RsaPrivateKeyParameters>(privateKeyJson).ToRsaPrivateCrtKeyParameters();

    var cipher = CipherUtilities.GetCipher(Algorithm);
    cipher.Init(false, decryptionKey);

    int blockSize = decryptionKey.Modulus.BitLength / 8;

    var dataToDecrypt = Convert.FromBase64String(encryptedData);
    var decryptedData = ApplyCipher(dataToDecrypt, cipher, blockSize);
    return Encoding.UTF8.GetString(decryptedData);
}
```

### `ApplyCipher` Helper
`ApplyCipher` is a helper method that accepts an array of bytes, cipher and block size. It creates an input stream and applies the cipher on each block of the input and returns the transformed bytes to the caller.

Complete code of the method is below
```csharp
private byte[] ApplyCipher(byte[] data, IBufferedCipher cipher, int blockSize)
{
    var inputStream = new MemoryStream(data);
    var outputBytes = new List<byte>();

    int index;
    var buffer = new byte[blockSize];
    while ((index = inputStream.Read(buffer, 0, blockSize)) > 0)
    {
        var cipherBlock = cipher.DoFinal(buffer, 0, index);
        outputBytes.AddRange(cipherBlock);
    }

    return outputBytes.ToArray();
}
```

Complete code for the wrapper class that implements encryption and decryption using RSA can be found at [RsaBcCrypto.cs](https://github.com/kashifsoofi/crypto-sandbox/blob/master/dotnet/src/Sandbox.Crypto/RsaBcCrypto.cs). Unit tests for the wrapper class can be found at [RsaBcCryptoTests.cs](https://github.com/kashifsoofi/crypto-sandbox/blob/master/dotnet/test/Sandbox.Crypto.Tests/RsaBcCryptoTests.cs). Complete project as class library along with tests is at [CryptoSandbox](https://github.com/kashifsoofi/crypto-sandbox/tree/master/dotnet).

[RsaCryptoTests.cs](https://github.com/kashifsoofi/crypto-sandbox/blob/master/dotnet/test/Sandbox.Crypto.Tests/RsaCryptoTests.cs) is also updated to include the tests cases for interoperability between Microsoft RSA cryptography and BouncyCastle.

## References
[https://en.wikipedia.org/wiki/RSA_(cryptosystem)](https://en.wikipedia.org/wiki/RSA_(cryptosystem))
[http://www.bouncycastle.org/csharp/](http://www.bouncycastle.org/csharp/)
[https://src-bin.com/en/q/e7ddf](https://src-bin.com/en/q/e7ddf)
And many more