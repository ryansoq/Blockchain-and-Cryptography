# 4.nlockTime address

指定時間後該地址才可進行交易

```
redeemScript = unixTime + OP_CHECKLOCKTIMEVERIFY + OP_DROP + publickey + OP_CHECKSIG;
```

```js
var crypto = require('crypto');
var ecdh = crypto.createECDH('secp256k1');
var bs58 = require('bs58');

// 查表
const OP_2 = "52";
const OP_3 = "53";
const OP_CHECKMULTISIG = "ae";

var hash2 = crypto.randomBytes(32)
console.log('--------')
console.log('私鑰')
console.log(hash2); //私鑰，64位十六進制數 //使用hash2.toString('hex')即可看到16進位字串
console.log('--------')


// ECDH和ECDSA產生公私鑰的方式都相同
var publickey = ecdh.setPrivateKey(hash2,'hex').getPublicKey('hex')
console.log('公鑰')
console.log(publickey); //公鑰(通過橢圓曲線算法可以從私鑰計算得到公鑰)
console.log('--------')

var unixTime = Date.now() + 60*60*24*1000*7; // 時間設定在七天後
const OP_CHECKLOCKTIMEVERIFY = "b1";
const OP_DROP = "75";
const OP_CHECKSIG = "ac";

redeemScript = unixTime + OP_CHECKLOCKTIMEVERIFY + OP_DROP + publickey + OP_CHECKSIG;

//把公鑰以sha256加密後再用ripemd160加密，取得publickeyHash
var hash = crypto.createHash('sha256').update(redeemScript).digest();
hash = crypto.createHash('ripemd160').update(hash).digest();
console.log('redeemScriptHash')
console.log(hash);
console.log('--------')

//在publickeyHash前面加上一个05前缀
var version = new Buffer('05', 'hex');
var checksum = Buffer.concat([version, hash]);
//兩次256雙重加密
checksum = crypto.createHash('sha256').update(checksum).digest();
checksum = crypto.createHash('sha256').update(checksum).digest();

//取前4位得到效驗碼
checksum = checksum.slice(0, 4);
console.log('checksum')
console.log(checksum);
console.log('--------')

//把publickeyHash前面一樣加上00而後面加上剛才算出的checksum
var address = Buffer.concat([version, hash, checksum]);
console.log('編碼前地址')
console.log(address);
console.log('--------')

// 最後使用base58進行編碼
address = bs58.encode(address);
console.log('編碼後的nlockTime比特幣地址')
console.log(address);
console.log('--------')
```

# 5.HD \(hierarchical deterministic\) wallet address 

參考下圖:

![](/assets/derivation.png)

> 圖片來源:[https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki)

根據[BIP39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki) 從12個隨機單字產生了128-bit master seed，之後繼續往下階層式的產生出許多地址

## 第一步驟

1.先用 seed 產生一個master key，而 seed 的長度需要是 128、256 或 512 bits

2.然後將 seed 去執行 HMAC-SHA512 加密，之後會產生一個長度為 512 bits 的結果

> 前 256 bits 為 master private key，後 256 bits 為 master chain code ，master chain code 代表 entropy\(熵\)，之後再往下產生 child keys 時會用到。

