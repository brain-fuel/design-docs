# ERD DSL Specification (Sigils-First)

A concise, text-based DSL for defining entities, fields, and relationships in a relational model. All types map directly to native SQL types or their parameterized variants.

---

## 1. Lexical Conventions

* **Entity Declaration**: Begins with a leading colon (`:`) and an identifier (letters, digits, `_`, `/`, `.`).

  ```text
  :User
  :organization.example.com
  ```

* **Field Declaration**: Omit or include the `has` keyword; both forms are equivalent.

  ```text
  # Concise
  UUID    $.id   PK
  # Verbose
  has UUID    $.id   PK
  ```

* **Types**: Must be valid SQL types or parameterized forms:

  * `UUID`, `BOOLEAN`, `DATE`, `TIMESTAMP`, `TEXT`, `JSON`, `JSONB`
  * Parametric: `VARCHAR(n)`, `CHAR(n)`, `NUMERIC(p,s)`, `DECIMAL(p,s)`, `INTERVAL <literal>`

* **Sigils** prefix field names to denote cardinality:

  * `$` **Scalar** (exactly one value)
  * `@` **List** (zero-or-more values)
  * `%` **Set/Map** (zero-or-more unique values)

* **Constraints** (space-separated after field):

  * `PK` — Primary key (implies `NOT NULL` + `UNIQUE`)
  * `PK(<f1>,<f2>,...)` — Composite primary key on listed fields
  * `UNIQUE` — Unique constraint
  * `NOT NULL` — Disallows nulls
  * `DEFAULT <literal>` — Default value (e.g. `DEFAULT 0`, `DEFAULT now()`, `DEFAULT 'foo'`)
  * `FK` — Foreign key reference to the entity named by the field’s type
  * `CHECK(<expression>)` — Boolean condition

* **Comments** begin with `#` or `;;`.

* **Formatting**: Indentation for readability; newlines separate statements.

---

## 2. Core Examples

### 2.1 Scalar & 1\:Many Relationships

```text
:Organization
    UUID           $.id          PK                            # Primary key
    VARCHAR(255)   $.domain      UNIQUE NOT NULL              # Unique domain
    VARCHAR(100)   $.name        NOT NULL                     # Name

:User
    UUID           $.id          PK
    VARCHAR(255)   $.email       UNIQUE NOT NULL              # Unique email
    UUID           $.org_id      FK                            # 1:many FK → Organization.id
    VARCHAR(50)    $.role        DEFAULT 'member'             # Default role
    INTEGER        $.age         NOT NULL CHECK(age >= 18)    # Age constraint
```

### 2.2 Many-to-Many via Join Entities

```text
:Post
    UUID           $.id          PK
    VARCHAR(200)   $.title       NOT NULL
    TEXT           $.body        NOT NULL
    UUID           $.author_id   FK                            # FK → User.id
    TIMESTAMP      $.created_at  DEFAULT now()

:Tag
    UUID           $.id          PK
    VARCHAR(50)    $.label       UNIQUE NOT NULL

:PostTags         # Join table for Post ↔ Tag
    UUID           $.post_id     FK                            # FK → Post.id
    UUID           $.tag_id      FK                            # FK → Tag.id
    INTEGER        $.priority    DEFAULT 0 CHECK(priority >= 0)
    PK(post_id,tag_id)                                        # Composite PK
```

### 2.3 Composite Keys & Partial Unique

```text
:OrderItem
    UUID           $.order_id    FK                            # FK → Order.id
    UUID           $.product_id  FK                            # FK → Product.id
    INTEGER        $.quantity    NOT NULL                     # Quantity
    PK(order_id,product_id)                                    # Composite PK

:Member
    UUID           $.id          PK
    VARCHAR(255)   $.email       NOT NULL
    VARCHAR(50)    $.group       NOT NULL
    UNIQUE(email,group)                                       # Multi-column unique
    CHECK(group IN ('public','private'))                     # Allowed values
```

