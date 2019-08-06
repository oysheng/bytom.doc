## Bytom插件wasm接口说明

----

## `createKey`

Create encrypted key.

### Parameters

`Object`:

- `String` - *alias*, name of the key.
- `String` - *auth*, password of the key.

### Returns

`Object`:

- `Object` - *encrypted-key-json*, encrypted key json, stored in web database.

```js
// Request
{
  "alias": "default",
  "auth": "123456"
}

// Result
{
  "crypto": {
    "cipher": "aes-128-ctr",
    "ciphertext": "3d998c92d8b8e030a516258a212d0fb35ce1b74921df56d7d7f567567bf5c3df5a6dbf1d348c8fde5fef56d5fc5dd58e1eaf07fb5efdbdf7c4669689758ecdde",
    "cipherparams": {
      "iv": "5792aefe29dd15d4da406eb1f1e3e6d9"
    },
    "kdf": "scrypt",
    "kdfparams": {
      "dklen": 32,
      "n": 4096,
      "p": 6,
      "r": 8,
      "salt": "3e9ece5e9bc3fe6e92a2565b20e4bca63584a261bbe6ed0c4ccb8f1fb8a56b84"
    },
    "mac": "b9d3e9091027372f49954b20452c0f4c26c0553836b86f2bf7cd4a543ff350fe"
  },
  "id": "9362aad3-f499-48f2-9658-b529a225d2e0",
  "type": "bytom_kd",
  "version": 1,
  "alias": "default",
  "xpub": "a4d4f09a04371516d37e1d27f92c9cb41e4b1e7f62762cf23ed3904a9dfd2d794195862fffd00bf7ac373e5891c8d2eb660dc5ff9c040ec4e01f973bbfd31c23"
}
```

----

## `resetKeyPassword`

Reset key password.

### Parameters

`Object`:

- `String` - *rootXPub*, root pubkey of the key.
- `String` - *oldPassword*, old password of the key.
- `String` - *newPassword*, new password of the key.
- `Object` - *encrypted-key-json*, encrypted key json, get by web database.

### Returns

`Object`:

- `Object` - *encrypted-key-json*, encrypted key json.

```js
// Request
{
  "rootXPub": "a4d4f09a04371516d37e1d27f92c9cb41e4b1e7f62762cf23ed3904a9dfd2d794195862fffd00bf7ac373e5891c8d2eb660dc5ff9c040ec4e01f973bbfd31c23",
  "oldPassword": "123456",
  "newPassword": "654321"
}

// Result
{
  "crypto": {
    "cipher": "aes-128-ctr",
    "ciphertext": "3d998c92d8b8e030a516258a212d0fb35ce1b74921df56d7d7f567567bf5c3df5a6dbf1d348c8fde5fef56d5fc5dd58e1eaf07fb5efdbdf7c4669689758ecdde",
    "cipherparams": {
      "iv": "5792aefe29dd15d4da406eb1f1e3e6d9"
    },
    "kdf": "scrypt",
    "kdfparams": {
      "dklen": 32,
      "n": 4096,
      "p": 6,
      "r": 8,
      "salt": "3e9ece5e9bc3fe6e92a2565b20e4bca63584a261bbe6ed0c4ccb8f1fb8a56b84"
    },
    "mac": "b9d3e9091027372f49954b20452c0f4c26c0553836b86f2bf7cd4a543ff350fe"
  },
  "id": "9362aad3-f499-48f2-9658-b529a225d2e0",
  "type": "bytom_kd",
  "version": 1,
  "alias": "default",
  "xpub": "a4d4f09a04371516d37e1d27f92c9cb41e4b1e7f62762cf23ed3904a9dfd2d794195862fffd00bf7ac373e5891c8d2eb660dc5ff9c040ec4e01f973bbfd31c23"
}
```

----

## `createAccount`

create account.

### Parameters

`Object`:

- `String` - *alias*, account alias.
- `Integer` - *quorum*, the quorum of xpubs.
- `String` - *rootXPub*, root xpub.
- `Integer` - *nextIndex*, index.

### Returns

`Object`:

- `String` - *alias*, account alias.
- `String` - *id*, account ID.
- `String` - *type*, type, it can be empty.
- `Integer` - *quorum*, the quorum of xpubs.
- `Integer` - *key_index*, index.
- `Object` - *xpubs*, array of xpub.