![](/assets/1_ChWUKm31L2WEEpeEB7kzPQ1.png)圖片來源:[https://github.com/bitcoinbook/bitcoinbook](https://github.com/bitcoinbook/bitcoinbook)

## 第二步驟

往下一層，產生該層各個child private key 的時候，要用到 child key derivation \(CKD\) 這個方法

這個方法一樣是做HMAC-SHA512 加密演算法，但他會需要先輸入三個參數。

```
1.parent public key：拿上一層的 private key 來產生的 public key即是。
2.parent chain code：上一層key的 chain code，為一個entropy(熵)。
3.index：代表著產生下一層的第幾個 key，例如 0 則代表為下一層的第一個key。
```

> 之後會產生跟上層一樣的512bits的key，同樣的，前後256bits分別為 child private key 與 child chain code。

![](/assets/1_ni33v4GKL12m2M4m_5GIQQ.png)圖片來源:[https://github.com/bitcoinbook/bitcoinbook](https://github.com/bitcoinbook/bitcoinbook)

> 由於HMAC-SHA512是Hash function，過程是不可逆的，所以我們不會知道parent是什麼，以及也不會知道自己鄰近的其他child是什麼

下面我們會用到十六進位轉二進位，所以需下載『 big-integer-converter 』模組

```
npm install big-integer-converter
```

```js
var crypto = require('crypto');
var bigInt = require('big-integer-converter');
var ecdh = crypto.createECDH('secp256k1');

var seed = crypto.randomBytes(64)

HMACseed = crypto.createHmac('sha512', 'secret for HMAC').update(seed).digest('hex');
// 16進位轉為2進位
HMACseed_Binary = bigInt.hexToBin(HMACseed);

//切割前後各256bits
Master_Prvkey = HMACseed_Binary.substring(0,256);
Master_ChainCode = HMACseed_Binary.substring(256,512);
var Master_pubkey = ecdh.setPrivateKey(bigInt.binToHex(Master_Prvkey), 'hex').getPublicKey('hex')
console.log('---Master公鑰---')
console.log(Master_pubkey); //公鑰(通過橢圓曲線算法可以從私鑰計算得到公鑰)
console.log('-----------------')

function CKD(parentPub, parentChainCode, childIdx){
  let input = parentPub + parentChainCode + childIdx;
  return crypto.createHmac('sha512', 'secret for HMAC').update(input).digest('hex')
}

// 第一層derivation的第一個child
let derivation0_child0 = CKD(Master_pubkey, Master_ChainCode, '00000000'); // 第三個參數，因index number規定要32bits，所以填入十六進位的八個數字
// 第二層derivation的第一個child
let derivation0_child1 = CKD(Master_pubkey, Master_ChainCode, '00000001');
console.log('---第0層derivation的第一個child---')
console.log(derivation0_child0) //長度為十六進位的128個字，所以為512bits
console.log('---第0層derivation的第二個child---')
console.log(derivation0_child1)
console.log('-----------------')

// 下面我們再示範用第一層的第一個child產生第二層

console.log('---第0層的第一個child的public key （m/1）---')
derivation0_child0_Prvkey = derivation0_child0.substring(0,64); // 因為十六進位，所以取一半是取到第64個字 相當於256bits
var derivation0_child0Pub = ecdh.setPrivateKey(derivation0_child0_Prvkey, 'hex').getPublicKey('hex')
console.log(derivation0_child0Pub)
console.log('------')

console.log('---第0層的第一個child的chain code （m/1）---')
derivation0_child0_ChainCode = derivation0_child0.substring(64,128); //取後面的一半
console.log(derivation0_child0_ChainCode)
console.log('------')

let derivation1_child1 = CKD(Master_pubkey, Master_ChainCode, '00000001');
let derivation1_child10 = CKD(Master_pubkey, Master_ChainCode, '00000010');

console.log('-----------------')
console.log('---第1層derivation的第一個child長出來的第一個child (m/1/1)---')
console.log(derivation1_child1) //長度為十六進位的128個字，所以為512bits
console.log('---第1層derivation的第一個child長出來的第十個child (m/1/10)---')
console.log(derivation1_child10)
console.log('-----------------')
```

# 查看地址的相關資料與交易紀錄

[https://blockchain.info/rawaddr/1GbVUSW5WJmRCpaCJ4hanUny77oDaWW4to](https://blockchain.info/rawaddr/1GbVUSW5WJmRCpaCJ4hanUny77oDaWW4to)

```js
var http = require('http');
http.get({
    host: 'blockchain.info',
    path: '/rawaddr/1GbVUSW5WJmRCpaCJ4hanUny77oDaWW4to'
}, function (response) {
    var body = '';
    response.on('data', function (d) {
        body += d;
    });
    response.on('end', function () {
        console.log(body);
    });
});
```

會出現如下訊息

```json
{
    "hash160":"ab0fcc2fb04ee80d29a00a80140b16323bed3d6e",
    "address":"1GbVUSW5WJmRCpaCJ4hanUny77oDaWW4to",
    "n_tx":3589,
    "total_received":4629912224989,
    "total_sent":4577511878052,
    "final_balance":52400346937,
    "txs":[

{
   "ver":1,
   "inputs":[
      {
         "sequence":4294967295,
         "witness":"01200000000000000000000000000000000000000000000000000000000000000000",
         "script":"03b58f070004fcfd145a0450e2b80708985d085a00001e06092f426974667572792f"
      }
   ],
   "weight":700,
   "block_height":495541,
   "relayed_by":"0.0.0.0",
   "out":[
      {
         "spent":false,
         "tx_index":303487002,
         "type":0,
         "addr":"1GbVUSW5WJmRCpaCJ4hanUny77oDaWW4to",
         "value":1284013028,
         "n":0,
         "script":"76a914ab0fcc2fb04ee80d29a00a80140b16323bed3d6e88ac"
      },

      .....
```

# 地址不建議重複使用

地址如果重複使用容易造成個人隱私上的暴露，所以許多交易所的地址會經常更換，其他相關資訊可參考如下連結:

[https://en.bitcoin.it/wiki/Address\_reuse](https://en.bitcoin.it/wiki/Address_reuse)
