---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: /bg.jpeg
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki

fonts:
    sans: "HiraMaruProN-W4"
---

# NFTのインデクシングとGraphQLのすゝめ

Presentation by @_serinuntius

---

# 今日話すこと
- 自己紹介
- NFTのインデクシングとは？
- なぜインデクシングが必要なのか？
- インデクシングの実装
- GraphQLのすゝめ
- まとめ

---

# 自己紹介
<div grid="~ cols-2 gap-4">
  <div>
  
  - 芹川葵: [@_serinuntius](https://twitter.com/_serinuntius)
  - no plan inc. CTO
  - JPYC 技術顧問
  - 趣味:
    - キャンプ
    - スノボ
  
  </div>
  <div class="">
    <img class="m-4 h-40" src="/serinuntius.jpeg" alt="serinuntiusのプロフィール画像">
    <img class="m-4 h-30" src="/noplan_logo_horizontal.png" alt="no plan inc. のプロフィール画像">
    <img class="m-4 p-2 h-20 bg-white" src="/JPYC.png" alt="JPYCのプロフィール画像">
  </div>
</div>

---

# 興味ある技術とか、最近やってる技術とか（雑）

<div class="grid grid-cols-5 gap-8 bg-gray-400 p-8">
    <img src="/fav/rust.png">
    <img src="/fav/cloud_run.jpeg">
    <img src="/fav/firebase.png">
    <img src="/fav/graphql.png">
    <img src="/fav/react.png">
    <img src="/fav/hardhat.jpeg">
    <img src="/fav/intel_sgx.webp">
    <img src="/fav/nestjs.png">
    <img src="/fav/nextjs.png">
    <img src="/fav/secret_network.png">
    <img src="/fav/solidjs.png">
    <img src="/fav/tauri.svg">
    <img src="/fav/vercel.png">
    <img src="/fav/circom.png">
    <img src="/fav/cosmwasm.webp">
</div>

---

# NFTのインデクシングとは？
<div grid="~ cols-2 gap-4">
  <div>
    <ul>
        <li>NFTが持つ情報や付随する情報を、検索し易いようにDBに格納すること</li>
    </ul>
  </div>
  <div>
    <table class="table_border">
        <thead>
            <tr>
                <th>tokenId</th>
                <th>address</th>
                <th>metadata</th>
            </tr>
        </thead>
        <tbody>
            <tr>
                <td>1</td>
                <td>0xabc</td>
                <td>{"image": "ipfs://..."}</td>
            </tr>
            <tr>    
                <td>2</td>
                <td>0x123</td>
                <td>{"image": "ipfs://..."}</td>
            </tr>
            <tr>
                <td>3</td>
                <td>0x456</td>
                <td>{"image": "ipfs://..."}</td>
            </tr>
            <tr>
                <td>4</td>
                <td>0x789</td>
                <td>{"image": "ipfs://..."}</td>
            </tr>
        </tbody>
    </table>
  </div>
</div>

--- 

# なぜインデクシングが必要なのか？

- 一般的にNFTの情報にアクセスするには、コントラクトに生えてるview関数を叩く


```solidity {0|1-5|7-12|all}
function ownerOf(uint256 tokenId) public view virtual override returns (address) {
    address owner = _owners[tokenId];
    require(owner != address(0), "ERC721: invalid token ID");
    return owner;
}

function tokenURI(uint256 tokenId) public view virtual override returns (string memory) {
    _requireMinted(tokenId);

    string memory baseURI = _baseURI();
    return bytes(baseURI).length > 0 ? string(abi.encodePacked(baseURI, tokenId.toString())) : "";
}
```

<v-clicks>

- 情報が欲しくなった時にview関数を叩きまくるのは色々と現実的ではない
  - ノードの負担増🆙, Infura, Alchemy等のノードプロバイダーのコスト増🆙
  - 時間もかかる
- ERC1155対応の闇😈

</v-clicks>


---

# ERC1155対応の闇😈

<v-clicks>

- ERC721みたいな `ownerOf(tokenId)` が生えてないので、 `tokenId` の所有者情報をview関数だけでは調べることができない😇
- `name()` が生えてないのも地味にだるい
  - ノードの情報だけでは何のコントラクトなのかわからない
  - ERC721は規格としてマストにはなってないものの、OpenZeppelinの `ERC721Metadata` が普及しているため、大体取れる気がする

</v-clicks>

---
layout: center
--- 

# どうやってインデクシングするのか？


---
layout: center
--- 

# 闇の魔術😈に対する防衛術をお教えします👍

---

# インデクシングの実装

<v-clicks>

- EVMには **「event」** という概念がある
- ノードに対してイベントをリッスンしておくことで、コントラクトからイベントが発火した時に何かを実行できる
- ex) ERC721

</v-clicks>


<div v-after>

```solidity
event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
```

</div>


<v-clicks>

- `indexed` というキーワードをつけると、その変数は検索できるようになる
- `eth_getLogs` というAPIを呼び出すことによってイベントを取得できる
  
- どのブロック高でそのイベントが発生したかがわかる

</v-clicks>


---

# インデクシングの実装 - ERC721

<v-clicks>

- 1ブロック目から、 

```solidity
event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
```

</v-clicks>

<v-clicks>

- これを順に追って行けば良い
- 最後に同期したブロック高をdbに入れて、次回以降は差分更新で良い

</v-clicks>

---


# インデクシングの実装 - ERC1155

<v-click>

- ERC1155の場合はイベントが２種類ある

</v-click>


<v-click>

```solidity {0|1|3-9|all}
event TransferSingle(address indexed operator, address indexed from, address indexed to, uint256 id, uint256 value);

event TransferBatch(
    address indexed operator,
    address indexed from,
    address indexed to,
    uint256[] ids,
    uint256[] values
);
```

</v-click>



<v-clicks>

- これらを順に追って行けば良い

</v-clicks>

---

# GraphQLのすゝめ (時間なかった)
- 先程行ったインデクシングでDBに保管してGraphQLで返すように実装してみました
- 型付のクライアントが自動生成できるのでオススメ🔥
- Nestjs / PlanetScale / Prisma / Cloud Run 辺りで実装しています

---
layout: center
---

# この辺りを全部気をつけて<br>実装するの面倒では？🤔🤔🤔


---
layout: center
---

# その通り！

---
layout: center
---

# 面倒ごとを全部うちが引き受けて<br>インデクシングするSaaS始めます！

---

# NFT GraphQL API (仮)

<v-clicks>

- ⛓ 任意のEVM互換のチェーンに対応できる（サポートして欲しいチェーンがあればお気軽に連絡ください）
- 💰 実装コストやら、ノードのメンテコストやら考えると圧倒的コスパ
- ¥ ボラの高い草コインじゃなくて、日本円で払えます（円も実質草コイ・・・🤫
- 🚀 もう少しでリリースするのでちぇけら

</v-clicks>


---
layout: center
---

# 興味ある方はこちら

<img class="m-4 h-64" src="/form_qr.svg" alt="serinuntiusのプロフィール画像">

---

# 参考文献
- [EIP721](https://eips.ethereum.org/EIPS/eip-721)
- [EIP1155](https://eips.ethereum.org/EIPS/eip-1155)
- [OpenZeppelin - IERC721.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC721/IERC721.sol)
- [OpenZeppelin - IERC1155.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC1155/IERC1155.sol)

---
layout: center
---

# ご静聴ありがとうございました🙌 
