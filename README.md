

# ReactiveSQLite

**A lightweight, reactive, domain-agnostic SQLite ORM for native Android (Java).**

---

## Overview

ReactiveSQLite is a small, self-contained SQLite ORM for native Android applications written in pure Java. It provides automatic table creation, type-safe CRUD, a fluent query builder, real transactions, entity relations, reactive live queries, and a bidirectional JavaScript bridge for WebView-based frontends.

The ORM is **100% domain-agnostic**: it knows nothing about `Product`, `Sale`, `Customer`, or any business concept. It only knows generic entities, tables, columns, primary keys, indexes, queries, transactions, and relations.

Zero external dependencies. Only the Android SDK and the Java standard library are required.

---

## Why ReactiveSQLite?

Most Android ORMs either require annotation processing (Room), bring a large dependency graph (Realm, ObjectBox), or force a specific architecture (Room + ViewModel + LiveData). ReactiveSQLite takes a different path:

- **No code generation** — everything is resolved at runtime via reflection.
- **No external dependencies** — only `android.database.sqlite` and `org.json` from the SDK.
- **No domain coupling** — your entities stay plain old Java objects.
- **Reactive by design** — live queries notify subscribers when the underlying data changes.
- **WebView-friendly** — an optional bridge exposes the entire ORM to JavaScript with a promise-based API.
- **Small enough to read** — the entire ORM fits in about a dozen files.

It is not meant to replace Room or SQLDelight in every scenario. It is meant for projects that want a **small, predictable, hackable** persistence layer with first-class support for WebView hybrid apps.

---

## Features

- Annotation-driven schema: `@Table`, `@Column`, `@PrimaryKey`, `@NotNull`, `@Unique`, `@Indexed`, `@Ignore`
- Entity relations: `@OneToMany` and `@ManyToOne` with batch loading
- Automatic `CREATE TABLE` and `CREATE INDEX` generation
- Generic repository and fluent `QueryBuilder`
- Real SQLite transactions (`beginTransaction` / `setTransactionSuccessful` / `endTransaction`)
- Reactive live queries with `LiveQuery<T>` and `Subscription`
- Bidirectional WebView bridge (`ReactiveBridge`) with JSON-over-JS interface
- Custom command registration for exposing domain handlers to JavaScript
- Database export/import via `MediaStore` (no storage permissions on Android 10+)
- Automatic model discovery via `DexFile` package scanning
- Full JSON serialization via the native `org.json` API

---

## Architecture

```

┌─────────────────────────────────────┐
│           Your Application          │
│         (Activities, Fragments)     │
└──────────────────┬──────────────────┘
│
┌──────────────────▼──────────────────┐
│          ReactiveSQLite             │
│  ┌───────────────────────────────┐  │
│  │        ModelRegistry          │  │
│  │        ModelDiscovery         │  │
│  │        EntityMapper           │  │
│  │        QueryBuilder           │  │
│  │        GenericRepository      │  │
│  │        LiveQueryManager       │  │
│  │        RelationResolver       │  │
│  │        Transactions           │  │
│  └───────────────────────────────┘  │
│  ┌───────────────────────────────┐  │
│  │         SQLiteEngine          │  │
│  └───────────────────────────────┘  │
└──────────────────┬──────────────────┘
│
SQLite Database

Optional WebView bridge:

┌──────────────────▼──────────────────┐
│          ReactiveBridge             │
│  BridgeJsInterface  (JS → Java)     │
│  BridgeEventBus     (Java → JS)     │
│  BridgeRuntimeInjector              │
│  BridgeLifecycleObserver            │
└─────────────────────────────────────┘

```

---

## Installation

ReactiveSQLite is distributed as source. Copy the `reactivesqlite/` package into your Android project under `src/main/java/`.

The only requirement is **Android API 19+**.

No Gradle plugin, no annotation processor, no KAPT.

```

app/src/main/java/
└── com/
└── yourpackage/
└── reactivesqlite/
├── ReactiveSQLite.java
├── SQLiteEngine.java
├── ModelRegistry.java
├── ModelMeta.java
├── ModelDiscovery.java
├── EntityMapper.java
├── QueryBuilder.java
├── GenericRepository.java
├── LiveQuery.java
├── LiveQueryManager.java
├── Subscription.java
├── TypeMapper.java
├── SqlTemplates.java
├── JsonEngine.java
├── ReactiveSQLiteManager.java
├── annotations/
├── relations/
├── bridge/
└── backup/

```

---

## Quick Start

### 1. Define an entity

```java
import com.yourpackage.reactivesqlite.annotations.Table;
import com.yourpackage.reactivesqlite.annotations.Column;
import com.yourpackage.reactivesqlite.annotations.PrimaryKey;
import com.yourpackage.reactivesqlite.annotations.Indexed;

@Table(name = "products")
public class Product {

    @PrimaryKey(autoIncrement = true)
    public long id;

    @Column(name = "product_name")
    public String name;

    @Indexed
    public double price;

    public int stock;
}
```

2. Initialize the ORM

```java
ReactiveSQLite db = ReactiveSQLite.with(this)
    .database("store.db")
    .register(Product.class)
    .build();
```

Or, with automatic model discovery:

```java
ReactiveSQLite db = ReactiveSQLite.with(this)
    .database("store.db")
    .registerAll("com.yourpackage.models")
    .build();
```

3. Use CRUD

```java
Product p = new Product();
p.name = "Tornillo";
p.price = 1.5;
p.stock = 100;

db.insert(p);           // p.id is now populated

p.price = 1.75;
db.update(p);

Product found = db.findById(Product.class, p.id);

List<Product> all = db.findAll(Product.class);

db.delete(p);
```

---

Query Builder

