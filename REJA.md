# Reja: O'zbekiston Soliq kodeksi bo'yicha Telegram bot (RAG)

> Vizual (chiroyli) ko'rinishi uchun `reja.html` faylini brauzerda oching.

## Kontekst (nima uchun va nima qilamiz)

Foydalanuvchi odamlar va kompaniyalar O'zbekiston Respublikasi Soliq
kodeksi bo'yicha savol berganda — oddiy savol bo'lsin yoki murakkab
keys/holat bo'lsin — kodeks asosida javob beradigan Telegram bot
yaratmoqchi.

**Muhim qaror — fine-tuning EMAS, RAG:** Foydalanuvchi modelni "train
qilish" kerak deb o'ylaydi. Aslida modelni qayta o'qitish (fine-tuning)
qimmat, qiyin va xato (hallyutsinatsiya) beradi. Buning o'rniga eng oson,
arzon va aniq yo'l — **RAG (Retrieval-Augmented Generation)**:

1. Soliq kodeksi matnini moddalarga bo'lib bazaga (vektor bazaga) saqlaymiz.
2. Foydalanuvchi savol berganda, savolga eng mos moddalarni bazadan topamiz.
3. Topilgan moddalarni Claude modeliga beramiz va u shu moddalar asosida
   javob yozadi, qaysi moddaga tayanganini ko'rsatib.

Bu yondashuvning afzalliklari: model o'ylab topib gapirmaydi (faqat
berilgan moddaga tayanadi), javobda modda raqami ko'rsatiladi (foydalanuvchi
tekshira oladi), kodeks o'zgarsa — faqat matnni yangilaymiz, modelni qayta
o'qitish shart emas.

