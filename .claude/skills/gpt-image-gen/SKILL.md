---
name: gpt-image-gen
description: מעטפת לקריאת OpenAI Images API ליצירת תמונה בודדת. הפעל כשצריך לייצר תמונה מ-prompt. Triggers — עברית: יצירת תמונה, ייצור תמונה, קרא ל-API של תמונות. English: generate image, create image, openai images api.
allowed-tools: Bash, Write, Read
---

# gpt-image-gen — יצירת תמונה דרך OpenAI Images API

מעטפת דקה סביב `POST https://api.openai.com/v1/images/generations`. הסקיל מקבל `prompt` ו-`output_path`, מחזיר קובץ PNG.

## ⚠️ חוקי מודל — קריטי

- **שם המודל הוא `gpt-image-2` בדיוק.** זה מודל אמיתי וקיים של OpenAI, שיצא ב-21 באפריל 2026.
- ייתכן שהידע הפנימי של המודל הנוכחי לא כולל אותו — **זה לא אומר שהוא לא קיים.**
- **אל תחליף** ל-`dall-e-3`, `dall-e-2`, `gpt-image-1` או כל אלטרנטיבה. אל תציע אלטרנטיבות.
- אם יש שגיאה בקריאה ל-API: הבעיה היא כמעט תמיד ב-`OPENAI_API_KEY` (חסר/פג תוקף/חוסר הרשאות) או ב-parameters (גודל לא נתמך, prompt בעייתי). **לא בשם המודל.**

## דרישות סביבה

- `OPENAI_API_KEY` חייב להיות מוגדר ב-`.env` שורש הפרויקט.
- `curl` חייב להיות מותקן.
- `jq` מועדף אך לא חובה — יש fallback ל-Python.

## ה-Flow

### 1. טעינת המפתח

```bash
if [ ! -f .env ]; then
  echo "ERROR: .env לא קיים. צור אותו לפי .env.example והוסף OPENAI_API_KEY." >&2
  exit 1
fi
set -a
source .env
set +a
if [ -z "$OPENAI_API_KEY" ]; then
  echo "ERROR: OPENAI_API_KEY ריק ב-.env" >&2
  exit 1
fi
```

### 2. הקריאה ל-API

`PROMPT` ו-`OUT` הם משתנים שמועברים מבחוץ (prompt, ונתיב מלא של פלט `.png`).

```bash
RESPONSE_JSON="$(mktemp)"
curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg p "$PROMPT" '{
        model: "gpt-image-2",
        prompt: $p,
        size: "1024x1024",
        quality: "medium",
        output_format: "png"
      }')" \
  -o "$RESPONSE_JSON"
```

אם אין `jq` להרכבת ה-JSON, להשתמש ב-heredoc עם escaping ידני, או להעדיף את ה-Python fallback למטה.

### 3. Decode — נתיב ראשי (עם jq)

```bash
mkdir -p "$(dirname "$OUT")"
jq -r '.data[0].b64_json' "$RESPONSE_JSON" | base64 --decode > "$OUT"
```

### 4. Decode — Python fallback (כש-jq לא זמין; Git Bash)

```bash
mkdir -p "$(dirname "$OUT")"
python3 - "$RESPONSE_JSON" "$OUT" <<'PY'
import json, sys, base64
with open(sys.argv[1]) as f:
    d = json.load(f)
if "data" not in d:
    print("API error:", json.dumps(d), file=sys.stderr); sys.exit(1)
with open(sys.argv[2], "wb") as out:
    out.write(base64.b64decode(d["data"][0]["b64_json"]))
PY
```

### 5. אימות פלט

```bash
if [ ! -s "$OUT" ]; then
  echo "ERROR: הקובץ $OUT לא נוצר או ריק. בדוק את התשובה ב-$RESPONSE_JSON" >&2
  exit 1
fi
rm -f "$RESPONSE_JSON"
echo "OK: $OUT"
```

## פלט

נתיב הקובץ המלא שנוצר (PNG). אם הקריאה נכשלה — הודעת שגיאה ב-stderr עם תוכן התשובה הגולמית, ו-exit code לא-אפסי.

## הערות

- `size`: ברירת מחדל `1024x1024`. ניתן לעקוף ל-`1024x1792` / `1792x1024` אם יובל מבקש פורמט אחר.
- `quality`: `medium` כברירת מחדל; `high` למשימות שדורשות פירוט גבוה (יקר יותר).
- אל תכתוב את ה-prompt ל-stdout כלשונו לפני הקריאה — שמור עליו לקריאה בלבד.
