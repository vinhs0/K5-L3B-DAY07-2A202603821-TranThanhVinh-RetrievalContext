# Báo cáo Day 7

Sinh viên: Trần Thanh Vinh, MSSV 2A202603821. Ngày chạy: 2026-11-16. Mọi số liệu lấy từ các file trong `results/` (đường dẫn ghi kèm); chi tiết thiết kế ở `docs/setup.md`, `docs/retrieval.md`, `docs/agent.md`, `docs/test_eval.md`.

## Dữ liệu và benchmark

- Nguồn, quyền sử dụng, số trang và số chunk: giáo trình OpenStax *Introduction to Sociology 3e* (giấy phép CC BY 4.0, được phép chia sẻ), dùng lại từ Day 6. Trang PDF vật lý 131-212 (chương 5-7), **69 trang có nội dung** (`data/processed/pages.jsonl` trong Day 6; 1 tài liệu, `source_id = soc3e`). Collection **B** `soc3e_B`: **241 chunk** (gộp theo section/đoạn, tối đa 1200 ký tự; `data/chunks.json`). Collection A `soc3e_A` (468 chunk cố định 500 ký tự) chỉ dùng để thử nghiệm. Corpus chỉ có 1 nguồn nên filter metadata chỉ có `chapter`/`section`/`document_type`.
- Vector store, embedding model, phiên bản và số chiều: ChromaDB 1.5.9, cosine HNSW, collection `soc3e_B`. Embedding `BAAI/bge-small-en-v1.5` (chạy local, `sentence-transformers 6.3.0`, revision `5c38ec7c405ec4b44b94cc5a9bb96e735b38267a`), **384 chiều**, chuẩn hóa L2, query có prefix `Represent this sentence for searching relevant passages: `. Chi phí 0 USD. Repo Day 7 tự dựng lại store được (`scripts/build_index.py`, khoảng 29 s trên CPU). Kiểm tra tương đương với store Day 6 trong `results/index_parity.json`: 241/241 id, document và metadata giống nhau, sai khác embedding lớn nhất 0.0, top-10 giống nhau cho 30/30 câu hỏi.
- Agent model, cách chạy và phân chia dev/test: model thật `gemini-3.5-flash-lite` (Gemini API free tier, `google-genai 3.3.0`, temperature 0, thinking_budget 1024, function calling thủ công). Model mặc định `gemini-2.5-flash` và các model dự phòng trả 404 (không còn phục vụ với key này); `gemini-3.5-flash` có quota 20 request/ngày nên bị 429, các lần chạy dở đó đã xóa và chạy lại toàn bộ trên `flash-lite` (0 lỗi provider ở các kết quả báo cáo). Benchmark 30 câu (`benchmark/questions.json` cho agent, `benchmark/gold/gold.json` chỉ cho bộ chấm; sinh bởi `scripts/build_benchmark.py`): **dev 19 câu** (16 có đáp án gồm 1 multi-hop `d13`, 3 thiếu evidence), **test 11 câu** (8 có đáp án gồm 1 multi-hop `t08`, 3 thiếu evidence). Đạt mức tối thiểu của BENCHMARK.md (dev 15+2, test 5+2) và SUBMISSION.md (20+4). Câu hỏi viết từ chunk thật, không dùng lại bộ test Day 6. Dev dùng để thử và chọn cấu hình; cấu hình đóng băng trong `configs/final.json` (`frozen: true`, 2026-11-16) trước khi chạy test; agent không nhận gold.
- Cách xác định câu có/thiếu evidence và tiêu chí chấm: câu có đáp án ghi `chunk_id`, trang, section, đoạn trích evidence (kiểm tra là chuỗi con chính xác của chunk) và `evidence_mode` (`any` hoặc `all`; `d13`, `t08` cần **cả hai** chunk). Câu thiếu evidence là câu gần chủ đề nhưng thiếu thông tin chính (vd. t11: Egalia có trong sách nhưng không có số học sinh); kiểm tra bằng đếm từ khóa không phân biệt hoa thường trên 241 chunk và 69 trang = 0, cộng đọc tay trang lân cận (trong `gold.json`, `absence_check`); phát biểu "thiếu" chỉ đúng với bộ tài liệu này. Câu thiếu evidence không nằm trong mẫu số metric retrieval. Chấm agent theo thang BENCHMARK.md (grounded 0/1/2, citation hợp lệ, abstention, context 0/1/2). **Nhãn do một AI (Claude) đọc câu trả lời, chunk được trích dẫn và gold rồi chấm (trường `labeler` trong `reviews.json`); chưa có người chấm thứ hai, sinh viên cần kiểm tra mẫu.**

