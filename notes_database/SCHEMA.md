# Notes Database Schema (PostgreSQL)

This database stores **notes**, **tags**, and their many-to-many relationship (**note_tags**).

Connection command is defined in:
- `notes_database/db_connection.txt`

Example (current):
```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp
```

## Extensions

This schema enables trigram search support:

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -v ON_ERROR_STOP=1 -c "CREATE EXTENSION IF NOT EXISTS pg_trgm"
```

> Note: `pg_trgm` enables efficient `ILIKE '%query%'` searches via GIN trigram indexes.

## Tables

### `notes`

- `id` UUID PK (default `gen_random_uuid()`)
- `title` text
- `content` text
- `is_archived` boolean
- `created_at` / `updated_at` timestamptz

Created with:

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -v ON_ERROR_STOP=1 -c "CREATE TABLE IF NOT EXISTS notes (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), title TEXT NOT NULL DEFAULT '', content TEXT NOT NULL DEFAULT '', is_archived BOOLEAN NOT NULL DEFAULT FALSE, created_at TIMESTAMPTZ NOT NULL DEFAULT now(), updated_at TIMESTAMPTZ NOT NULL DEFAULT now())"
```

### `tags`

- `id` UUID PK (default `gen_random_uuid()`)
- `name` text
- `created_at` timestamptz

Created with:

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -v ON_ERROR_STOP=1 -c "CREATE TABLE IF NOT EXISTS tags (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), name TEXT NOT NULL, created_at TIMESTAMPTZ NOT NULL DEFAULT now())"
```

Case-insensitive uniqueness:

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -v ON_ERROR_STOP=1 -c "CREATE UNIQUE INDEX IF NOT EXISTS ux_tags_name_lower ON tags ((lower(name)))"
```

### `note_tags` (join table)

- `note_id` UUID FK → `notes(id)` ON DELETE CASCADE
- `tag_id` UUID FK → `tags(id)` ON DELETE CASCADE
- `created_at` timestamptz
- PK `(note_id, tag_id)`

Created with:

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -v ON_ERROR_STOP=1 -c "CREATE TABLE IF NOT EXISTS note_tags (note_id UUID NOT NULL REFERENCES notes(id) ON DELETE CASCADE, tag_id UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE, created_at TIMESTAMPTZ NOT NULL DEFAULT now(), PRIMARY KEY (note_id, tag_id))"
```

## Indexes (search-friendly)

General querying:

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -v ON_ERROR_STOP=1 -c "CREATE INDEX IF NOT EXISTS ix_notes_updated_at ON notes (updated_at DESC)"
psql postgresql://appuser:dbuser123@localhost:5000/myapp -v ON_ERROR_STOP=1 -c "CREATE INDEX IF NOT EXISTS ix_notes_is_archived ON notes (is_archived)"
psql postgresql://appuser:dbuser123@localhost:5000/myapp -v ON_ERROR_STOP=1 -c "CREATE INDEX IF NOT EXISTS ix_note_tags_tag_id ON note_tags (tag_id)"
psql postgresql://appuser:dbuser123@localhost:5000/myapp -v ON_ERROR_STOP=1 -c "CREATE INDEX IF NOT EXISTS ix_note_tags_note_id ON note_tags (note_id)"
```

Trigram search (fast `ILIKE`):

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -v ON_ERROR_STOP=1 -c "CREATE INDEX IF NOT EXISTS ix_notes_title_trgm ON notes USING GIN (title gin_trgm_ops)"
psql postgresql://appuser:dbuser123@localhost:5000/myapp -v ON_ERROR_STOP=1 -c "CREATE INDEX IF NOT EXISTS ix_notes_content_trgm ON notes USING GIN (content gin_trgm_ops)"
psql postgresql://appuser:dbuser123@localhost:5000/myapp -v ON_ERROR_STOP=1 -c "CREATE INDEX IF NOT EXISTS ix_tags_name_trgm ON tags USING GIN (name gin_trgm_ops)"
```

## Minimal seed data (optional)

Only minimal seed data was inserted for validation:

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -v ON_ERROR_STOP=1 -c "INSERT INTO tags (name) VALUES ('welcome') ON CONFLICT DO NOTHING"
psql postgresql://appuser:dbuser123@localhost:5000/myapp -v ON_ERROR_STOP=1 -c "INSERT INTO notes (title, content) VALUES ('Welcome', 'This is your first note.')"
psql postgresql://appuser:dbuser123@localhost:5000/myapp -v ON_ERROR_STOP=1 -c "INSERT INTO note_tags (note_id, tag_id) SELECT (SELECT id FROM notes ORDER BY created_at DESC LIMIT 1), (SELECT id FROM tags WHERE lower(name)=lower('welcome') LIMIT 1) ON CONFLICT DO NOTHING"
```

## Validation queries

List latest notes with tags:

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "SELECT n.id, n.title, array_agg(t.name ORDER BY t.name) AS tags FROM notes n LEFT JOIN note_tags nt ON nt.note_id=n.id LEFT JOIN tags t ON t.id=nt.tag_id GROUP BY n.id, n.title ORDER BY n.created_at DESC LIMIT 5"
```

Example `ILIKE` search that should benefit from trigram indexes:

```bash
psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "SELECT id, title FROM notes WHERE title ILIKE '%wel%' OR content ILIKE '%wel%' ORDER BY updated_at DESC LIMIT 20"
```
