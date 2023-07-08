---
title: "Database Is Sacred"
date: 2023-07-06T20:07:09+10:00
draft: false
---

# Database is Sacred: An "Engineer's" Testament

What does the word 'database' make you think of? It's probably difficult to generalize, but I'm guessing a good chunk of you would think of a 'relational' flavor. Well, there's no shortage of 'flavors' among databases today, and I feel like talking — or venting — about how often relational databases are used beneath their potential today

## (Relational) Databases are awesome

My (not so distant) future self will probably negate this title😁, but for the purpose of this blog or the context in my head today, yes - relational databases are awesome. They are sturdy and efficient enough to do what you need them to do: store and manipulate data. Yes, I said manipulate.

During the past decade or two, the development of new frameworks and ORMs has led us to treat databases as "bags to store data in." Most tutorials are one-dimensional, urging us to put all business logic in the same place (your backend/API layer). Sometimes this works fine; other times, it is slower than a pack of snails. This slowdown doesn’t happen immediately, and we usually figure it out only when it's too late to revert poor decisions. You write a function that processes some simple data, stores the result in the database, and displays it to the user. Fifty change requests later, we have a 500-line function, doing nested loops through data to calculate the tax amount. And the function that once completed the job in a millisecond now takes 20 minutes to run.

Picture this: a blank canvas, a fresh project, the excitement of architecting a brand-new system. Two schools of thought emerge: database-driven development (let's call this DbDD, because DDD would upset the domain 😬) versus application code-driven development (and let's call this ACDD. Porque no). And of course, there are 'full stack developers,' right? (coughs, JavaScript). I did some digging and apparently, the idea of a "full stack" developer has been around since the early 2000s, and it's steadily gaining momentum as a standard in the startup sphere. With nearly a [third of developers](https://survey.stackoverflow.co/2023/#developer-profile-developer-roles) considering themselves full-stackers, the term has become one of the most popular hiring buzzwords of the decade. Again, it's tough to generalize, but I would argue that this concept is nebulous and is potentially one of the reasons for 'tech debt' as that fresh project/idea starts to become real. And here's my attempt to shed some light on how, in the name of delivering rapid value, our 'full stack developers' often cut corners and lead that 'brand-new system' into becoming (pardon my French) a pile of garbage.

*Many* (take this with a grain of salt; I know it's wrong to generalize) full-stack developers are lured by the hypnotic allure of ACDD. It's the shiny new car that entices with the promise of speed. With its application-first, think-later approach, it boasts an illusion of rapid development. The element that's surprisingly overlooked here is that a database comes with many pre-packed features, ready to use. But, no, it's the 'higher-order functions' that take precedence ¯\_(ツ)_/¯. Here's my take on how we let it slip.

## Schema is a thing

I'll talk through an arbitrary example of an e-commerce startup (it's always e-commerce in examples, somehow ¯\_(ツ)_/¯). When designing an application schema, it's all too easy to take the path of least resistance with ACDD, defining users and orders tables, and then throwing in some JSON columns for flexibility:

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT NOT NULL,
    order_content JSON NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```
We might expect the `order_content` to be an array of items, each with an itemId and quantity:

```json
[
  { "itemId": 1, "quantity": 3 },
  { "itemId": 2, "quantity": 2 }
]
```

While this approach gives you the illusion of progress, it will soon lead to a dead-end. JSON fields in PostgreSQL (😫) aren't as performant for querying as proper relational tables. Searching within JSON fields, filtering data, or maintaining consistency and integrity within a JSON field can quickly become a nightmare😵.

('Full-stack development' is all about embracing JavaScript, right? In that spirit, I'll use TypeScript for the application code examples here.) The ACDD approach forces you to handle complex and error-prone parsing and validation of JSON fields:

```typescript
interface OrderContent {
  itemId: number;
  quantity: number;
}

async function handleNewOrder(userId: number, orderContent: OrderContent[]) {
  const orderContentJson = JSON.stringify(orderContent);

  await db.query(
    "INSERT INTO orders (user_id, order_content) VALUES ($1, $2)",
    [userId, orderContentJson]
  );
}
```

Without a contract for the JSON column, nothing stops a developer from inserting inconsistent data, such as an item without a quantity or even non-item data:

```json
[
  { "itemId": 3 },
  { "itemId": 4, "quantity": "two" },
  { "message": "This is not an item." }
]
```

Inconsistencies here lead to runtime errors. However, a 'full-stack dev' who wants to handle this in the application layer might approach it like this:

```typescript
interface OrderContent {
  itemId: number;
  quantity: number;
}

