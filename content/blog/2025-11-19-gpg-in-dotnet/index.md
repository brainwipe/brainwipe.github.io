---
title: "PGP in .NET with GnuPG and Bouncy Castle"
date: 2025-11-19
categories: pgp gpg encryption bouncy castle .net core
---
In this blog we are going to encrypt and decrypt some text using PGP. Keys will be asymmetric and made by [GnuPG (GPG)](https://www.gnupg.org/) but the act of encrypt and decrypt will be a .NET method using C# .NET [Bouncy Castle](https://www.bouncycastle.org/). We're doing this on Windows using the [Ubuntu WSL2](https://learn.microsoft.com/en-us/windows/wsl/install) in [Windows Terminal](https://github.com/microsoft/terminal) for GPG, because it already has everything installed.

> Code is offered under GPL without warranty.

This is the sort of thing I forget straight after I learn it, which is why I'm writing it here.

## Background
When calling an API from .NET, you might be required to decrypt the contents of the body using PGP. It protects against [man in the middle attacks](https://en.wikipedia.org/wiki/Man-in-the-middle_attack) and mis-configuration of the endpoints. Think of it as a last line of defence.

GPG uses the [AES256](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard) algorithm by default, which we're not going to touch here. Before GPG, I would have used [openssl](https://www.openssl.org/) to generate a key pair but it's better to use the "key ring" system of GPG.

## Generate Keys
In asymmetric encryption, you generate a pair of keys at the same time. The keys are called public and private. The public key is shorter and is used to encrypt the data, the private key is longer and used to decrypt it. You send the public key to the person encrypting the data and keep the private key safe. The private key has a "passkey", which is a password that you need to use it.

We're going to use GPG to generate the keys for PGP. GPG stores the keys we generate in  `/root/.gnupg/pubring.kbx` but you don't get them from there, you must export them. We'll come onto that.

On Windows open Terminal with Ubuntu WSL2. We'll be using that for all the command line actions below.

Start the key generation process by selecting RSA and RSA.

```bash
# gpg --full-gen-key
gpg (GnuPG) 2.2.27; Copyright (C) 2021 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
  (14) Existing key from card
Your selection? 1
```

For key length balance between security (bigger is better) and resources/speed (bigger is slower). For this example, our payload is irregular and large, so a bigger key is fine.

NB: If you choose 4096 then you won't be able to store it in the AWS Parameter store because the size of the file is 4096 *and* extra file furniture.

We're going to set the key to never expire as if the private key were to be compromised, it would be changed immediately. Check with you infosec for their risk appetite.

```bash
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (3072) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 0
Key does not expire at all
Is this correct? (y/N) y
```

Fill in the user ids - some systems will want to use the id to pick the key from the store. If you have multiple systems with different keys (test and production, for example) then make sure the naming of the keys are unique.

For example, the user Id:
`The Plastic Neuron (Blog Example Key) <brainwiped@gmail.com>` 

Might be better with a different comment to make it unique:
`The Plastic Neuron (Test System) <brainwiped@gmail.com>` 
`The Plastic Neuron (Production System) <brainwiped@gmail.com>` 

```bash
GnuPG needs to construct a user ID to identify your key.

Real name: The Plastic Neuron
Email address: brainwiped@gmail.com
Comment: Blog Example Key
You selected this USER-ID:
    "The Plastic Neuron (Blog Example Key) <brainwiped@gmail.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? o
```

You will then be asked for a passphrase (password) to secure the private key. Keep this secret too. Never send it anywhere. Your private key cannot be used without it. You will be asked to confirm it after.

```bash
    ┌──────────────────────────────────────────────────────┐
    │ Please enter the passphrase to                       │
    │ protect your new key                                 │
    │                                                      │
    │ Passphrase: ________________________________________ │
    │                                                      │
    │       <OK>                              <Cancel>     │
    └──────────────────────────────────────────────────────┘
```

Now GPG will generate the keys.

```bash
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

gpg: key 3B5B7E567A83D4F6 marked as ultimately trusted
gpg: revocation certificate stored as '/root/.gnupg/openpgp-revocs.d/D751039CF43E0D022585F2F63B5B7E567A83D4F6.rev'
public and secret key created and signed.

pub   rsa4096 2025-11-12 [SC]
      D751039CF43E0D022585F2F63B5B7E567A83D4F6
uid           The Plastic Neuron (Blog Example Key) <brainwiped@gmail.com>
sub   rsa4096 2025-11-12 [E]
```

The keys have now been created and stored as binary in `/root/.gnupg/pubring.kbx`. Before we go onto export, let's look at some useful functions.

## Useful GPG Functions

### Listing Keys

```bash
# gpg --list-keys
gpg: checking the trustdb
gpg: marginals needed: 3  completes needed: 1  trust model: pgp
gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 2u
/root/.gnupg/pubring.kbx
------------------------
pub   rsa4096 2025-11-12 [SC]
      D751039CF43E0D022585F2F63B5B7E567A83D4F6
uid           [ultimate] The Plastic Neuron (Blog Example Key) <brainwiped@gmail.com>
sub   rsa4096 2025-11-12 [E]

```

The most important part is the 40 character fingerprint, but for completeness, here's the detail.

At the top are the details of the gpg store. Marginals are keys that need more validation to fully trust their source. Completes are keys where you have verified the ID of the sender.

- `pub   rsa4096 2025-11-12` 
	- `pub` main public key
	- RSA is the key algorithm, 4096 is the size and the date is on the end. 
	- `[SC]` means that the key is used for Signing and Certifying
- `D751039CF43E0D022585F2F63B5B7E567A83D4F6`
	- the fingerprint, it's how we're going to select the key for export
-  ` [ultimate] The Plastic Neuron (Blog Example Key)` unique ID for the key
- `sub   rsa4096 2025-11-12` 
	- the sub key
	- used for encryption

### Deleting keys
Must do the secret (private) key first. Then you can delete the public key.

```
gpg --delete-secret-key D751039CF43E0D022585F2F63B5B7E567A83D4F6
gpg --delete-key D751039CF43E0D022585F2F63B5B7E567A83D4F6
```

## Export Keys

Before we can use them in our C# code, we must export them. The default export format for a key is a binary format but we're going to pass them in as strings, so use the `--armor` keyword. Although the file extension is `.gpg`, it's still a 

Private and public keys have separate commands:

```bash
# gpg --output blog-example-pub-ascii.gpg --armor --export D751039CF43E0D022585F2F63B5B7E567A83D4F6
```

Make sure you have a safe place to store the private key you're going to create. Do not commit it to your codebase.

```bash
# gpg --output blog-example-private-ascii.gpg --armor --export-secret-keys D751039CF43E0D022585F2F63B5B7E567A83D4F6
```

## Using GnuPG Keys with Bouncy Castle

Once the keys are made and stored somewhere sensible.

Import the [Bouncy Castle](https://www.nuget.org/packages/BouncyCastle.Cryptography/) crypto nuget package. Make sure you get the official one and that it's up to date.

To use this code, new up the class (or inject it using dependency injection) and call `Encrypt` or `Decrypt`.

```csharp
﻿using Org.BouncyCastle.Bcpg.OpenPgp;
using Org.BouncyCastle.Crypto;
using Org.BouncyCastle.Crypto.Encodings;
using Org.BouncyCastle.Crypto.Engines;
using System.IO;
using System.Text;

namespace Lang.Encryption;

// THIS CODE IS NOT PRODUCTION READY, PUT IN ERROR HANDLING AND LOGGING!
public class PgpEncryption()
{
	public string Encrypt(string input, string publicKey)
	{
		var inputBytes = Encoding.UTF8.GetBytes(input);
		var cipher = new Pkcs1Encoding(new RsaEngine());
		cipher.Init(true, LoadPgpPublicKey(publicKey));
		int blockSize = cipher.GetInputBlockSize();
		var output = new List<byte>();
		for (int chunkPosition = 0; chunkPosition < inputBytes.Length; chunkPosition += blockSize)
		{
			var chunkSize = blockSize;
			if (chunkPosition + blockSize > inputBytes.Length)
			{
				chunkSize = inputBytes.Length - chunkPosition;
			}
			output.AddRange(cipher.ProcessBlock(inputBytes, chunkPosition, chunkSize));
		}
		
		return Convert.ToBase64String(output.ToArray());
	}

	public string Decrypt(string encrypted, string privateKey, string passKey)
	{
		var encryptedBytes = Convert.FromBase64String(encrypted);
		var cipher = new Pkcs1Encoding(new RsaEngine());
		cipher.Init(false, LoadPgpPrivateKey(privateKey, passKey));
		int blockSize = cipher.GetInputBlockSize();
		var decryptedBytes = new List<byte>();
		for (int chunkPosition = 0; chunkPosition < encryptedBytes.Length; chunkPosition += blockSize)
		{
			var chunkSize = blockSize;
			if (chunkPosition + blockSize > encryptedBytes.Length)
			{
				chunkSize = encryptedBytes.Length - chunkPosition;
			}

			decryptedBytes.AddRange(cipher.ProcessBlock(encryptedBytes, chunkPosition, chunkSize));
		}
		return Encoding.UTF8.GetString(decryptedBytes.ToArray());
	}

	private static ICipherParameters LoadPgpPublicKey(string armoredKeyString)
	{
		using var decoderStream = PgpUtilities.GetDecoderStream(new MemoryStream(Encoding.ASCII.GetBytes(armoredKeyString)));
		var pgpPubRingBundle = new PgpPublicKeyRingBundle(decoderStream);

		foreach (var keyRing in pgpPubRingBundle.GetKeyRings())
		{
			foreach (var key in keyRing.GetPublicKeys())
			{
				if (key.IsEncryptionKey)
				{
					return key.GetKey();
				}
			}
		}
		
		throw new ArgumentException("No encryption key found in the provided PGP public key block.");
	}

	private static ICipherParameters LoadPgpPrivateKey(string armoredKeyString, string passphrase)
	{
		using var keyStream = new MemoryStream(Encoding.ASCII.GetBytes(armoredKeyString));
		using var decoderStream = PgpUtilities.GetDecoderStream(keyStream);

		var pgpSecRingBundle = new PgpSecretKeyRingBundle(decoderStream);

		foreach (var keyRing in pgpSecRingBundle.GetKeyRings())
		{
			foreach (var secretKey in keyRing.GetSecretKeys())
			{
				var privateKey = secretKey.ExtractPrivateKey(passphrase.ToCharArray());
				if (privateKey != null)
				{
					return privateKey.Key;
				}
			}
		}

		throw new ArgumentException("No private key could be found in the provided PGP block.");
	}
}
```

## Helpful reference

If found this article particularly helpful:

- [Comprehensive Yet Simple Guide for GPG Key/Subkey Encryption, Signing, Verification & Other Common Operations](https://medium.com/code-oil/comprehensive-yet-simple-guide-for-gpg-key-subkey-encryption-signing-verification-other-common-c28fd868cbe7)