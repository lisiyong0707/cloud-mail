

# cloud-mail 控制台批量创建邮箱账号指南

本指南介绍如何在不修改 cloud-mail 任何后端和前端源码的情况下，利用浏览器自带的开发者工具控制台（Console）安全、稳定地批量生成邮箱账号。

## 📌 1. 为什么使用此方式？

  - 安全性高：脚本仅对接口进行标准数据写入，不涉及任何系统底层的修改，绝不会导致您的 Cloudflare KV 数据库或 D1
    数据库绑定失效，也不会损坏任何已有数据。
  - 开发量极低：无需编写后端批量接口，也无需在 Vue 页面中添加繁琐的 UI 输入框与表单，一键即用。
  - 完全本地运行：脚本通过读取浏览器缓存中的 Token 自动进行身份认证，不会在任何地方泄露您的隐私令牌。

## 🛠️ 2. 完整操作步骤

### 第一步：打开浏览器控制台

1.  打开您的云邮系统（例如 https://mail.suks.uk）并正常登录。
2.  在网页任意空白处点击右键，选择 “检查”（Inspect），或者直接按下键盘上的 F12 键（Mac 上请按 Cmd + Option + I）。
3.  切换到顶部的 “控制台”（Console） 标签页。

### 第二步：复制并注入批量创建脚本

## 将下面这段 JavaScript 代码完整复制，粘贴到控制台空白处，并按下 回车（Enter） 执行：
```
async function bulkCreateEmails(prefix, start, end, domain) {
    // 自动从本地缓存中提取最新的安全令牌
    const token = localStorage.getItem('token');
    if (!token) {
        console.error("❌ 未在 LocalStorage 中找到 token，请确认您已处于登录状态！");
        return;
    }

    console.log(`🚀 开始批量创建邮箱，范围: ${prefix}${start}@${domain} 至 ${prefix}${end}@${domain}`);
    
    for (let i = start; i <= end; i++) {
        const emailAddress = `${prefix}${i}@${domain}`;
        const payload = {
            email: emailAddress,
            token: "" // 验证负载，默认为空
        };

        // 完美复刻 Cloudflare Pages 部署环境所需的专属身份标头 (无 Bearer 前缀)
        const headers = {
            'Content-Type': 'application/json',
            'Accept': 'application/json, text/plain, */*',
            'Authorization': token, // 直接传入原始 Token 字符串
            'accept-language': 'zh'
        };

        try {
            const response = await fetch('/api/account/add', {
                method: 'POST',
                headers: headers,
                body: JSON.stringify(payload)
            });

            const result = await response.json().catch(() => ({}));

            // 同时兼容业务状态码 200 或 0 的成功返回
            if (response.ok && (result.code === 200 || result.code === 0 || result.success !== false)) {
                console.log(`%c[成功] 已创建: ${emailAddress}`, "color: green; font-weight: bold;");
            } else {
                console.error(`❌ [失败] 创建 ${emailAddress} 失败，原因:`, result.message || result.msg || response.statusText);
            }
        } catch (error) {
            console.error(`❌ [错误] 请求 ${emailAddress} 时发生异常:`, error);
        }

        // 每次请求后停顿 500 毫秒，防止请求频率过快而导致接口被拦截
        await new Promise(resolve => setTimeout(resolve, 500));
    }

    console.log('%c🎉 批量创建任务全部完成！现在请手动刷新网页查看左侧列表。', "color: blue; font-weight: bold; font-size: 14px;");
}
```
### 第三步：运行批量创建命令

脚本注入成功后，直接在控制台中输入调用命令。您可以根据实际需求修改其中的参数，然后按 回车（Enter）：

指令格式：
```
bulkCreateEmails('前缀', 起始序号, 结束序号, '域名');
```
示例一：批量创建 cf46@suks.uk 到 cf100@suks.uk
```
bulkCreateEmails('cf', 46, 100, 'suks.uk');
```
### 第四步：刷新查看结果

待控制台输出 🎉 批量创建任务全部完成！ 提示后，手动按下键盘的 F5 键刷新网页。

系统重新载入并从后端拉取数据后，新创建的所有账号就会整齐地出现在您的左侧列表中。

## 🔍 3. 参数说明

  - prefix (字符串)：邮箱账号名称的前缀。如：'cf'、'user'。
  - start (数字)：生成的起始数字。如：46。
  - end (数字)：生成的截止数字。如：100。
  - domain (字符串)：您的目标绑定域名。如：'suks.uk'。

## 🔒 4. 额度与安全性提醒

### 1. 数据库安全性

  - 无破坏性：本脚本只对数据库执行
    INSERT（追加写入）操作，绝不执行任何修改（Update）、删除（Delete）或清空（Drop）操作，不会对您的现有数据、数据库配置或
    KV 绑定产生任何不良破坏。
  - 频率受控：脚本中默认设置了 500 毫秒 的延时，避免因并发请求过高把系统冲垮。

### 2. 免费额度限制

如果您使用的是 Cloudflare 免费版计划，每次批量创建账号都会消耗您的每日免费读写配额：

  - Cloudflare KV：每天 1,000 次免费写入次数。
  - Cloudflare D1：每天 5,000 次免费写入次数。
  - 小建议：在一天内，请尽量避免频繁地重复删除并批量创建超过 500
    个以上的账号。正常使用（每日数十或百余次）是绝对安全且无法超出限额的。如果您在当天不慎超限，只需静等次日 0
    点额度重置即可。
