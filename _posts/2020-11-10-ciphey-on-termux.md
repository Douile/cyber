# A quick guide for how to install [ciphey](https://github.com/Ciphey/Ciphey) on [termux](https://github.com/termux/termux-app)

## Build python3.8
_On a machine with docker installed_
1. `git clone https://github.com/termux/termux-packages`

2. `cd termux-packages`

3. Checkout python version 3.8 `git checkout a1765e4908334748595d33ce6a96896090508044 -- ./packages/python`

4. Build package `./scripts/run-docker.sh ./build-package.sh python`
    - For faster build skip building dependencies `./scripts/run-docker.sh ./build-package.sh -I python`
    - This takes a really long time

## Build/install ciphey
_Inside your termux install_
1. Install build deps `pkg install boost cmake build-essential swig`

2. Install python3.8
    - `debs/python_3.8.6_aarch64.deb`
    
3. Install poetry
    - `python3 -m pip install poetry`

4. Clone cipheycore `git clone https://github.com/Ciphey/Cipheycore`

5. Checkout latest version
    - `git checkout v0.3.2`

6. Build code (taken from [ciphey docs](https://github.com/Ciphey/CipheyCore#building))
    - `rm -rf build && mkdir build && cd build`
    - `cmake .. -DCMAKE_BUILD_TYPE=Release -DCIPHEY_CORE_TEST=OFF`
    - `cmake --build . -t ciphey_core`

7. Build python whl
    - `cmake --build . -t ciphey_core_py --config Release`
    - `poetry build`

8. Install ciphey core
    - `python3 -m pip install -U ./dist/cipheycore-0.3.2-cp38-cp38-linux_aarch64.whl`

9. Install ciphey
    - `python3 -m pip install ciphey`
