# astrbot_plugin_steamshot - Ubuntu环境运行说明

## 修复内容

本插件已修复Windows特有代码的问题，主要修改：

1. 将Windows特有的`winreg`模块导入改为条件导入，仅在Windows系统上尝试导入
2. 增强了浏览器版本检测逻辑，支持多种Linux环境下Chrome的安装路径
3. 使用`os.path`确保路径的跨平台兼容性
4. 为Ubuntu环境添加了必要的ChromeDriver选项
5. 确保仅在Windows系统上使用Windows特有的参数
6. 优化了无头环境下的ChromeDriver配置，提高兼容性
7. 改进了错误处理逻辑，提供更详细的错误信息和调试输出

## Ubuntu环境依赖安装

在Ubuntu系统上运行此插件前，请安装以下依赖：

```bash
# 更新包列表
sudo apt update

# 安装必要的系统依赖
sudo apt install -y wget gnupg2 apt-transport-https ca-certificates unzip

# 安装额外的图形库依赖
sudo apt install -y libxss1 libappindicator1 libindicator7 libgconf-2-4 libnss3 libgbm-dev

# 安装Google Chrome
sudo wget -q -O - https://dl.google.com/linux/linux_signing_key.pub | sudo apt-key add -
sudo sh -c 'echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google-chrome.list'
sudo apt update
sudo apt install -y google-chrome-stable

# 安装Python依赖（如果AstrBot环境中尚未安装）
pip install selenium requests bs4 webdriver-manager jinja2

# 安装Xvfb（无图形界面环境下运行Chrome必须）
sudo apt install -y xvfb

# 安装字体支持（解决中文显示问题）
sudo apt install -y fonts-wqy-zenhei fonts-wqy-microhei fonts-arphic-uming
```

## Xvfb 详细配置指南

在无图形界面的Ubuntu服务器上，**必须**正确配置和启动Xvfb虚拟显示器。以下是详细步骤：

### 基本启动方式

```bash
# 设置显示环境变量
export DISPLAY=:99

# 启动Xvfb，设置合适的分辨率和颜色深度
Xvfb :99 -screen 0 1920x1080x24 -ac -nolisten tcp -nolisten unix &

# 确认Xvfb正在运行
ps aux | grep Xvfb

# 然后启动AstrBot
```

### 设置为系统服务（推荐）

为了确保Xvfb在系统重启后自动启动，建议将其配置为系统服务：

```bash
# 创建Xvfb服务文件
sudo nano /etc/systemd/system/xvfb.service
```

将以下内容添加到文件中：

```
[Unit]
Description=X Virtual Framebuffer Service
After=network.target

[Service]
ExecStart=/usr/bin/Xvfb :99 -screen 0 1920x1080x24 -ac -nolisten tcp -nolisten unix
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

然后启用并启动服务：

```bash
# 重新加载systemd配置
sudo systemctl daemon-reload

# 启用服务（开机自启）
sudo systemctl enable xvfb.service

# 启动服务
sudo systemctl start xvfb.service

# 检查服务状态
sudo systemctl status xvfb.service
```

## Chrome和ChromeDriver版本匹配

**非常重要**：Chrome和ChromeDriver的版本必须严格匹配。不匹配会导致渲染器连接错误。

### 检查Chrome版本

```bash
# 查看已安装的Chrome版本
google-chrome --version

# 或者
chromium-browser --version
```

### 手动下载匹配的ChromeDriver

虽然插件会尝试自动下载兼容的ChromeDriver，但在某些情况下可能需要手动下载：

1. 访问ChromeDriver下载页面：https://chromedriver.chromium.org/downloads
2. 下载与你的Chrome版本完全匹配的ChromeDriver
3. 解压并放置到合适位置：

```bash
# 解压下载的ChromeDriver
unzip chromedriver_linux64.zip

# 移动到系统PATH中
chmod +x chromedriver
sudo mv chromedriver /usr/local/bin/

# 验证ChromeDriver版本
chromedriver --version
```

## 运行注意事项

1. **权限问题**：确保插件目录和截图目录具有正确的写入权限

   ```bash
   mkdir -p ~/AstrBot/data/plugins/astrbot_plugin_steamshot/screenshots
   chmod -R 755 ~/AstrBot/data/plugins/astrbot_plugin_steamshot
   ```

2. **环境变量配置**：确保AstrBot运行时能访问DISPLAY环境变量

   ```bash
   # 在启动AstrBot前设置环境变量
