# 修复 HTTP Basic 认证模式下 Android 浏览器下载 APK 失败的问题

## 相关 Issue

- #136: 开启http认证方式后,安卓浏览器下载apk失败

## 问题描述

### 重现步骤

1. 开启 HTTP Basic 认证启动服务器:
   ```bash
   ./gohttpserver --auth-type http --auth-http admin:admin
   ```
2. 上传 APK 文件
3. 使用 Android 手机扫码或浏览器直接访问(已登录后)点击下载或安装
4. 华为手机浏览器返回「链接失效,文件下载失败」错误

### 预期行为

- 开启 HTTP Basic 认证后,所有文件类型(包括 APK)都应该能正常下载
- 与其他文件类型(如 txt)的行为保持一致

### 实际行为

- 开启认证后,APK 文件无法下载
- 关闭认证后 APK 可以正常下载
- txt 文件在认证开启时可以正常浏览

## 根本原因分析

Android 浏览器在下载 APK 文件时,当处于 HTTP Basic Authentication 环境下,对 APK MIME 类型(`application/vnd.android.package-archive`)的响应有特殊的安全检查机制。

某些 Android 设备(特别是华为手机)在这种情况下会拒绝下载 APK 文件,认为链接不安全。这是因为:

1. Android 浏览器对 APK 文件有特殊的安全策略
2. 在 HTTP 认证环境下,浏览器可能会对某些文件类型执行额外的验证
3. 如果响应头中没有明确的下载指示,浏览器可能会拒绝下载

## 解决方案

### 代码修改

在 `httpstaticserver.go` 的 `hIndex` 函数中,在调用 `http.ServeFile` 之前,为所有 APK 文件强制设置 `Content-Disposition: attachment` 响应头。

**关键改动:**

```go
// 修复 #136: Android浏览器在HTTP认证下无法下载APK
// 强制APK文件作为附件下载
if filepath.Ext(path) == ".apk" {
    w.Header().Set("Content-Disposition", "attachment; filename="+strconv.Quote(filepath.Base(path)))
}
```

### 改动说明

1. 检测文件扩展名是否为 `.apk`
2. 如果是 APK 文件,强制设置 `Content-Disposition: attachment` 响应头
3. 使用 `strconv.Quote` 确保文件名正确编码,避免特殊字符问题
4. 这会告诉浏览器将文件下载到本地,而不是尝试在浏览器中打开或安装
5. 与现有的 `?download=true` 参数逻辑保持一致

### 影响范围

- **仅影响**: APK 文件的 GET 请求响应行为
- **不受影响**:
  - 目录列表
  - JSON API
  - 上传/删除功能
  - IPA plist 路由
  - 其他文件类型(图片、文本等)
- **兼容性**: 与现有的 `download=true` 参数使用相同的响应头模式
- **向后兼容**: 使用 `strconv.Quote` 进行安全的文件名编码

## 测试验证

### 测试场景

详细的测试步骤请参考项目根目录的 `FIX_DESCRIPTION.md` 和 `TEST_STEPS.md` 文件。

主要测试包括:

1. **开启 HTTP 认证后测试 APK 下载**
   - 直接访问 APK 文件
   - 使用 `?download=true` 参数
   - 通过二维码扫描下载

2. **关闭认证对比测试**
   - 确认无认证环境下功能正常

3. **验证其他文件类型不受影响**
   - 图片文件应正常显示
   - 文本文件应正常显示
   - IPA 文件功能不变

4. **不同 Android 设备兼容性测试**
   - 华为手机(华为浏览器、Chrome)
   - 小米手机(小米浏览器、Chrome)
   - 三星手机(Samsung Internet、Chrome)

### 快速本地验证

```bash
# 编译
go build -o /tmp/gohttpserver .

# 启动服务器(开启HTTP认证)
/tmp/gohttpserver --port 8080 --auth-type http --auth-http admin:admin --upload &

# 检查响应头
curl -sI -u admin:admin "http://127.0.0.1:8080/your.apk" | grep -i content-disposition
# 期望输出: Content-Disposition: attachment; filename="your.apk"
```

### 成功标准

- ✅ HTTP 认证开启时,Android 浏览器可以正常下载 APK
- ✅ HTTP 认证开启时,其他文件类型行为正常
- ✅ HTTP 认证关闭时,所有功能正常
- ✅ 不同品牌 Android 设备都能正常下载
- ✅ 直接访问文件和通过二维码访问都正常
- ✅ 与 `download=true` 参数兼容

## 相关文件

- `httpstaticserver.go` - 主要修复文件(hIndex 函数)
- `FIX_DESCRIPTION.md` - 完整的中文修复说明和测试步骤
- `TEST_STEPS.md` - 详细的测试步骤文档

## 待办事项 (针对维护者)

- [ ] 在 Android 设备上使用 HTTP 认证测试前后行为(参考 TEST_STEPS.md)
- [ ] 确认非 APK 文件(图片、文本)仍正常显示或下载
- [ ] 合并后关闭或链接 https://github.com/codeskyblue/gohttpserver/issues/136

---

Co-Authored-By: Oz <oz-agent@warp.dev>