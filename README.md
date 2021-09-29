# HPB Batch Sending Tool

A simple tool to create batch-transactions to send HPB (not HRC20) on the HPB network
to multiple destinations in a single transaction.

![HPB batch sender](http://url/to/img.png)

## Features

- allows you to use less than 21000 gas per receiver, when sending HPB to 4 addresses or more
- validates addresses before sending
- checks if addresses fulfill all needed state-requirements (and does not allow to send if these 
  are not fulfilled).


## Limitations

- [MetaMask](https://metamask.io/) must be installed in the browser 
- It's only cheaper than single transactions when sending to 4 or more destinations at once
- It only supports private key addresses **NO SmartContracts**!
- The destination addresses **must have been used before on chain** (must have had some 
  transaction in the past or some balance --> see below for technical details about this)

## HPB technical details

The tool simply creates a "contract deployment" transaction (i.e. transaction where `to:` is `null`).
The `data` field ist constructed with HPB [EVM OP-Codes](https://ethervm.io/) which send HPB 
to all the destinations.

In essence it just uses one OP-Code [`CALL`](https://ethervm.io/#F1) per destination to send the HPB
and at the end one final OP-Code [`SELFDESTRUCT`](https://ethervm.io/#FF) to collect and refund the 
remaining HPB to the sender and also to benefit from the gas refund (each `SELFDESTRUCT` OP-Code
reduces the gas-counter of a transaction by 24000).

All the remaining OP-Codes are just `PUSH` or `DUP` OP-Codes to set up the values on the stack
which are expected by `CALL` and `SELFDESTRUCT`.

### Per destination address

The code in `data` per destination address looks like this (`xxx...xxx` stands for the address):

```
60 00  80 80 80   68 0000abcdef12345678  73 xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx 82 f1
```

This is in detail:

- `60 00` = `PUSH1 00`: --> push `0x00` onto the stack
- `80` = `DUP1`: --> duplicate the last value on stack (the `0x00` from above)
- `80` = `DUP1`: --> duplicate the last value on stack (the `0x00` from above)
- `80` = `DUP1`: --> duplicate the last value on stack (the `0x00` from above)
- `68 0000abcdef12345678` = `PUSH9 xxxxxxx`: --> push a 9-byte long value to the stack, which will be 
   the amount to send (in wei, as a hexadecimal number, so the `0000abcdef12345678` in this example 
   represent `0.048358647703819896` HPB to be sent)
- `73 xxxx…xxxxx` = `PUSH20 xxx…xxxx`: --> push a 20-byte long value to the stack, which will be 
   the destination address
- `82` = `DUP3` --> duplicate the 3rd value (copy another `0x00` from the ones we have pushed/dup'ed above) 

At this point the EVM stack looks like this (compare with arguments of OP-Code [`CALL`](https://ethervm.io/#F1):

```
00         // "gas": if we set 0, the default minimum if 2300 will be used, is ok as we only send to addresses (no contracts)
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx    // "addr": destination 
0000abcdef12345678         // "value" wei to send with this call
00                         // "argsOffset": no arguments
00                         // "argsLength": no arguments
00                         // "retOffset": no return data expected
00                         // "retLength": no return data expected
```

- And then one final OP-Code: `f1` --> `CALL`: This will execute this transfer and send the ETH
  to this address (and will also place one element on the stack). For efficiency reason we ignore
  the returned `success` element and do neither check it, nor `POP` it from the stack.

All of this will be added **per destination address**!


### Collect remaining HPB at the end:

After the last destination the following code is added:

```
73  xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx  ff
```

This is in detail:

- `73 xxxx…xxxxx` = `PUSH20 xxx…xxxx`: --> push a 20-byte long value to the stack, which will be 
  the address where the remaing funds of the transaction should be sent to.
- `ff` = [`SELFDESTRUCT`](https://ethervm.io/#FF): Selfdestruct will send all remaining funds to
  the last address on the stack (which we pushed there in the step before) and self-destructs the
  contract, which will reduce the transaction's gas counter again by 24000.


### Note about `CALL`
When using `CALL` to send ETH there is one important detail: if the receiver account of `CALL` does
not exist yet (is a "new" account) then there will be extra gascosts of 25000 gas 
(see `G_newaccount` in Appendix G in the [Yellow paper](https://ethereum.github.io/yellowpaper/paper.pdf))!

"New account" in this context means, that the account does not exist in the Ethereum state trie yet.
So any address which never had sent any transaction (`nonce` still 0) or has never had any balance
does not exist in the state trie yet, and will be therefore much more expensive to send to.

--> This is the rationale why this tool checks before sending if all destination addresses have
    been used before (otherwise sending in batch would be more expensive than a single transaction).


## Development setup

This project was built with [VueJS 3](https://v3.vuejs.org/guide/introduction.html).

```
npm install
```

### Compiles and hot-reloads for development
```
npm run serve
```

### Compiles and minifies for production
```
npm run build
```

### Lints and fixes files
```
npm run lint
```