export DISPLAY=:99

# 或者将其添加到AstrBot的启动脚本中
```

3. **内存优化**：在资源有限的环境中，调整Chrome的内存使用至关重要

   ```bash
   # 检查系统内存使用情况
   free -h
   top -o %MEM
   ```

4. **系统限制调整**：在某些系统上，可能需要调整系统限制

   ```bash
   # 查看当前文件描述符限制
   ulimit -n
   
   # 临时增加限制
   ulimit -n 65535
   
   # 永久增加限制，编辑/etc/security/limits.conf
   sudo nano /etc/security/limits.conf
   # 添加以下行
   * soft nofile 65535
   * hard nofile 65535
   ```

## 故障排除：解决"session not created from disconnected: unable to connect to renderer"错误

这个错误是在无头环境中运行ChromeDriver时最常见的问题。以下是详细的解决方案：

### 1. 确认Xvfb正确运行

```bash
# 检查Xvfb是否正在运行
sudo systemctl status xvfb.service

# 或者
ps aux | grep Xvfb

# 测试Xvfb是否正常工作
export DISPLAY=:99
xset -q  # 应该输出Xvfb的配置信息
```

### 2. 检查Chrome和ChromeDriver版本匹配

```bash
google-chrome --version
chromedriver --version
```

### 3. 尝试使用更完整的Chrome启动选项

修改代码中的`create_driver`函数（如果需要自定义）：

```python
options.add_argument("--headless")
options.add_argument("--disable-gpu")
options.add_argument("--no-sandbox")
options.add_argument("--disable-dev-shm-usage")
options.add_argument("--disable-setuid-sandbox")
options.add_argument("--disable-features=site-per-process")
options.add_argument("--window-size=1920,1080")
options.add_argument("--user-agent=Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36")
```

### 4. 检查系统资源

```bash
# 检查内存使用
free -m

# 检查磁盘空间
df -h

# 检查临时目录空间
df -h /tmp
```

### 5. 尝试清除Chrome缓存和临时文件

```bash
# 清除Chrome临时文件
rm -rf /tmp/.com.google.Chrome*
rm -rf ~/.cache/google-chrome/
```

### 6. 检查防火墙设置

确保没有防火墙规则阻止ChromeDriver的通信：

```bash
sudo ufw status
```

## 高级调优技巧

### 1. 限制Chrome的内存使用

在`create_driver`函数中添加以下选项：

```python
options.add_argument("--js-flags=--max-old-space-size=256")  # 限制JavaScript堆内存
options.add_argument("--disable-extensions")
options.add_argument("--disable-images")  # 禁用图片加载以节省资源
```

### 2. 优化截图性能

```python
# 减少截图尺寸以提高性能
options.add_argument("--window-size=1280,800")

# 禁用不必要的功能
options.add_argument("--disable-javascript")  # 如果只需要静态内容
```

### 3. 使用系统服务管理AstrBot和依赖

创建一个启动脚本，确保所有服务按正确顺序启动：

```bash
#!/bin/bash

# 启动Xvfb（如果未作为系统服务运行）
if ! pgrep -x "Xvfb" > /dev/null; then
    export DISPLAY=:99
    Xvfb :99 -screen 0 1920x1080x24 -ac -nolisten tcp -nolisten unix &
    sleep 2
fi

# 设置环境变量
export DISPLAY=:99

# 启动AstrBot
cd /path/to/astrbot
python main.py
```

## 已知限制

1. 在资源极其有限的环境中（如1GB内存的VPS），Chrome可能会频繁崩溃
2. 在某些网络环境中，Steam网站可能需要更长的加载时间，可能需要调整超时设置
3. Docker容器中运行时需要额外配置共享内存和权限

## 最佳实践

1. 使用至少2GB RAM的服务器以确保稳定运行
2. 定期更新Chrome和ChromeDriver到最新版本
3. 监控系统资源使用情况，避免资源耗尽
4. 对于长期运行的服务，配置适当的日志轮转以避免磁盘空间问题