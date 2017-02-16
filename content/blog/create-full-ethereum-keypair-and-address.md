+++
date = "2017-02-15T13:36:41+01:00"
title = "create full ethereum keypair and address"
draft = true

+++

## Coming soon

# Generating a usable Ethereum address and its corresponding keys

This article is a guide on how to generate an ECDSA private key and derive its Ethereum address. 

using the terminal and using python (ou autre)

## Generating the curves
1. Generate Elliptic Curve secp256k1 keypair and display its properties using openssl commands ecparam and ec. First part generates the private key and the second one prints the private and public parts as hexadecimal.
more info here: https://wiki.openssl.org/index.php/Command_Line_Elliptic_Curve_Operations 
```
openssl ecparam -name secp256k1 -genkey -noout | openssl ec -text -noout
```

2. export the private key as an hexadecimal string (remove the heading 00 if any)
```

```

3. export the public key as an hexadecimal string *without the 0x04 prefix*

4. import it in geth

5. that the keccak-256sum -x of it (last 20 bytes)

openssl ecparam -name secp256k1 -genkey -noout |
openssl ec -text -noout 2> /dev/null | grep priv:$ -A 3 | tail -n +2 | tr -d '\n :'

