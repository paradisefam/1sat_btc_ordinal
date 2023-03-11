# 1 Sat Ordinals

An ordinals protocol implementation on Bitcoin SV.

## Motivation

There are several major differences between chains like BTC where ordinals originated, and a classic Bitcoin system like BSV. Most notably, BSV does not have taproot or segwit, so does not need a workaround to store large data files. BSV also has a pre-ordinals data protocol known as "B". The on-chain data structure is almost identical to ordinal inscriptions.

BSV has several other interesting protocols for tagging on-chain data that can be used to extend the capabilities of ordinals much further. 1Sat Ordinals can be created using many existing protocols for inscription. This enables things like minting large videos (> 10MB) and attaching them to a real world geolocation.

Ordinals target a specific satoshi but in practice a range is typically used to satisfy dust limits on other chains. Since BSV has no dust limit, we can dramatically simplify things by truly inscribing a single satoshi.

This satoshi is the ordinal and can be sent to any wallet that recognizes it.

## Important note for wallet implementors

It is very importent for wallets to keep single satoshi unspent outputs seperate from the rest of your UTXOs. To help isolate ordinals and help prevent accidental loss, we separate funding addresses from destinations meant for holding 1sat ordinals.

Those familliar with the Run token protocol will recall a "purse" was used to fund jigs. We use the same termonology here. A purse funds the network fees for inscriptions and transfers, while a seperat wallet holds the 1sat ordinals as to not comingle them with funding sats.

# Inscriptions

## Creating an Inscription

Creating an inscription is much easier than on other blockchains. To summarize the transaction template:

```bash
Input - Any valid utxo
Output #1 - P2PKH (1 Satoshi)
Output #2 - OP_RETURN INSCRIPTION
```

The output with this script must lock exactly 1 Sat.

### 1SAT P2PKH

Spending a 1sat ordinal is as simple as sending the 1 sat output to another address and retaining the "1sat" prefix. `1SAT_P2PKH` refers to a standard p2pkh output with a single sat value and the porefix included. If the value is higher, or the prefix is missing, it will no longer be picked up by 1Sat Ordinal indexers. You can revive these transactions be spending them to 1sat outputs containing the prefix again.

```bash
OP_PUSHDATA1 <"1sat"> OP_DROP OP_DUP OP_HASH160 <pubkeyhash> OP_EQUALVERIFY OP_CHECKSIG
```

### OP_RETURN

Now that the 1sqt output is created, add a second output for the inscription. `OP_FALSE OP_RETURN` and the ordinal protocol identifier `ord` to tag the inscription.

```
o1 - 1SAT_P2PKH
o2 - OP_FALSE OP_RETURN 0x6F7264
```

Notice we do not add `OP_FALSE` so we can spend this transaction.

### Inscription

Next, inscribe a data file by appending the B protocol to the script. This is a typical way to inscribe a data file of any kind on Bitcoin. B is a Bitcom protocol that uses the prefix "19HxigV4QyBv3tHpQVcUEQyq1pzZVdoAut".

```bash
o1 - 1SAT_P2PKH
o2 - OP_FALSE OP_RETURN "19HxigV4QyBv3tHpQVcUEQyq1pzZVdoAut" <data> <encoding> <content-type>
```

Note the protocol separator "|". This is a convention used to separate composable protocols in OP_RETURN.

### All Together

Here's a plain text "Hello world" inscription bringing it all together. This output script creates an ordinal and sends it to a recipient. The output value must be exactly 1 satoshi.

Output #1:

```bash
OP_DUP OP_HASH160 <pubkeyhash> OP_EQUALVERIFY OP_CHECKSIG
```

Output #2:

```bash
OP_FALSE OP_RETURN 0x31736174 0x7c 0x31394878696756345179427633744870515663554551797131707A5A56646F417574 0x48656C6C6F20776F726C6421 0x7574662D38 0x746578742F706C61696E
```

## Handling Large Files - BCAT

