# API'dan ma'lumot yig'ish va tahlil qilish (API → Pandas → CSV/Excel → SQL)

Bu konspektda API'dan ma'lumot olishdan boshlab, tozalash, faylga saqlash, tahlil qilish va SQL bazasiga yozishgacha bo'lgan to'liq jarayon kod bilan ko'rsatilgan.

## Jarayon bosqichlari

| № | Bosqich | Vazifasi | Asosiy vosita |
|---|---|---|---|
| 1 | API'dan ma'lumot yig'ish | Internetdagi xizmatdan JSON ma'lumot olish | `requests` |
| 2 | Tozalash | Ustun nomlari, dublikat, bo'sh qiymat, tur xatolarini tuzatish | `pandas` |
| 3 | Saqlash | Tozalangan ma'lumotni CSV yoki Excel faylga yozish | `to_csv`, `to_excel` |
| 4 | Tahlil va grafiklar | Guruhlash, hisoblash, vizualizatsiya | `pandas`, `matplotlib`, `seaborn` |
| 5 | SQL bazasiga yozish | Ma'lumotni jadvalga joylash | `SQLAlchemy` |
| 6 | SQL orqali tahlil | SELECT so'rovlari bilan tahlil | SQL |
| 7 | Dashboard | Power BI'da hisobot yaratish | Power BI |

---

## Loyiha tuzilishi

```
api-project/
├── data/
│   ├── raw/            # xom ma'lumot (JSON/CSV)
│   └── clean/          # tozalangan ma'lumot
├── charts/             # saqlangan grafiklar
├── 01_fetch.py         # API'dan olish
├── 02_clean.py         # tozalash
├── 03_analysis.py      # tahlil va grafiklar
├── 04_to_sql.py        # SQL bazaga yozish
├── requirements.txt
├── .gitignore
└── README.md
```

## Kutubxonalarni o'rnatish

```bash
pip install requests pandas numpy matplotlib seaborn openpyxl sqlalchemy psycopg2-binary
```

`requirements.txt` fayli:

```
requests
pandas
numpy
matplotlib
seaborn
openpyxl
sqlalchemy
psycopg2-binary
```

---

## 1-bosqich. API'dan ma'lumot yig'ish

**API** (Application Programming Interface) bu dastur orqali ma'lumot olish usuli. Biz so'rov (request) yuboramiz, server javob (response) qaytaradi, odatda **JSON** formatida.

Misol uchun ochiq va bepul API ishlatamiz: `https://fakestoreapi.com/products`

```python
# 01_fetch.py
import json
import requests
import pandas as pd

URL = "https://fakestoreapi.com/products"


def fetch_data(url: str) -> list:
    """API'ga GET so'rov yuboradi va JSON qaytaradi."""
    response = requests.get(url, timeout=10)
    response.raise_for_status()   # xato bo'lsa (404, 500) to'xtatadi
    return response.json()


if __name__ == "__main__":
    data = fetch_data(URL)
    print(f"Olingan yozuvlar soni: {len(data)}")

    # Xom ma'lumotni saqlab qo'yamiz
    with open("data/raw/products.json", "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

    # JSON'ni jadvalga aylantirish
    df = pd.json_normalize(data)
    print(df.head())
    print(df.columns.tolist())
```

**Muhim tushunchalar:**

| Tushuncha | Ma'nosi |
|---|---|
| `requests.get()` | Ma'lumot olish uchun so'rov |
| `status_code` | 200 = muvaffaqiyatli, 404 = topilmadi, 500 = server xatosi |
| `raise_for_status()` | Xato bo'lsa dasturni to'xtatadi |
| `timeout` | Javob kutish vaqti (soniya) |
| `json_normalize()` | Ichma-ich JSON'ni tekis jadvalga aylantiradi |

### Parametrli va kalitli API

Ko'p API'lar parametr yoki API kalit talab qiladi:

```python
params = {"limit": 10, "category": "electronics"}
headers = {"Authorization": "Bearer SIZNING_KALITINGIZ"}

response = requests.get(URL, params=params, headers=headers, timeout=10)
```