### Foydalanuvchi tanlovlari (kelishilgan)
- **Kodeks matni manbasi:** lex.uz (saytdan olib, faylga saqlaymiz).
- **Til:** O'zbek + Rus (foydalanuvchi qaysi tilda yozsa, shu tilda javob).
- **Hosting:** Eng arzon / oddiy boshlash (avval o'z kompyuterda, keyin server).

---

## Texnik arxitektura (umumiy ko'rinish)

```
lex.uz matni  ──►  ingest.py (bir marta ishlaydi)
                   - matnni moddalarga bo'ladi (o'zbek + rus)
                   - har modda uchun embedding (vektor) yaratadi
                   - ChromaDB ga saqlaydi (kompyuterda, disk faylida)

Foydalanuvchi  ──►  bot.py (doimiy ishlaydi)
(Telegram)          - savol tilini aniqlaydi (o'zbek / rus)
                    - savolga eng mos 5-8 moddani ChromaDB dan topadi
                    - moddalar + savolni Claude ga yuboradi
                    - Claude javobni modda raqamlari bilan qaytaradi
                    - javob foydalanuvchiga Telegram orqali boradi
```

### Texnologiyalar (barchasi oddiy va arzon)
- **Til:** Python 3.11+
- **Telegram:** `python-telegram-bot` (bepul, kengtarqalgan)
- **AI model:** Anthropic Claude — `anthropic` SDK
  - Default: `claude-sonnet-4-6` (arzon, ikki tilda yaxshi, yuridik
    savollar uchun yetarli). Murakkab keyslar uchun `claude-opus-4-8`
    ga o'tish mumkin — model nomi `.env` da sozlanadi.
- **Vektor baza:** `chromadb` — kompyuterda fayl sifatida saqlanadi,
  alohida server kerak emas, bepul.
- **Embedding (vektor) modeli:** `sentence-transformers` orqali lokal,
  ko'p tilli model `intfloat/multilingual-e5-base` — **bepul**, o'zbek va
  rus tillarini qo'llab-quvvatlaydi, faqat CPU da ishlaydi. Bu embedding
  uchun hech qanday pul to'lanmaydi; faqat Claude javoblariga pul ketadi.

### Nima uchun xarajat juda kam
- Embedding: bepul (lokal model).
- Vektor baza: bepul (lokal Chroma).
- Telegram bot: bepul.
- Yagona xarajat: Claude API — har savolga taxminan bir nechta sent
  (sonnet bilan). Hosting: avval o'z kompyuter (bepul), keyin arzon VPS.

---

## Bosqichma-bosqich amalga oshirish rejasi

### 1-bosqich: Tayyorgarlik (kalit va token olish)
- **Anthropic API kaliti:** console.anthropic.com da hisob ochib, API key
  olinadi → `.env` ga `ANTHROPIC_API_KEY` sifatida yoziladi.
- **Telegram bot tokeni:** Telegramda `@BotFather` ga yozib, `/newbot`
  buyrug'i bilan yangi bot yaratiladi → token olinadi → `.env` ga
  `TELEGRAM_BOT_TOKEN` sifatida yoziladi.

### 2-bosqich: Loyiha skeletini yaratish
Quyidagi fayllar yaratiladi (repozitoriy hozir bo'sh):
- `requirements.txt` — kutubxonalar: `anthropic`, `python-telegram-bot`,
  `chromadb`, `sentence-transformers`, `python-dotenv`, `beautifulsoup4`,
  `requests`, `langdetect`.
- `.env.example` — kalitlar uchun namuna (`ANTHROPIC_API_KEY`,
  `TELEGRAM_BOT_TOKEN`, `CLAUDE_MODEL=claude-sonnet-4-6`).
- `.gitignore` — `.env`, `chroma_db/`, `data/`, `__pycache__/` ni
  git ga tushirmaslik uchun (kalitlar maxfiy qolishi shart).
- `README.md` — o'zbekcha oddiy qo'llanma: o'rnatish va ishga tushirish.

### 3-bosqich: Kodeks matnini olish (lex.uz)
`fetch_taxcode.py` skripti yoziladi:
- Soliq kodeksining **o'zbekcha** va **ruscha** versiyalarini lex.uz dan
  oladi (ikki alohida hujjat). lex.uz JavaScript ishlatgani uchun matnni
  olishda ikki yo'l ko'rsatiladi:
  1. **Avtomatik:** `requests` + `beautifulsoup4` bilan sahifa(lar)ni
     yuklab, matnni ajratib olish.
  2. **Qo'lda (zaxira yo'l):** agar avtomatik ishlamasa, hujjatni lex.uz
     dan Word/PDF ko'rinishida yuklab, `data/uz.txt` va `data/ru.txt`
     fayllariga saqlash. Reja shu zaxira yo'lni ham qo'llab-quvvatlaydi.
- Natija: `data/uz.txt` (o'zbekcha to'liq kodeks) va `data/ru.txt`
  (ruscha to'liq kodeks).
- **Eslatma:** Soliq kodeksi vaqti-vaqti bilan o'zgaradi — matnni
  yangilab turish kerak (bu skriptni qayta ishga tushirish bilan).

### 4-bosqich: Matnni bazaga yuklash (`ingest.py`)
- `data/uz.txt` va `data/ru.txt` ni **moddalarga** (modda / статья) bo'ladi
  — har bir modda alohida bo'lak (chunk) bo'ladi. Modda — eng tabiiy
  bo'linish birligi, chunki javoblar modda raqami bilan beriladi.
- Juda uzun moddalar qism-qismga bo'linadi (taxminan 800-1000 token).
- Har bo'lakka metadata: `{lang: "uz"/"ru", article_no: "183", title: ...}`.
- `intfloat/multilingual-e5-base` bilan embedding yaratiladi.
- ChromaDB ga (`chroma_db/` papkasiga) saqlanadi. Bu **bir marta**
  ishlaydi; matn yangilanganda qayta ishga tushiriladi.

### 5-bosqich: Bot logikasi (`bot.py`)
- `python-telegram-bot` bilan xabarlarni qabul qiladi.
- `/start` buyrug'i — qisqacha tanishtirish + ogohlantirish (bot
  ma'lumot beradi, lekin rasmiy yuridik maslahat o'rnini bosmaydi).
- Foydalanuvchi savol yozganda:
  1. `langdetect` bilan til aniqlanadi (o'zbek / rus).
  2. Savol embedding ga aylantiriladi, ChromaDB dan **shu tildagi** eng
     mos 5-8 modda topiladi (metadata `lang` bo'yicha filtrlash).
  3. Tizim promti + topilgan moddalar + savol Claude ga yuboriladi.
     Tizim promtida: "Faqat berilgan moddalarga tayan, har bir da'voda
     modda raqamini ko'rsat, agar moddalar yetarli bo'lmasa ochiq ayt"
     degan ko'rsatma bo'ladi (hallyutsinatsiyani oldini olish).
  4. Claude javobi modda raqamlari bilan foydalanuvchiga yuboriladi.
- Uzun javoblar uchun Claude `stream` rejimida chaqiriladi (timeout
  bo'lmasligi uchun).

### 6-bosqich: Sinash va ishga tushirish
- Lokalda: `python ingest.py` (bir marta) → `python bot.py` (botni yoqadi).
- Telegramda botga test savollar beriladi (o'zbekcha va ruscha), javoblar
  va modda iqtiboslari tekshiriladi.
- Keyinchalik doimiy ishlashi uchun arzon VPS (masalan, oddiy Linux
  server) yoki shu kabi joyga ko'chiriladi — bu alohida, keyingi qadam.

---

## Asosiy fayllar (yaratiladi)
- `requirements.txt`, `.env.example`, `.gitignore`, `README.md`
- `fetch_taxcode.py` — lex.uz dan matn olish
- `ingest.py` — matnni moddalarga bo'lib ChromaDB ga yuklash
- `bot.py` — Telegram bot + RAG + Claude javob logikasi
- `rag.py` (ixtiyoriy) — qidiruv va prompt yig'ish logikasi (toza kod uchun)

## Muhim yuridik/amaliy nuqtalar
- **Har doim modda raqamini iqtibos qilish** — foydalanuvchi tekshira olishi uchun.
- **Ogohlantirish (disclaimer):** bot ma'lumot beradi, rasmiy yuridik
  xulosa emas — murakkab holatlarda mutaxassisga murojaat tavsiya etiladi.
- **Yangilab turish:** kodeks o'zgarsa, `fetch_taxcode.py` + `ingest.py`
  qayta ishga tushiriladi.
- **Maxfiylik:** API kalitlar `.env` da, hech qachon git ga tushmaydi.

## Tekshirish (verification)
1. `python ingest.py` ishlaydi va `chroma_db/` da moddalar soni kutilganday
   (kodeksdagi moddalar soniga yaqin) ekanini chiqaradi.
2. `python bot.py` ishga tushadi, xatosiz Telegramga ulanadi.
3. Telegramda 4-5 ta test savol (2 ta o'zbekcha, 2 ta ruscha, 1 ta murakkab
   keys) — har javobda to'g'ri modda iqtibos qilinganini qo'lda tekshirish.
4. Mavjud bo'lmagan mavzu so'ralganda bot "kodeksda topilmadi" deb to'g'ri
   javob berishini tekshirish (hallyutsinatsiya yo'qligi).
