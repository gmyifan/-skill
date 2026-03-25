# 视频平台页面结构速查表

已验证的平台结构信息，供抓取时快速参考，避免重复探索。

---

## 爱奇艺 (iqiyi.com)

**验证日期**: 2026-03-25

### 列表页入口
使用旧版列表页 `list.iqiyi.com`（服务端渲染，数据更完整）：

| 类型 | URL |
|------|-----|
| 电视剧 | `https://list.iqiyi.com/www/2/-------------11-1-1-iqiyi--.html` |
| 电影 | `https://list.iqiyi.com/www/1/-------------11-1-1-iqiyi--.html` |
| 电影(独播筛选) | `https://list.iqiyi.com/www/1/----------4---11-1-1-iqiyi--.html` |
| 综艺 | `https://list.iqiyi.com/www/6/-------------11-1-1-iqiyi--.html` |
| 动漫 | `https://list.iqiyi.com/www/4/-------------11-1-1-iqiyi--.html` |

### URL 模式
```
/www/{频道ID}/{排序}-{地区}-{类型}-{子类型}-{风格}-{年份1}-{年份2}-{费用}-{状态}-{规格}-{新类型}-{其他}-{其他}-11-{页码}-1-iqiyi--.html
```

频道ID: 1=电影, 2=电视剧, 4=动漫, 6=综艺

### 内容加载机制
- **无限滚动加载**（非URL分页）
- 每次滚动加载约 **48 条**
- 每类上限约 **768 条**（16批 x 48条）
- 通过 `window.scrollTo(0, document.body.scrollHeight)` 触发加载
- 停止条件：连续 3 次滚动后 `.qy-mod-li` 数量不增加

### DOM 结构
- 列表项选择器: `.qy-mod-li`
- 标题/链接: `.link-txt`（title 属性=片名，href 属性=链接）
- 链接格式: `//www.iqiyi.com/v_xxx.html`（需补 `https:`）

### 标签标识
- **独播**: `img[src*="only.png"]`（角标图片 `https://www.iqiyipic.com/common/fix/site-v4/video-mark-aura3/only.png`）
- **VIP**: `img[src*="vip_100000"]`

### JS 提取脚本
```javascript
() => {
    const items = document.querySelectorAll('.qy-mod-li');
    return Array.from(items).map(item => {
        const linkEl = item.querySelector('.link-txt');
        const title = linkEl ? (linkEl.getAttribute('title') || linkEl.textContent.trim()) : '';
        const href = linkEl ? (linkEl.getAttribute('href') || '') : '';
        const isExclusive = !!item.querySelector('img[src*="only.png"]');
        return { title, href, isExclusive };
    });
}
```

### 抓取结果参考（2026-03-25）
- 电视剧: 768条加载 → 317条独播
- 电影: 768条加载 → 84条独播
- 综艺: 768条加载 → 252条独播
- 动漫: 768条加载 → 89条独播
- 总计: 742条独播内容

### 注意事项
- 电影频道有 URL 级别的独播筛选（规格参数=4），但结果仍包含非独播内容，需用 `only.png` 二次过滤
- 新版频道页（`www.iqiyi.com/dianshiju/`）为 SPA，数据通过 JS 动态加载，不推荐直接抓取

---

## 腾讯视频 (v.qq.com)

**状态**: 待探索

---

## 优酷 (youku.com)

**状态**: 待探索

---

## 芒果TV (mgtv.com)

**状态**: 待探索

---

## B站 (bilibili.com)

**状态**: 待探索