```js
// Request
{
  "alias": "alice",
  "quorum": 1,
  "rootXPub": "a4d4f09a04371516d37e1d27f92c9cb41e4b1e7f62762cf23ed3904a9dfd2d794195862fffd00bf7ac373e5891c8d2eb660dc5ff9c040ec4e01f973bbfd31c23",
  "nextIndex": 1
}

// Result
{
  "alias": "alice",
  "id": "08FO663C00A02",
  "key_index": 1,
  "quorum": 1,
  "xpubs": [
    "2d6c07cb1ff7800b0793e300cd62b6ec5c0943d308799427615be451ef09c0304bee5dd492c6b13aaa854d303dc4f1dcb229f9578786e19c52d860803efa3b9a"
  ]
}
```

----

## `createAccountReceiver`

create account address.

### Parameters

`Object`:

- `Object` - *account*, account object.
  - `String` - *alias*, account alias.
  - `String` - *id*, account ID.
  - `String` - *type*, type, it can be empty.
  - `Integer` - *quorum*, the quorum of xpubs.
  - `Integer` - *key_index*, index.
  - `Object` - *xpubs*, array of xpub.
- `Integer` - *nextIndex*, index.

### Returns

`Object`:

- `String` - *control_program*, control program.
- `String` - *address*, address.

```js
// Request
{
  "account": {
    "alias": "alice",
    "id": "08FO663C00A02",
    "key_index": 1,
    "quorum": 1,
    "xpubs": [
      "2d6c07cb1ff7800b0793e300cd62b6ec5c0943d308799427615be451ef09c0304bee5dd492c6b13aaa854d303dc4f1dcb229f9578786e19c52d860803efa3b9a"
    ]
  },
  "nextIndex": 1
}

// Result
{
    "address": "bm1q5u8u4eldhjf3lvnkmyl78jj8a75neuryzlknk0",
    "control_program": "0014a70fcae7edbc931fb276d93fe3ca47efa93cf064"
}
```

----

## `signTransaction1`

sign transaction.

### Parameters

`Object`:

- `String` - *raw_transaction*, raw transaction.
- `Object` - *signing_instructions*, sign array.
  - `Object` - *derivation_path*, derivation path array.
  - `Object` - *sign_data*, sign data array.
- `Object` - *key*, encrypted key json, get by web database.

### Returns

`Object`:

- `String` - *control_program*, control program.
- `String` - *address*, address.

```js
// Request
{
  "transaction": {
    "raw_transaction": "07010000020161015fb6a63a3361170afca03c9d5ce1f09fe510187d69545e09f95548b939cd7fffa3ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff80fc93afdf01000116001426bd1b851cf6eb8a701c20c184352ad8720eeee90100015d015bb6a63a3361170afca03c9d5ce1f09fe510187d69545e09f95548b939cd7fffa33152a15da72be51b330e1c0f8e1c0db669269809da4f16443ff266e07cc43680c03e0101160014489a678741ccc844f9e5c502f7fac0a665bedb25010003013effffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff80a2cfa5df0101160014948fb4f500e66d20fbacb903fe108ee81f9b6d9500013a3152a15da72be51b330e1c0f8e1c0db669269809da4f16443ff266e07cc43680dd3d01160014cd5a822b34e3084413506076040d508bb12232c70001393152a15da72be51b330e1c0f8e1c0db669269809da4f16443ff266e07cc436806301160014a3f9111f3b0ee96cbd119a3ea5c60058f506fb1900",
    "signing_instructions": [
      {
        "derivation_path": [
          "010100000000000000",
          "0500000000000000"
        ],
        "sign_data": [
          "62a73b6b7ffe52b6ad782b0e0efdc8309bf2f057d88f9a17d125e41bb11dbb88"
        ]
      }
    ]
  },
  "password": "123456",
  "key": {
    "crypto": {
      "cipher": "aes-128-ctr",
      "ciphertext": "3d998c92d8b8e030a516258a212d0fb35ce1b74921df56d7d7f567567bf5c3df5a6dbf1d348c8fde5fef56d5fc5dd58e1eaf07fb5efdbdf7c4669689758ecdde",
      "cipherparams": {
        "iv": "5792aefe29dd15d4da406eb1f1e3e6d9"
      },
      "kdf": "scrypt",
      "kdfparams": {
        "dklen": 32,
        "n": 4096,
        "p": 6,
        "r": 8,
        "salt": "3e9ece5e9bc3fe6e92a2565b20e4bca63584a261bbe6ed0c4ccb8f1fb8a56b84"
      },
      "mac": "b9d3e9091027372f49954b20452c0f4c26c0553836b86f2bf7cd4a543ff350fe"
    },
    "id": "9362aad3-f499-48f2-9658-b529a225d2e0",
    "type": "bytom_kd",
    "version": 1,
    "alias": "default",
    "xpub": "a4d4f09a04371516d37e1d27f92c9cb41e4b1e7f62762cf23ed3904a9dfd2d794195862fffd00bf7ac373e5891c8d2eb660dc5ff9c040ec4e01f973bbfd31c23"
  }
}

// Result
{
  "raw_transaction": "07010000020161015fb6a63a3361170afca03c9d5ce1f09fe510187d69545e09f95548b939cd7fffa3ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff80fc93afdf01000116001426bd1b851cf6eb8a701c20c184352ad8720eeee90100015d015bb6a63a3361170afca03c9d5ce1f09fe510187d69545e09f95548b939cd7fffa33152a15da72be51b330e1c0f8e1c0db669269809da4f16443ff266e07cc43680c03e0101160014489a678741ccc844f9e5c502f7fac0a665bedb25010003013effffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff80a2cfa5df0101160014948fb4f500e66d20fbacb903fe108ee81f9b6d9500013a3152a15da72be51b330e1c0f8e1c0db669269809da4f16443ff266e07cc43680dd3d01160014cd5a822b34e3084413506076040d508bb12232c70001393152a15da72be51b330e1c0f8e1c0db669269809da4f16443ff266e07cc436806301160014a3f9111f3b0ee96cbd119a3ea5c60058f506fb1900",
  "signatures": [
    "0d432e6f0e22da3168d76552273e60d23d432d61b4dac53e6769d39a1097f1cd1bd8e54c7d39eda334803e5c904bc2de2f27ff29748166e0334dcfded20e980b"
  ]
}
```

