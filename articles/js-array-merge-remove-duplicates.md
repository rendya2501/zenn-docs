---
title: "【JavaScript】配列をマージして重複を削除する方法"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [js]
published: true
---

## はじめに <BR/> sssss

よくわからないけど動く。  

```jsx
const member1 = ["2020-08-01", "2020-08-02", "2020-08-03", "2020-08-04"];
const member2 = ["2020-08-03", "2020-08-04", "2020-08-05", "2020-08-06"];
const result = [...member1,...member2].filter((element, index, self) => self.indexOf(element) === index);

console.log(result);
// [
//     '2020-08-01',
//     '2020-08-02',
//     '2020-08-03',
//     '2020-08-04',
//     '2020-08-05',
//     '2020-08-06'
// ]
```

## `filter` の引数

```jsx
array.filter(callback(element, index, array))
```

- `element`：現在処理している要素
- `index`：現在処理している要素のインデックス
- `array`（`self`）：`filter` が適用されている元の配列

## コードの動作解析

1. `member1` と `member2` を結合する：

    ```jsx
    [...member1, ...member2]
    // ["2020-08-01", "2020-08-02", "2020-08-03", "2020-08-04", "2020-08-03", "2020-08-04", "2020-08-05", "2020-08-06"]
    ```

2. `filter` を適用：

    ```jsx
    filter((element, index, self) => self.indexOf(element) === i)
    ```

    - `self.indexOf(element) === index` の意味：
      - `self.indexOf(element)` は `element` が配列 `self` の中で最初に出現するインデックスを返す。
      - `index` は `filter` の現在のインデックス。
      - もし `index` が `indexOf(element)` の値と一致するなら、その要素は最初に登場したものである。
      - それ以降に出現する重複要素は `indexOf(element) !== index` になるので `false` になり `filter` によって削除される。

## 実際の `filter` の処理の流れ

| index | element | self.indexOf(element) | self.indexOf(element) === index | 残るか？ |
| --- | --- | --- | --- | --- |
| 0 | "2020-08-01" | 0 | ✅ (0 === 0) | 残る |
| 1 | "2020-08-02" | 1 | ✅ (1 === 1) | 残る |
| 2 | "2020-08-03" | 2 | ✅ (2 === 2) | 残る |
| 3 | "2020-08-04" | 3 | ✅ (3 === 3) | 残る |
| 4 | "2020-08-03" | 2 | ❌ (2 !== 4) | 削除 |
| 5 | "2020-08-04" | 3 | ❌ (3 !== 5) | 削除 |
| 6 | "2020-08-05" | 6 | ✅ (6 === 6) | 残る |
| 7 | "2020-08-06" | 7 | ✅ (7 === 7) | 残る |

というわけで、重複のない配列が得られます！

## `Set` を使おう

`filter` を使う方法も良いですが、ES6 の `Set` を使うとより簡単に書けます：

```jsx
const member1 = ["2020-08-01", "2020-08-02", "2020-08-03", "2020-08-04"];
const member2 = ["2020-08-03", "2020-08-04", "2020-08-05", "2020-08-06"];
const result = [...new Set([...member1, ...member2])];

console.log(result);
// [
//     '2020-08-01',
//     '2020-08-02',
//     '2020-08-03',
//     '2020-08-04',
//     '2020-08-05',
//     '2020-08-06'
// ]
```

`Set` は重複を自動で削除するので、この方が簡潔で効率的です！
