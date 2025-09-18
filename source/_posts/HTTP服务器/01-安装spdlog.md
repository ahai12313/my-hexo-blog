---
title: 01_安装spdlog
date: 2025-09-18 22:49:25
tags:
categories: HTTP服务器
---
# 1 安装
```bash
git clone https://github.com/gabime/spdlog.git
cd spdlog
mkdir build && cd build
cmake ..
make
sudo make install
```

```cmake
find_package(spdlog REQUIRED)
target_link_libraries(your_target PRIVATE spdlog::spdlog)
```