# OSPP-2025-Reproduce-Build
Instructions to reproduce the OSPP-2025 Triton-cpu + Triton-shared + LLVM build


## 1. Python

Only tested with python3.9 up to python3.11

```shell
pip install setuptools>=40.8.0
pip install wheel
pip install "cmake>=3.18,<4.0"
pip install ninja>=1.11.1
pip install pybind11>=2.13.1
pip install lit
pip install nanobind
pip install numpy
```
Note1: using -i https://pypi.tuna.tsinghua.edu.cn/simple will probably speedup your downloading speed on China based machines.

## 2. LLVM

We need to use LLVM 19.1.7 from open euler

```shell
git clone https://gitee.com/openeuler/llvm-project.git -b dev_19.1.7
cd llvm-project
```

There are many patches that need to be applied they are in the patches dir and should be applied in order here is a table with some information about the patches.

| Patch | Gitee PR                                                           | Upstream PR                                                                                               |   |   |
|-------|--------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------|---|---|
| 0001  | [link](https://gitee.com/openeuler/llvm-project/pulls/236)         | -                                                                                                         |   |   |
| 0002  | [link](https://gitee.com/openeuler/llvm-project/pulls/288)(merged) | -                                                                                                         |   |   |
| 0003  | [link](https://gitee.com/openeuler/llvm-project/pulls/243)         | -                                                                                                         |   |   |
| 0004  | -                                                                  | [link](https://github.com/llvm/llvm-project/pull/107005/commits/9e1383f2c3a69d5df5beaef8fff522af0bd389a0) |   |   |
| 0005  | [link](https://gitee.com/openeuler/llvm-project/pulls/234/files)   | -                                                                                                         |   |   |
| 0006  | [link](https://gitee.com/openeuler/llvm-project/pulls/266)         | Several, see Gitee PR                                                                                     |   |   |
| 0007  | -                                                                  | -                                                                                                         |   |   |
| 0008  | [link](https://gitee.com/openeuler/llvm-project/pulls/269)         | Several, see Gitee PR                                                                                     |   |   |
| 0009  | [link](https://gitee.com/openeuler/llvm-project/pulls/280)         |                                                                                                           |   |   |
| 0010  | [link](https://gitee.com/openeuler/llvm-project/pulls/255)         | [link](https://github.com/llvm/llvm-project/commit/df0d249b6511289f1e8c1389f4fd33d7b4c083fa)              |   |   |
| 0011  | [link](https://gitee.com/openeuler/llvm-project/pulls/290)         | [link](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Fllvm%2Fllvm-project%2Fpull%2F140793)      |   |   |
| 0012  | [link](https://gitee.com/openeuler/llvm-project/pulls/291)         |                                                                                                           |   |   |
| 0013  | [link](https://gitee.com/openeuler/llvm-project/pulls/293)         |                                                                                                           |   |   |

to apply all patches in order, simply:

```shell
ls ../OSPP-2025-Reproduce-Build/patches/*.patch | sort | xargs -n 1 git apply --whitespace=nowarn
```

and then compile:

```shell
mkdir build; cd build
cmake -G Ninja -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_ASSERTIONS=ON ../llvm -DLLVM_ENABLE_PROJECTS="mlir;llvm;clang;clang-tools-extra;lld" -DLLVM_TARGETS_TO_BUILD="host;NVPTX;AMDGPU;AArch64" -DMLIR_ENABLE_BINDINGS_PYTHON=ON -DPython3_EXECUTABLE=$(which python3) -DMLIR_INCLUDE_INTEGRATION_TESTS=ON -DLLVM_ENABLE_RTTI=ON -DBUILD_SHARED_LIBS=OFF
ninja
```

Make sure python bindings have been generated. 
```
file tools/mlir/python_packages/mlir_core
```
If it does not exist, force build with : `ninja check-mlir-python`

It is important to note that we are using the MLIR python bindings so the python version used to compile llvm (in this case the one pointed by `$(which python3)`) must be the same we use to run triton.

Troubleshooting: If you encounter some error such as "mlir/Dialect/Math/Transforms/Passes.h.inc" does not exist, build withouth patches first, apply patches and rebuild using `ninja`. There must be a dependency missing in CMakeLists.


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
export PYTHONPATH=$LLVM_BUILD_DIR/tools/mlir/python_packages/mlir_core
export PATH=$LLVM_BUILD_DIR/bin:$PATH
export TRITON_BUILD_WITH_CLANG_LLD=true
export TRITON_PLUGIN_DIRS=$(pwd)/triton-shared
pip install --no-build-isolation -v -e python
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
export TRITON_SHARED_OPT_PATH=$YOUR_WORKDIR/triton-cpu/python/build/cmake.linux-{arch}-cpython-{version}/third_party/triton_shared/tools/triton-shared-opt/triton-shared-opt
```

The first just activates the triton-shared backend, and the next two point to important files that triton-shared needs to use. Depending on the architecture of the server and the python version the last path will be different.

## 5. Tips for debugging and testing

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

Most debugging happens around `ttshared.mlir` as it is the crucial step between triton and standard MLIR. Another **VERY IMPORTANT** thing to take into account is to **ALWAYS DELETE THE TRITON CACHE** before running as triton may pick the code stored in cache and not apply any of your new changes. The best way to do it's to just append the remove command before your python execution like this:

```shell
rm -rf ~/.triton/cache/ && python program.py
```

This is also essential when running test, to run test triton uses pytest

```shell
pip install pytest-xdist
pip install torch
```

Then:

```shell
rm -rf ~/.triton/cache && python3 -m pytest -n32 --device=cpu python/test/unit/language/test_core.py -m cpu
```

to run all the core tests, again making sure to **remove the triton cache**.

## 6. Failing test.

There are 96 failing test in `test_core.py` there is a file named `failed.txt` in this repo that contains the name of all the failing test. To run a specific test for example let's say `test_reduce[1-argmax-float32-shape149-0-True]` we can just do:

```shell
rm -rf ~/.triton/cache && python3 -m pytest -n32 --device=cpu python/test/unit/language/test_core.py:: test_reduce[1-argmax-float32-shape149-0-True]-m cpu
```

the text between the square brackets are the parameters passed to the test. There may be instances where not all of the tests fail and some just fails with certain parameters, this is also the case of `test_reduce` as if we run all of the possible `test_reduce` with all possible parameters (same thing as before but without square braces):

```shell
rm -rf ~/.triton/cache && python3 -m pytest -n32 --device=cpu python/test/unit/language/test_core.py::test_reduce -m cpu
```

We see that just the tests with the `float32` parameters fail pointing us to a better path to fix the solution. 

## 7. Work left on the SVE pipeline.

There is still work left to do on the SVE pipeline as it produces some errors, the code related to it it's on the `sve-pipe` branch so on triton-cpu:

```shell
git checkout sve-pipe
```

To test the pipeline I recommend to run either the cpu-matrix multiplication example:

```shell
rm -rf ~/.triton/cache/ && python python/tutorials/03-matrix-multiplication-cpu.py
```

or the `test_dot`

```shell
rm -rf ~/.triton/cache && python3 -m pytest -n0 -x --device=cpu python/test/unit/language/test_core.py::test_dot -m cpu
```

Some errors come from `pipeline_schedule` you can try and comment it out. The easist way is to just comment this part (line 845 `compiler.py`)

```python
include5 = transform.IncludeOp(
  [],
  FlatSymbolRefAttr.get("main_type1_pipeline"),
  transform.FailurePropagationMode.Propagate,
  [sequence.bodyTarget],
)
```