> API kalitni kodga yozmang. `.env` faylga saqlang va `.gitignore`ga qo'shing.

---

## 2-bosqich. Python va Pandas yordamida tozalash

```python
# 02_clean.py
import pandas as pd
import json

with open("data/raw/products.json", encoding="utf-8") as f:
    data = json.load(f)

df = pd.json_normalize(data)

# 1. Ma'lumotni ko'zdan kechirish
print(df.info())
print(df.describe())
print(df.isna().sum())          # har ustundagi bo'sh qiymatlar

# 2. Ustun nomlarini tartiblash
df = df.rename(columns={
    "rating.rate": "rating_rate",
    "rating.count": "rating_count",
})

# 3. Keraksiz ustunlarni olib tashlash
df = df.drop(columns=["image", "description"])

# 4. Dublikatlarni olib tashlash
df = df.drop_duplicates(subset="id")

# 5. Matnlarni tozalash
df["title"] = df["title"].str.strip()
df["category"] = df["category"].str.strip().str.title()

# 6. Turlarni to'g'rilash
df["price"] = pd.to_numeric(df["price"], errors="coerce")
df["rating_count"] = df["rating_count"].astype(int)

# 7. Bo'sh qiymatlar bilan ishlash
df = df.dropna(subset=["price"])              # narxi yo'q qatorlarni olib tashlash
df["rating_rate"] = df["rating_rate"].fillna(df["rating_rate"].mean())

# 8. Yangi ustun qo'shish
df["loaded_at"] = pd.Timestamp.now()

print(df.head())
```

**Tozalash funksiyalari xulosasi:**

| Funksiya | Vazifasi | Namuna |
|---|---|---|
| `rename` | Ustun nomini o'zgartiradi | `df.rename(columns={"a": "b"})` |
| `drop` | Ustunni olib tashlaydi | `df.drop(columns=["image"])` |
| `drop_duplicates` | Takrorlarni olib tashlaydi | `df.drop_duplicates(subset="id")` |
| `str.strip` | Boshi/oxiridagi bo'sh joyni olib tashlaydi | `df["title"].str.strip()` |
| `str.title` | Har so'zni bosh harf bilan yozadi | `df["category"].str.title()` |
| `to_numeric` | Sonli turga o'tkazadi | `pd.to_numeric(df["price"])` |
| `dropna` | Bo'sh qatorlarni olib tashlaydi | `df.dropna(subset=["price"])` |
| `fillna` | Bo'shlarni to'ldiradi | `df["x"].fillna(0)` |

> Bu funksiyalar SQL'dagi `TRIM`, `UPPER`, `LOWER`, `REPLACE` kabi string funksiyalariga o'xshash ishlaydi.

---

## 3-bosqich. CSV yoki Excel fayliga saqlash

```python
# 02_clean.py davomi
df.to_csv("data/clean/products.csv", index=False, encoding="utf-8-sig")
df.to_excel("data/clean/products.xlsx", index=False, sheet_name="Products")

print("Fayllar saqlandi")
```

| Parametr | Ma'nosi |
|---|---|
| `index=False` | Qator raqamlarini yozmaydi |
| `encoding="utf-8-sig"` | Excel'da o'zbekcha va rus harflari to'g'ri ochiladi |
| `sheet_name` | Excel varaq nomi |

Saqlangan faylni qayta o'qish:

```python
df = pd.read_csv("data/clean/products.csv")
# yoki
df = pd.read_excel("data/clean/products.xlsx")
```

---

## 4-bosqich. Tahlil qilish va grafiklar yaratish

