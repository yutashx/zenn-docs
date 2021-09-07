---
title: 'Open3DのCUDA対応buildの備忘録'
emoji: "🙂"
type: "tech"
topics: [Open3D, CUDA]
published: true
---

# Open3DのCUDA対応buildの備忘録
## 背景
自分の調べた限りOpen3DのCUDA対応のバイナリは提供されていないため自分でbuildする必要がある。
大体この[ページ](http://www.open3d.org/docs/release/compilation.html)に書いてある通りにbuildする。

## 作業環境
- OS: Ubuntu 20.04.3
- CPU: Intel(R) Core(TM) i7-2600 CPU @ 3.40GHz
- GPU: NVIDIA GeForce GTX 1060 6GB
- Memory: 8GB
- Swap: 16GB
- CUDA Version: 11.4  
- NVIDIA-SMI: 470.57.02
- Driver Version: 470.57.02
- Open3D: 0.13.0+d29a56854

## 必要なもの
- C++14 compiler
- CMake 3.18 or later
ここでレポジトリを追加せずに`apt install cmake`をするとCMake 3.16がインストールされる。
これだとCMakeのバージョン違いでbuildに失敗するので手動でインストールする。
CMakeの[公式サイト](https://cmake.org/download/)からバイナリをダウンロードするのが早いと思われる。

## 手順
### Cloning Open3D
```
git clone --recursive https://github.com/intel-isl/Open3D
```

### Install Dependencies
```
cd Open3D
./util/install_deps_ubuntu.sh # install dependencies
```

### Config
```
mkdir build
cd build
cmake \
    -DBUILD_EXAMPLES=ON \
    -DBUILD_BENCHMARKS=ON \
    -DCMAKE_INSTALL_PREFIX=/usr/local/ \
    -DBUILD_CUDA_MODULE=ON \
    -DBUILD_COMMON_CUDA_ARCHS=ON \
    -DBUILD_CACHED_CUDA_MANAGER=ON \
    -DUSE_SYSTEM_LIBREALSENSE=ON \
    -DBUILD_PYTHON_MODULE=ON \
    -DPYTHON_EXECUTABLE=/path/to/python \
    ..
```
Open3Dのbuild optionは[CMakeLists.txt](https://github.com/isl-org/Open3D/blob/master/CMakeLists.txt)の28行目から86行目あたりに載っているので適宜変更する。

### build
```
make -j2
```
リファレンスではmakeで使用するプロセスの数を`nproc`で指定しているが、自分の環境ではメモリが少なかったためプロセス数を2に制限した。自分の環境では最終的にbuild完了するまでに3時間ほどかかった。
(というのも最初nprocでプロセス数を指定したところメモリ不足でOSがフリーズしてしまった。ちなみにそのときはメモリ7.92GB、Swap領域8GB使用していた...)

### Install
#### Open3DのC++ Libraryのインストール
```
make install
```

#### 現在のPython環境にpip packageをインストールする
```
make install-pip-package
```

#### Python packageを`build/lib`下に作成する
```
make python-package
```

#### pip wheelを`build/lib`下に作成する
```
make pip-package
```

#### conda packageを`build/lib`下に作成する
```
make conda-package
```
インストール先を必要に応じて選択する。

## 感想
buildめっちゃ時間かかる。潤沢な計算資源が欲しい。