## Retrieval và thử nghiệm

Cùng corpus, cùng index `soc3e_B`, query gốc = nguyên văn câu hỏi, không filter (trừ khi nêu), k=5, `n/N` trên câu có đáp án (dev N=16, test N=8). Hit@k = có chunk gold trong top-k. Trích từ `results/retrieval/{dev,test}/comparison.md`. Cột "Context Evidence Rate": với hàng chỉ retrieval không có bước chọn context nên ghi "—" (Recall@5 bằng Hit@5 được nêu để tham khảo); số context cuối của agent nằm ở bảng Agent bên dưới (và được gộp vào hàng test "đã chọn").

| Cấu hình | Phương pháp, top-k, filter, rerank, context budget | Giả thuyết / thay đổi | Split | Hit@1 | Evidence Hit@3 | Context Evidence Rate | Thời gian / chi phí |
|---|---|---|---|---|---|---|---|
| Baseline (`dense_k5`) | Dense bge-small trên Chroma Day 6, k=5, không filter, không rerank; không chọn context | Mốc Day 6 | dev | 13/16 = 0.812 | 15/16 = 0.938 | — (chỉ retrieval; Recall@5 = 15/16) | 46 ms median (p95 84); 0 USD |
| Phương pháp 2 (`bm25_k5`) | BM25Okapi (k1=1.5, b=0.75), token hóa lowercase + stopword + stemmer nhỏ, k=5 | H0: cạnh tranh ở thực thể có tên | dev | 14/16 = 0.875 | 15/16 = 0.938 | — (Recall@5 = 15/16) | 1 ms; 0 USD |
| Phương pháp 3 (`hybrid_k5`) | Dense top-20 + BM25 top-20 hợp nhất RRF (k=60), k=5 | H0: hybrid >= max(dense, bm25) | dev | 15/16 = 0.938 | 15/16 = 0.938 | — (Recall@5 = 15/16) | 52 ms; 0 USD |
| Cải thiện 1 (`imp4_hybrid_rerank_pool30_k5`; vừa là phương pháp mới) | Hybrid, pool 30 ứng viên, rerank bằng cross-encoder `ms-marco-MiniLM-L-6-v2` (local), k=5 | H4/H6: sửa lỗi d09 (gold nằm ngoài pool 20); đọc cặp query-đoạn nên Hit@1 tăng | dev | 15/16 = 0.938 | 16/16 = 1.000 | — (Recall@5 = 16/16) | 1090 ms median (p95 1494), khoảng 20 lần hybrid; 0 USD (tải model 88 MB một lần) |
| Cải thiện 2 (`imp6_hybrid_rerank_pool30_demote_k5`) | Như trên + `demote_backmatter` (chunk key_terms/summary/review_questions xếp sau chunk thân bài, chỉ dùng metadata) | H7: sửa lỗi d07 (chunk glossary đứng trước chunk gold) | dev | 16/16 = 1.000 | 16/16 = 1.000 | — (Recall@5 = 16/16; agent: xem bảng sau) | 1033 ms median (p95 1148); 0 USD |
| Cấu hình đã chọn (`imp6` làm retrieval của agent `final`) | `hybrid_rerank`, pool/fuse depth 30, RRF k=60, demote, k=5; agent: tối đa 2 tool call, context budget 8000 ký tự | Chốt trên dev, chạy test một lần sau khi đóng băng | test | 7/8 = 0.875 | 8/8 = 1.000 | 8/8 (agent `final`, context cuối chứa gold; t08 cần cả 2 chunk) | Retrieval 944 ms median (p95 1515); agent 4063 token/câu, 1.36 tool call/câu, 0 USD |
| Baseline (`dense_k5`; agent `base` dùng hybrid k5) | Dense k=5 như trên | Mốc đối chứng trên test | test | 7/8 = 0.875 | 7/8 = 0.875 | 8/8 (agent `base`) | Retrieval 23 ms median (p95 30); agent `base` 5697 token/câu, 1.82 tool call/câu, 0 USD |

