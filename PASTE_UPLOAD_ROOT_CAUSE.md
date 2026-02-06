# 复制粘贴上传失败的真正原因 - 已解决

## 问题诊断

浏览器开发者工具显示：
- **POST /upload 返回 400 (Bad Request)**
- 错误在前端的 `m.send(v)` 处
- 涉及 `FileReader` 处理

## 根本原因

**前端粘贴上传时，使用 FileReader 将图片转换为 Base64 字符串，而不是直接发送 Blob/File 对象。**

### 对比

| 上传方式 | 发送格式       | 后端处理              |
| -------- | -------------- | --------------------- |
| 文件选择 | File/Blob 对象 | ✅ 正常处理            |
| 粘贴上传 | Base64 字符串  | ❌ 无法处理 → 400 错误 |

## 解决方案

已在 `functions/upload.js` 中添加 Base64 字符串处理逻辑：

```javascript
// 处理 Base64 字符串的情况（粘贴上传可能发送 Base64）
if (typeof uploadFile === 'string') {
    console.log('Received Base64 string, converting to Blob');
    try {
        // 移除 data:image/...;base64, 前缀
        const base64Data = uploadFile.includes(',') ? uploadFile.split(',')[1] : uploadFile;
        // 使用 fetch 将 Base64 转换为 Blob
        const response = await fetch(`data:image/png;base64,${base64Data}`);
        uploadFile = await response.blob();
    } catch (e) {
        console.error('Failed to convert Base64 to Blob:', e);
        throw new Error('Invalid image data format');
    }
}
```

## 修改内容

1. **检测 Base64 字符串** - 判断 `uploadFile` 是否为字符串
2. **转换为 Blob** - 使用 `fetch` API 将 Base64 转换为 Blob 对象
3. **统一处理** - 之后的逻辑与 File/Blob 对象相同

## 为什么 GitHub 版本正常？

GitHub 版本使用 **Telegraph API**，它对请求格式更宽松，可能直接接受 Base64 字符串。而 **Telegram Bot API** 要求严格的 multipart/form-data 格式，必须是真实的 Blob/File 对象。

## 测试

现在粘贴上传应该能正常工作了。如果仍有问题，检查：
1. 浏览器控制台是否有新的错误信息
2. 后端日志中是否显示 "Received Base64 string, converting to Blob"
3. Telegram Bot Token 和 Chat ID 是否正确配置