async function handleNewOrder(userId: number, orderContent: any[]) {
  // Validate orderContent
  if (!Array.isArray(orderContent)) {
    throw new Error("Invalid orderContent: must be an array.");
  }

  const validatedOrderContent: OrderContent[] = [];

  for (const item of orderContent) {
    if (typeof item.itemId !== 'number' || typeof item.quantity !== 'number') {
      throw new Error("Invalid item: must have an itemId and a quantity.");
    }

    validatedOrderContent.push(item);
  }

  const orderContentJson = JSON.stringify(validatedOrderContent);

  await db.query(
    "INSERT INTO orders (user_id, order_content) VALUES ($1, $2)",
    [userId, orderContentJson]
  );
}
```

That right there is the first turn towards a 'pile of garbage'. The point is, before assigning `JSON` or, worse, `TEXT` type to house unstructured data in a table of your relational database, consider if that's your cue to adopt [normalization](https://en.wikipedia.org/wiki/Database_normalization).

So, what could and should be the alternative approach? (Looks at DbDD). Simple - use your database to its strength. In this case, normalize instead of relying on the `JSON` type.


```sql
CREATE TABLE IF NOT EXSITS orders (
    id SERIAL PRIMARY KEY,
    user_id INT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXSITS items (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price NUMERIC(6,2) NOT NULL,
    category_id INT NOT NULL
);

CREATE TABLE IF NOT EXSITS order_items (
    order_id INT NOT NULL,
    item_id INT NOT NULL,
    quantity INT NOT NULL,
    PRIMARY KEY(order_id, item_id),
    FOREIGN KEY (order_id) REFERENCES orders (id) ON DELETE CASCADE,
    FOREIGN KEY (item_id) REFERENCES items (id) ON DELETE CASCADE
);
```

> 💥 Food for thought: Don't overlook modelling schema and rush to 'dump' data that you can handle 'later'. 

## SQL - Embrace it, way around it is chaotic

Unintentionally, I think ORMs are often perceived as a way to avoid touching SQL 😠. Don't get me wrong, ORMs (especially mature products like Hibernate or Entity Framework) are great, handling a lot of underlying complexity and security. But those too, like everything else, are 'tools'. A tool is meant to be used for a task or set of tasks, and unfortunately, it's not a magic wand.

Some frameworks (looking at Ruby on Rails) even go to the extent of being database agnostic, where you could easily work on multiple database systems in different environments. Consider having SQLite for your development and test environments, and PostgreSQL as your deployment environment. This approach, in my humble opinion, is another turn that leads to a 'pile of garbage'. I am a proponent of embracing the data layer for its strengths; abstracting it would be a mistake leading to the loss of some of the most important features a database has to offer.

While various ORMs have abstracted SQL away from developers, SQL has become some esoteric super language (which it isn't) that's reserved for wizards (which it's not). SQL is super simple and easy to learn. What it allows us to do is manipulate the data at the source. 

When you are writing your data processing on the database itself, you can think less about N+1 issues or memory bloats because you're instantiating every record under the sun to sum a few columns on them. On the contrary, ACDD often relies on the application layer for processing data. Continuing from the e-commerce example, if you need to find the top-selling items, you might fetch all the orders and items into your application and then process them using loops and conditionals:


```typescript
interface ItemCount {
  itemId: number;
  count: number;
}

async function getTopSellingItems(): Promise<ItemCount[]> {
  const res = await db.query("SELECT order_content FROM orders");
  const orderContents: OrderContent[][] = res.rows.map((row) => JSON.parse(row.order_content));

  const itemCounts: { [itemId: number]: number } = {};

  for (const orderContent of orderContents) {
    for (const { itemId, quantity } of orderContent) {
      if (!itemCounts[itemId]) {
        itemCounts[itemId] = 0;
      }

      itemCounts[itemId] += quantity;
    }
  }

  return Object.entries(itemCounts)
    .map(([itemId, count]) => ({ itemId: parseInt(itemId), count }))
    .sort((a, b) => b.count - a.count);
}
```

Here, we're looping over all `order_contents` and keeping a count of `itemId` quantities. This approach involves excessive data transfer from the database to the application, relatively high memory usage, and increased CPU usage due to application-side processing.

That entire TypeScript logic can be condensed into this single SQL query:

```sql
SELECT oi.item_id, SUM(oi.quantity) as total_quantity 
FROM order_items oi
GROUP BY oi.item_id
ORDER BY total_quantity DESC
LIMIT 5;
```

This SQL query performs aggregation on the database side, returning only the final result to the application, which leads to a much more efficient use of resources. The corresponding application code then becomes incredibly simple:

```typescript
interface ItemCount {
  itemId: number;
  count: number;
}

async function getTopSellingItems(): Promise<ItemCount[]> {
  const res = await db.query(
    "SELECT oi.item_id, SUM(oi.quantity) as total_quantity FROM order_items oi GROUP BY oi.item_id ORDER BY total_quantity DESC LIMIT 5"
  );

  return res.rows.map((row) => ({ itemId: row.item_id, count: row.total_quantity }));
}
```

> 💥 Food for thought: Perform data ops closest to the source - which is the database. Don't fall for fetch first, filter later approach.

## Migrations - Time machines are real

Migrations are an essential part of every database-centric application. They represent the changes made to your database over time and serve as the source of truth for your database schema. Their importance cannot be overstated. In my opinion, there are three crucial aspects to the migration process: auditability, avoidance of fragmentation, and accounting for the source of truth.

#### Auditability
Auditability implies that every change to the database schema is tracked in the form of migrations, which are version-controlled. You should be able to answer questions such as "Who made a particular change?" and "When was this change made?" by looking at the migrations.

While the primary role of migrations is to encapsulate the changes made to the database schema, there's more to it - migrations should also handle data changes within a migration.

To highlight the importance of auditability, let's consider an anti-pattern where a developer manually adds a new column to the database without a migration, but by connecting to the database directly 😈:

```sql
ALTER TABLE orders ADD COLUMN delivery_address TEXT;
```

While this change is quick and easy, it introduces a host of problems. There is no record of who made the change, when it was made, or why it was made. When deploying the application to a new environment, the new column will be missing, potentially leading to runtime errors.

This change should instead be a migration (choose a migration agent of your preference, my current personal preference is [sqitch](https://sqitch.org/)), with every deployment having a corresponding revert, tracked in source control.

#### Avoiding Fragmentation
Migration fragmentation refers to the practice of having many small, piecemeal migrations, each making minor changes to the database. This practice can lead to a disorganized codebase and can make understanding the evolution of the database schema difficult.

Consider an example where a developer creates three separate migrations to add a new table, add a column to it, and then add an index to that column. In this case, these changes are spread across multiple migrations in the filesystem, potentially combined with other unrelated changes:

```sql
-- Version: 1
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Version: 2
ALTER TABLE orders ADD COLUMN delivery_address TEXT;

-- Version: 3
CREATE INDEX idx_orders_delivery_address ON orders(delivery_address);
```

Choice of your migration agent plays a pivotal role in avoiding fragmentation. [Rework](https://sqitch.org/docs/manual/sqitch-rework/) in sqitch or [update](https://www.liquibase.org/get-started/quickstart) in liquibase are some great tools to help avoid fragmentation.

#### Maintaining source of truth

Migrations serve as the source of truth for your database schema. They define the state of your database at any given point in time, and this state should be the same regardless of the environment (development, staging, or production).

To maintain this source of truth, migrations need to be both deterministic and idempotent. Being deterministic means the migration will produce the same result given the same input. Being idempotent means you can run the migration multiple times without changing the result beyond the initial application.

Consider an example where a developer creates a migration to add a unique index to a column:

```sql
CREATE UNIQUE INDEX idx_users_email ON users(email);

```

If the users table contains duplicate email addresses, this migration will fail. What's even worse is if the migration is run on a table without duplicate emails, and then run again after duplicates have been introduced, the results will be inconsistent.

A better approach would be to handle potential duplicates within the migration:
```sql
DELETE FROM users
WHERE id IN (
  SELECT id
  FROM (
    SELECT id, ROW_NUMBER() OVER(partition BY email ORDER BY id) AS rnum
    FROM users
  ) t
  WHERE t.rnum > 1
);

CREATE UNIQUE INDEX idx_users_email ON users(email);
```
This migration first removes duplicate email addresses and then creates the unique index. The migration is deterministic (it always results in a table where email addresses are unique) and idempotent (it can be run multiple times without causing errors or inconsistencies).

> 💥 Food for thought: Schema (& potentially data only) changes to database are best placed to run as migrations. Migrations should be auditable, avoid fragmentation & be the source of truth.

## Transactions must be ACID

Transactions are another critical aspect of working with a database, and they're often overlooked in the ACDD approach. When an operation involves multiple steps, all of which must either succeed or fail together, they should be included in a single transaction. This ensures that the database remains in a consistent and predictable state. Consistently handling data is of paramount importance in any database-centric application, and ACID (Atomicity, Consistency, Isolation, Durability) transactions are the mechanism for achieving this.

This concept is better explained with a banking or payments example rather than an e-commerce example. Or maybe it's just that payments are my thing, so I'll go with a payments example. It's my blog, right? 😎

Imagine a banking system where you need to transfer money from Account A to Account B. This operation consists of two separate actions: deducting from the balance of Account A and crediting to the balance of Account B.

 
#### A Non-ACID Approach: The Road to Inconsistency

In a non-transactional, or non-ACID approach, these two actions would be performed separately:

```typescript
async function transferFunds(sourceId: number, targetId: number, amount: number) {
  // Reduce balance of source account
  await db.query(
    "UPDATE accounts SET balance = balance - $1 WHERE id = $2",
    [amount, sourceId]
  );

  // Increase balance of target account
  await db.query(
    "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
    [amount, targetId]
  );
}
```

But what happens if the operation fails right after the first query 💩? What if there's a server crash, a network failure, or an error in the second query? This could lead to a situation where the balance was deducted from Account A, but not credited to Account B. It's a data integrity nightmare: the total balance across the system has suddenly decreased. In a real-world banking system, this would mean that money has simply disappeared!💫

#### The ACID Approach: Ensuring Consistency

In an ACID transactional approach, we perform the two actions within a single transaction:

```typescript
async function transferFunds(sourceId: number, targetId: number, amount: number) {
  // Begin the transaction
  await db.query("BEGIN");

  try {
    // Reduce balance of source account
    const res1 = await db.query(
      "UPDATE accounts SET balance = balance - $1 WHERE id = $2 RETURNING balance",
      [amount, sourceId]
    );

    // If the source account doesn't have enough funds, throw an error
    if (res1.rows[0].balance < 0) {
      throw new Error("Insufficient funds");
    }

    // Increase balance of target account
    await db.query(
      "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
      [amount, targetId]
    );

    // Commit the transaction
    await db.query("COMMIT");
  } catch (error) {
    // If there's any error, rollback the transaction
    await db.query("ROLLBACK");
    throw error;
  }
}
```
Here, we initiate a transaction using the BEGIN command, then attempt to perform the two actions. If there's an error (such as insufficient funds in the source account, or a network issue), we catch it and roll back the transaction using the ROLLBACK command. This action undoes all changes made within the transaction, preserving the consistency of our database.

If both actions are successful, we commit the transaction using the COMMIT command. This step finalizes the changes made in the transaction. Regardless of any subsequent failures, the money will be correctly and atomically transferred from Account A to Account B, thereby maintaining the integrity of our data.


> 💥 Food for thought: Data integrity is pivotal, interactions with database must account for ACID compliance & transactions are your friends.

## Stored Procedures: Server-side Powerhouse

Another underappreciated feature of databases includes stored procedures and functions. With ACDD, developers often neglect these elements, preferring to execute corresponding logic within the application code instead.

Consider a scenario where there's a need to update the stock quantity after placing an order. An ACDD developer might approach this task as follows:


```typescript
interface Order {
  userId: number;
  items: Array<{itemId: number, quantity: number}>;
}

async function placeOrder(order: Order) {
  const client = await db.connect();

  try {
    await client.query('BEGIN');

    const orderId = await createOrder(client, order.userId);

    for (const item of order.items) {
      await createOrderItem(client, orderId, item.itemId, item.quantity);
      await updateStock(client, item.itemId, -item.quantity);
    }

    await client.query('COMMIT');
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;
  } finally {
    client.release();
  }
}
```

In this instance, the transaction logic and error handling are explicitly executed in the application layer. While this offers flexibility, it simultaneously adds to the complexity of the application.

On the other hand, a DbDD approach might leverage `stored procedures`. The aforementioned transaction could be managed through a stored procedure like this:


```sql
CREATE OR REPLACE PROCEDURE place_order(user_id INT, order_items_order_id INT, item_id INT, quantity INT)
LANGUAGE plpgsql
AS $$
BEGIN
  INSERT INTO orders(user_id) VALUES(user_id) RETURNING id INTO order_id;

  FOR item IN order_items LOOP
    INSERT INTO order_items(order_id, item_id, quantity) VALUES(order_id, item.item_id, item.quantity);
    UPDATE items SET stock = stock - item.quantity WHERE id = item.item_id;
  END LOOP;

  COMMIT;
EXCEPTION
WHEN OTHERS THEN
  ROLLBACK;
  RAISE;
END $$;
```

And, the app code becomes:

```typescript
async function placeOrder(order: Order) {
  await db.query('CALL place_order($1, $2, $3, $4)', [...]);
}
```

> 💥 Food for thought: By using stored procedures, the complexity of the transaction logic is moved into the database. Your application code becomes cleaner and more focused on its primary responsibilities.

## Views - They are cheap, but effective

If not carefully handled, manipulating data to meet specific presentation requirements can lead to bloated and disorganized code in your application layer. This scenario frequently arises when adopting an ACDD approach. To elaborate, well, here goes an e-commerce example, ...again:

Imagine we need to compile a summary of each user's orders, including the total number of orders placed, total quantity of items ordered, and the total amount spent.

Using the ACDD approach, one might retrieve all the orders for a user, loop through them, and reduce the orders array to calculate these totals. This process necessitates considerable data processing within the application code:


```typescript
interface Order {
  quantity: number;
  price: number;
}

async function getUserOrderSummary(userId: number) {
  const orders = await Order.findAll({
    where: { userId: { [Op.eq]: userId } }
  });

  let orderCount = 0;
  let totalQuantity = 0;
  let totalAmount = 0;

  orders.forEach(order => {
    orderCount++;
    totalQuantity += order.quantity;
    totalAmount += order.quantity * order.price;
  });

  return {
    orderCount,
    totalQuantity,
    totalAmount,
  };
}
```

This could instead be:

```sql
CREATE VIEW user_order_summary AS
SELECT 
  user_id, 
  COUNT(*) AS order_count, 
  SUM(quantity) AS total_quantity, 
  SUM(quantity * price) AS total_amount
FROM orders
GROUP BY user_id;
```

And in the application code, we could simply do this (irrespective of the ORM):

```typescript
async function getUserOrderSummary(userId: number) {
  const [result] = await db.query("SELECT * FROM user_order_summary WHERE user_id = $1", {
    replacements: [userId],
    type: db.QueryTypes.SELECT,
  });

  return result;
}
```

> 💥 Food for thought: Leverage views to account for navigating away from complex app code data wrangling.

## Mat. Views - Escaping the ORM overhead

Another frequent pitfall of ACDD architectures is an overemphasis on the application layer to carry out demanding computational tasks. Although the ease of ORM libraries might lure you into managing everything within your application layer, this practice can cause performance bottlenecks and lead to code that is difficult to maintain.

Let's consider another e-commerce example. Suppose we need a comprehensive report of the top-selling products over the past month. This report should include the product's ID, name, total sales count, total revenue, and the average order value for each product.

The ACDD approach might appear like this: 

```typescript
import { Sequelize, Op, literal } from "sequelize";

async function getTopSellingProducts() {
  const oneMonthAgo = new Date();
  oneMonthAgo.setMonth(oneMonthAgo.getMonth() - 1);

  // Fetch the products
  const products = await Product.findAll({
    include: [{
      model: OrderItem,
      include: [{
        model: Order,
        where: { createdAt: { [Op.gte]: oneMonthAgo } }
      }]
    }]
  });

  // Prepare the report
  const report = products.map(product => {
    const sales_count = product.orderItems.reduce((total, orderItem) => total + orderItem.quantity, 0);
    const total_revenue = product.orderItems.reduce((total, orderItem) => total + (orderItem.quantity * product.price), 0);
    const avg_order_value = total_revenue / sales_count;
    
    return { id: product.id, name: product.name, sales_count, total_revenue, avg_order_value };
  });

  // Sort and limit the report
  report.sort((a, b) => b.sales_count - a.sales_count);
  return report.slice(0, 10);
}
```
This approach marks another detour towards a 'pile of garbage' because it leads to performance overhead, extended load times during peak traffic, and burdensome maintenance requirements for the code.

What could be a better alternative? Offloading computational tasks to the database, especially for heavy data processing tasks that aren't time-sensitive. This is where the strength of PostgreSQL's materialized views comes into play. Unlike regular views, materialized views store their result set as a physical table in the database and can be indexed for faster access. Here's how we could implement the same report using a materialized view:

```sql
CREATE MATERIALIZED VIEW top_selling_products AS
SELECT 
  p.id, 
  p.name, 
  COUNT(*) as sales_count,
  SUM(oi.quantity * p.price) as total_revenue,
  AVG(oi.quantity * p.price) as avg_order_value
FROM 
  products p
JOIN 
  order_items oi ON oi.product_id = p.id
JOIN 
  orders o ON o.id = oi.order_id
WHERE 
  o.created_at >= NOW() - INTERVAL '1 month'
GROUP BY 
  p.id, p.name
ORDER BY 
  sales_count DESC
LIMIT 10;
```

Fetching the report becomes a straightforward, highly efficient operation:

```typescript
async function getTopSellingProducts() {
  return db.query("SELECT * FROM top_selling_products", {
    type: db.QueryTypes.SELECT,
  });
}
```

This alternative accounts for performance, data manipulation at source, increased user experience, less application code overhead.

> 💥 Consider materialized views computationally heavy tasks, refresh mat. views with a background task.

## Rules - becuase, they are good 😱

[The PostgreSQL rule system](https://www.postgresql.org/docs/current/rules.html) provides a powerful tool for automatic, database-level responses to specific events. However, developers using an ACDD approach often overlook these capabilities, instead opting to handle such actions in the application code. 

Imagine a scenario where, for auditing purposes, your application needs to keep a log of all order creation events. This audit log is a record in a separate audit_log table for each insertion into the orders table.

In an ACDD approach using an ORM like Sequelize, you might handle this within your application code:

```typescript
import { Sequelize, Transaction } from 'sequelize';

async function createOrder(userId: number, itemId: number, quantity: number) {
  const transaction = await sequelize.transaction();

  try {
    // Insert into the orders table
    await Order.create({
      userId: userId,
      itemId: itemId,
      quantity: quantity
    }, { transaction });

    // Insert into the audit_log table
    await AuditLog.create({
      action: 'INSERT',
      tableName: 'orders',
      userId: userId,
      actionTime: new Date()
    }, { transaction });

    await transaction.commit();
  } catch (error) {
    await transaction.rollback();
    throw error;
  }
}
```
Perfectly functional approach, but risks data integrity, potentially leading to reduced performance since the audit log insertion in the application layer adds an extra network round trup and a potential point of failure. 

A better alternative? In DbDD approach, we could use the database's inherent capabilities to maintain data integrity. In PostgreSQL, we can achieve the desired behavior using rules. Let's create a rule that automatically inserts into the audit_log table whenever a new row is inserted into the orders table:

```sql
CREATE OR REPLACE RULE order_audit_log AS
    ON INSERT TO orders
    DO INSERT INTO audit_log (action, table_name, user_id, action_time)
       VALUES ('INSERT', 'orders', NEW.user_id, NOW());
```

With this rule in place, inserting an order becomes a simple operation:

```typescript
async function createOrder(userId: number, itemId: number, quantity: number) {
  await db.query(
    "INSERT INTO orders (user_id, item_id, quantity) VALUES ($1, $2, $3)",
    [userId, itemId, quantity]
  );
}
```
Isn't that simplified application logic, guaranteed data integrity and performance?

## A few more often overlooked database features:
### Harnessing Indices:
Indexing is a database technique that accelerates data retrieval operations on a database table. Much like the index found at the end of a book, it allows the database to find data without scanning the entire table. Here are several crucial points to bear in mind for effective index use:

1. Pinpointing Frequent or Slow Queries: Indices aren't a universal solution, and they should be customized to your specific use-cases. A common strategy is to scrutinize your application logs to identify which queries are executed most frequently and which ones take an extended time to run. Adding indices to cater to these specific operations can significantly boost your application's performance.

2. Grasping Data Distribution: An index's effectiveness hinges on the data distribution in your table. If a column has a high degree of variance (i.e., the data in the column is extremely diverse), an index on that column can substantially quicken query performance. However, if the data is predominantly homogeneous (i.e., there are numerous repeating values), the advantages of indexing dwindle.

3. Using Partial Indexes for Specific Queries: PostgreSQL allows the creation of partial indices, where an index is built on a subset of data that meets a certain criterion. This can be a significant boon when your queries typically involve a specific subset of data.

4. Creating Multicolumn Indices: PostgreSQL enables the creation of an index on multiple columns. This can be beneficial when your queries frequently involve filtering or sorting on these columns simultaneously. However, the sequence of columns in this index matters as it influences the queries that the index can expedite.

5. Handling Index Maintenance and Overhead: While indices can hasten data retrieval, they entail some overhead. They consume extra disk space and make write operations slower because each insert, update, or delete operation also requires an index update. Therefore, it's crucial to strike a balance and avoid over-indexing your tables.

### Utilizing Table Partitioning:
Table partitioning is a strategy where a table is divided into smaller, more manageable segments, and each piece of data is inserted into the appropriate partition. This technique can significantly enhance performance, simplify maintenance, and provide faster query execution by allowing the system to scan less data. Here are some key insights on when and where to use table partitioning:

1. Sizeable Tables: Partitioning is most potent for tables that are large and would ordinarily take a substantial time to query.

2. Predetermined Access Patterns: If the table data features a column often used in where clauses and exhibits a good distribution of values, it could be an ideal candidate for partitioning.

3. Data Age: If older data in the table is infrequently accessed, then range partitioning on a date/time column can be extremely effective. This also facilitates the efficient implementation of data retention policies, where old partitions can be quickly dropped.

4. Maintenance Operations: Partitioning can accelerate maintenance operations. For instance, dropping a partition (which is a metadata operation) is much faster than deleting rows from a table (which necessitates individual row deletion and generates undo and redo logs).

### Preferring FTS over LIKE:

Full-text search (FTS) is a technique employed for searching text content. It transcends the capabilities of traditional pattern-matching commands like LIKE and ILIKE, offering a more potent and flexible toolkit for text-based searches. Here are several tips on effective use:

1. Natural Language Search: Full-text search interprets the search text as a natural language query, considering language-specific features such as stemming, synonyms, etc., to deliver more pertinent results.

2. Relevance Ranking: Results can be ranked not merely on whether they match the search terms, but on the degree of match. You can customize the ranking function to suit your needs.

3. Complex Query Capabilities: Full-text search supports intricate queries including Boolean operators, phrase searching, and proximity searching.

4. Performance: Full-text search can be much swifter than LIKE and ILIKE for extensive text fields as it uses a tsvector data type for efficient text analysis.

5. Customizable Parsing and Indexing: Full-text search allows customization in how text is parsed and indexed, including support for different languages and configurations.



## Conclusion
Well, firstly, that was a bit of a brain dump and an outlet for some pent-up frustration 😤. 

Embracing Database-Driven Development means viewing your database as more than just a storage system — it's a high-performance powerhouse for data processing, a sentinel for your business logic, a time capsule of your application's history, and so much more. Database-driven development urges you to unleash the full potential of your database system, whether it's PostgreSQL or another robust RDBMS. From harnessing the capabilities of SQL, migrations, and stored procedures to mastering the science of performance monitoring and optimization, DbDD covers a broad range of practices and concepts. When applied correctly, these can significantly enhance your application architecture.

Are you rushing to 'finish' a full-stack application? PLEASE, pause and reflect before donning your 'full stack developer' cap. As with everything else in real life, 'strong foundations' are pivotal to the long-term success of your project or product. And in the realm of software systems, nothing is more foundational than your database. Treat your database with reverence, not as a landfill 😏.
















