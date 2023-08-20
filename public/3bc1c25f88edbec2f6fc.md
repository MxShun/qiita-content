---
title: ドットファイルに入門する
tags:
  - 入門
  - dotfiles
private: false
updated_at: '2022-12-14T07:01:17+09:00'
id: 3bc1c25f88edbec2f6fc
organization_url_name: null
slide: false
---
【完走賞】MxShun ひとりマラソン 🏃 Advent Calendar 2022 毎週水曜日は Advent Calendar を機になにかに入門しよう！2記事目です。

転職をし開発環境が Mac になったことを機に、先週は Zsh プラグインに入門しました。同じ理由で今回はドットファイルに入門してみます。
というのも、Mac ではシンボリックリンクが使えるのでドットファイル管理リポジトリとシームレスに調整ができるうまみが大きいからです。

https://github.com/MxShun/dotfiles

### 管理するファイル
実際にユーザディレクトリ下にあるバージョン管理したいドットファイルを洗い出します。
VSCode や IntelliJ IDEA など、バージョン管理しなくてもサービスごとに設定が同期されているものは省きました。結果、ドットファイルとして管理したいものはそーんなになかったですね。

- `.gitconfig`
- `.vimrc`
- `.zshrc`（ Mac 用）
- `.bashrc`（ Windows 用）

### インストーラー
ドットファイル初期導入・更新用のシェルを用意しました。

`uname` コマンドを活用し、Windows と Mac それぞれ個別のセットアップも定義しています。

```bash
#!/bin/bash
set -eux

BASEDIR=$(dirname $0)
cd $BASEDIR

git pull

files=`find . -mindepth 2 -type f -name ".*" -not -path ".git/*"`
if [ "$(uname)" == "Darwin" ]; then
    for file in $files; do
        [[ $file == "./bash/.bashrc" ]] && continue

        ln -snfv ${PWD}/$file ~/
    done
elif [ "$(expr substr $(uname -s) 1 5)" == "MINGW" ]; then
    for file in $files; do
        [[ $file == "./zsh/.zshrc" ]] && continue

        cp ${PWD}/$file ~/
    done

    # Colorscheme for Mintty
    curl https://raw.githubusercontent.com/mavnn/mintty-colors-solarized/master/sol.dark > ~/dev/theme/mintty-colors-solarized/sol.dark
else
    echo Do not supported
fi

# Colorscheme for Vim
curl https://raw.githubusercontent.com/altercation/solarized/master/vim-colors-solarized/colors/solarized.vim > ~/.vim/colors/solarized.vim
```

一旦雛形はできたので、引き続き更新をしていきます。