Các thử nghiệm khác trên dev không giúp (`results/retrieval/dev/comparison.md`): hybrid depth 10/50 giống hệt depth 20; chapter filter suy luận từ query không đổi metric; collection A (chunk 500 ký tự) tệ hơn (hybrid Hit@3 0.875, mất multi-hop d13, d07, d08); rerank pool 20 (MRR 0.906) thấp hơn hybrid (0.938). Đơn vị thời gian: retrieval = ms/câu trên CPU, model đã nạp, một lần chạy, nhiễu (cùng model rerank đo 326 ms đến 1282 ms tùy tải máy), chỉ tin bậc độ lớn. Agent = giây (xem bảng dưới). Chi phí tiền = 0 USD (embedding, rerank chạy local; Gemini free tier), token/câu là đại lượng thay thế. Context Evidence Rate của retrieval-only không đo được vì chưa có bước chọn context; số liệu context nằm ở agent.

**Hai lỗi dev, nguyên nhân, cách thử, lý do chọn cấu hình** (chi tiết: `docs/retrieval.md` mục 5, `docs/agent.md` mục 5):

1. **d09** ("minimum number of people needed to form a group"; gold `B-soc3e-p0161-001`). Dense, BM25, hybrid và rerank pool 20 đều trượt. Nguyên nhân: lệch từ vựng ("minimum number needed" so với "at least two people"), các chunk mục 6.2 về kích thước nhóm lấn át; gold ở hạng 12 (dense), 24 (BM25) và ngay ngoài pool 20 sau fusion. Cách thử: pool rerank 20 -> 30. Trước/sau: Evidence Hit@3 15/16 -> 16/16 (0.938 -> 1.000), MRR 0.906 -> 0.969, đánh đổi chậm khoảng 20 lần (khoảng 1 s/query).
2. **d07** ("What is anticipatory socialization?"; gold `B-soc3e-p0147-003` ở hạng 2). Hạng 1 là chunk glossary "Key Terms" `B-soc3e-p0150-000` (định nghĩa ngắn, cross-encoder ưa hơn, không có ngữ cảnh section). Cách thử: demote chunk back-matter theo `document_type`. Trước/sau: Hit@1 0.938 -> 1.000 (15/16 -> 16/16), MRR 0.969 -> 1.000. Không demote thì hybrid không đổi. Rủi ro: một sự kiện chỉ có trong glossary bị đẩy xuống (vẫn truy xuất được vì chỉ xếp lại, không lọc).
3. Lỗi phụ trong agent: validator citation quá chặt (F1/E1, d10 và d13) và tìm kiếm thừa trước khi từ chối (E2); xem mục Agent.

**Lý do chọn cấu hình.** Tiêu chí khai báo trước khi chạy dev (`docs/retrieval.md` mục 0; `docs/agent.md` mục 0): chất lượng trước (answer grade, citation, abstention, rồi Context Evidence Rate), sau đó chi phí token/tool call và độ trễ; hiệu một câu (1/16) coi là nhiễu. Trên dev 7 cấu hình agent hòa hoàn toàn ở tiêu chí 1-2 (16/16 mọi chỉ số), nên chỉ còn chi phí quyết định. **Quy tắc tie-break "chọn token thấp nhất, trừ cấu hình chỉ 1 tool call" được thêm sau khi thấy các cấu hình hòa (post-hoc, ghi rõ ở `docs/agent.md` mục 0b).** `rerank_one` bị loại theo quy tắc đó; `rerank_small` rẻ hơn 7% (3613 so với 3865 token) nhưng tôi không chọn vì tiết kiệm nhờ cắt context còn 3 chunk (chỉ vô hại trên dev), vẫn còn lỗi E1 và chênh lệch nhỏ hơn nhiễu; đây là lệch nhẹ so với cách đọc đúng chữ tiêu chí 3, đã gắn cờ trong `docs/agent.md` mục 7. Chọn `rerank_fix` đóng băng thành `configs/final.json`. Giữ rerank dù trên dev `base` (hybrid không rerank) đã đạt như nhau là quyết định phòng ngừa rủi ro, không phải lợi ích đo được.

## Agent và đánh giá

