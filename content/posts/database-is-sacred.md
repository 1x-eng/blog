---
title: "Database Is Sacred"
date: 2023-07-06T20:07:09+10:00
draft: false
---

# Database is Sacred: An "Engineer's" Testament

What does the word 'database' make you think of? It's probably difficult to generalize, but I'm guessing a good chunk of you would think of a 'relational' flavor. Well, there's no shortage of 'flavors' among databases today, and I feel like talking — or venting — about how often relational databases are used beneath their potential today

## (Relational) Databases are awesome

My (not so distant) future self will probably negate this title😁 but, for the point of this blog or the context in my head today, yes - relational databases are awesome. They are all fine and strong enough to do what you need them to do: store and manipulate data. Yes, I said manipulate.

During the previous decade or two, the development of new frameworks and ORMs made us treat databases as “bags to store data in”. Most tutorials are one-dimensional, wanting to put all business logic in the same place (your backend/api layer).Sometimes this works fine, sometimes it is slower than a pack of snails. This slowdown doesn’t happen immediately, and we usually figure it out only when it is too late to revert poor decisions. You write a function that processes some simple data and stores the result in the database while also displaying it to the user. Fifty change requests later, we have 500 lines of code, doing nested loops through data to calculate the tax amount. And the function that completed the job in a millisecond now takes 20 minutes to run.

Picture this: a blank canvas, a fresh project, the excitement of architecting a brand new system. Two schools of thought emerge: database-driven development (let's call this DbDD, becuase DDD would upset domain 😬) versus application code-driven development (and, let's call this ACDD. Porque no). And, ofcourse there is 'full stack developers' right? (coughs, javascript). I did some digging and apparently, the idea of a "full stack" developer has been around since the early 2000s, and it's steadily gaining momentum as a standard in the startup sphere. With nearly a [third of developers](https://survey.stackoverflow.co/2023/#developer-profile-developer-roles) considering themselves full stackers, the term has become one of the most popular hiring buzzwords of the decade. Again, tough to generalise but I would argue that this concept is nebulous and is potentially one of the reasons for 'tech debt' as that fresh project/idea starts to become real. And, here's my attempt to shed some light on how frequently, in the name of delivering rapid value our 'full stack developers' cut corners and lead that 'brand new system' into (pardon my french) a pile of garbage. 

*Many* (take this with a grain of salt, I know its wrong to generalise) full stack developers get lured by the hypnotic allure of ACDD, it's the shiny new car that entices with the promise of speed. With its application-first, think-later approach, it boasts an illusion of rapid development. The element thats surprisingly overlooked here - a database comes with many packed features, ready to use. But, nooooooo its the 'higher order functions' that takes precedence ¯\_(ツ)_/¯ Here's my take on how we let it slip.

## Schema is a thing

I'll talk through an arbitrary example of an e-commerce startup (Its always e-commerce, in examples. somehow ¯\_(ツ)_/¯). When designing an application schema, it's all too easy to take the path of least resistance with ACDD, defining a users and orders table, and then throwing in some JSON columns for flexibility:

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

While this approach gives you the illusion of progress, it'll soon lead to a dead-end. JSON fields in PostgreSQL (😫) aren't as performant for querying as proper relational tables. Searching within JSON fields, filtering data, or maintaining consistency and integrity within a JSON field can quickly become a nightmare😵.

