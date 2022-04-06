---
title: "Installing ciphey in termux"
tags: guides
---

# A quick guide for how to install [ciphey](https://github.com/Ciphey/Ciphey) on [termux](https://github.com/termux/termux-app)

## Build python3.8
_On a machine with docker installed_
1. Clone the packages repo 
```bash
git clone https://github.com/termux/termux-packages
```

2. Change directory to repository 
```bash
cd termux-packages
```

3. Checkout python version 3.8 
```bash
git checkout a1765e4908334748595d33ce6a96896090508044 -- ./packages/python
```

4. Build the python package 
```bash
# This takes a long time
./scripts/run-docker.sh ./build-package.sh python
# For a faster build you can skip building dependencies 
# ./scripts/run-docker.sh ./build-package.sh -I python
```

5. Move the built package to your termux install
    - If build was successful deb file should be at `./debs/python_3.8.6_aarch64.deb`

## Build/install ciphey
_Inside your termux install_
1. Install build dependencies 
```bash
pkg install boost cmake build-essential swig
```

2. Install python3.8
```bash
pkg install ./python_3.8.6_aarch64.deb
```
    
3. Install poetry
```bash
python3 -m pip install poetry
```

4. Clone cipheycore 
```bash
git clone https://github.com/Ciphey/Cipheycore
cd CipheyCore
```

5. Checkout latest version
```bash
git checkout v0.3.2 # You can use "git tag" to list all version tags
```

6. Build code (taken from [ciphey docs](https://github.com/Ciphey/CipheyCore#building))
```bash
rm -rf build && mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release -DCIPHEY_CORE_TEST=OFF
cmake --build . -t ciphey_core
```

7. Build cipheycore whl
```bash
cmake --build . -t ciphey_core_py --config Release
poetry build
```

8. Install ciphey core
```bash
python3 -m pip install -U ./dist/cipheycore-0.3.2-cp38-cp38-linux_aarch64.whl
```

9. Install ciphey
```bash
python3 -m pip install ciphey
```