Inscribes a data file using the BCAT protocol. It lists, in order, the transaction ids required to assemble the data file. BCAT is a Bitcom protocol that uses the prefix "15DHFxWZJT58f9nhyGnsRBqrgwK4W6h4Up". You can find more information about BCAT protocol [here](https://bcat.bico.media/).

```bash
o1 - 1SAT_P2PKH
o2 - OP_FALSE OP_RETURN <BCAT fields...> tx2 tx3 tx4...
```

## Adding Metadata

You can use metadata to add context to ordinals. For example, you can tag an ordinal with a geohash to place it on a physical location. Metadata is written using "Magic Attribute Protocol" which has a prefix of "1PuQa7K62MiKCtssSLKy1kh56WWU7MtUR5".

```bash
o1 - 1SAT_P2PKH
o2 - OP_FALSE OP_RETURN <B fields...> | MAP SET app <platform_name> type "ord" context "geohash" geohash "dhmgdqvr7"
```

Similarly, you can attach an ordinal to a particular url:

```bash
o1 - 1SAT_P2PKH
o2 - OP_FALSE OP_RETURN <B fields...> | MAP SET app <platform_name> type "ord" context "url" url "https://google.com"
```

## Ordinal Lock

You can lock an ordinal, creating an on-chain public listing for sale. If the listing value is transferred to the specified address, the ordinal is released to the destination given by the buyer. The listing can be canceled.

Below is the sCrypt contract. Provide a payment output, and a sellec pubkey and compile the contract.

```js
export class OrderLock extends SmartContract {
    @prop()
    seller: PubKey

    @prop()
    payOutput: ByteString

    constructor(seller: PubKey, payOutput: ByteString) {
        super(...arguments)

        this.seller = seller;
        this.payOutput = payOutput;
    }

    @method()
    public purchase(selfOutput: ByteString, trailingOutputs: ByteString) {
        assert(hash256(selfOutput + this.payOutput + trailingOutputs) == this.ctx.hashOutputs)
    }

    @method()
    public cancel(sig: Sig) {
        assert(this.checkSig(sig, this.seller), 'signature check failed')
    }
}
```

Once the script is compiled, use it to mint and list an ordinal in a single 1 sat output:

```bash
o1 - ORDINAL_LOCK (1 sat)
o2 - OP_FALSE OP_RETURN "ord" | <B> <data> <encoding> <content-type>
```

## Signing

It is possible to sign ordinals to link an identity to the creation of the token. Separating identity signatures from funding signatures is more flexible. This is useful for verified minting. Signatures use "Author Identity Protocol" which has a Bitcom prefix of "15PciHG22SNLQJXMoSUaWVi7WSqc7hCfva".

### User Signatures

To sign when inscribing:

```bash
o1 - 1SAT_P2PKH
o2 - OP_FALSE OP_RETURN "ord" | <B> <data> <encoding> <content-type> | <AIP> <signing_address> "BITCOIN_ECDSA" <sig>
```

### Platform Signatures

You can not only use AIP signatures to prove a particular identity created an inscription, but you can sign once more to prove that a particular platform was used to create the transaction as well.

```bash
o1 - 1SAT_P2PKH
o2 - OP_RETURN "ord" | <B> <data> ... | MAP SET app <mint_platform> type ord | AIP <user_address> <user_sig> | AIP <platform_address> <platform_sig>
```

## Examples

### Web Pages

To inscribe a webpage, simply:

```bash
o1 - 1SAT_P2PKH
o2 - OP_FALSE OP_RETURN "ord" | <B> <html_data> "utf-8" "text/html"
```

### o:// - Ordinal URL References

You can embed references to other ordinals inside the on-chain HTML or markdown.

An ordinalID is the precise satoshi identifier.

```html
<img src="o://ordinalID" />
```

Or in markdown

```markdown
[My Ordinal!]("o://ordinalID")
```

You can also address the specific inscription by txid and output index.

```html
<img src="b://txid{64}.o2" />
```

Complex example: To inscribe a video that exists across multiple transactions and place it
on a specific geohash with identity signature
tx1 - o1 - OP_PUSH ord OP_DROP OP_FALSE OP_RETURN BCAT tx2 tx3 tx4... | MAP SET context geohash geohash <geohash> | AIP <sig...>
tx1 - o2 - 1 sat

tx2 - o1 - BCAT_PART <tx2_data>

### Inscription Collection

You can also mint lots of ordinals at once, and link them to a specific collection.

o1 - 1SAT_P2PKH
o2 - OP_FALSE OP_RETURN <B> <data> ... | MAP SET app <mint_platform> type ord collection <unique-collection-name>
o3 - 1SAT_P2PKH
o4 - OP_FALSE OP_RETURN <B> <data> ... | MAP SET app <mint_platform> type ord collection <unique-collection-name>

## TRANSFERS

You can either just send the sat or append to the inscription
i1 - 1 sat
i2 - funding utxo
o1 - 1 sat
o2 - "ord" | <B> <data> ... | AIP <sig...> append to inscriptions... 0 sat
o3 - change

TOKENIZED 3D MODEL ON BITCOIN
o1 - 1 sat
o2 - ord | B <gltf_data> <binary> <gltf>

"All 1 sat outputs"

ATTACHING TO REAL WORLD LOCATION

Create with initial location
o1 - 1 sat
o2 - "ord" | MAP app <appname> type "ordinal" context geohash geohash <geohash> | AIP <sig...>

Spend the ordinal with MAP geohash - updates are checked by indexer
i1 - 1 sat o2 from inscription tx
o1 - 1 sat
o2 - "ord" | MAP geohash <geohash> | AIP <sig...>

OP_PUSH <ord> OP_DROP

we can append metadata to the script w OP_RETURN if needed

## Common Questions

### Can we merge items?

    Yes via API not via script no need for so complicate

### Why use OP_RETURN?

    Its a perfectly good data carrier when you don't need the data to be evaluated by script. It also allows nodes to prune the data more easily.

### Why set context twice?

    The `context` field is used to specify a field name to use as the context for this transaction.

    Once you set the contextfield, you must still set the cooresponding field value.
    This helps indexers find the most relevent data, allowing us to build faster indexers.

### Why not just use Run?

    Run is no longer in development, and adds some extra complexity we didn't need. Issues building the Run-Sdk in webpack bundled projects and the requirement for JavaScript to interprate token transactions also contributed to the need for a new token protocol.

### Why not just use Stas?

    Stas scripts are larger than ordinals. Both systems ultimately require some level of indexer-based validation, and both can use OP_RETURN based token data. Stas protocol is an owned and licensed protocol. We wanted something that would be publicly available without an individual licensing requirement.

## Needs

- [ ] API Endpoint - Get ordinals by address (similar to endpoint for p2pkh utxos)
- [ ] Paymail - Need a capability for 1sat Ordinals
- [ ] BUX Wallet Support
