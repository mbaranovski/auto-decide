**Decision:** Going with Option B (PostgreSQL full-text search with tsvector) -- it delivers every requested feature (name search, category and price filtering, and true relevance/price/date sorting) without requiring additional infrastructure, whereas Option C drops real relevance sorting and Option A introduces an Elasticsearch cluster dependency that isn't justified for a product catalog search.

## Implementation Plan

Here is how to implement the search functionality using PostgreSQL full-text search:

### 1. Database Setup

Add a `tsvector` column to the products table and create a GIN index for fast lookups:

```sql
-- Add tsvector column for full-text search
ALTER TABLE products ADD COLUMN search_vector tsvector;

-- Populate the search vector from product name (and optionally description)
UPDATE products SET search_vector = to_tsvector('english', coalesce(name, ''));

-- Create GIN index for fast full-text search
CREATE INDEX idx_products_search ON products USING GIN(search_vector);

-- Create index on category for fast filtering
CREATE INDEX idx_products_category ON products (category);

-- Create index on price for range filtering and sorting
CREATE INDEX idx_products_price ON products (price);

-- Create index on created_at for "newest" sorting
CREATE INDEX idx_products_created_at ON products (created_at DESC);
```

### 2. Auto-Update Trigger

Keep the search vector in sync whenever a product name changes:

```sql
CREATE OR REPLACE FUNCTION products_search_vector_update() RETURNS trigger AS $$
BEGIN
  NEW.search_vector := to_tsvector('english', coalesce(NEW.name, ''));
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_products_search_vector
  BEFORE INSERT OR UPDATE OF name ON products
  FOR EACH ROW
  EXECUTE FUNCTION products_search_vector_update();
```

### 3. Search Query

The core query that supports all requested features -- name search, category filter, price range filter, and sorting by relevance, price, or newest:

```sql
SELECT
  p.id,
  p.name,
  p.category,
  p.price,
  p.created_at,
  ts_rank(p.search_vector, plainto_tsquery('english', :search_term)) AS relevance
FROM products p
WHERE
  -- Full-text search on product name
  (:search_term IS NULL OR p.search_vector @@ plainto_tsquery('english', :search_term))
  -- Filter by category
  AND (:category IS NULL OR p.category = :category)
  -- Filter by price range
  AND (:min_price IS NULL OR p.price >= :min_price)
  AND (:max_price IS NULL OR p.price <= :max_price)
ORDER BY
  CASE WHEN :sort_by = 'relevance' THEN ts_rank(p.search_vector, plainto_tsquery('english', :search_term)) END DESC,
  CASE WHEN :sort_by = 'price_asc' THEN p.price END ASC,
  CASE WHEN :sort_by = 'price_desc' THEN p.price END DESC,
  CASE WHEN :sort_by = 'newest' THEN p.created_at END DESC
LIMIT :page_size
OFFSET :offset;
```

### 4. Application Layer (Example in Python)

```python
from dataclasses import dataclass
from enum import Enum
from typing import Optional


class SortBy(str, Enum):
    RELEVANCE = "relevance"
    PRICE_ASC = "price_asc"
    PRICE_DESC = "price_desc"
    NEWEST = "newest"


@dataclass
class SearchFilters:
    search_term: Optional[str] = None
    category: Optional[str] = None
    min_price: Optional[float] = None
    max_price: Optional[float] = None
    sort_by: SortBy = SortBy.RELEVANCE
    page: int = 1
    page_size: int = 20


def search_products(db_conn, filters: SearchFilters):
    """Search products with full-text search, filtering, and sorting."""
    params = {
        "search_term": filters.search_term,
        "category": filters.category,
        "min_price": filters.min_price,
        "max_price": filters.max_price,
        "sort_by": filters.sort_by.value,
        "page_size": filters.page_size,
        "offset": (filters.page - 1) * filters.page_size,
    }

    query = """
        SELECT
            p.id, p.name, p.category, p.price, p.created_at,
            ts_rank(p.search_vector, plainto_tsquery('english', %(search_term)s)) AS relevance
        FROM products p
        WHERE
            (%(search_term)s IS NULL OR p.search_vector @@ plainto_tsquery('english', %(search_term)s))
            AND (%(category)s IS NULL OR p.category = %(category)s)
            AND (%(min_price)s IS NULL OR p.price >= %(min_price)s)
            AND (%(max_price)s IS NULL OR p.price <= %(max_price)s)
        ORDER BY
            CASE WHEN %(sort_by)s = 'relevance'
                 THEN ts_rank(p.search_vector, plainto_tsquery('english', %(search_term)s)) END DESC,
            CASE WHEN %(sort_by)s = 'price_asc' THEN p.price END ASC,
            CASE WHEN %(sort_by)s = 'price_desc' THEN p.price END DESC,
            CASE WHEN %(sort_by)s = 'newest' THEN p.created_at END DESC
        LIMIT %(page_size)s OFFSET %(offset)s
    """

    cursor = db_conn.cursor()
    cursor.execute(query, params)
    return cursor.fetchall()
```

### 5. Feature Checklist

| Requirement | Delivered | How |
|---|---|---|
| Search by product name | Yes | `tsvector` full-text search with `plainto_tsquery` |
| Filter by category | Yes | SQL `WHERE p.category = :category` |
| Filter by price range | Yes | SQL `WHERE p.price BETWEEN :min AND :max` |
| Sort by relevance | Yes | `ts_rank()` scoring from PostgreSQL full-text search |
| Sort by price | Yes | `ORDER BY p.price` |
| Sort by newest | Yes | `ORDER BY p.created_at DESC` |

All six requested features are fully implemented with no deferred work and no additional infrastructure beyond PostgreSQL.
