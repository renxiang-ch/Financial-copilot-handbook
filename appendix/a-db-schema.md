# Appendix A · Database Schema

This appendix does not duplicate the schema — a copied-out SQL file drifts
from the real one the moment either changes. Instead:

**Authoritative source**: `src/copilot/storage/schema.py` in repo A, at the
`v1.0-teaching` tag. Read `CREATE_TABLES_SQL` and the `migrate_*` functions
that `create_tables()` calls.

**To see the schema actually in effect** (including anything a migration
added after the initial `CREATE TABLE`):

```sql
\d companies
\d filings
\d financial_facts
\d text_chunks
\d supply_edges
```

<!-- TODO: once the tag is cut, paste the \d output here as a point-in-time
     reference, clearly dated, with a note that schema.py is still the
     source of truth if the two ever disagree -->
