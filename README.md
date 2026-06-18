# YWRT

> 基于 ImmortalWrt 的高通平台固件，专注支持京东云亚瑟 (RE-CS-07) 与京东云雅典娜 (RE-CS-02)。

## 支持机型

| 机型 | 设备名 | SoC | 说明 |
|------|--------|-----|------|
| 京东云雅典娜 | JDCloud RE-CS-02 | IPQ6010 | eMMC，带屏幕 |
| 京东云亚瑟 | JDCloud RE-CS-07 | IPQ6010 | eMMC |

## 分支说明

- `main`：高通专用，带满血 NSS 驱动。
- `owrt`：多平台通用，可编译高通平台，但没有 NSS 驱动。

## 编译方法

```bash
# 1. 安装编译依赖（Ubuntu/Debian）
sudo apt update
sudo apt install -y build-essential flex bison gawk libncurses-dev zlib1g-dev \
    libssl-dev libelf-dev qemu-user rsync unzip

# 2. 更新 feeds
./scripts/feeds update -a
./scripts/feeds install -a

# 3. 选择机型（京东云雅典娜 RE-CS-02）
make defconfig
# 或使用预置 seed
cp config-athena-seed .config
make defconfig

# 4. 编译
make -j$(nproc) V=s

# 5. 编译产物位于 bin/
```

## 致谢

高通部分源码取自以下项目：

- https://github.com/LiBwrt/openwrt-6.x.git
- https://github.com/qosmio/openwrt-ipq.git
