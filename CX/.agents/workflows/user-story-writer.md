---
description: Viết User Story và Acceptance Criteria chuẩn INVEST + Given-When-Then cho BA/PO
---

# User Story & AC Writer Workflow

Sử dụng workflow này để tạo mới, cải thiện (refine) hoặc bổ sung Acceptance Criteria (AC) cho các User Story. Quy trình này dựa trên bộ kỹ năng từ BA Zone.

## Hướng dẫn sử dụng

Khi user yêu cầu viết User Story hoặc AC, hãy thực hiện theo các bước sau:

### 1. Kích hoạt Skill
Đọc và tuân thủ các chỉ dẫn trong file skill:
`file:///.agents/skills/user-story-writer/SKILL.md`

### 2. Các bước thực hiện

1. **Xác định Chế độ (Mode):**
   - **Mode A - Viết mới**: Từ mô tả tính năng (feature description).
   - **Mode B - Refine**: Review và sửa US/AC có sẵn.
   - **Mode C - Bổ sung AC**: Thêm chi tiết cho US đã có.

2. **Thu thập Input (nếu thiếu):**
   - **Persona**: Ai dùng?
   - **Goal**: Làm gì?
   - **Business Value**: Tại sao?
   - **Context**: Thuộc feature/module nào?

3. **Sinh User Story:**
   - Format: `As a [persona], I want to [goal], So that [value]`.
   - Đảm bảo chuẩn **INVEST** (Independent, Negotiable, Valuable, Estimable, Small, Testable).
   - Tham khảo: `.agents/skills/user-story-writer/references/invest-criteria.md`

4. **Sinh Acceptance Criteria:**
   - Format: **Given-When-Then** (Gherkin syntax).
   - Tối thiểu 3 AC: Happy path, Edge case, Negative path.
   - Tham khảo: `.agents/skills/user-story-writer/templates/ac-template.md`

5. **Kiểm tra chất lượng (Self-check):**
   - Sử dụng bảng INVEST Self-check.
   - Đối chiếu với checklist: `.agents/skills/user-story-writer/: checklists/quality-checklist.md`

### 3. Đầu ra (Output)
Trình bày kết quả theo cấu trúc:
1. **User Story** (As a / I want / So that)
2. **INVEST Self-check table** (✅/⚠️)
3. **Acceptance Criteria** (AC1, AC2, AC3...)
4. **Notes** (Dependency, Assumption, Questions)

## Tham khảo
- Ví dụ mẫu: `.agents/skills/user-story-writer/references/examples.md`
- Template US: `.agents/skills/user-story-writer/templates/user-story-template.md`