```java
List<Product> cheap = db.query(Product.class)
    .where("price").lt(10.0)
    .and("stock").gt(0)
    .orderBy("price").asc()
    .limit(50)
    .list();

long count = db.query(Product.class)
    .where("name").contains("tornillo")
    .count();

Product first = db.query(Product.class)
    .where("price").gte(100.0)
    .first();
```

Supported operators: eq, neq, gt, gte, lt, lte, like, notLike, startsWith, endsWith, contains, between, in, notIn, isNull, isNotNull.

---

Transactions

```java
db.transaction(() -> {
    db.insert(productA);
    db.insert(productB);
    db.update(productC);
});
```

Any exception inside the block triggers a ROLLBACK. On success, COMMIT.

Transactions with a return value:

```java
Sale sale = db.transaction(() -> {
    product.stock -= quantity;
    db.update(product);
    Sale s = new Sale();
    s.productId = product.id;
    s.quantity = quantity;
    return db.insert(s);
});
```

---

Relations

```java
@Table(name = "customers")
public class Customer {
    @PrimaryKey
    public long id;

    public String name;

    @OneToMany(target = Order.class, mappedBy = "customerId")
    public List<Order> orders;
}

@Table(name = "orders")
public class Order {
    @PrimaryKey
    public long id;

    @Indexed
    public long customerId;

    @ManyToOne(target = Customer.class)
    public Customer customer;

    public double total;
}
```

Load relations individually or in batch:

```java
// Single entity
Customer c = db.findById(Customer.class, 1L);
db.loadRelations(c);

// Batch (avoids N+1)
List<Customer> customers = db.findAll(Customer.class);
for (Customer c : customers) db.loadRelations(c);

// Or via query
List<Customer> customers = db.query(Customer.class)
    .include("orders")
    .list();
```

---

Live Queries

Reactive queries notify subscribers when the underlying table changes.

```java
LiveQuery<List<Product>> live = db.query(Product.class)
    .where("stock").gt(0)
    .observe();

Subscription sub = live.subscribe(products -> {
    // Called immediately with current data
    // Called again whenever products table changes
    runOnUiThread(() -> adapter.submitList(products));
});

// Later
sub.cancel();
```

Live queries are backed by LiveQueryManager, which listens to insert/update/delete events emitted by the engine.

---

ReactiveSQLiteManager + WebView

For hybrid apps, ReactiveSQLiteManager wires the ORM to a WebView and exposes it as window.ReactiveSQLite.

```java
ReactiveSQLiteManager manager = new ReactiveSQLiteManager(activity, webView);
manager.initialize();
```

From JavaScript:

```javascript
// Insert
await ReactiveSQLite.insert("product", { name: "Tornillo", price: 1.5 });

// Query
const products = await ReactiveSQLite.query("product")
    .where("price").lt(10)
    .orderBy("price").asc()
    .list();

// Transactions
await ReactiveSQLite.transaction([
    { type: "insert", model: "product", data: { name: "A", price: 1 } },
    { type: "insert", model: "product", data: { name: "B", price: 2 } }
]);

// Live query
const live = ReactiveSQLite.query("product").live();
live.subscribe(products => {
    console.log(products);
});
```

Custom commands

Expose domain-specific handlers to JavaScript without polluting the ORM:

```java
manager.registerCommandHandlers(db -> {
    db.registerHandler("checkout", params -> {
        // Business logic lives here, not in the ORM
        return JsonEngine.toJson(result);
    });
});
```

```javascript
await ReactiveSQLite.execute({
    method: "customCommand",
    params: { command: "checkout", data: { cartId: 42 } }
});
```

---

Backup & Restore

Export the database to Downloads/<AppName>/ without requesting storage permissions:

```java
String fileName = DatabaseExporter.exportSimple(context, databasePath);
```

Import a .db file selected via the Storage Access Framework:

```java
DatabaseFileImportController controller =
    new DatabaseFileImportController(activity, webView, manager);
// Exposes window.SQLiteFileImport.pickAndImport()
```

---

Design Principles

1. Domain-agnostic. The ORM knows entities, not business concepts.
2. Zero dependencies. Android SDK + Java standard library.
3. Reflection-based. No annotation processing, no code generation.
4. Small. The entire library fits in a dozen files.
5. Reactive. Live queries observe data changes without polling.
6. Parameterized SQL. Every value goes through a placeholder.
7. Real transactions. A single SQLiteDatabase handle per operation scope.
8. WebView-native. The bridge is optional but first-class.

---

What ReactiveSQLite Is Not

· It is not a replacement for Room when you need compile-time verified queries and Jetpack integration.
· It is not a full ORM with lazy proxies, identity maps, or change tracking.
· It does not manage migrations beyond CREATE TABLE IF NOT EXISTS. Schema evolution must be handled explicitly.
· It does not include a JavaScript runtime. The WebView bridge targets apps that already ship a WebView.

---

Roadmap

· Explicit, versioned migrations
· Composite primary keys
· @ManyToMany relations
· Kotlin DSL
· Kotlin coroutines integration for live queries
· Optional coroutine-based async CRUD

---

Contributing

Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

When contributing, please:

· Keep the ORM domain-agnostic.
· Avoid new external dependencies.
· Preserve the package-visible API of SQLiteEngine.
· Add tests when possible (a plain JUnit module is enough).

---

License

MIT License.

```
Copyright (c) 2024 <your name or organization>

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

Acknowledgments

ReactiveSQLite was designed for hybrid Android apps that pair a native persistence layer with a WebView frontend. Its architecture intentionally mirrors the shape of the .NET port (ReactiveSQLite.NET) so that the same mental model — entity → repository → query → live query → SQLite — works across platforms.

If you find it useful, a ⭐ on the repository is appreciated.
