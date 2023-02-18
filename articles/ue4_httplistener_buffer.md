---
title: 'Unreal Engine 4のHttpListenerのバッファサイズの変更'
emoji: "🪣"
type: "tech"
topics: [UE4, unrealengine, Linux, Ubuntu]
published: true
---

# Unreal Engine 4のHttpListenerのバッファの変更
## TL;DR
プロジェクトの`Config/DefaultEngine.ini`に`DeafaultBufferSize=<適切な値>`を追記することで解決する。

## 環境
- Ubuntu 22.04.1 LTS
- Unreal Engine 4.27.2

## 事象
UE4のエディタでFile -> Package Project -> Linux -> Linux -> ゲームのバイナリを置きたいディレクトリを選択、の順序でLinux用のパッケージを作成し、パッケージを実行したところ次のエラーを吐いてセグフォした。

```txt
[2023.02.18-09.15.37:089][  0]LogWorld: Bringing World /Game/VehicleAdvCPP/Maps/VehicleAdvExampleMap.VehicleAdvExampleMap up for play (max tick rate 0) at 2023.02.18-18.15.37
[2023.02.18-09.15.37:089][  0]LogWorld: Bringing up level for play took: 0.000230
[2023.02.18-09.15.37:090][  0]LogGameMode: FindPlayerStart: PATHS NOT DEFINED or NO PLAYERSTART with positive rating
[2023.02.18-09.15.37:091][  0]LogTemp: PlayerStart is not found
[2023.02.18-09.15.37:092][  0]LogTemp: サーバー開始
[2023.02.18-09.15.37:092][  0]LogHttpServerModule: Starting all listeners...
[2023.02.18-09.15.37:092][  0]LogHttpListener: Warning: HttpListener unable to set desired buffer size (524288): Limited to 425984
[2023.02.18-09.15.37:092][  0]LogHttpListener: Created new HttpListener on 0.0.0.0:8082
[2023.02.18-09.15.37:092][  0]LogHttpServerModule: All listeners started
[2023.02.18-09.15.37:112][  0]LogCore: === Critical error: ===
Unhandled Exception: SIGSEGV: invalid attempt to read memory at address 0x0000000000000230

[2023.02.18-09.15.37:112][  0]LogCore: Fatal error!

0x0000000004d33788 Chicken7!UnknownFunction(0x4b33788)
0x000000000859f63e Chicken7!UnknownFunction(0x839f63d)
0x0000000009620089 Chicken7!UnknownFunction(0x9420088)
0x0000000008b3fe1c Chicken7!UnknownFunction(0x893fe1b)
0x00000000095ebc39 Chicken7!UnknownFunction(0x93ebc38)
0x00000000094cd76e Chicken7!UnknownFunction(0x92cd76d)
0x00000000094c9069 Chicken7!UnknownFunction(0x92c9068)
0x0000000008b1ac0f Chicken7!UnknownFunction(0x891ac0e)
0x0000000003ccd0df Chicken7!UnknownFunction(0x3acd0de)
0x0000000003ce158c Chicken7!UnknownFunction(0x3ae158b)
0x0000000009efe036 Chicken7!UnknownFunction(0x9cfe035)
0x00007f7403601d90 libc.so.6!UnknownFunction(0x29d8f)
0x00007f7403601e40 libc.so.6!__libc_start_main(+0x7f)
0x0000000003ccc489 Chicken7!UnknownFunction(0x3acc488)

[2023.02.18-09.15.37:119][  0]LogExit: Executing StaticShutdownAfterError
```

このセグフォでプログラムが落ちる直前にHttpListenerがメモリ不足の警告を出しているので、この周りを調べていく。

## 調査
まずwarningを吐き出す場所を特定するため、HttpLister.cpp (`Engine/Source/Runtime/Online/HTTPServer/Private/HttpListener.cpp`)を探す。
source code: https://github.com/EpicGames/UnrealEngine/blob/4.27/Engine/Source/Runtime/Online/HTTPServer/Private/HttpListener.cpp

warningを出した場所の該当コードを見つけた。
```cpp
int32 ActualBufferSize;
NewSocket->SetSendBufferSize(Config.BufferSize, ActualBufferSize);
if (ActualBufferSize < Config.BufferSize)
{
    UE_LOG(LogHttpListener, Warning, 
        TEXT("HttpListener unable to set desired buffer size (%d): Limited to %d"),
        Config.BufferSize, ActualBufferSize);
}
``` 

この`Config`は同じファイルの62行目ので初期化される。
source code: https://github.com/EpicGames/UnrealEngine/blob/4.27/Engine/Source/Runtime/Online/HTTPServer/Private/HttpListener.cpp#L62
```cpp
Config = FHttpServerConfig::GetListenerConfig(ListenPort);
```

FHttpServerConfigは`Engine/Source/Runtime/Online/HTTPServer/Private/HttpServerConfig.h`で定義されている。
source code: https://github.com/EpicGames/UnrealEngine/blob/4.27/Engine/Source/Runtime/Online/HTTPServer/Private/HttpServerConfig.h

このためBufferSizeは512KBとして定義されている。

`FHttpServerConfig`はメソッドを持つ構造体であり、そのメソッドについて見る。
`Engine/Source/Runtime/Online/HTTPServer/Private/HttpServerConfig.cpp`
source code:  https://github.com/EpicGames/UnrealEngine/blob/4.27/Engine/Source/Runtime/Online/HTTPServer/Private/HttpServerConfig.cpp
```cpp
GConfig->GetInt(*IniSectionName, TEXT("DefaultBufferSize"), Config.BufferSize, GEngineIni);
```
20行目にある通り、GConfigを通してバッファサイズを変更できる模様。

## 解決策
プロジェクトの`Config/DefaultEngine.ini`に`DeafaultBufferSize=<適切な値>`を追記する。
warningではバッファの上限は425984（おそらく単位はバイト）と言われているので、それ未満の値、今回は425000にする。
今回は下の例のように記述した。

```ini
[HTTPServer.Listeners]
DefaultBindAddress="0.0.0.0"
DefaultBufferSize=425000
```
これで警告`LogHttpListener: Warning: HttpListener unable to set desired buffer size (524288): Limited to 425984`を消すことができた。

## 最後に
今回Unreal Engineのライセンスの都合で、Engine本体のコードは一部GitHubで参照してもらう形にしています（[Unreal® Engineエンドユーザーライセンス契約](https://www.unrealengine.com/ja/eula-reference/unreal-ja）)。
Zennなどで共有する場合は次の項目に気をつける必要があります。
 > ii.  公開ディスカッションのためのEngineコードの共有
お客様は、最大30行までのEngineコードのスニペットを、かかるスニペットの内容について議論する目的でのみ、オンライン上の公開フォーラムに投稿すること、またはライセンス技術用のサポートパッチおよびプラグインに関してかかるスニペットを配布することが認められます。ただし、Engineコードのライセンスなしに第三者にEngineコードを使用もしくは修正できるようにすること、またはEngineコードのより広範囲な部分を集約、再結合もしくは再構築することを目的としない場合に限られます。

そのためUnreal Engine 4のソースコードを閲覧する際には、Epic Games Developerに登録する必要があります。

またこのライセンスが原因か分かりませんが、Unreal EngineのC++周りの情報はインターネット上にあまりないです。
そのためUnreal EngineをC++で書いている人々はどのようにエラーを解決しているのか気になります。

この記事以外にも自分のUnreal Engine 4をC++で書く際の知見は[こちら](https://zenn.dev/yutashx/scraps/aef914c793b092)のスクラップにまとめているので、もしよければご覧ください。
