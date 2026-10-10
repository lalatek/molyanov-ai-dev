---
# Creation date (YYYY-MM-DD)
created: 2026-10-10

# Status: draft | approved
status: draft

# Work type: feature | bug | refactoring
type: feature
---

# User-spec: ui-preview-approval

> **Executor instruction.** If the project has Project Knowledge, first read its main `SKILL.md`,
> then only the materials it routes to for this task. Read `decisions.md` if it exists. Work from
> the root of the project this spec belongs to. Implement the entire user-spec. Use the execution
> skills appropriate to the work.

## What We Are Building

Thêm một bước mới — **UI Mockup Preview & Approval Step** — vào skill `user-spec-planning` (Step 7,
chạy ngay sau Step 6 hiện tại), áp dụng cho mọi dự án dùng methodology này. Sau khi user-spec được
approve và trước khi implementation (`layout-writing`/`code-writing`) bắt đầu, với mọi feature thực
sự có UI, AI sinh ra một hoặc hai mockup HTML/CSS tĩnh độ trung thực cao (một file cho mỗi platform
feature nhắm tới — web và/hoặc mobile), mỗi file bao quát toàn bộ màn hình/flow của feature. Người
dùng xem và duyệt (approve/reject) mockup trước khi code bắt đầu; sau khi approve, mockup trở thành
nguồn visual bắt buộc đối chiếu khi implement.

## Why

Hiện tại, với các dự án có UI (web, mobile), không có cách nào để hình dung/preview UI trước khi
vào implementation — chỉ có UI/UX spec dạng text. Người dùng không thể đánh giá UI dự kiến có đúng
ý hay không trước khi code được viết, nên các sai lệch chỉ bị phát hiện sau khi đã implement xong,
gây tốn công sửa lại.

## Expected Behavior