- Tool, số lượt gọi, điều kiện dừng, cách chọn context và xử lý lỗi (`docs/agent.md` mục 2, `src/agent.py`):
  - Tool: `search(query, top_k 1-8, chapter 5/6/7 tùy chọn, filter_reason)` dùng phương pháp retrieval cố định theo config (model chỉ chọn query, k, chapter; `filter_reason` được log), và `finish(abstain, answer, citations, evidence_quote)`. `get_neighbors` chỉ có ở config thử `rerank_nb`, không dùng ở `final`.
  - Số lượt gọi: `final` tối đa 2 tool call (một search + một lần viết lại hoặc tìm sự kiện thứ hai); `base` tối đa 3. Hết giới hạn thì ép buộc gọi `finish` (`tool_config mode ANY`).
  - Điều kiện dừng: model gọi `finish` (trả lời hoặc `abstain=true`). Văn bản thuần không có `finish` bị nhắc một lần.
  - Chọn context: các chunk của mỗi lượt xếp theo hạng liên quan, khử trùng theo id, ngân sách tổng 8000 ký tự văn bản chunk (chunk không vừa bị bỏ và ghi `budget_dropped`), mỗi chunk giữ id/trang/section/score; context model thấy = `final_context` trong trace, nên citation chỉ hợp lệ nếu chunk thực sự được đưa vào. Không nén hay tóm tắt. Prompt coi nội dung chunk là dữ liệu không đáng tin, không phải chỉ thị.
  - Xử lý lỗi: validator chỉ nhận `finish` khi có câu trả lời, citations không rỗng, mọi id nằm trong context và xuất hiện trong ngoặc vuông ở câu trả lời (`final` chấp nhận ngoặc nhóm `[id1, id2]`); sai thì cho một lần sửa, sai lần hai thì chuyển thành abstain `invalid_citations`. Lỗi provider: retry/backoff cho 429/5xx, phát hiện hết quota ngày và dừng với `status: "error"`; các lần chạy báo cáo có 0 lỗi, nên cơ chế này chỉ kích hoạt ở lần chạy bị hủy. Retrieval lỗi raise `RetrievalError`, không trả kết quả giả.

Bảng tổng hợp (nhãn thủ công bởi AI; `results/agent/{dev,test}/<config>/summary.json`, `reviews.json`; đã đối chiếu `results/agent/{dev,test}/comparison.md`). "Dev `final`" là cấu hình dev `rerank_fix` (cùng nội dung `configs/final.json`). Context Evidence Rate (gold trong context cuối): 16/16 dev cả hai, 8/8 test cả hai. Thời gian = thời gian hoạt động (LLM + tool, không tính 6.5 s nghỉ giữa các lần gọi) median, wall-clock xem `comparison.md`.

| Cấu hình / split | Tool use | Số lượt gọi | Grounded answer 0/1/2 và trung bình | Citation hợp lệ | Câu có citation | Abstention đúng | Từ chối nhầm |
|---|---|---|---|---|---|---|---|
| `base` / dev | 19/19 | 1.32 /câu (1 lượt: 16 câu; 3 lượt: 3 câu); 4349 token/câu; active 5.4 s | 0/0/16, TB 2.000 | 19/19 | 16/16 | 3/3 | 0/16 |
| `final` (= `rerank_fix`) / dev | 19/19 | 1.21 /câu (1 lượt: 15; 2 lượt: 4); 3865 token/câu; active 5.2 s | 0/0/16, TB 2.000 | 18/18 | 16/16 | 3/3 | 0/16 |
| `base` / test | 11/11 | 1.82 /câu (1/2/3 lượt = 5/3/3); 5697 token/câu; active 7.1 s (p95 22.2) | 0/0/8, TB 2.000 | 9/9 | 8/8 | 3/3 | 0/8 |
| `final` / test | 11/11 | 1.36 /câu (1/2/3 lượt = 7/4/0); 4063 token/câu; active 9.5 s (p95 21.2) | 0/0/8, TB 2.000 | 9/9 | 8/8 | 3/3 | 0/8 |

Per-query test của cấu hình `final` (`results/agent/test/final/traces.json`, nhãn trong `reviews.json`). Hầu hết câu hỏi chỉ cần 1 search nên tool call chủ yếu là một query; "base x" là số lượt của `base`; `base` trích dẫn cùng chunk và trả lời tương đương trên mọi câu (`docs/test_eval.md`). Hạng = hạng của chunk gold trong kết quả search. Citation viết tắt `p0133-004` = `B-soc3e-p0133-004`.

