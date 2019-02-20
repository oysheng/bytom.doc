## bufferserver API:

### list-utxos

    http://127.0.0.1:3100/dapp/list-utxos

#### request
```js
{
  "asset": "d8202143b5ae3c607fbeb0b9f920149f551ed13c1a1c70f23f61a2cd19cc6c6f",
  "program": "2022e829107201c6b975b1dc60b928117916285ceb4aa5c6d7b4b8cc48038083e074037caa8700c0",
  "sort": {
    "by": "amount",
    "order": "desc"
  }
}
```

#### response
```js
[
  {
    "hash": "cc4e6501ce7d566e337523eeee0a696b38a20a040c9621b66fe4abaf86dedd81",
    "asset": "d8202143b5ae3c607fbeb0b9f920149f551ed13c1a1c70f23f61a2cd19cc6c6f",
    "amount": 200
  },
  {
    "hash": "cc4e6501ce7d566e337523eeee0a696b38a20a040c9621b66fe4abaf86dedd81",
    "asset": "d8202143b5ae3c607fbeb0b9f920149f551ed13c1a1c70f23f61a2cd19cc6c6f",
    "amount": 100
  }
]
```

-----

### list-balances

    http://127.0.0.1:3100/dapp/list-balances

#### request
```js
{
  "address": "sm1qg5pq4qvk79h6nxt5ksqvu45rt7a9cstcfxctwq",
  "asset": "d8202143b5ae3c607fbeb0b9f920149f551ed13c1a1c70f23f61a2cd19cc6c6f"
}
```

#### response
```js
[
  {
    "address": "sm1qg5pq4qvk79h6nxt5ksqvu45rt7a9cstcfxctwq",
    "asset": "d8202143b5ae3c607fbeb0b9f920149f551ed13c1a1c70f23f61a2cd19cc6c6f",
    "amount": 10000,
    "create_at": 1550650095
  },
  {
    "address": "sm1qg5pq4qvk79h6nxt5ksqvu45rt7a9cstcfxctwq",
    "asset": "d8202143b5ae3c607fbeb0b9f920149f551ed13c1a1c70f23f61a2cd19cc6c6f",
    "amount": -10000,
    "create_at": 1550650095
  }
]
```

-----

### update-base

    http://127.0.0.1:3100/dapp/update-base

#### request
```js
{
	"program":"001445020a8196f16fa99974b400ce56835fba5c4178",
	"asset":"ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff"
}
```

#### response

null

-----

### update-utxo

    http://127.0.0.1:3100/dapp/update-utxo

#### request
```js
{
  "hash": "db2dfa5e98dc9c94596b0689abb18b34e04003ad613a731d8ccf0c56b49400f3"
}
```

#### response

null

-----

### update-balance

    http://127.0.0.1:3100/dapp/update-balance

#### request
```js
{
  "address": "sm1qg5pq4qvk79h6nxt5ksqvu45rt7a9cstcfxctwq",
  "asset": "d8202143b5ae3c607fbeb0b9f920149f551ed13c1a1c70f23f61a2cd19cc6c6f",
  "amount": 300000000,
  "tx_id": "d9b5acb305a8272d6ee29c606e85c84da258bacf7e6fd92fde59fcb5b0a1a17a"
}
```

#### response

null