```python
# 03_analysis.py
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns

df = pd.read_csv("data/clean/products.csv")
sns.set_theme(style="whitegrid")

# --- Tahlil ---

# 1. Kategoriya bo'yicha statistika
category_stats = (
    df.groupby("category")
      .agg(
          products=("id", "count"),
          avg_price=("price", "mean"),
          avg_rating=("rating_rate", "mean"),
      )
      .round(2)
      .sort_values("avg_price", ascending=False)
      .reset_index()
)
print(category_stats)

# 2. Eng qimmat 5 ta mahsulot
top5 = df.nlargest(5, "price")[["title", "price", "category"]]
print(top5)

# 3. Reytingi 4 dan yuqori mahsulotlar
high_rated = df[df["rating_rate"] > 4]
print(f"Reytingi 4 dan yuqori: {len(high_rated)} ta")

# --- Grafiklar ---

# Grafik 1: kategoriya bo'yicha o'rtacha narx
plt.figure(figsize=(8, 5))
sns.barplot(data=category_stats, x="category", y="avg_price")
plt.title("Kategoriya bo'yicha o'rtacha narx")
plt.xlabel("Kategoriya")
plt.ylabel("O'rtacha narx ($)")
plt.xticks(rotation=20)
plt.tight_layout()
plt.savefig("charts/avg_price_by_category.png", dpi=150)
plt.show()

# Grafik 2: narxlar taqsimoti
plt.figure(figsize=(8, 5))
sns.histplot(df["price"], bins=15, kde=True)
plt.title("Narxlar taqsimoti")
plt.xlabel("Narx ($)")
plt.tight_layout()
plt.savefig("charts/price_distribution.png", dpi=150)
plt.show()

# Grafik 3: narx va reyting bog'liqligi
plt.figure(figsize=(8, 5))
sns.scatterplot(data=df, x="price", y="rating_rate", hue="category")
plt.title("Narx va reyting")
plt.tight_layout()
plt.savefig("charts/price_vs_rating.png", dpi=150)
plt.show()
```

**Grafik turlari:**

| Grafik | Qachon ishlatiladi | Funksiya |
|---|---|---|
| Bar chart | Kategoriyalarni solishtirish | `sns.barplot` |
| Histogram | Qiymatlar taqsimotini ko'rish | `sns.histplot` |
| Scatter plot | Ikki ko'rsatkich bog'liqligi | `sns.scatterplot` |
| Line chart | Vaqt bo'yicha o'zgarish | `sns.lineplot` |

---

# Keyingi bosqich: `api-to-sql-project`

Endi ma'lumotni fayl o'rniga **SQL bazasiga** yozamiz va tahlilni SQL orqali qilamiz.

```
API'dan ma'lumot olish → Ma'lumotlarni SQL bazasiga yozish → SQL orqali tahlil qilish → Power BI dashboard
```

## 5-bosqich. Ma'lumotlarni SQL bazasiga yozish

Avval PostgreSQL'da baza yarating:

```sql
CREATE DATABASE api_db;
```

Parolni kodga yozmaslik uchun `.env` fayl yarating:

```
DB_USER=postgres
DB_PASSWORD=sizning_parolingiz
DB_HOST=localhost
DB_PORT=5432
DB_NAME=api_db
```

```bash
pip install python-dotenv
```

```python
# 04_to_sql.py
import os
import pandas as pd
from dotenv import load_dotenv
from sqlalchemy import create_engine

from importlib import import_module

load_dotenv()

# Bazaga ulanish
engine = create_engine(
    f"postgresql+psycopg2://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}"
    f"@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"
)

# Tozalangan ma'lumotni o'qish
df = pd.read_csv("data/clean/products.csv")

# SQL jadvalga yozish
df.to_sql(
    name="products",
    con=engine,
    if_exists="replace",   # "append" = qo'shib boradi, "replace" = qayta yozadi
    index=False,
)

print(f"{len(df)} ta qator 'products' jadvaliga yozildi")
```

> Boshlash uchun PostgreSQL o'rnatmasdan ham sinab ko'rish mumkin: `engine = create_engine("sqlite:///api_db.sqlite")`

| `if_exists` | Ma'nosi |
|---|---|
| `fail` | Jadval bor bo'lsa xato beradi |
| `replace` | Jadvalni o'chirib, qaytadan yozadi |
| `append` | Mavjud jadvalga qo'shadi |

## 6-bosqich. SQL orqali tahlil qilish

