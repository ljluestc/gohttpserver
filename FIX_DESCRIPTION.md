# 修复 Android 浏览器在 HTTP 认证下无法下载 APK 的问题

## 问题描述

### 现象
- 开启 HTTP Basic 认证后(`--auth-type http --auth-http admin:admin`)
- Android 浏览器(特别是华为手机)无法下载 APK 文件
- 浏览器返回"链接失效,文件下载失败"错误
- 其他文件类型(如 txt)在认证开启时可以正常浏览
- 关闭认证后 APK 可以正常下载

### 复现步骤
1. 启动服务器: `./gohttpserver --auth-type http --auth-http admin:admin`
2. 上传 APK 文件
3. 在 Android 浏览器中访问 APK 文件链接(已登录后)
4. 点击下载或安装
5. 华为手机浏览器显示"链接失效,文件下载失败"

## 问题原因

Android 浏览器在下载 APK 文件时,当处于 HTTP Basic Authentication 环境下,对 APK MIME 类型(`application/vnd.android.package-archive`)的响应有特殊的安全检查机制。某些 Android 设备在这种情况下会拒绝下载 APK 文件,认为链接不安全。

解决方法是强制浏览器将 APK 文件作为附件下载,而不是尝试内联显示。

## 修复方案

### 代码修改

在 `httpstaticserver.go` 的 `hIndex` 函数中,为所有 APK 文件添加 `Content-Disposition: attachment` 响应头:

```go
func (s *HTTPStaticServer) hIndex(w http.ResponseWriter, r *http.Request) {
    path := mux.Vars(r)["path"]
    realPath := s.getRealPath(r)
    // ... 其他代码 ...
    
    log.Println("GET", path, realPath)
    if r.FormValue("raw") == "false" || isDir(realPath) {
        if r.Method == "HEAD" {
            return
        }
        renderHTML(w, "assets/index.html", s)
    } else {
        if filepath.Base(path) == YAMLCONF {
            auth := s.readAccessConf(realPath)
            if !auth.Delete {
                http.Error(w, "Security warning, not allowed to read", http.StatusForbidden)
                return
            }
        }
        
        // 修复 #136: Android浏览器在HTTP认证下无法下载APK
        // 强制APK文件作为附件下载
        if filepath.Ext(path) == ".apk" {
            w.Header().Set("Content-Disposition", "attachment; filename="+strconv.Quote(filepath.Base(path)))
        }
        
        if r.FormValue("download") == "true" {
            w.Header().Set("Content-Disposition", "attachment; filename="+strconv.Quote(filepath.Base(path)))
        }
        http.ServeFile(w, r, realPath)
    }
}
```

### 修改说明
1. 检测文件扩展名是否为 `.apk`
2. 如果是 APK 文件,强制设置 `Content-Disposition: attachment` 响应头
3. 这会告诉浏览器将文件下载到本地,而不是尝试在浏览器中打开或安装
4. 使用 `strconv.Quote` 确保文件名正确编码,避免特殊字符问题

### 影响范围
- 仅影响 APK 文件的响应行为
- 其他文件类型(图片、文本等)不受影响
- 与现有的 `download=true` 参数完全兼容
- 不影响已有的下载功能

## 本地测试步骤

### 1. 编译测试程序

```bash
cd /home/calelin/dev/gohttpserver

# 编译
go build -o /tmp/gohttpserver-test
```

### 2. 测试场景 A:开启 HTTP 认证后测试 APK 下载

#### 步骤 A1: 启动服务器

```bash
# 启动服务器,开启 HTTP 认证和上传功能
/tmp/gohttpserver-test --port 8080 --auth-type http --auth-http admin:admin --upload
```

#### 步骤 A2: 上传测试文件

1. 在浏览器中访问 `http://localhost:8080`
2. 输入认证信息: 用户名 `admin`, 密码 `admin`
3. 点击 "Upload" 按钮
4. 上传一个测试 APK 文件(如 `test.apk`)

#### 步骤 A3: Android 浏览器直接访问

1. 获取当前电脑 IP:
   ```bash
   ip addr show | grep 'inet ' | grep -v 127.0.0.1
   # 输出示例: 192.168.1.100
   ```

2. 在 Android 手机浏览器中直接访问:
   ```
   http://192.168.1.100:8080/test.apk
   ```

3. 输入认证信息 `admin:admin`

4. **预期结果**: APK 文件开始下载,显示下载进度

#### 步骤 A4: 测试下载链接

访问:
```
http://192.168.1.100:8080/test.apk?download=true
```

**预期结果**: 同样能够正常下载

#### 步骤 A5: 测试二维码

1. 在电脑浏览器中访问 `http://localhost:8080`
2. 点击 APK 文件旁边的二维码图标
3. 用 Android 手机扫描二维码
4. **预期结果**: 直接下载 APK,不显示"链接失效"错误

### 3. 测试场景 B:关闭认证对比测试

#### 步骤 B1: 启动服务器(无认证)

```bash
/tmp/gohttpserver-test --port 8080 --upload
```

#### 步骤 B2: 重复测试 A2-A4

按照场景 A 的步骤 2-4 测试

**预期结果**: 与开启认证时的行为一致,APK 能正常下载

### 4. 测试场景 C:验证其他文件类型不受影响

#### 测试图片文件

1. 上传图片文件(如 `test.jpg`)
2. Android 浏览器访问: `http://192.168.1.100:8080/test.jpg`

**预期结果**: 图片正常显示,不触发下载

#### 测试文本文件

1. 上传文本文件(如 `test.txt`)
2. Android 浏览器访问: `http://192.168.1.100:8080/test.txt`

**预期结果**: 文本内容正常显示,不触发下载

#### 测试 IPA 文件

1. 上传 IPA 文件(如 `test.ipa`)
2. Android 浏览器访问: `http://192.168.1.100:8080/test.ipa`

**预期结果**: 正常下载或显示(如之前的行为)

### 5. 测试场景 D:不同 Android 设备

推荐测试以下设备和浏览器组合:

- **华为手机**: 华为浏览器、Chrome、Firefox
- **小米手机**: 小米浏览器、Chrome
- **三星手机**: Samsung Internet、Chrome
- **其他**: OPPO、vivo 等品牌

### 6. 观察服务器日志

服务器会输出访问日志,格式如下:
```
2026/03/28 16:30:15 httpstaticserver.go:151: GET /test.apk /path/to/test.apk
```

如果下载失败,检查日志中是否有错误信息。

## 成功标准

所有以下条件都应该满足:

- ✅ HTTP 认证开启时,Android 浏览器可以正常下载 APK
- ✅ HTTP 认证开启时,其他文件类型(图片、文本)行为正常
- ✅ HTTP 认证关闭时,所有功能正常
- ✅ 不同品牌 Android 设备都能正常下载
- ✅ 直接访问文件和通过二维码访问都正常
- ✅ 与 `download=true` 参数兼容
- ✅ 不影响 IPA 文件下载功能

## 如果测试失败

请记录以下信息以便调试:

1. 具体错误消息
2. Android 设备型号和系统版本
3. 浏览器名称和版本
4. 网络环境(WiFi/4G)
5. HTTP 响应头(可通过 Chrome DevTools 查看)
6. 服务器日志输出

## 附加说明

此修复仅影响 APK 文件的下载行为,通过设置 `Content-Disposition: attachment` 响应头来绕过 Android 浏览器的安全检查。这是一个最小侵入的修改,不会影响其他功能。