| Query ID | Loại câu | Tool call và kết quả | Câu trả lời và citation | Nhãn, lý do |
|---|---|---|---|---|
| t01 | Có đáp án | 1 (base 1): "Erik Erikson stages development", chapter 5; gold p0133-004 hạng 1 | Erikson có 8 giai đoạn, từ sinh đến chết [p0133-004] | Answer 2, citation đúng, context 2: chunk nêu nguyên văn |
| t02 | Có đáp án | 1 (base 2): "Emile Durkheim suicide rates differences", không filter; gold p0134-002 hạng 1 | Durkheim: nguyên nhân xã hội (khác biệt tôn giáo), không phải tâm lý [p0134-002] | 2, đúng, ctx 2. Query đầu của `base` trượt gold, `final` trúng ngay |
| t03 | Có đáp án | 1 (base 2): "reunited twins nature nurture", chapter 5; p0137-001 hạng 1, p0137-002 hạng 3 | Elyse Schein và Paula Bernstein, cặp sinh đôi gặp lại [p0137-001, p0137-002] | 2, cả hai citation đúng, ctx 2: 001 = nghiên cứu sinh đôi và chia tách 1968, 002 = tên và gặp lại 2003 |
| t04 | Có đáp án | 1 (base 1): "peer group definition"; gold p0141-000 hạng 1 | Peer group: tuổi và địa vị xã hội tương đồng, chung sở thích [p0141-000] | 2, đúng, ctx 2: định nghĩa nguyên văn |
| t05 | Có đáp án | 2 (base 3): "McDonaldization of society" rồi "George Ritzer"; gold p0174-000 hạng 1 cả hai lần | McDonaldization gắn với Ritzer (1993), mô hình thức ăn nhanh lan vào các thiết chế [p0174-000] | 2, đúng, ctx 2; lượt 2 là thừa (gold đã hạng 1) |
| t06 | Có đáp án | 1 (base 1): "Edwin Sutherland deviant behavior theory", chapter 7; p0196-001 hạng 1 | Sutherland: differential association theory [p0196-001] | 2, đúng, ctx 2 |
| t07 | Có đáp án | 1 (base 1): "difference between prison and jail", chapter 7; p0203-002 hạng 1 | Jail: giam tạm, nhỏ, địa phương; prison: lớn, án trên 1 năm, bang hoặc liên bang [p0203-002] | 2, đúng, ctx 2: đủ mọi vế trong chunk |
| t08 (multi-hop) | Có đáp án, cần cả 2 chunk | 1 (base 1): "Uniform Crime Reports National Crime Victimization Survey higher crime rate why", chapter 7; p0200-001 hạng 1, p0201-000 hạng 2 | NCVS cao hơn UCR vì ghi nhận cả tội phạm không báo cảnh sát (từ "likely" giữ nguyên) [p0201-000] | 2, đúng, ctx 2 (cả hai chunk trong context). Câu trả lời chỉ cần 0201-000 nên multi-hop ít được thử |
| t09 | Thiếu evidence | 2 (base 3): "Zimbardo Stanford prison experiment participants" rồi "Zimbardo", không citation | Từ chối: không đủ bằng chứng về thí nghiệm nhà tù Stanford | Abstention đúng, không khẳng định thừa; gold: không được nhắc trong sách |
| t10 | Thiếu evidence | 2 (base 3): "death penalty states" rồi "death penalty capital punishment", chapter 7 | Từ chối: không đủ bằng chứng về số bang còn dùng án tử hình | Abstention đúng |
| t11 | Thiếu evidence (hard negative) | 2 (base 2): "Egalia preschool Sweden children" rồi "... number children how many", chapter 5 | Từ chối: không có số trẻ học ở Egalia | Abstention đúng: Egalia có ở trang 136 nhưng không có số liệu |

Context 0/1/2 trên test (8 câu có đáp án; đọc context cuối trong `reviews.json`, trường `context_grade`, `context_reason`):

| Cấu hình | Mức 0 | Mức 1 | Mức 2 | Lý do |
|---|---|---|---|---|
| Baseline (`base`) | 0 | 0 | 8 | Chunk gold (kèm trang, section) có trong context cuối và đủ để trả lời; t08 có cả p0200-001 và p0201-000 |
| Cấu hình chọn (`final`) | 0 | 0 | 8 | Như trên; t03 cả p0137-001 (khung nghiên cứu sinh đôi) và p0137-002 (tên) đều trong context |