**Local / interactive mode:**
1. `user-spec-planning` Step 6 duyệt nội dung spec text như hiện tại (không đổi).
2. Nếu feature có UI thực sự (topic `ui_ux_requirements` không rỗng), Step 7 mới sinh ra
   `mockup-web.html` và/hoặc `mockup-mobile.html` trong `work/{feature}/` — chỉ sinh file cho
   platform feature thực sự nhắm tới (feature chỉ-web → 1 file; feature cross-platform → 2 file).
   Mỗi file là HTML/CSS tĩnh, độ trung thực cao, bao quát toàn bộ màn hình/flow của feature trong
   một file duy nhất, bám theo design system/style token sẵn có của dự án nếu có (theo "Evidence
   Order" đã có của `layout-writing/references/design-decisions.md`), hoặc theo general convention
   nếu dự án chưa có style guide.
3. Agent trình bày mockup cho người dùng xem (mở file trong browser).
4. Người dùng approve, hoặc yêu cầu sửa — Step 7 sinh lại mockup và trình lại, không giới hạn số
   lần lặp.
5. Sau khi approve, mockup path(s) được đưa vào phần hand-off cho implementation.
6. `layout-writing`/`code-writing` implement dựa trên mockup làm nguồn visual, coi mockup như
   nhánh "Claude Design export" đã có trong `layout-writing/references/reproduce.md` (không thêm
   source type thứ 4).
7. `layout-writing` dùng lại cơ chế diff sẵn có (`capture.mjs`/`overlay.mjs`, review bởi
   `layout-reviewer`) để xác nhận UI đã implement khớp với mockup đã approve.

**online-pipeline (headless, chạy qua GitHub Actions):**
1. Stage `userspec-turn` chạy Step 1–5 như hiện tại.
2. Trước khi ghi `status: ready_for_review`, nếu feature có UI, agent sinh mockup(s), chụp
   screenshot từng mockup bằng `capture.mjs` (Playwright) — Playwright được cài trong một thư mục
   tooling riêng trong checkout của workflow, độc lập với `package.json`/stack của target project
   (không phụ thuộc target project phải là dự án Node) — rồi commit các file PNG screenshot vào
   cùng branch spec (`userspec/{slug}`) cùng với mockup.html. Agent **không tự mở hay comment lên
   PR** (giữ đúng nguyên tắc hiện có: agent không gọi `gh pr create`); agent chỉ commit file lên
   branch rồi ghi `ready_for_review` và exit như cũ — vì tại thời điểm agent còn chạy, PR spec có
   thể chưa tồn tại (PR chỉ được mở sau khi agent exit, bởi bước YAML ở bước 3).
3. Bước deterministic YAML "Open or update the spec PR" (chạy sau khi agent exit, lúc này PR đã
   chắc chắn tồn tại) được mở rộng: sau khi mở/update PR như hiện tại, đọc các file PNG đã commit
   trên branch và post thêm 1 comment lên đúng PR đó, chứa ảnh markdown
   `![mockup](.../blob/{branch}/{path}?raw=true)` cho mỗi screenshot (bắt buộc dùng cú pháp ảnh
   markdown, không phải link trần, để GitHub render inline) — GitHub render ảnh inline khi viewer
   đã đăng nhập.
4. Một comment `/approve` duy nhất trên PR đó duyệt cả spec text và mockup cùng lúc (không cần sửa
   logic Worker ở repo `control-plane` ngoài repo này, vốn chỉ nhận diện 2 loại PR hiện có).
5. Yêu cầu sửa đi theo route bình thường (issue comment mới → Resume path → sinh lại mockup cùng
   với spec, PR update lại kèm comment ảnh mới).

Với feature không có UI thực sự (backend-only), Step 7 bị skip hoàn toàn ở cả 2 mode — không sinh
mockup, không có gì chặn implementation.

## Acceptance Criteria

- [ ] Với feature có UI thực sự, sau Step 7, `work/{feature}/` chứa `mockup-web.html` và/hoặc
      `mockup-mobile.html` (chỉ platform feature nhắm tới), mỗi file bao quát mọi màn hình/flow
      nêu trong user-spec.
- [ ] Implementation (`layout-writing`/`code-writing`) không được bắt đầu trước khi mockup được
      approve — xác nhận trong chat (local) hoặc comment `/approve` trên spec PR kèm screenshot đã
      đính (online-pipeline). *(Nhánh online-pipeline: verify ở lần chạy thật đầu tiên sau merge,
      xem Risk 4 — không verify được trước merge trong session planning này.)*
- [ ] Với online-pipeline, sau khi bước YAML mở/update PR spec, PR đó có comment kèm ảnh markdown
      `![mockup](.../blob/{branch}/{path}?raw=true)` cho mỗi mockup; không có PR/branch thứ 2 nào
      được tạo cho mockup. *(Verify ở lần chạy thật đầu tiên sau merge, xem Risk 4.)*
- [ ] Sau khi implementation tồn tại, cơ chế diff sẵn có của `layout-writing`
      (`capture.mjs`/`overlay.mjs`) chạy đối chiếu với mockup đã approve làm nguồn visual.
- [ ] Feature không có UI (không có `ui_ux_requirements`) bỏ qua hoàn toàn Step 7 — không sinh file
      mockup, không có gì chặn implementation.
- [ ] Mockup file (và screenshot PNG ở online-pipeline) được giữ lại sau khi implement xong, không
      bị xoá.
- [ ] `skills/methodology/SKILL.md` phản ánh bước mới (lifecycle diagram, danh sách bước "Plan the
      Feature", cây thư mục `work/{feature}/`, bảng Skill Responsibilities).

## Constraints

- Không có tool nào ghi trực tiếp vào Figma trong môi trường hiện tại — mockup phải là artifact
  HTML/CSS tĩnh, không phải file `.fig` thật.
- Cơ chế `/approve` ở online-pipeline nằm ở repo `control-plane` (ngoài repo này, ngoài khả năng
  sửa trong phạm vi feature này) và chỉ nhận diện 2 loại PR hiện có (`userspec/{slug}`,
  `feature/{slug}`) — không thể mở PR/branch thứ 3 riêng cho mockup; mockup phải đi theo cùng PR
  spec hiện có.
- `capture.mjs` resolve Playwright qua `package.json` của project được chỉ định. Vì methodology
  này stack-agnostic (dự án đích có thể là Python/Ruby/PHP/Go, không có `package.json`), workflow
  `userspec-turn` phải cài Playwright vào một thư mục tooling riêng trong CI checkout (độc lập
  stack của target project) và trỏ `capture.mjs` vào thư mục đó khi chạy trong online-pipeline —
  không trỏ vào root của target project.
- `gh` CLI không có cách upload ảnh trực tiếp vào comment. Giải pháp: commit file PNG vào branch
  spec, reference bằng ảnh markdown `![mockup](.../blob/{branch}/{path}?raw=true)` (không phải link
  trần — cú pháp markdown ảnh là bắt buộc để GitHub render inline) trong comment. PNG được giữ lại
  trên branch cùng mockup.html, không xoá sau khi approve.
- Agent không tự mở hay comment lên PR (giữ nguyên tắc hiện có: chỉ workflow YAML deterministic
  mới thao tác PR, agent không gọi `gh pr create`). Việc post comment kèm ảnh screenshot phải do
  bước YAML "Open or update the spec PR" thực hiện (chạy sau khi agent đã exit và PR chắc chắn tồn
  tại), không phải do agent — vì tại thời điểm agent đang chạy lần đầu, PR spec có thể chưa tồn
  tại.
- Nếu `capture.mjs`/Playwright screenshot thất bại trong CI (ví dụ render timeout), xử lý như bất
  kỳ lỗi bước CI khác trong workflow đó — job fail và báo lỗi qua cơ chế issue-comment
  error-reporting hiện có, không có fallback âm thầm nào.
- Thêm bước cài Playwright vào CI workflow `userspec-turn` làm tăng thời gian chạy CI cho mọi
  feature có UI — đã được người dùng chấp nhận.

## Risks

- **Risk 1:** Mockup lệch với nội dung spec text nếu người duyệt chỉ đọc 1 trong 2 tài liệu.
  **Mitigation:** mockup phải được sinh bám sát đúng user-spec đã approve, không tự thêm scope mới.
- **Risk 2:** Thêm thời gian/chi phí CI cho mọi feature có UI, kể cả feature nhỏ.
  **Mitigation:** Step 7 (và bước cài Playwright trong CI) chỉ chạy khi feature thực sự có UI
  (`ui_ux_requirements` không rỗng); feature backend-only không bị ảnh hưởng.
- **Risk 3 (limitation đã chấp nhận, không cần mitigation):** Ở online-pipeline, việc gộp approval
  spec + mockup vào 1 comment `/approve` nghĩa là người duyệt không thể approve riêng spec text mà
  reject riêng mockup (hoặc ngược lại) — muốn sửa phải yêu cầu sửa cả 2 rồi duyệt lại cùng lúc.
- **Risk 4:** Nhánh online-pipeline (cài Playwright trong CI, chạy `capture.mjs` trong CI, commit
  PNG + comment link, gộp `/approve`) chưa được verify thật trong session planning này (không
  emulate được GitHub Actions thật). **Mitigation:** verify bằng cách chạy thật trên feature UI
  đầu tiên đi qua online-pipeline sau khi feature này merge, theo dõi sát lần chạy đầu để phát hiện
  và sửa lỗi kịp thời.

## Accepted Decisions

- Chọn mockup HTML/CSS tĩnh thay vì file Figma thật vì không có tool nào trong môi trường hiện tại
  có thể ghi vào Figma.
- Chọn chụp screenshot mockup bằng `capture.mjs` (Playwright) có sẵn của `layout-writing` thay vì
  xây cơ chế render mới, và đính ảnh vào PR spec hiện có thay vì mở PR/branch thứ 3, để tránh phải
  sửa logic ở repo `control-plane` ngoài phạm vi repo này.
- Chọn để `layout-writing` tái dùng nhánh xử lý "Claude Design export" đã có trong `reproduce.md`
  cho mockup, thay vì định nghĩa thêm một source type thứ 4.
- Chọn tách riêng 2 file mockup cho web và mobile (không dùng 1 file chung có phone-frame mô
  phỏng) khi feature nhắm tới cả 2 platform.
- Chọn cách xử lý khác nhau giữa local (2 bước duyệt riêng: Step 6 duyệt spec, Step 7 duyệt
  mockup, đều trong chat) và online-pipeline (gộp 1 bước duyệt chung cho cả spec + mockup trên
  cùng PR) — vì online-pipeline bị giới hạn bởi logic Worker ở `control-plane`, còn local không bị
  giới hạn này.
- Chọn để bước YAML deterministic "Open or update the spec PR" (không phải agent) đảm nhiệm việc
  post comment kèm ảnh screenshot — vì agent không được tự ý mở/comment PR theo nguyên tắc hiện có
  của online-pipeline, và PR chỉ chắc chắn tồn tại sau khi agent đã exit. Agent chỉ commit
  mockup/PNG vào branch; YAML lo phần thao tác PR, giữ đúng sự phân chia trách nhiệm hiện có.
- Chọn chấp nhận giới hạn: nhánh online-pipeline của feature này chỉ được verify bằng chạy thật sau
  khi merge, không verify được trong session planning (không có cách emulate GitHub Actions thật ở
  đây).

## Testing

**Unit tests:** không cần — đây là thay đổi ở skill instructions (markdown), không có logic code
mới cần unit test.

**Integration tests:** không cần theo nghĩa test code tự động — thay vào đó, dry-run thủ công Step
7 mới trên một feature mẫu có UI thực sự ở local/interactive mode: xác nhận mockup.html(s) sinh
đúng, approval gate chặn implementation cho tới khi approve, và `layout-writing` lấy đúng mockup đã
approve làm nguồn visual.

**E2E tests:** không áp dụng. Nhánh online-pipeline (cài Playwright trong CI, chạy `capture.mjs`
trong CI, commit PNG + comment link, gộp `/approve`) không thể emulate trong session planning này —
đây là known limitation đã được người dùng chấp nhận (xem Risk 4), sẽ verify bằng cách chạy thật
trên feature UI đầu tiên đi qua online-pipeline sau khi merge.

## Verification

### Agent Verification

| Step | Expected Result |
|------|-----------------|
| 1. Dry-run Step 7 mới trên 1 feature mẫu có UI, local/interactive mode | `mockup-web.html`/`mockup-mobile.html` sinh đúng trong `work/{feature}/`, bao quát đủ màn hình/flow của feature mẫu |
| 2. Yêu cầu sửa mockup trong dry-run | Step 7 sinh lại mockup và trình lại, không có giới hạn số lần lặp |
| 3. Approve mockup trong dry-run | Implementation (`layout-writing`) nhận đúng mockup path(s) làm nguồn visual |
| 4. Chạy `capture.mjs`/`overlay.mjs` sau khi implement xong UI mẫu | Diff xác nhận UI đã code khớp với mockup đã approve |
| 5. Dry-run với 1 feature mẫu backend-only (không có `ui_ux_requirements`) | Step 7 bị skip hoàn toàn, không sinh file mockup, implementation không bị chặn |
| 6. Review các SKILL.md mới/sửa (`user-spec-planning`, `layout-writing`, `online-pipeline`, `methodology`) bằng agent `skill-checker`, `skill-logic-reviewer`, và `skill-simplicity-reviewer` (3 lane theo đúng Review Model của `methodology/SKILL.md`) | Không có finding nghiêm trọng về cấu trúc/logic/over-engineering của skill |
| 7. Sau khi implement xong 1 feature mẫu có UI, kiểm tra lại `work/{feature}/` | File mockup (và PNG screenshot nếu online-pipeline) vẫn còn, không bị xoá bởi bước implementation/cleanup nào |

### User Verification

- Xem thử mockup sinh ra trong dry-run (mở `mockup-web.html`/`mockup-mobile.html` trong browser) —
  xác nhận mockup thực sự trung thực cao, dễ đọc, và đúng ý tưởng trước khi feature này được coi là
  hoàn tất — vì "trung thực cao, dễ duyệt" là tiêu chí chủ quan, cần người dùng tự đánh giá thẩm mỹ,
  không phải thứ agent tự verify được.
- Sau khi merge, theo dõi lần chạy online-pipeline đầu tiên có feature UI thực sự: xác nhận
  Playwright cài đúng trong CI, `capture.mjs` chạy được, PNG commit + comment link render đúng trên
  GitHub, `/approve` gộp duyệt cả spec + mockup — vì nhánh này chưa được verify trong session
  planning (known limitation đã chấp nhận).