---

## 3. Extended Patterns

* **Enum Types**

  ```text
  enum Status { pending, active, archived }
  :Task
      UUID       $.id       PK
      Status     $.status   DEFAULT pending
  ```

* **Arrays & JSON/JSONB**

  ```text
  :Product
      UUID           $.id         PK
      TEXT[]         $.tags       DEFAULT ARRAY[]::TEXT[]    # Text array
      JSON           $.raw_data   DEFAULT '{}'                # JSON text
      JSONB          $.metadata   DEFAULT '{}'                # JSONB blob
  ```

* **Numeric Precision**

  ```text
  :Invoice
      UUID             $.id         PK
      NUMERIC(10,2)    $.amount     NOT NULL                  # Currency
      DECIMAL(12,4)    $.rate       NOT NULL                  # Exchange rate
  ```

* **Inheritance (Table-Per-Type)**

  ```text
  :Vehicle
      UUID     $.id         PK
      VARCHAR(50) $.make    NOT NULL

  :Car       # Subtype of Vehicle
      UUID     $.id         FK UNIQUE                   # PK+FK → Vehicle.id
      INTEGER  $.doors      NOT NULL                     # Door count
  ```

* **Indexes & Views**

  ```text
  INDEX idx_user_email ON User(email)
  MATERIALIZED VIEW recent_posts AS
      SELECT * FROM Post WHERE created_at > now() - INTERVAL '7 days'
  ```

* **Triggers & Procedures**

  ```text
  TRIGGER audit_log BEFORE UPDATE ON Task
      EXECUTE PROCEDURE log_change();
  ```

---

## 4. Advanced Database Features

* **Partitioning**

  ```text
  :EventLog
      UUID         $.id         PK
      TIMESTAMP    $.ts         NOT NULL
      JSONB        $.payload    NOT NULL
      PARTITION BY RANGE(ts) INTERVAL '1 month'
  ```

* **Time-To-Live (TTL)**

  ```text
  :Session
      UUID         $.id             PK
      TIMESTAMP    $.created_at     NOT NULL
      TTL(created_at, '30 days')    # Automatic expiry
  ```

* **Full-Text Search (FTS)**

  ```text
  :Article
      UUID      $.id           PK
      TEXT      $.content      NOT NULL
      FTS(content) USING GIN   # GIN index for FTS
  ```

* **Spatial Types**

  ```text
  EXTENSION postgis
  :Place
      UUID        $.id         PK
      GEOGRAPHY   $.location   NOT NULL
      INDEX       idx_place_loc ON location USING GIST
  ```

* **Row-Level Security (RLS)**

  ```text
  :Document
      UUID      $.id            PK
      UUID      $.owner_id      FK
      RLS ENABLE
      POLICY owner_only USING (owner_id = current_user())
  ```

* **Auditing & Soft Deletes**

  ```text
  :User
      UUID       $.id          PK
      TIMESTAMP  $.created_at  DEFAULT now()
      TIMESTAMP  $.deleted_at  NULL
      AUDITABLE                     # created_at/updated_at triggers
      SOFT_DELETE($.deleted_at)     # marks deletion timestamp
  ```

* **Extensions**

  ```text
  EXTENSION uuid-ossp     # UUID generation
  ```

---

## 5. Grammar Sketch (EBNF)

```ebnf
file       ::= { entityDef }
entityDef  ::= ":" identifier NEWLINE { ["has" SP] type SP sigilField [SP constraintList] NEWLINE }

sigilField ::= sigil "." identifier
sigil      ::= "$" | "@" | "%"

type       ::= identifier ["(" literal {"," literal} ")"] | "enum" enumName
identifier ::= letter { letter | digit | '_' | '/' | '.' }

constraintList ::= constraint { SP constraint }
constraint     ::= "PK" | "PK(" identifier {"," identifier} ")" | "UNIQUE" | "NOT NULL" | ("DEFAULT" SP literal) | "FK" | ("CHECK(" expr ")")
literal        ::= number | string | list_literal
list_literal   ::= "[" [literal {"," SP literal}] "]"
expr           ::= any string without unbalanced parentheses
```