- Đường dẫn trace, nhãn, lệnh chạy, thời gian và chi phí: trace `results/agent/{dev,test}/<config>/traces.json` (model, tool call, tham số, kết quả, context, câu trả lời, citation, token, latency, lỗi); nhãn `reviews.json`; tổng hợp `summary.json`, `comparison.md`; retrieval `results/retrieval/{dev,test}/<config>/`. Lệnh (từ thư mục gốc, `.venv/Scripts/python.exe`, `PYTHONPATH=.`):
  - Cài đặt và index: `python -m venv .venv`, `pip install -r requirements.txt`, `python scripts/build_index.py` (thêm `--rebuild` để dựng lại; `python scripts/verify_index.py` cần repo Day 6); benchmark: `python scripts/build_benchmark.py`. Cần `GEMINI_API_KEY` trong `.env` (không đưa vào repo).
  - Retrieval: `python scripts/eval_retrieval.py --split dev --configs dense_k5,bm25_k5,hybrid_k5,...`; test: `python scripts/eval_retrieval.py --split test --allow-test --configs dense_k5,bm25_k5,hybrid_k5,hybrid_rerank_k5,imp6_hybrid_rerank_pool30_demote_k5`.
  - Agent: `python scripts/run_agent.py --split dev --config <base|rerank|rerank_nb|rerank_strict|rerank_small|rerank_one|rerank_fix>`; test: `python scripts/run_agent.py --split test --config final --allow-test` rồi `--config base --allow-test`; in nội dung để chấm tay: `python scripts/review_dump.py test final,base t01,...,t11`; tổng hợp: `python scripts/eval_agent.py --split test --configs final,base --allow-test`.
  - Unit test: `python -m unittest discover -s tests -v` (mock chỉ kiểm code; 44 test pass).
  - Thời gian và chi phí: một lần chạy test hai cấu hình khoảng 6 phút (kể cả 6.5 s nghỉ giữa các lần gọi để giữ free-tier); mỗi config dev khoảng 19 truy vấn, tổng khoảng 330 lệnh gọi LLM. Chi phí tiền 0 USD (free tier); token/câu: dev 3865-5095, test 4063 (`final`) và 5697 (`base`). Wall-clock median test: 31.4 s (`final`) và 26.1 s (`base`), chủ yếu do pacing.
- Kết quả trước/sau, ví dụ lỗi, đánh đổi và giới hạn:
  - Lỗi dev của agent. **E1** (`rerank`, d13): model viết `[id1, id2]` trong một ngoặc, validator tìm `[<id>]`, sinh `citation_check_failed` và thêm một lần gọi LLM (+2.3k token), có thể thành từ chối nhầm; sửa bằng `lenient_citation_format`, vòng sửa 2 -> 0, citation vẫn 18/18 hỗ trợ. **E2** (d17-d19, mọi cấu hình cap 3): lặp search cùng chunk trước khi từ chối (d18: "recidivism" có và không filter trả cùng danh sách), 8.5-9.4k token; hạ cap 3 -> 2 giảm token câu thiếu evidence 8920 -> 5596 (-37%), vẫn 3/3 abstention. Lỗi nhẹ **E3**: model trả văn bản thuần thay vì `finish` (d02 trong `rerank_fix`), cần một lần nhắc; chưa sửa.
  - Kết quả bão hòa: trên dev mọi cấu hình agent đều 16/16 context, grade trung bình 2.000, 3/3 abstention nên dev không phân biệt được cấu hình về chất lượng, chỉ phân biệt về chi phí. Test cũng vậy (8/8, 2.000, 9/9, 3/3 cho cả `base` và `final`); khác biệt thật duy nhất là hiệu quả: `final` dùng 1.36 so với 1.82 tool call/câu và khoảng 29% ít token hơn (4063 so với 5697), không cần lượt gọi thứ ba. Thời gian hoạt động median của `final` (9.5 s) lại cao hơn `base` (7.1 s) do rerank (khoảng 1 s mỗi search) và biến động API, nên không kết luận về độ trễ (nhiễu, sai khác dưới hệ số 2 không đáng tin).
  - Dev sang test: Hit@1 của cấu hình chọn giảm từ 1.000 (dev, cấu hình được chọn trên chính dev nên lạc quan) xuống 0.875 (7/8) trên test; Hit@3 vẫn 8/8.
  - t03: với query gốc là câu hỏi benchmark, mọi phương pháp không rerank (dense, BM25, hybrid) đều trượt t03 trong top-5, `hybrid_rerank` pool 20 phục hồi ở top-5 nhưng chỉ cấu hình chọn có Hit@3 8/8. Tuy vậy điều đó không làm tăng grade vì agent tự viết lại query ("reunited twins nature nurture") nên tìm được cả hai chunk (context 8/8 cho cả hai cấu hình). Tương tự trên dev, viết lại query của LLM đã tự khắc phục lỗi d09 vốn là lý do dùng rerank. Do đó metric retrieval trên câu hỏi gốc và metric context của agent đo hai thứ khác nhau, và lợi ích đo được của rerank trong agent là bằng 0 trên cả dev và test.
  - Đánh đổi: rerank thêm khoảng 1 s CPU mỗi search (so với khoảng 25-50 ms của hybrid) và agent cần 2-3 lệnh gọi LLM, khoảng 4-5.7k token mỗi câu so với 0 token cho retrieval thuần; abstention không thể dựa vào ngưỡng score (score top-1 của câu thiếu evidence chồng lấn với câu có đáp án trên cả dev lẫn test, `comparison.md`), nên agent phải đọc văn bản.
  - Giới hạn: N rất nhỏ (8 + 3 câu test, 16 + 3 câu dev; một câu = 6.25 điểm phần trăm), kết quả chạm trần nên không xếp hạng được cấu hình về độ chính xác; **chỉ một người chấm, là AI (Claude), chưa có người chấm hai hay chấm kiểm tra độc lập; điểm tuyệt đối là đáng ngờ và sinh viên cần chấm lại mẫu (d09, d10, d13, d15, t03, t08 và một câu abstention)**; phần lớn câu hỏi chỉ cần một chunk, multi-hop gần như chưa được thử (t08 trả lời được chỉ với một chunk); tiêu chí tie-break là post-hoc; các số liệu thời gian nhiễu (provider, pacing 6.5 s, retrieval đo lúc agent đang chạy); model `gemini-3.5-flash-lite` không phải model mặc định của đề và có thể bị gỡ như dòng 2.x. Corpus chỉ có 69 trang một tài liệu tiếng Anh.

