# Arabic Document Intelligence

نظام بيستقبل **PDF / صورة / Scanned document** ويخلّيك تسأله بالعربي أو الإنجليزي: تلخيص، أسئلة، واستخراج تواريخ وأسماء وأرقام.

```
Upload ─► Detect (text layer? / scanned?)
            │                    │
       PyMuPDF text         OpenCV preprocess ─► Tesseract (ara+eng) ─► [conf منخفضة] Vision-LLM fallback
            └──────────┬─────────┘
                  Arabic Cleaning (NFKC, reversed-fix, whitespace)
                       ▼
        Structure-aware Chunking (+ page metadata)
                       ▼
        BGE-M3 Embeddings ─► Chroma     +     BM25 (normalized Arabic)
                       ▼
        Hybrid Retrieval (RRF) ─► Reranker (اختياري)
                       ▼
   Query Router ─► qa | summary (map-reduce) | purpose | extract (JSON schema + regex)
                       ▼
                      LLM  ─► إجابة + citations بأرقام الصفحات
```

## التشغيل

```bash
# system deps (Ubuntu): sudo apt install tesseract-ocr tesseract-ocr-ara
pip install -r requirements.txt
cp .env.example .env        # حط ANTHROPIC_API_KEY
export $(grep -v '^#' .env | xargs)

uvicorn app.main:app --reload          # API  → http://localhost:8000/docs
streamlit run ui/streamlit_app.py      # UI   → http://localhost:8501
```
أو Docker: `docker compose up --build`

## API

| Method | Path | وصف |
|---|---|---|
| POST | `/documents` | رفع ملف → `{doc_id, pages, chunks, sources}` |
| GET | `/documents/{id}/pages` | النص المستخرج لكل صفحة (مصدره + OCR confidence) |
| POST | `/documents/{id}/ask` | `{"question": "...", "route": null}` |

```bash
curl -F file=@contract.pdf localhost:8000/documents
curl -X POST localhost:8000/documents/<id>/ask -H 'content-type: application/json' \
     -d '{"question":"استخرج التواريخ والأسماء والأرقام المهمة."}'
```

## هيكل الكود

| ملف | وظيفته |
|---|---|
| `app/ingestion.py` | اكتشاف النوع، text layer vs OCR، كشف الـ encoding البايظ، OCR fallback |
| `app/ocr.py` | deskew + denoise + binarize + Tesseract مع confidence |
| `app/arabic_clean.py` | NFKC، تطويل، أرقام هندية، لمّ الأسطر، كشف/إصلاح النص المعكوس |
| `app/chunking.py` | فقرات → جمل، overlap، `page_start/page_end` لكل chunk |
| `app/retrieval.py` | dense + BM25 + RRF + reranker |
| `app/router.py` | تحديد نوع السؤال |
| `app/tasks.py` | الـ 4 strategies |
| `eval/` | CER/WER، Recall@k، MRR، مقارنة بدون/مع reranker |

## قرارات تصميم تذكرها في التقرير

1. **النص الأصلي vs المُطبَّع**: الـ chunks بتتخزن بالنص الأصلي (للعرض والـ citations)، والـ normalization (أ/إ/آ، ى/ي، ة/ه، تشكيل) بيتطبق بس وقت الـ BM25.
2. **Router**: "لخص" بتتعمل map-reduce على المستند كله، مش top-k، لأن الـ retrieval هيضيّع أجزاء. الاستخراج بيجمع LLM (schema) + regex للأنماط الثابتة.
3. **OCR fallback**: لو متوسط confidence أقل من `OCR_MIN_CONF` الصفحة بتروح لـ vision LLM (بيتفعّل بس لو فيه API key).
4. **Per-page detection**: PDF ممكن يبقى نصفه text ونصفه scanned، فالقرار بيتاخد لكل صفحة.

## Evaluation

```bash
# جهّز 30-50 سؤال: {"question": "...", "gold_page": N} (شوف eval/questions.example.jsonl)
python -m eval.run_retrieval <doc_id> eval/questions.jsonl
```
للـ OCR استخدم `eval.metrics.cer / wer` على صفحات عندك ground-truth ليها، وقارن Tesseract بـ preprocessing وبدونه وبالـ vision fallback.

```bash
pytest -q    # tests للـ cleaning / chunking / router / metrics
```

## Roadmap
- Surya / PaddleOCR كمحركات بديلة للمقارنة مع Tesseract
- Heading detection في الـ chunking
- LLM-based router للأسئلة اللي مش بتطابق الـ rules
- Faithfulness eval (LLM-as-judge) على الإجابات
- Multi-document collections