SQL so'rovlarini bazada (pgAdmin, DBeaver) yoki Python orqali ishga tushirish mumkin.

```sql
-- 1. Jadvalni tekshirish
SELECT * FROM products LIMIT 10;

-- 2. Kategoriya bo'yicha statistika
SELECT
    category,
    COUNT(*)                   AS products,
    ROUND(AVG(price)::numeric, 2)       AS avg_price,
    ROUND(AVG(rating_rate)::numeric, 2) AS avg_rating
FROM products
GROUP BY category
ORDER BY avg_price DESC;

-- 3. Eng qimmat 5 ta mahsulot
SELECT title, price, category
FROM products
ORDER BY price DESC
LIMIT 5;

-- 4. Reytingi 4 dan yuqori mahsulotlar
SELECT title, rating_rate, rating_count
FROM products
WHERE rating_rate > 4
ORDER BY rating_rate DESC;

-- 5. Har kategoriyada eng qimmat mahsulot (oyna funksiyasi)
SELECT category, title, price
FROM (
    SELECT
        category,
        title,
        price,
        ROW_NUMBER() OVER (PARTITION BY category ORDER BY price DESC) AS rn
    FROM products
) t
WHERE rn = 1;
```

Python orqali SQL natijasini olish:

```python
import pandas as pd
from sqlalchemy import text

query = text("""
    SELECT category, ROUND(AVG(price)::numeric, 2) AS avg_price
    FROM products
    GROUP BY category
    ORDER BY avg_price DESC
""")

result = pd.read_sql(query, engine)
print(result)
```

> `LIMIT` PostgreSQL va SQLite'da ishlaydi. SQL Server'da `SELECT TOP 5 ...` yoziladi.

## 7-bosqich. Power BI dashboard (keyingi qadam)

1. Power BI Desktop → **Get Data** → **PostgreSQL database**
2. Server: `localhost`, Database: `api_db`
3. `products` jadvalini tanlab **Load** bosing
4. Vizuallar yarating:

| Vizual | Ma'lumot |
|---|---|
| Card | Mahsulotlar soni, o'rtacha narx |
| Bar chart | Kategoriya bo'yicha o'rtacha narx |
| Donut chart | Kategoriyalar ulushi |
| Table | Eng qimmat 5 ta mahsulot |
| Slicer | Kategoriya filtri |

---

## Barcha jarayonni bitta skriptda yig'ish

```python
# main.py
import requests
import pandas as pd
from sqlalchemy import create_engine

URL = "https://fakestoreapi.com/products"


def extract(url):
    r = requests.get(url, timeout=10)
    r.raise_for_status()
    return pd.json_normalize(r.json())


def transform(df):
    df = df.rename(columns={"rating.rate": "rating_rate", "rating.count": "rating_count"})
    df = df.drop(columns=["image", "description"])
    df = df.drop_duplicates(subset="id")
    df["title"] = df["title"].str.strip()
    df["category"] = df["category"].str.strip().str.title()
    df["price"] = pd.to_numeric(df["price"], errors="coerce")
    df = df.dropna(subset=["price"])
    df["loaded_at"] = pd.Timestamp.now()
    return df


def load(df, engine):
    df.to_csv("data/clean/products.csv", index=False, encoding="utf-8-sig")
    df.to_sql("products", engine, if_exists="replace", index=False)


if __name__ == "__main__":
    engine = create_engine("sqlite:///api_db.sqlite")
    data = extract(URL)
    clean = transform(data)
    load(clean, engine)
    print(f"Tayyor: {len(clean)} ta qator")
```

Bu uslub **ETL** deyiladi: **E**xtract (olish), **T**ransform (tozalash), **L**oad (yuklash).

---

## `.gitignore`

```
.env
__pycache__/
*.sqlite
.ipynb_checkpoints/
```

## Xulosa

```
API (requests) → JSON → Pandas (tozalash) → CSV/Excel → Tahlil + grafik
                                          ↓
                                   SQL baza (to_sql) → SQL tahlil → Power BI
```
