# APK 下载修复测试步骤

## 编译测试程序

```bash
# 仓库根目录
cd /home/calelin/dev/gohttpserver

# 编译
go build -o /tmp/gohttpserver-test
```

## 测试场景 1:开启 HTTP 认证后下载 APK

### 步骤 1: 启动服务器

```bash
# 启动服务器,开启 HTTP 认证
/tmp/gohttpserver-test --port 8080 --auth-type http --auth-http admin:admin --upload
```

### 步骤 2: 上传测试 APK 文件

1. 在浏览器中访问 `http://localhost:8080`
2. 输入用户名和密码: `admin:admin`
3. 点击 "Upload" 按钮
4. 上传一个测试 APK 文件(假设文件名为 `test.apk`)

### 步骤 3: Android 浏览器测试

1. 获取当前电脑的 IP 地址:
   ```bash
   ip addr show | grep 'inet ' | grep -v 127.0.0.1
   ```
   假设 IP 为 `192.168.1.100`

2. 在 Android 手机浏览器中直接访问 APK 文件:
   ```
   http://192.168.1.100:8080/test.apk
   ```

3. 输入认证信息: `admin:admin`

4. **预期结果**:
   - APK 文件应该开始下载
   - 浏览器应该显示下载进度
   - 下载完成后应该提示安装 APK

5. 如果失败,测试使用下载链接:
   ```
   http://192.168.1.100:8080/test.apk?download=true
   ```

### 步骤 4: 使用二维码测试

1. 在电脑浏览器中访问 `http://localhost:8080`
2. 点击 APK 文件旁边的二维码图标
3. 用 Android 手机扫描二维码
4. **预期结果**:
   - 应该正常下载 APK 文件
   - 不应该显示"链接失效"错误

## 测试场景 2:关闭认证比较

### 步骤 1: 启动服务器(无认证)

```bash
/tmp/gohttpserver-test --port 8080 --upload
```

### 步骤 2: 重复测试

按照上述步骤 2-4 测试相同的 APK 文件

**预期结果**:
- 与开启认证时的行为一致
- APK 文件能够正常下载

## 测试场景 3:验证其他文件类型不受影响

### 测试图片文件

1. 上传一张图片(如 `test.jpg`)
2. 在 Android 浏览器中访问: `http://192.168.1.100:8080/test.jpg`
3. **预期结果**:
   - 图片应该正常显示
   - 不应该触发下载

### 测试文本文件

1. 上传一个文本文件(如 `test.txt`)
2. 在 Android 浏览器中访问: `http://192.168.1.100:8080/test.txt`
3. **预期结果**:
   - 文本内容应该正常显示
   - 不应该触发下载

## 测试不同的 Android 设备

推荐测试以下设备和浏览器:

1. **华为手机**:
   - 华为浏览器
   - Chrome
   - Firefox

2. **小米手机**:
   - 小米浏览器
   - Chrome

3. **其他 Android 设备**:
   - Samsung Internet
   - Chrome

## 验证修复效果

### 成功标准

- ✅ HTTP 认证开启时,Android 浏览器可以正常下载 APK
- ✅ HTTP 认证开启时,其他文件类型行为正常
- ✅ HTTP 认证关闭时,所有功能正常
- ✅ 不同品牌 Android 设备都能正常下载
- ✅ 直接访问文件和通过二维码访问都正常

### 失败情况处理

如果仍然出现问题,请记录:

1. 具体的错误消息
2. Android 设备型号和系统版本
3. 浏览器名称和版本
4. 网络环境(WiFi/4G)
5. HTTP 响应头(可以通过 Chrome DevTools 查看)

## 查看日志

服务器会输出访问日志:

```
2026/03/28 <time>:<line> GET /test.apk /path/to/test.apk
```

如果下载失败,检查日志中是否有错误信息。

## 测试不同认证方式

目前修复了 HTTP Basic Auth,也可以测试其他认证方式:

```bash
# OpenID 认证
/tmp/gohttpserver-test --port 8080 --auth-type openid --auth-openid https://your-openid-provider.com/openid
```

虽然 issue 主要针对 HTTP Basic Auth,但验证 OpenID 认证也能正常工作,可以增强信心。