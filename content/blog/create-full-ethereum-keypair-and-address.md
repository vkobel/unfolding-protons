+++
date = "2017-02-15T13:36:41+01:00"
title = "create full ethereum keypair and address"
draft = false

+++

# Generating a usable Ethereum wallet and its corresponding keys

### Contents
- [Generating the EC private key](#generating-the-ec-private-key)
- [Derive the Ethereum address from the public key](#derive-the-ethereum-address-from-the-public-key)
- [Importing the private key to geth](#importing-the-private-key-to-geth)
- [Complete example](#complete-example)

This article is a guide on how to generate an ECDSA private key and derive its Ethereum address. Using OpenSSL and keccak-256sum from a bash terminal.

You can find a working implementation of keccak-256sum [here](https://github.com/maandree/sha3sum). Cool thing, it exists as a package in the [Arch User Repository as well](https://aur.archlinux.org/packages/sha3sum/).

**Warning SHA3 != keccak**. Ethereum is using the keccak-256 algorithm and not the standard sha3. More info on [Stackoverflow](http://ethereum.stackexchange.com/questions/550/which-cryptographic-hash-function-does-ethereum-use).

I have a [repository](https://github.com/vkobel/ethereum-generate-wallet) with complete scripts in both bash and python if you'd like.

## Generating the EC private key
First of all we use OpenSSL ecparam command to generate an elliptic curve private key. Ethereum standard is to use the secp256k1 curve. Bitcoin is using this curve as well.

This command will print the private key in PEM format (using the wonderful ASN.1 key structure) on stdout. If you want more OpenSSL info on elliptic curves, please feel free to [dig further](https://wiki.openssl.org/index.php/Command_Line_Elliptic_Curve_Operations).

```bash
> openssl ecparam -name secp256k1 -genkey -noout
-----BEGIN EC PRIVATE KEY-----
MHQCAQEEIFDLYO9KuwsC4ej2UsdA4SYk7s3lb8aZuW+B8rjugrMmoAcGBSuBBAAK
oUQDQgAEsNjwhFoLKLXGBxfpMv3ILhzg2FeySRlFhtjfi3s8YFZzJtmckVR3N/YL
JLnUV7w3orZUyAz77k0ebug0ILd1lQ==
-----END EC PRIVATE KEY-----
```

On its own this command is not what we want, but if you pipe it with the ec command it will display both private and public part in hexadecimal format, exactly what we want! Let's do it:

```bash
> openssl ecparam -name secp256k1 -genkey -noout | openssl ec -text -noout
read EC key
Private-Key: (256 bit)
priv:
    20:80:65:a2:47:ed:be:5d:f4:d8:6f:bd:c0:17:13:
    03:f2:3a:76:96:1b:e9:f6:01:38:50:dd:2b:dc:75:
    9b:bb
pub:
    04:83:6b:35:a0:26:74:3e:82:3a:90:a0:ee:3b:91:
    bf:61:5c:6a:75:7e:2b:60:b9:e1:dc:18:26:fd:0d:
    d1:61:06:f7:bc:1e:81:79:f6:65:01:5f:43:c6:c8:
    1f:39:06:2f:c2:08:6e:d8:49:62:5c:06:e0:46:97:
    69:8b:21:85:5e
ASN1 OID: secp256k1
```

This command decodes the ASN.1 structure and derives the public key from the private one. Sometimes, OpenSSL is adding a null byte (0x00) in front of the private part, I don't know why it does that but you have to trim any leading zero bytes in order to use it with Ethereum. 

The private key **must be 32 bytes and not begin with 0x00** and the public one **must be uncompressed and 64 bytes long** or 65 with the constant 0x04 prefix. More on that on the next section. 

## Derive the Ethereum address from the public key

The public key is what we need in order to derive its Ethereum address. Every EC public key begins with the 0x04 prefix before giving the location of the two point on the curve. You should remove this leading 0x04 byte in order to hash it correctly. I recommend the excellent [Cloudflare article on elliptic curves](https://blog.cloudflare.com/a-relatively-easy-to-understand-primer-on-elliptic-curve-cryptography/).

Use any method you like to get it in the form of an hexadecimal string (without line return nor semicolon). Here is an example of extraction using *grep*, *tail*, *tr* and *sed*. I'm sure there's an easier way to do but I'm not a bash guru. You can find the scripts (python as well) on [my github repository](https://github.com/vkobel/ethereum-generate-wallet).

**The examples below assume you saved the output of the openssl commands in a file named 'Key'**

```bash
# Extract the public key and remove the EC prefix 0x04
> cat Key | grep pub -A 5 | tail -n +2 | 
            tr -d '\n[:space:]:' | sed 's/^04//' > pub
836b35a026743e823a90a0ee3b91bf615c6a757e2b60b9e1dc1826fd0dd16106f7bc1e8179f665015f43c6c81f39062fc2086ed849625c06e04697698b21855e
```
The pub file now contains the hexadecimal value of the public key without the 0x04 prefix.

An Ethereum address is made of 20 bytes (40 hex characters long), it is commonly represented by adding the 0x prefix. In order to derive it, one should take the keccak-256 hash of the hexadecimal form of a public key, then keep only the last 20 bytes (aka get rid of the first 12 bytes).

Simply pass the file containing the public key in hexadecimal format to the keccak-256sum command. **Do not forget to use the '-x' option** in order to interpret it as hexadecimal and not a simple string.i

In the below snippet, the '-l' option is to output as lowercase. Then I use *tr* to delete the trailing ' -' from the stdout of the keccak command. Finally, I take only the last 40 characters using the **tail** command. Yes, I specify 41 in order to get 40 characters, probably because of a line return.

```bash
# Generate the hash and take the address part
> cat pub | keccak-256sum -x -l | tr -d ' -' | tail -c 41
0bed7abd61247635c1973eb38474a2516ed1d884
```
Which gives us the Ethereum address `0x0bed7abd61247635c1973eb38474a2516ed1d884`

## Importing the private key to geth

Let's import the private key to geth and therefore validating the derivation of the address.

To begin with, we format the private key the same way we did for the public one above. The only exception is removing the leading 0x00 instead of the 0x04:
```bash
# Extract the private key and remove the leading zero byte
> cat Key | grep priv -A 3 | tail -n +2 | 
            tr -d '\n[:space:]:' | sed 's/^00//' > priv
208065a247edbe5df4d86fbdc0171303f23a76961be9f6013850dd2bdc759bbb
```
To import it to geth, use the `account import` feature like this:

```bash
> geth account import priv
Your new account is locked with a password. Please give a password. Do not forget this password.
Passphrase: 
Repeat passphrase: 
Address: {0bed7abd61247635c1973eb38474a2516ed1d884}
```
**YAY!**, the address retured by geth is the same we computed. Note that geth will ask you a passphrase to encrypt your private key.

You're now ready to use the new account with geth. Of course, it would be easier to take advantage of the `geth account new` feature in order to quickly setup an Ethereum account. But using this way gives you the power of knowing your public and unencrypted private keys. In addition, this would be useful to generate a secure Ethereum account completely off chain. 

## Complete example

```bash
# Generate the private and public keys
> openssl ecparam -name secp256k1 -genkey -noout | 
  openssl ec -text -noout > Key

# Extract the public key and remove the EC prefix 0x04
> cat Key | grep pub -A 5 | tail -n +2 |
            tr -d '\n[:space:]:' | sed 's/^04//' > pub

# Extract the private key and remove the leading zero byte
> cat Key | grep priv -A 3 | tail -n +2 | 
            tr -d '\n[:space:]:' | sed 's/^00//' > priv

# Generate the hash and take the address part
> cat pub | keccak-256sum -x -l | tr -d ' -' | tail -c 41 > address

# (Optional) import the private key to geth
> geth account import priv

```

