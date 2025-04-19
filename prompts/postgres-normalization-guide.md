# Database Normalization Best Practices for PostgreSQL with Prisma ORM

## Introduction

This guide outlines best practices for database normalization when working with PostgreSQL and Prisma ORM. Proper normalization creates an efficient, maintainable database structure that works optimally with Prisma's capabilities.

## Database Normalization Fundamentals

### First Normal Form (1NF)
- **Atomic values**: Each cell in a table must contain a single value
- **No repeating groups**: Related data that might be repeated should be stored in separate tables
- **Primary key**: Each table must have a primary key that uniquely identifies each row

### Second Normal Form (2NF)
- **Meet 1NF requirements**
- **No partial dependencies**: Non-key attributes must depend on the entire primary key, not just part of it
- This is particularly important for tables with composite primary keys

### Third Normal Form (3NF)
- **Meet 2NF requirements**
- **No transitive dependencies**: Non-key attributes must not depend on other non-key attributes
- Each non-key column should depend only on the primary key

## Prisma-Specific Implementation

### Primary Keys

Always define clear primary keys for every model:

```prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String
}
```

**Best practices:**
- Use integers with `autoincrement()` for simple, high-performance keys
- Consider UUIDs for distributed systems or when IDs might be exposed:
  ```prisma
  id String @id @default(uuid())
  ```
- Avoid using natural keys (like email) as primary keys; use surrogate keys instead

### Relationships

Properly model entity relationships to maintain normalization:

#### One-to-Many Relationships

```prisma
model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String?
  authorId  Int
  author    User     @relation(fields: [authorId], references: [id])
}

model User {
  id    Int    @id @default(autoincrement())
  name  String
  posts Post[]
}
```

#### Many-to-Many Relationships

Always use junction tables for many-to-many relationships:

```prisma
model Post {
  id    Int       @id @default(autoincrement())
  title String
  tags  PostTag[]
}

model Tag {
  id    Int       @id @default(autoincrement())
  name  String    @unique
  posts PostTag[]
}

model PostTag {
  id      Int   @id @default(autoincrement())
  postId  Int
  tagId   Int
  post    Post  @relation(fields: [postId], references: [id])
  tag     Tag   @relation(fields: [tagId], references: [id])
  @@unique([postId, tagId])
}
```

#### One-to-One Relationships

```prisma
model User {
  id       Int       @id @default(autoincrement())
  name     String
  profile  Profile?
}

model Profile {
  id       Int       @id @default(autoincrement())
  bio      String?
  userId   Int       @unique
  user     User      @relation(fields: [userId], references: [id])
}
```

### Database Constraints

Implement constraints to maintain data integrity:

```prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String
  age       Int?     @db.SmallInt
  
  @@index([email])
}
```

**Key constraints:**
- `@unique`: Ensures no duplicate values
- Foreign key constraints (automatically added by Prisma in relations)
- Check constraints (use `@db.Constraint` for PostgreSQL-specific constraints)

### Indexing Strategy

Create appropriate indexes for performance:

```prisma
model Order {
  id         Int      @id @default(autoincrement())
  customerId Int
  orderDate  DateTime
  status     String
  
  customer   Customer @relation(fields: [customerId], references: [id])
  
  @@index([customerId])
  @@index([orderDate])
  @@index([status])
}
```

**Indexing best practices:**
- Index foreign key columns
- Index frequently queried fields
- Create composite indexes for columns that are often queried together
- Don't over-index; each index adds overhead to write operations

### Enumeration Types

Use enums for fields with predetermined values:

```prisma
enum Role {
  USER
  ADMIN
  MODERATOR
}

model User {
  id   Int   @id @default(autoincrement())
  role Role  @default(USER)
}
```

PostgreSQL supports enum types natively, making this an efficient approach.

### Metadata and Audit Fields

Include metadata for tracking and auditing:

```prisma
model Document {
  id        Int      @id @default(autoincrement())
  title     String
  content   String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  createdBy Int
  updatedBy Int?
  
  creator   User     @relation("DocumentCreator", fields: [createdBy], references: [id])
  editor    User?    @relation("DocumentEditor", fields: [updatedBy], references: [id])
}
```

### Avoid Redundancy

- Store each piece of data in exactly one place
- Use views or Prisma's query capabilities for complex read operations instead of denormalizing
- When denormalization is necessary for performance, document it clearly and maintain data consistency

## Schema Migration Considerations

Prisma Migrate helps manage schema evolution:

**Best practices:**
- Use Prisma's migration tools rather than manually altering the database
- Create migrations for each logical change to the schema
- Test migrations on development databases before applying to production
- Keep migration history in version control
- Consider data transformation needs when changing schema

## Performance Optimization

- Use appropriate data types for columns (e.g., `@db.SmallInt` for small numbers)
- Consider partitioning large tables by date or other logical divisions
- Use PostgreSQL's specialized types like `jsonb` for semi-structured data when appropriate
- Create materialized views for complex, frequently-accessed read operations

## Common Anti-Patterns to Avoid

- Storing comma-separated values in a single field (violates 1NF)
- Using natural keys (like email) as primary keys
- Denormalizing prematurely without measuring performance
- Overusing JSONB fields when structured relations would be more appropriate
- Ignoring index maintenance as data grows

## Conclusion

Following these normalization best practices with PostgreSQL and Prisma ORM will help you create a database structure that is efficient, maintainable, and scalable. Properly normalized databases are easier to extend, query, and maintain over time, resulting in better long-term application performance and developer experience.
