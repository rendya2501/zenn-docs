---
title: "【Visual Studio】で同じ単語を同時選択するショートカット【マルチカーソル】"
emoji: "🖱️"
type: "tech"
topics: ["visualstudio", "shortcuts", "tips", "editor"]
published: true
---

調べてもすんなり出て来なかったので、メモ代わりにまとめておく。

## Visual Studio のマルチカーソル系ショートカット

| やりたいこと | 操作 | 補足 |
| --- | --- | --- |
| 同じ単語を1つずつ追加 | `Shift + Alt + .` | 選択した場所から順番に進む |
| 選択を1つずつ解除 | `Shift + Alt + ,` | 選択したものがない場合、ビープ音が鳴る |
| 同じ単語をすべて一括選択 | `Shift + Alt + ;` or `Shift + Alt + :` | 環境によって `;` か `:` の場合がある |
| 任意の場所にカーソル追加 | `Ctrl + Alt + クリック` | バラバラな行を同時編集したいときに有効 |

## VS Code / Cursor との比較

| やりたいこと | VS Code | Cursor | Visual Studio |
| --- | --- | --- | --- |
| 同じ単語を1つずつ追加 | `Ctrl + D` | `Ctrl + D` | `Shift + Alt + .` |
| 選択を1つずつ解除 | `Ctrl + U` | `Ctrl + U` | `Shift + Alt + ,` |
| 同じ単語をすべて一括選択 | `Ctrl + Shift + L` | `Ctrl + F2` | `Shift + Alt + ;` or `:` |
| 任意の場所にカーソル追加 | `Alt + クリック` | `Alt + クリック` | `Ctrl + Alt + クリック` |
