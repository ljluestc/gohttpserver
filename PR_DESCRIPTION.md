# 修复HTTP认证模式下Android浏览器下载APK失败的问题

## 问题复现
1. 开启 HTTP 认证: `./gohttpserver --auth-type http --auth-http admin:admin`
2. 上传 APK 文件
3. 使用 Android 手机扫码或浏览器直接访问(已登录后)点击下载或安装
4. 华为手机浏览器返回"链接失效,文件下载失败"

## 预期行为
- 开启 HTTP 认证后,所有文件类型(包括 APK)都应该能正常下载
- 与其他文件类型(如 txt)的行为保持一致

## 实际行为
- 开启认证后,APK 文件无法下载
- 关闭认证后 APK 可以正常下载
- txt 文件在认证开启时可以正常浏览

## 解决方案

Android 浏览器在下载 APK 文件时,对 HTTP 响应头有特殊要求。当使用 HTTP Basic Authentication 且 Content-Type 为 `application/vnd.android.package-archive`(APK 的 MIME 类型)时,Android 浏览器会返回错误。

**修改方案:**
1. 在 `httpstaticserver.go` 的 `hIndex` 函数中,为 APK 文件下载添加特殊的 Content-Disposition 头
2. 强制 APK 文件作为附件下载,而不是尝试在浏览器中打开
3. 确保响应头包含明确的文件名和下载行为

## 技术细节
- Android 浏览器对 APK 文件下载有特殊的安全检查机制
- 某些 Android 设备在 HTTP 认证下对 APK 的 MIME 类型处理有问题
- 通过设置适当的响应头,可以绕过这个限制

## 相关 issue
- #136: 开启http认证方式后,安卓浏览器下载apk失败

## 测试计划
1. 测试开启 HTTP 认证后下载 APK 文件
2. 测试华为、小米等不同品牌的 Android 设备
3. 确认其他文件类型下载不受影响
4. 确认关闭认证后功能正常