==========================================================

待开发的API接口：

----

## `createPubkey`

create pubkey.

### Parameters

`Object`:

- `String` - *xpub*, xpub.
- `Integer` - *seed*, seed, default is 1.

### Returns

`Object`:

- `String` - *xpub*, xpub.
- `String` - *pubkey*, public key.
- `Object` - *derived_path*, derived path array.

```js
// Request
{
  "xpub": "2d6c07cb1ff7800b0793e300cd62b6ec5c0943d308799427615be451ef09c0304bee5dd492c6b13aaa854d303dc4f1dcb229f9578786e19c52d860803efa3b9a",
  "seed": 1
}

// Result
{
  "xpub": "2d6c07cb1ff7800b0793e300cd62b6ec5c0943d308799427615be451ef09c0304bee5dd492c6b13aaa854d303dc4f1dcb229f9578786e19c52d860803efa3b9a",
  "pubkey": "ba5a63e7416caeb945eefc2ce874f40bc4aaf6005a1fc792557e41046f7e502f",
  "derived_path": [
    "010100000000000000",
    "0500000000000000"
  ]
}
```

----

## `convertArgument`

convert argument.

### Parameters

`Object`:

- `Obejct Array`
  - `String` - *type*, type.
  - `String` - *value*, value.

### Returns

`Object`:

- `Obejct Array`
  - `String` - *type*, data type.
  - `String` - *value*, value.

```js
// Request
{
  "type": "data",
  "raw_data": {
    "value": "ba5a63e7416caeb945eefc2ce874f40bc4aaf6005a1fc792557e41046f7e502f"
  }
}

//or 

{
  "type": "integer",
  "raw_data": {
    "value": 100
  }
}

//or 

{
  "type": "string",
  "raw_data": {
    "value": "string"
  }
}

//or

{
  "type": "boolean",
  "raw_data": {
    "value": true
  }
}

//or

{
  "type": "address",
  "raw_data": {
    "value": "bm1q5u8u4eldhjf3lvnkmyl78jj8a75neuryzlknk0"
  }
}
```

```js
// Result
{
  "data": "ba5a63e7416caeb945eefc2ce874f40bc4aaf6005a1fc792557e41046f7e502f"
}

//or 

{
  "data": "64"
}

//or

{
  "data": "737472696e67"
}

//or 

{
  "data": "01"
}

//or 

{
  "data": "0014a70fcae7edbc931fb276d93fe3ca47efa93cf064"
}
```