('full stack development' is all about embracing javascript, right? In that spirit, I'll consider typescript for application code examples here.) The ACDD approach forces you to handle complex and error-prone parsing and validating of JSON fields:

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

Without a contract to JSON column, nothing stops a developer from inserting inconsistent data, such as an item without a quantity or even non-item data:

```json
[
  { "itemId": 3 },
  { "itemId": 4, "quantity": "two" },
  { "message": "This is not an item." }
]
```

Inconsistencies here = run time errors. But, 'full stack dev' wants to handle this in application layer, might look like this - 

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

That, right there - is the first turn towards 'pile of garbage'. Point is, before assigning `JSON` or worse `TEXT` type to house unstructured data in a table of your relational database, consider if thats your cue for adopting [normalisation](https://en.wikipedia.org/wiki/Database_normalization)

What could and should be the alternative approach then? (Looks at DbDD) Simple - use your database to its strength. In this case, normalise instead of relying on `JSON` type.

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

Unintentionally, I think ORMs are understood as an option for not touching SQL😠. Don't get me wrong, ORM's (especially mature products like Hibernate / Entity framework) are great, handling a lot of underlying complexity and security. But that too; like everything else is a 'tool'. A tool that is meant to be used for a task/set of tasks and unfortunately not a magic wand.

Some frameworks (looking at Ruby on Rails) even go to an extent of being database agnostic, where you could easily work on multiple database systems in different environments. Think about having sqlite for your development and test environments, and Postgres as your deployment environment. This too, imho is a turn that leads to a 'garbage of pile'. I am a proponent of embracing data layer to its strength, abstracting it would be a mistake leading to loss of some of the most important features a database has to offer.

While various ORMs abstracted the SQL away from developers, it (SQL) became some esoteric super language (it isn’t) that was reserved for wizards (it’s not). SQL is super simple and easy to learn. What it allows us is manipulating the data at the source. 

When you are writing your data processing on the database itself, you can think less about N+1 issues or memory bloats because you are instantiating every record under the sun to sum a few columns on them. On the contrary, ACDD often relies on the application layer for processing data. Continuing from ecommerce example, If you need to find the top-selling items, you might fetch all the orders and items into your application and then process them using loops and conditionals:

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

Here, we're looping over all `order_contents` and maintaining a count of `itemId` quantities. This approach involves excessive data transfer from the database to the application, relatively high memory usage, and increased CPU usage due to application-side processing.

That whole typescript logic boils down to this single SQL query - 

```sql
SELECT oi.item_id, SUM(oi.quantity) as total_quantity 
FROM order_items oi
GROUP BY oi.item_id
ORDER BY total_quantity DESC
LIMIT 5;
```

This SQL query performs aggregation on the database side, returning only the final result to the application. This leads to a much more efficient use of resources. The corresponding application code becomes incredibly simple:

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

Migrations are an essential part of every database-centric application. They represent the changes made to your database over time. They are the source of truth for your database schema, and their importance cannot be overstated. IMHO, there are 3 crucial aspects to the migration process: auditability, avoiding fragmentation & accounting for source of truth. 

#### Auditability
Auditability means that every change to the database schema is tracked in the form of migrations, which are version-controlled. You should be able to answer the questions like "Who made a particular change?" and "When was this change made?" by looking at the migrations.

The primary role of migrations is to encapsulate the changes made to the database schema. But, there's more to it - migrations should also be responsible for handling data changes within a migration.

To highlight the importance of auditability, let's look at an anti-pattern where a developer manually adds a new column to the database without a migration, but by connecting to the database directly 😈:

```sql
ALTER TABLE orders ADD COLUMN delivery_address TEXT;
```

While this change is quick and easy, it introduces a host of problems. There is no record of who made the change, when it was made, and why it was made. When deploying the application to a new environment, the new column will be missing, potentially leading to runtime errors. 

This change should instead be a migration (chose a migration agent of your choice, my current personal preference is [sqitch](https://sqitch.org/)) with every deploy having a corresponding revert, tracked in source control.

#### Avoiding Fragmentation
Migration fragmentation refers to the practice of having many small, piecemeal migrations that each make minor changes to the database. This practice can lead to a disorganized codebase and can make it difficult to understand the evolution of the database schema.

Consider an example where a developer creates three separate migrations to add a new table, add a column to it, and then add an index to that column where these changes are spread across multiple migrations in filesystem, potentially combined with other unrelated changes:

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

Migrations are the source of truth for your database schema. They define the state of your database at any point in time, and the state should be the same regardless of the environment (development, staging, production).

To maintain the source of truth, migrations need to be deterministic and idempotent. Deterministic means the migration will produce the same result given the same input, and idempotent means you can run the migration multiple times without changing the result beyond the initial application.

Consider an example where a developer creates a migration to add a unique index to a column:

```sql
CREATE UNIQUE INDEX idx_users_email ON users(email);

```

If the users table contains duplicate email addresses, this migration will fail. Even worse, if the migration is run on a table without duplicate emails, then run again after duplicates have been introduced, the results will be inconsistent.

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

Transactions are another crucial aspect of working with a database that's often overlooked in the ACDD approach. If an operation involves multiple steps that must all succeed or fail together, they should be part of a single transaction to ensure database remains in a consistent/predictable state. Handling data consistently is paramount in any database-centric application. ACID (Atomicity, Consistency, Isolation, Durability) transactions are the mechanism to achieve this. 

This is better explained with a banking/payments example than an e-commerce (or, payments was my thing, so I'll take a payments example. My blog, right 😎)

Imagine a banking system where you need to transfer money from Account A to Account B. This operation consists of two separate actions: reducing the balance of Account A and increasing the balance of Account B.
 
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

But what if the operation fails right after the first query 💩? Say, because of a server crash, network failure, or an error in the second query? Leads to a situation where the balance was reduced from Account A, but not added to Account B. This is a data integrity nightmare: the sum of all balances in the system has suddenly decreased, which in a real-world banking system would mean money has simply disappeared!💫

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
Here, we begin a transaction using the BEGIN command, then try to perform the two actions. If there's any error (like insufficient funds in the source account, or a network error), we catch it and rollback the transaction using the ROLLBACK command. This undoes all changes made within the transaction, leaving our database in a consistent state.

If both actions succeed, we commit the transaction using the COMMIT command. This makes the changes made in the transaction permanent. Regardless of any subsequent failure, the money will be correctly and atomically transferred from Account A to Account B, maintaining the integrity of our data.

> 💥 Food for thought: Data integrity is pivotal, interactions with database must account for ACID compliance & transactions are your friends.

## Stored Procedures: Server-side Powerhouse

Another underutilized feature of databases are stored procedures and functions. With ACDD, developers might shun them and favor performing the corresponding logic in application code instead.

Say there's a requirement to update the stock quantity after an order is placed. An ACDD developer might handle it this way:

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

Here, the transaction logic and error handling are explicitly done in the application layer. While it gives flexibility, it increases the application's complexity😵.

In contrast, the DbDD approach could tap on the shoulders of `stored procedures`. The above transaction can be handled in a stored procedure like this:

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

Manipulating data for specific presentation needs can lead to verbose and messy code in your application layer if not managed correctly. This can often be the case when an ACDD approach is taken. Let me unwrap this with e-commerce example:

Suppose we want to retrieve a summary of each user's orders, which includes the total number of orders placed, total quantity of items ordered, and the total amount spent.

An ACDD approach might fetch all the orders for a user, then loop through them and reduce the orders array to calculate these totals. This results in substantial data processing in the application code:

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

 Another common downfall of ACDD architectures is over-reliance on the application layer to perform heavy computational tasks. While the convenience of ORM libraries may tempt you to handle everything in your application layer, this can lead to performance bottlenecks and less maintainable code.

 Considering another e-commerce example, lets say we want a detailed report of top-selling products in the last month. This report includes the product's ID, name, total sales count, total revenue, and the average order value for each product.

ACDD approach might look like this - 

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
This is another turn towards `pile of garbage' becuase, this accounts for performance overhead, increased load time during peak traffic and this code needs to be 'maintained'.

What this could instead be? Offloading computational tasks to the database, particularly for non-real-time, heavy data processing tasks. This is where PostgreSQL's materialized views shine. In contrast to regular views, materialized views store their result set as a physical table in the database and can be indexed for faster access. Here's how we could implement the same report using a materialized view:

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

What this could instead be? In DbDD approach, we could use the database's inherent capabilities to maintain data integrity. In PostgreSQL, we can achieve the desired behavior using rules. Let's create a rule that automatically inserts into the audit_log table whenever a new row is inserted into the orders table:

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
### Embracing Indices:
Indexing is a database technique that speeds up data retrieval operations on a database table. It works much like the index at the end of a book, allowing the database to find data without scanning the entire table. Here are some crucial points to consider when using indices effectively:

1. Identifying Frequent or Slow Queries: Indices are not a one-size-fits-all solution, and they should be tailored to your specific use-cases. One common approach is to look at your application logs to identify which queries are run most frequently and which ones take the longest time to execute. Adding indices that cater to these specific operations can greatly enhance your application's performance.

2. Understanding Data Distribution: The effectiveness of an index depends on the distribution of data in your table. If a column has a high degree of variance (i.e., the data in the column is very diverse), an index on that column can significantly speed up query performance. However, if the data is very homogeneous (i.e., there are many repeating values), the benefits of indexing are reduced.

3. Partial Indexes for Specific Queries: PostgreSQL supports the creation of partial indices, where an index is built on a subset of data that meets a certain condition. This can be a significant advantage when your queries usually involve a specific subset of data.

4. Multicolumn Indices: In PostgreSQL, you can create an index on multiple columns. This can be advantageous when your queries often involve filtering or sorting on these columns together. However, the order of columns in this index matters as it affects the queries that the index can speed up.

5. Index Maintenance and Overhead: While indices can speed up data retrieval, they come with some overhead. They consume additional disk space and make write operations slower because each insert, update, or delete operation also needs to update the index. Therefore, it's important to find a balance and not over-index your tables.

### Utilizing Table Partitioning:
Table partitioning is a technique where a table is split into smaller, more manageable pieces, and each piece of data is inserted into the appropriate partition. This approach can greatly improve performance, ease maintenance, and provide quicker query execution by allowing the system to scan fewer data. Here are some key points on when and where to use table partitioning:

1. Large Tables: Partitioning is most effective for tables that are large in size and would otherwise take a long time to query.

2. Defined Access Patterns: If the table data has a column that is often used in where clauses and has a good distribution of values, it could be a good candidate for partitioning.

3. Age of Data: If older data in the table is infrequently accessed, then range partitioning on a date/time column can be very effective. This also allows for efficient implementation of data retention policies, where old partitions can be easily dropped.

4. Maintenance Operations: Partitioning can speed up maintenance operations. For example, it's much faster to drop a partition (which is a metadata operation) than to delete rows from a table (which requires individual row deletion and generates undo and redo logs).

### Leveraging FTS instead of LIKE

Full-text search is a technique for searching text content. It goes beyond the capabilities of traditional pattern-matching commands like LIKE and ILIKE to offer a more powerful and flexible toolset for text-based searches. Here are a few pointers on effective use:

1. Natural Language Search: Full-text search interprets the search text as a natural language query, considering language-specific features like stemming, synonyms, etc., to provide more relevant results.

2. Relevance Ranking: Results can be ranked not just on whether they match the search terms, but on how closely they match. You can customize the ranking function based on your needs.

3. Complex Query Capabilities: Full-text search supports complex queries including Boolean operators, phrase searching, and proximity searching.

4. Performance: Full-text search can be much faster than LIKE and ILIKE for large text fields as it uses a tsvector data type for efficient text analysis.

5. Customizable Parsing and Indexing: Full-text search allows for customization in the way text is parsed and indexed, including support for different languages and configurations.


## Conclusion
Well, firstly, that was a bit of a brain dump & the completion of a venting session 😤 

Embracing Database-Driven Development means seeing your database as more than a storage system — it's a high-performance engine for data processing, a gatekeeper of your business logic, a time machine through your application's history, and much more. Database-driven development encourages you to tap into the full potential of your database system, be it PostgreSQL or any other robust RDBMS. From harnessing the power of SQL, migrations, and stored procedures to mastering the art of performance monitoring and optimization, DbDD encompasses a wide spectrum of practices and concepts that, when used correctly, can elevate your application architecture significantly.

Racing to 'complete' a full-stack application? Take a step back and PLEASE think before you put on your 'full stack developer' hat. Like everything else in real life - 'building strong foundations' make or break your project/product in the long run. And there is nothing more foundational than your database, in the context of software systems. Treat your database as if its sacred, and not a dumping ground 😏.















