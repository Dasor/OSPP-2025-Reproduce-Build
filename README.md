# OSPP-2025-Reproduce-Build
Instructions to reproduce the OSPP-2025 Triton-cpu + Triton-shared + LLVM build


## 1. Python

Only tested with python3.9 up to python3.11

```shell
pip install setuptools>=40.8.0
pip install wheel
pip install cmake>=3.18,<4.0
pip install ninja>=1.11.1
pip install pybind11>=2.13.1
pip install lit
pip install nanobind
```

## 2. LLVM

We need to use LLVM 19.1.7 from open euler

```shell
git clone https://gitee.com/openeuler/llvm-project.git -b dev_19.1.7
cd llvm-project
```

There are many patches that need to be applied they are in the patches dir and should be applied in order here is a table with some information about the patches.

| Patch | Gitee PR                                                         | Upstream PR                                                                                               |   |   |
|-------|------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------|---|---|
| 0001  | [link](https://gitee.com/openeuler/llvm-project/pulls/236)       | -                                                                                                         |   |   |
| 0002  | -                                                                | -                                                                                                         |   |   |
| 0003  | [link](https://gitee.com/openeuler/llvm-project/pulls/243)       | -                                                                                                         |   |   |
| 0004  | -                                                                | [link](https://github.com/llvm/llvm-project/pull/107005/commits/9e1383f2c3a69d5df5beaef8fff522af0bd389a0) |   |   |
| 0005  | [link](https://gitee.com/openeuler/llvm-project/pulls/234/files) | -                                                                                                         |   |   |
| 0006  | [link](https://gitee.com/openeuler/llvm-project/pulls/266)       | Several, see Gitee PR                                                                                     |   |   |
| 0007  | -                                                                | -                                                                                                         |   |   |
| 0008  | [link](https://gitee.com/openeuler/llvm-project/pulls/269)       | Several, see Gitee PR                                                                                     |   |   |
| 0009  | [link](https://gitee.com/openeuler/llvm-project/pulls/280)       |                                                                                                           |   |   |

to aplly all patches in order just:

```shell
ls ../patches/*.patch | sort | xargs -n 1 git apply --whitespace=nowarn
```

and then compile:

```shell
cmake -G Ninja -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_ASSERTIONS=ON ../llvm -DLLVM_ENABLE_PROJECTS="mlir;llvm" -DLLVM_TARGETS_TO_BUILD="host;NVPTX;AMDGPU;AArch64" -DMLIR_ENABLE_BINDINGS_PYTHON=ON -DPython3_EXECUTABLE=$(which python3) -DMLIR_INCLUDE_INTEGRATION_TESTS=ON -DLLVM_ENABLE_RTTI=ON -DBUILD_SHARED_LIBS=ON
ninja
```


It is important to note that we are using the MLIR python bindings so the python version used to compile llvm (in this case the one pointed by `$(which python3)`) must be the same we use to run triton.


## 3. Triton-cpu + Triton-shared

```shell
git clone https://gitee.com/Dasor/triton-cpu
cd triton-cpu
git checkout dev
git submodule init
git submodule update
export LLVM_BUILD_DIR=$YOUR_WORKDIR/llvm-project/build
export LLVM_INCLUDE_DIRS=$LLVM_BUILD_DIR/include
export LLVM_LIBRARY_DIR=$LLVM_BUILD_DIR/lib
export LLVM_SYSPATH=$LLVM_BUILD_DIR
export PATH=$LLVM_BUILD_DIR/bin:$PATH
export TRITON_BUILD_WITH_CLANG_LLD=true
export TRITON_PLUGIN_DIRS=$(pwd)/triton-shared
pip install -e --no-build-isolation python
```

## 4. Using triton-shared (MLIR Backend)

To use triton-shared we need to set some enviroment variables, first:

```shell
export TRITON_DISABLE_LINE_INFO=1
```

We need this enviroment variable for triton to work no matter the backend we are using, then:

```shell
export TRITON_USE_SHARED_BACKEND=1
export LLVM_BINARY_DIR=$YOUR_WORKDIR/llvm-project/build/bin/
export TRITON_SHARED_OPT_PATH=$YOUR_WORKDIR/triton_cpu/python/build/cmake.linux-{arch}-cpython-{version}/third_party/triton_shared/tools/triton-shared-opt/triton-shared-opt
```

The first just activates the triton-shared backend, and the next two point to important files that triton-shared needs to use, depending on the archquitecture of the server and the python version the last path will be different.

## 4. Tips for debugging and testing

When debugging it's a really good idea to set the envrioment variable `TRITON_SHARED_DUMP_PATH` so you get the IR from all the intermediate steps, for example:

```shell
export TRITON_SHARED_DUMP_PATH=$YOUR_WORKDIR/dumps
``` 

Will generate in your chosen directory (from lower to higher abstraction level):

```shell
ll.ir # LLVM IR 
ll.mlir # LLVM IR in the MLIR dialect just before mlir-translate
ttshared.mlir # MLIR IR (this is usually the most important)
tt.mlir # Triton IR
```

Most debugging happends around `ttshared.mlir` as it is the crucial step between triton and standard MLIR. Another **VERY IMPORTANT** thing to take into account is to **ALWAYS DELETE THE TRITON CACHE** before running as triton may pick the code stored in cache and not apply any of your new changes. The best way to do it's to just append the remove command before your python execution like this:

```shell
rm -rf ~/.triton/cache/ && python program.py
```

This is also essential when running test, to run test triton uses pytest

```shell
pip install pytest-xdist
```

Then:

```shell
rm -rf ~/.triton/cache && python3 -m pytest -n32 --device=cpu python/test/unit/language/test_core.py -m cpu
```

to run all the core tests, again making sure to **remove the triton cache**.

## 5. Failing test.

There are 96 failing test in `test_core.py` there is a file named `failed.txt` in this repo that contains the name of all the failing test. To run a specific test for example let's say `test_reduce[1-argmax-float32-shape149-0-True]` we can just do:

```shell
rm -rf ~/.triton/cache && python3 -m pytest -n32 --device=cpu python/test/unit/language/test_core.py:: test_reduce[1-argmax-float32-shape149-0-True]-m cpu
```

the text between the square brackets are the parameters passed to the test. There may be instances where not all of the tests fail and some just fails with certain parameters, this is also the case of `test_reduce` as if we run all of the possible `test_reduce` with all possible parameters (same thing as before but without square braces):

```shell
rm -rf ~/.triton/cache && python3 -m pytest -n32 --device=cpu python/test/unit/language/test_core.py::test_reduce -m cpu
```

We see that just the tests with the `float32` parameters fail pointing us to a better path to fix the solution. 
