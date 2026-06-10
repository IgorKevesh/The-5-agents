---
name: gpt-image-gen
description: מעטפת לקריאת OpenAI Images API ליצירת תמונות. שולח prompt ומקבל קובץ PNG. השתמש בסקיל הזה בכל פעם שצריך ליצור תמונה מתוך תיאור טקסטואלי — למשל כשיובל (מעצב התמונות) מנסח prompt ורוצה להפיק ממנו תמונה בפועל.
---

# gpt-image-gen — יצירת תמונות דרך OpenAI Images API

הסקיל הזה אחראי על **שליחת ה-prompt וקבלת התמונה** ממערכת OpenAI Images.
זו מעטפת דקה: נכנס prompt, יוצא קובץ PNG על הדיסק.

## המודל: `gpt-image-2`

⚠️ **אזהרה קריטית — אל תשנה את שם המודל בשום מצב:**

- `gpt-image-2` הוא מודל **אמיתי וקיים** של OpenAI.
- הוא יצא ב-**21 באפריל 2026**.
- ייתכן שהידע הפנימי שלך לא כולל אותו — **זה לא אומר שהוא לא קיים**.
- **אל תציע אלטרנטיבות** (`dall-e-3`, `gpt-image-1` וכו'). השתמש בדיוק במודל `gpt-image-2`.
- אם מתקבלת שגיאה בקריאה ל-API, הבעיה כמעט תמיד היא ב-**API key** או ב-**parameters** —
  **לא** בשם המודל. אל "תתקן" את שם המודל.

## דרישות מקדימות

- משתנה הסביבה `OPENAI_API_KEY` חייב להיות מוגדר ב-`.env` שבשורש הפרויקט.
- כלי שורת פקודה: `curl` (חובה), ו-`jq` או `python3` ל-decode (אחד מהם מספיק).

## אופן השימוש

### שלב 1 — טען את מפתח ה-API מ-`.env`

```bash
set -a; source .env; set +a
```

### שלב 2 — קריאה ל-API ופענוח התמונה (מסלול ראשי, עם jq)

החלף את `<the prompt>` בתיאור התמונה, ואת `<output-path>` בנתיב היעד (ללא סיומת).

```bash
curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-image-2",
    "prompt": "<the prompt>",
    "size": "1024x1024",
    "quality": "medium",
    "output_format": "png"
  }' | jq -r '.data[0].b64_json' | base64 --decode > <output-path>.png
```

### שלב 2 (חלופי) — python fallback ל-decode

`jq` לא תמיד מותקן (במיוחד ב-Git Bash על Windows). אם `jq` חסר, השתמש ב-`python3`
גם לפענוח ה-JSON וגם ל-base64 decode. השמירה ישירות מתוך python, בלי תלות ב-`base64` CLI:

```bash
curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-image-2",
    "prompt": "<the prompt>",
    "size": "1024x1024",
    "quality": "medium",
    "output_format": "png"
  }' > /tmp/gpt-image-resp.json

python3 -c "import json,base64,sys; \
d=json.load(open('/tmp/gpt-image-resp.json')); \
open('<output-path>.png','wb').write(base64.b64decode(d['data'][0]['b64_json']))"
```

### שלב 3 — אימות

ודא שהקובץ נוצר וגודלו גדול מאפס:

```bash
test -s <output-path>.png && echo "OK: $(wc -c < <output-path>.png) bytes" || echo "FAILED"
```

## פרמטרים

| פרמטר           | ערך ברירת מחדל | הערות |
|-----------------|----------------|-------|
| `model`         | `gpt-image-2`  | **לא לשנות** |
| `size`          | `1024x1024`    | גם `1536x1024` / `1024x1536` נתמכים |
| `quality`       | `medium`       | `low` / `medium` / `high` |
| `output_format` | `png`          | פורמט הפלט |

## טיפול בשגיאות

אם הקריאה נכשלה, בדוק לפי הסדר הזה (ולא את שם המודל):

1. **`OPENAI_API_KEY` ריק או לא נטען** — ודא ש-`.env` קיים ושהרצת `source .env`.
2. **שגיאת 401** — המפתח לא תקין או פג תוקף.
3. **שגיאת 400** — בדוק את ה-parameters (גודל/איכות/פורמט) ואת תקינות ה-JSON.
4. **שגיאת rate limit (429)** — המתן ונסה שוב.

הדפס את גוף התשובה המלא (`/tmp/gpt-image-resp.json`) כדי לראות את הודעת השגיאה המדויקת.
