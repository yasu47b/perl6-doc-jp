# モジュール

## Perl6 モジュールの作り方、使い方、そしてその配布方法

### モジュールの作り方と使い方

モジュールは、単一のソースファイルもしくは複数のソースファイルのまとまりです。
それらのファイルから、Perl 6がどのようにモジュールを組み立てるのか見ることができます。
モジュールにはパッケージ(クラス、ロール、グラマー)とサブルーチン、そして時々変数が含まれています。
Perl 6でモジュールという言葉はmoduleキーワードで宣言されたバッケージにも使われます。（下部の例を参考）
このドキュメント内で「モジュール」という用語は、大抵の場合名前空間にあるソースファイルのまとまりを指しています。

### 基本構造

Perl 6のモジュールの配布方法は以前のPerlと同じです - 配布用のプロジェクトディレクトリにはREADMEとLICENSEファイル、メインとなるモジュール格納用のlibディレクトリ、テストのためのtディレクトリ、またそれらに加えて実行可能ブログラムとスクリプト用のbinディレクトリが含まれることもあります。

モジュールファイルにはたいてい.pm拡張子が、実行可能スクリプトには.pl拡張子がつきます。しかしながら、あなたがPerl 6のモジュールであるということを強調したいなら、モジュールファイルには.pm6、そして実行可能スクリプトには.p6の拡張子をつけることも出来ます。ただ、テストスクリプトに関しては今までどおり.t拡張子を使用してください。

### ローディングと基本的なインポート

#### need

need はコンパイル時にcompilation unitをロードします。

```
  need MyModule;
```

モジュール内部で定義されている名前空間のパッケージが利用可能になります。

```
  # MyModule.pm
  unit module MyModule;

  class MyModule::Class { ... }
```

クラス MyModule::Class はMyModule.pmがロードされた時に定義されます。

#### use

use はコンパイル時にcompilation unitをロードしてインポートします。

```
  use MyModule;
```

これは次の例と等価です。

```
  need MyModule;
  import MyModule;
```

#### require

require は実行時にcompilation unitiをロードし、限定的なシンボルをインポートします。

```
sub load-mymodule {
   say "loading MyModule";
   require MyModule;
}

load-mymodule();
```

間接的にcompilation unitを設定する場合は（以下のようにサブルーチン引数として）、compilation unit名は
実行時の変数に格納されます。

```
  sub load-a-module($name){
    require ::($name);
  }

  load-a-module('MyModule');
```
シンボルをインポートする場合は、コンパイル時に定義されてなければなりません。

```
  sub do-something {
    require MyModule <&something>;
    something() # &something will be defined here
  }

  do-something();
  # &something will not be defined here
```

もし、&something がエクスポートされなかった場合は、モジュールのインポート処理は失敗します。

## エクスポートと選択的インポート

### is export

パッケージ、サブルーチン、変数、定数と列挙型は is export トレイトによってエクスポートされます。


```
    unit module MyModule;
    our $var is export = 3;
    sub foo is export { ... };
    constant $FOO is export = "foobar";
    enum FooBar is export <one two three>;

    # Packages like classes can be exported too
    class MyClass is export {};

    # If a subpackage is in the namespace of the current package
    # it doesn't need to be explicitly exported
    class MyModule::MyClass {};
```

他のトレイトの使い方と同様に、"is export"をルーチンに適用する場合は引数リストの後に配置します。


```
  sub foo (Str $string) is export { ... }
```

エクスポート処理で、"is export"は名前付きのパラメータを受け取ることで、シンボルをグループ化できます。
そして、インポートをする側では、使いたいものだけを選択してインポートが出来ます。
その場合は、ALL, DEFAULT, MANDATORYの３つのタグが使用できます。

```
    # lib/MyModule.pm
    unit module MyModule;
    sub bag        is export             { ... }
    sub pants      is export(:MANDATORY) { ... }
    sub sunglasses is export(:day)       { ... }
    sub torch      is export(:night)     { ... }
    sub underpants is export(:ALL)       { ... }
```

```
    # main.pl
    use lib 'lib';
    use MyModule;          #bag, pants
    use MyModule :DEFAULT; #the same
    use MyModule :day;     #pants, sunglasses
    use MyModule :night;   #pants, torch
    use MyModule :ALL;     #bag, pants, sunglasses, torch, underpants
```

### UNIT::EXPORT::*

"is export"の内部では"EXPORT"の名前空間の"UNIT"パッケージスコープに
シンボルを追加しています。例えば、"is export(:Foo)"は"UNIT::EXPORT::FOO"パッケージ
が(UNIT::EXPORTパッケージに)追加されます。このようにPerl 6ではインポートする対象を
決定しています。


```
    unit module MyModule;

    sub foo is export { ... }
    sub bar is export(:other) { ... }
```

上記のコードは以下と等価です。

```
    unit module MyModule;

    my package EXPORT::DEFAULT {
        our sub foo { ... }
    }

    my package EXPORT::other {
        our sub bar { ... }
    }
```

"is export"とは対照的に、"EXPORT"パッケージは動的にシンボルをエクスポートしたい場合に
使用します。以下の様な例です。

```
    # lib/MyModule.pm
    unit module MyModule;

    my package EXPORT::DEFAULT {
       for <zero one two three four>.kv -> $number,$name {
          for <sqrt log> -> $func {
             OUR::{'&' ~ $func ~ '-of-' ~ $name } := sub { $number."$func"() };
          }
       }
    }

```

```
    # main.pl
    use MyModule;
    say sqrt-of-four; #-> 2
    say log-of-zero;  #-> -Inf
```