---

## 6. Codegen Mapping

* **PK** → `PRIMARY KEY` on a single column
* **PK(f1,f2,…)** → `PRIMARY KEY(f1, f2, …)` for composite keys
* **FK** → `FOREIGN KEY REFERENCES <Entity>(id)` (add `UNIQUE` for 1:1)
* **UNIQUE**, **NOT NULL**, **DEFAULT**, **CHECK** → DDL equivalents
* **List sigil (@)** → inverse relation, no direct column
* **Join entities** → explicit many-to-many tables

---

## 7. Composite Primary Keys Notation

To explicitly define composite primary keys, use the `PK(<field1>,<field2>,...)` constraint on any one field (or on a dedicated line) to indicate that the combination of listed fields forms the primary key. Examples:

```text
:Enrollment
    UUID $.student_id   FK                    # FK → Student.id
    UUID $.course_id    FK                    # FK → Course.id
    Int  $.grade        DEFAULT 0 CHECK(grade >= 0)
    PK(student_id,course_id)                 # Composite PK on (student_id, course_id)
```

```text
:OrderItem
    UUID $.order_id     FK                    # FK → Order.id
    UUID $.product_id   FK                    # FK → Product.id
    Int  $.quantity     NOT NULL
    PK(order_id,product_id)                  # Composite PK on (order_id, product_id)
```

---

## 8. Ops Integration & Minimal Lovable Product

To tame operational complexity while keeping the DSL feature‑complete for a Minimal Lovable Product (MLP), provide:

1. **Single Source of Truth**  
   - Store each spec in a `.erd` file alongside code.  
   - Name migrations after the spec (e.g. `20250801_create_user_table.sql` generated from `user.erd`).

2. **Codegen Hooks**  
   - CLI tool (`erdc`) that:
     - **Parses** `.erd` files and validates against the EBNF.  
     - **Generates** core DDL scripts for `CREATE TABLE`, FKs, indexes.  
     - **Emits** template migration stubs for advanced features (partitioning, TTL, triggers).  

3. **Sidecar Metadata (Optional)**  
   - Support an optional `entity.meta.yml` for each `.erd` to declare environment‑specific or ops concerns:  
     ```yaml
     # User.meta.yml
     rls: true
     partitions:
       by: range
       field: created_at
       interval: "1 month"
     migrations:
       post: | # appended SQL for audit triggers
         CREATE TRIGGER …;
     ```

4. **CI/Static Analysis**  
   - **Lint step**: run `erdc lint` to:
     - Enforce EBNF correctness.  
     - Run static checks (redundant PKs, missing FKs).  
     - Report composite key or index warnings.  
   - **Diff step**: compare generated DDL vs. last committed migration to catch drift.

5. **Migration Workflow**  
   - **Migration generation**: `erdc migrate --to v1.2.0` scaffolds a new SQL file combining core DDL + sidecar overrides.  
   - **Review & apply**: DevOps reviews the generated script, tweaks if needed, and applies via standard tools (Flyway, Liquibase).

6. **Extension & Plugin Support**  
   - Allow custom plugins to hook into parsing or codegen for DB‑specific features (e.g. MySQL sharding, Oracle partitions).  
   - Plugins register new keywords (e.g. `SHARD BY HASH()`) without modifying the core grammar.

---

## 9. Next Steps
1. Validate against your domain model.
2. Build a parser (e.g. Pest, ANTLR) using the EBNF.
3. Generate:
   - SQL migrations (tables, FKs, indexes)
   - Model & query code stubs
   - Documentation (Mermaid, markdown)
4. Iterate on advanced features (nested types, extension hooks).

