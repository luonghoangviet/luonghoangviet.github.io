# Playwright MCP: tự động hóa web kết hợp AI agent

## Playwright là gì?
Playwright là thư viện automation từ Microsoft giúp điều khiển browser (Chromium, Firefox, WebKit) theo cách đáng tin cậy và song song. Nó hỗ trợ context độc lập, isolation cookie/storage, network interception, tracing, headless/headful, và API đồng nhất đa trình duyệt.

### Ưu điểm so với WebdriverIO
- **Kiến trúc hiện đại**: Playwright dùng DevTools protocol trực tiếp, không cần Selenium Grid; khởi động nhanh, ít flake hơn.
- **Đa trình duyệt gốc**: Chromium/Firefox/WebKit được đội Playwright build kèm; phiên bản đồng bộ, giảm lệch API.
- **Isolation mạnh**: Browser context tạo nhanh, phù hợp test song song; fixture/test runner tích hợp.
- **Auto-wait thông minh**: Chờ element sẵn sàng/ổn định trước khi tương tác, giảm phải thêm sleep thủ công.
- **Network & storage control**: Dễ mock response, ghi/đọc storage state, cho phép warm start phiên đăng nhập.

## MCP (Model Context Protocol) và context engineering
- **MCP là gì**: Giao thức chuẩn để agent/LLM truy cập công cụ ngoài (tools, data sources) qua một API thống nhất (yml/json schema). Giúp tách “não LLM” và “tay chân công cụ”.
- **Context engineering**: Thiết kế ngữ cảnh cung cấp cho LLM (system prompt, tool schema, memory, tri thức) để hành động đúng nhiệm vụ, an toàn và kiểm soát được.
- **Bridge AI agent ↔ LLM ↔ Playwright**: MCP đóng vai lớp glue. LLM gọi tool `playwright` thông qua MCP server; server thực thi Playwright (mở trang, click, điền form, đọc DOM) và trả kết quả/ảnh/chụp accessibility tree về cho LLM. Nhờ đó agent vừa “đọc/hiểu” trang vừa “bấm” được.

## Playwright MCP
### Accessibility tree là gì, hoạt động ra sao?
Accessibility tree là cấu trúc cây mô tả giao diện theo góc nhìn trợ năng (role, name, state) được xây từ DOM + ARIA + thuộc tính accessibility. Playwright MCP có thể xuất cây này để LLM hiểu ngữ nghĩa: nút nào là button, label nào thuộc input nào, trạng thái checked/disabled ra sao.

### Ứng dụng của Playwright MCP
- **Tự động hóa có hiểu ngữ cảnh**: Agent đọc accessibility tree để chọn nút đúng thay vì dựa vào CSS fragile.
- **Kiểm thử end-to-end sinh mã tự động**: LLM sinh step click/điền form, chạy qua MCP, rồi tự sửa step nếu gặp lỗi.
- **Kiểm tra accessibility**: Dùng tree để phát hiện thiếu label/ARIA hoặc flow bàn phím.
- **Data collection an toàn**: Crawl có kiểm soát (tôn trọng robots, tốc độ), trả về dữ liệu cấu trúc cho LLM.

## Ví dụ nhanh
### Playwright (thuần)
```js
import { chromium } from 'playwright';

(async () => {
  const browser = await chromium.launch({ headless: true });
  const page = await browser.newPage();
  await page.goto('https://example.com');
  await page.getByRole('link', { name: 'More information' }).click();
  console.log(await page.title());
  await browser.close();
})();
```

### Playwright MCP (mẫu ý tưởng)
Giả sử MCP server expose tool `playwright.browse`:
```yaml
# tool schema (giản lược)
name: playwright.browse
params:
  url: string
  actions: array # chuỗi hành động yêu cầu
  returnAccessibilityTree: boolean
```
LLM gọi tool:
```json
{
  "tool": "playwright.browse",
  "params": {
    "url": "https://example.com",
    "actions": [
      { "type": "click", "selector": "role=link[name='More information']" }
    ],
    "returnAccessibilityTree": true
  }
}
```
MCP server chạy Playwright, trả kết quả:
- Ảnh chụp màn hình hoặc HTML snippet sau khi click
- Accessibility tree để LLM hiểu cấu trúc
- Trạng thái (thành công/thất bại) và log để LLM tự sửa bước tiếp theo

## Tóm tắt
- Playwright: automation đa trình duyệt nhanh, ổn định, auto-wait tốt.
- MCP: chuẩn nối LLM với công cụ; context engineering là thiết kế ngữ cảnh + tool để LLM dùng đúng.
- Playwright MCP: cho phép agent vừa “nhìn/hiểu” (accessibility tree) vừa “hành động” (click/điền form), hữu ích cho test tự động, RPA thông minh và kiểm tra accessibility.
