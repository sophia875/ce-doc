# Quality Checklist - Self Review trước khi xuất

> Skill phải tự kiểm tra theo checklist này trước khi xuất output cuối cùng.
> Nếu fail bất kỳ mục nào → fix trước khi trả về cho user.
>
> Biên soạn bởi **Phúc NT** · BA Zone · Digital School  
> Đã chỉnh cho dự án Cask Exchange

---

## ✅ Phần 1: User Story Quality

### Persona (As a...)
- [ ] Persona cụ thể, không dùng "user" chung
- [ ] Có mô tả trạng thái/điều kiện kèm theo (đã KYC, đã connect Stripe, đang có Bid Active, đang ở bước checkout nào...)
- [ ] Phân biệt được với các persona khác trong sản phẩm (Buyer / Seller / Admin / System). Nhớ: Buyer/Seller là vai trò trong 1 giao dịch, không phải loại account

### Goal (I want to...)
- [ ] Mô tả hành động cụ thể, không mơ hồ
- [ ] Dùng động từ rõ ràng (place a Bid, pay the deposit, upload bank transfer proof)
- [ ] Có thể đo lường khi nào "đã làm xong"
- [ ] Không chứa từ "manage", "handle" chung chung

### Business Value (So that...)
- [ ] Khác biệt rõ với phần "I want to"
- [ ] Mô tả outcome cho user/business, không phải feature
- [ ] Trả lời được câu "Nếu không làm thì Buyer / Seller / Admin hoặc Cask Exchange mất gì?"
- [ ] Tránh lặp lại nội dung của Goal

---

## ✅ Phần 2: INVEST Compliance

| Tiêu chí | Đã check? | Notes |
|----------|-----------|-------|
| Independent - Story chạy độc lập | [ ] | |
| Negotiable - Có chỗ thảo luận | [ ] | |
| Valuable - Giá trị rõ ràng | [ ] | |
| Estimable - Dev ước lượng được | [ ] | |
| Small - ≤ 5 ngày dev work | [ ] | |
| Testable - QA viết được test case | [ ] | |

**Nếu có ⚠️ ở Small**: đề xuất split thành các story con  
**Nếu có ⚠️ ở Estimable**: đề xuất tạo Spike story trước

---

## ✅ Phần 3: Acceptance Criteria Quality

### Số lượng
- [ ] Tối thiểu 3 AC (1 happy + 1 edge + 1 negative)
- [ ] Tối đa 7-8 AC (nếu nhiều hơn → split story)

### Format Given-When-Then
- [ ] Mỗi AC có đủ 3 phần Given/When/Then
- [ ] Given mô tả tiền điều kiện cụ thể, không mơ hồ
- [ ] When mô tả 1 hành động duy nhất của user
- [ ] Then mô tả kết quả đo lường được

### Coverage scenarios
- [ ] Có ít nhất 1 happy path
- [ ] Có ít nhất 1 edge case (boundary, race condition, timeout, offline...)
- [ ] Có ít nhất 1 negative path (lỗi, validation fail, permission denied)
- [ ] Edge case đặc thù Cask Exchange: 0, null, cancel, expire, partial match, 1:N checkout session, hai user thao tác cùng lúc (chọn case nào phù hợp)

### Đo lường được
- [ ] Có số liệu cụ thể khi cần (số tiền £, fee, deadline, số lượng)
- [ ] Nói rõ loại quantity (target / filled / remaining / matchable), không ghi chung chung "quantity"
- [ ] Có message text cụ thể (không "thông báo lỗi" chung chung)
- [ ] Có trạng thái rõ ràng, dùng đúng status trong `docs/project-summary.md` mục 3.3 (ví dụ Bid Active / Cancelled, session `pending` / `expired`)

### Tránh anti-patterns
- [ ] Không chứa từ mơ hồ: "nhanh", "đẹp", "user-friendly", "intuitive", "phù hợp"
- [ ] Không chứa logic implementation: API name, DB schema, framework
- [ ] Không mô tả UI cụ thể: màu, vị trí pixel, font size
- [ ] Không gộp nhiều scenario vào 1 AC

---

## ✅ Phần 4: Domain Cask Exchange

- [ ] Tên sản phẩm là "Cask Exchange" (không "CaskX")
- [ ] Dùng "vintage" trong US/AC, tên hiển thị `[Master Cask Name] [Vintage Year]`
- [ ] Không mâu thuẫn rule đã chốt (`docs/project-summary.md` mục 4). Nếu có thì ghi conflict vào Notes
- [ ] Ghi story nằm ở đâu trên core flow
- [ ] Chỗ chưa biết đã ghi `[OPEN]` kèm người cần trả lời

---

## ✅ Phần 5: Output Format

- [ ] Có ID story rõ ràng (US-FEATURE-001 format)
- [ ] Có metadata: Epic, Priority, Estimate (nếu có thông tin)
- [ ] Bảng INVEST self-check đầy đủ 6 dòng
- [ ] AC đánh số rõ ràng (AC1, AC2, AC3)
- [ ] Có section "Notes" cho dependency, assumption, open questions

---

## ✅ Phần 6: Sanity Check tổng thể

Trước khi xuất, đọc lại 1 lần và tự hỏi:

1. **Stakeholder hiểu được không?** PO không kỹ thuật đọc có hiểu không?
2. **Dev estimate được không?** Junior dev đọc có ước lượng được không?
3. **QA test được không?** QA có viết được tối thiểu 5 test case từ AC không?
4. **Có bị trùng story khác không?** Có overlap với story nào trong backlog không?
5. **"So that" có thật sự valuable không?** Hay chỉ là echo của "I want to"?

---

## 🚨 Red flags - Phải fix ngay nếu gặp

❌ Story description dài hơn 200 từ → quá chi tiết, mất tính Negotiable  
❌ AC dài hơn 10 dòng → quá phức tạp, nên split  
❌ Không có AC nào → không Testable  
❌ "So that" rỗng hoặc copy "I want to" → không Valuable  
❌ Persona là "user" hoặc "everyone" → quá generic  
❌ Có chữ "AND" trong tiêu đề → đang gộp 2 story  
❌ AC dùng từ "etc.", "v.v." → không cụ thể, không testable  

---

## 📋 Final Output Template

Khi mọi checklist đã pass, format output theo cấu trúc:

```markdown
## US-[ID]: [Title]

**As a** [persona]
**I want to** [goal]
**So that** [value]

### INVEST Self-check
[Bảng 6 dòng]

### Acceptance Criteria
**AC1: [Happy path name]**
[Given/When/Then]

**AC2: [Edge case name]**
[Given/When/Then]

**AC3: [Negative path name]**
[Given/When/Then]

### Notes
- Dependencies: ...
- Assumptions: ...
- Open Questions: ...
```

---
*Biên soạn bởi **Phúc NT** · BA Zone · Digital School*  
*Vui lòng dẫn nguồn khi sử dụng hoặc chia sẻ tài liệu này.*