## Thử thách nâng cao (nếu thực hiện)

Không thực hiện thử thách nâng cao để lấy điểm bonus: corpus chỉ có 69 trang, không đủ 100 trang, và không có benchmark bảng/ảnh/biểu đồ. Trong repo có thêm giao diện web và công cụ nạp PDF văn bản (`app/`, `scripts/ingest_pdf.py`, `scripts/build_library.py`) như phần mở rộng, nhưng không có benchmark, không dùng cho kết quả ở trên và không tính vào ứng cử bonus.

- Tài liệu, nguồn, quyền sử dụng, số trang và phiên bản: Không thực hiện.
- Cách xử lý text, bảng, ảnh, biểu đồ; phạm vi index và phần chưa xử lý: Không thực hiện.
- Cách chia benchmark, schema evidence, model và cấu hình đã chốt trên dev: Không thực hiện.

| Nhóm test | Đạt / tổng | Tỷ lệ | Lỗi / nhận xét |
|---|---|---|---|
| Bảng | — | — | Không thực hiện |
| Ảnh, sơ đồ | — | — | Không thực hiện |
| Biểu đồ | — | — | Không thực hiện |
| Multi-hop | — | — | Không thực hiện |
| Text | — | — | Không thực hiện |
| Thiếu bằng chứng | — | — | Không thực hiện |
| **Toàn bộ** | — | — | Không thực hiện |

- Trace, nhãn và evidence từng câu; thời gian, chi phí, lỗi hệ thống: Không thực hiện.
- Giới hạn, cách tái lập và đường dẫn benchmark: Không thực hiện. (Benchmark phần cơ bản: `benchmark/questions.json`, `benchmark/gold/gold.json`.)

## Tái lập và lưu ý môi trường

- Python 3.12 cài đặt trước đó bị hỏng, venv `.venv` được dựng lại trên Python 3.14.7; 44 unit test pass (`python -m unittest discover -s tests -v`).
- Cần `GEMINI_API_KEY` trong `.env` (không commit); embedding và rerank chạy local, tải model từ Hugging Face lần đầu (khoảng 40 s cho cross-encoder).
- Quy trình: chọn và chỉnh trên dev, đóng băng `configs/final.json` (2026-11-16), rồi mới chạy test một lần bằng `--allow-test`; test không được dùng để chỉnh, do đó đây là đánh giá độc lập theo BENCHMARK.md (nhưng chỉ với N nhỏ như nêu trên).
