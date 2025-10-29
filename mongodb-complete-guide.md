# Complete MongoDB Tutorial: Comprehensive Practical Guide

## Table of Contents

1. [Introduction to MongoDB](#introduction-to-mongodb)
2. [What is MongoDB?](#what-is-mongodb)
3. [Why Use MongoDB?](#why-use-mongodb)
4. [MongoDB Architecture](#mongodb-architecture)
5. [Core Components](#core-components)
6. [Installing MongoDB](#installing-mongodb)
7. [MongoDB Basics](#mongodb-basics)
8. [Data Types in MongoDB](#data-types-in-mongodb)
9. [CRUD Operations](#crud-operations)
10. [Query Operations](#query-operations)
11. [Indexing](#indexing)
12. [Aggregation Framework](#aggregation-framework)
13. [Data Modeling](#data-modeling)
14. [Replication](#replication)
15. [Sharding](#sharding)
16. [Security - Authentication & Authorization](#security-authentication-authorization)
17. [Backup and Restore](#backup-and-restore)
18. [Performance Optimization](#performance-optimization)
19. [Monitoring](#monitoring)
20. [Advantages and Disadvantages](#advantages-and-disadvantages)
21. [Best Practices](#best-practices)

---

## Introduction to MongoDB

MongoDB is a leading NoSQL database that stores data in flexible, JSON-like documents. Unlike traditional relational databases that use tables and rows, MongoDB uses collections and documents, making it easier to store and work with complex, hierarchical data structures.

**Created:** 2009 by MongoDB Inc. (formerly 10gen)  
**Written in:** C++, JavaScript, Python  
**Type:** Document-oriented NoSQL database  
**License:** Server Side Public License (SSPL)

MongoDB has become one of the most popular databases in the world, especially for modern web applications, mobile apps, and data analytics platforms.

---

## What is MongoDB?

MongoDB is an open-source, document-oriented NoSQL database that provides high performance, high availability, and easy scalability. It works on the concept of collections and documents instead of tables and rows.

### Key Characteristics

- **Document-Oriented Storage**: Stores data in BSON (Binary JSON) format
- **Schema-less**: No predefined schema required
- **Scalable**: Horizontal scaling through sharding
- **High Performance**: Indexing and embedded data models
- **High Availability**: Replication with automatic failover
- **Flexible Query Language**: Rich query capabilities
- **Aggregation Framework**: Powerful data processing pipelines
- **GridFS**: Store and retrieve large files

### Document Structure

MongoDB stores data as documents in BSON format:

```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "name": "John Doe",
  "age": 30,
  "email": "[email protected]",
  "address": {
    "street": "123 Main St",
    "city": "New York",
    "zipcode": "10001"
  },
  "hobbies": ["reading", "gaming", "hiking"],
  "registered": ISODate("2024-01-15T10:30:00Z")
}
```

---

## Why Use MongoDB?

### Use Cases

MongoDB is perfect for:

1. **Content Management Systems**: Flexible schema for diverse content types
2. **Real-time Analytics**: Fast queries and aggregations
3. **IoT Applications**: Handle massive streams of sensor data
4. **Mobile Applications**: Easy synchronization and offline support
5. **E-commerce Platforms**: Product catalogs with varying attributes
6. **Social Networks**: User profiles, posts, and relationships
7. **Gaming Applications**: Player profiles and game state management
8. **Big Data Applications**: Process large volumes of unstructured data

### When to Use MongoDB

✅ **Use MongoDB when you need:**
- Flexible, evolving schemas
- High-volume data ingestion
- Horizontal scalability
- Complex hierarchical data structures
- Rapid application development
- Real-time analytics
- Geographic data queries

❌ **Don't use MongoDB when you need:**
- Complex multi-row transactions (use PostgreSQL)
- Strong ACID guarantees for all operations
- Complex joins across many tables
- Mature reporting tools (traditional BI tools work better with SQL)

---

## MongoDB Architecture

### High-Level Architecture

```
┌────────────────────────────────────────────────────────┐
│                   Application Layer                     │
│              (Node.js, Python, Java, etc.)             │
└─────────────────────┬──────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────┐
│                    MongoDB Driver                        │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────┐
│                   MongoDB Server                         │
│  ┌────────────┐  ┌──────────────┐  ┌─────────────────┐│
│  │  Query     │  │  Aggregation │  │  Index Manager  ││
│  │  Engine    │  │  Framework   │  │                 ││
│  └────────────┘  └──────────────┘  └─────────────────┘│
│  ┌──────────────────────────────────────────────────┐ │
│  │            Storage Engine (WiredTiger)            │ │
│  └──────────────────────────────────────────────────┘ │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────┐
│                  Data Storage (Disk)                     │
│              BSON Documents + Indexes                    │
└─────────────────────────────────────────────────────────┘
```

### Sharded Cluster Architecture

```
                    ┌──────────────┐
                    │ Application  │
                    └──────┬───────┘
                           │
                  ┌────────┴────────┐
                  │     Mongos      │
                  │  (Query Router) │
                  └────────┬────────┘
                           │
           ┌───────────────┼───────────────┐
           ↓               ↓               ↓
    ┌─────────┐     ┌─────────┐     ┌─────────┐
    │ Shard 1 │     │ Shard 2 │     │ Shard 3 │
    │ (RS)    │     │ (RS)    │     │ (RS)    │
    └─────────┘     └─────────┘     └─────────┘
           ↑               ↑               ↑
           └───────────────┼───────────────┘
                           │
                  ┌────────┴────────┐
                  │  Config Servers │
                  │   (Replica Set) │
                  └─────────────────┘
```

---

## Core Components

### 1. Database

A database is a physical container for collections. Each database has its own set of files on the file system.

```bash
# Show all databases
show dbs

# Create/Switch to database
use myDatabase

# Show current database
db

# Drop database
db.dropDatabase()
```

### 2. Collection

A collection is a group of MongoDB documents. It's equivalent to a table in relational databases, but doesn't enforce a schema.

```bash
# Create collection
db.createCollection("users")

# Show all collections
show collections

# Drop collection
db.users.drop()
```

### 3. Document

A document is a record in a MongoDB collection. Documents are composed of field-value pairs in BSON format.

```javascript
{
  _id: ObjectId("..."),
  field1: "value1",
  field2: 123,
  field3: ["array", "values"],
  field4: {
    nested: "document"
  }
}
```

### 4. Field

A field is a name-value pair in a document (similar to a column in relational databases).

### 5. _id Field

Every document must have an `_id` field that acts as a primary key. If not provided, MongoDB automatically generates one.

```javascript
_id: ObjectId("507f1f77bcf86cd799439011")
```

### 6. Storage Engine (WiredTiger)

The storage engine manages how data is stored in memory and on disk:

- **Document-level concurrency control**
- **Compression** (reduces storage footprint)
- **Checkpointing** (ensures data durability)
- **Journaling** (recovery from crashes)

---

## Installing MongoDB

### Installation on Windows

#### Step 1: Download MongoDB

1. Visit: https://www.mongodb.com/try/download/community
2. Select Version: Latest
3. Platform: Windows x64
4. Package: MSI

#### Step 2: Run the Installer

1. Double-click the `.msi` file
2. Choose "Complete" installation
3. Install MongoDB as a Windows Service
4. Install MongoDB Compass (optional GUI)

#### Step 3: Set Environment Variables

1. Open System Properties → Environment Variables
2. Edit Path variable
3. Add: `C:\Program Files\MongoDB\Server\7.0\bin`

#### Step 4: Create Data Directory

```cmd
md C:\data\db
```

#### Step 5: Start MongoDB

```cmd
# Start MongoDB service
net start MongoDB

# Or run manually
mongod --dbpath C:\data\db
```

#### Step 6: Verify Installation

```cmd
mongod --version
```

### Installation on Ubuntu/Linux

#### Step 1: Import MongoDB Public Key

```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg \
   --dearmor
```

#### Step 2: Create List File

```bash
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
```

#### Step 3: Update Package Database

```bash
sudo apt-get update
```

#### Step 4: Install MongoDB

```bash
sudo apt-get install -y mongodb-org
```

#### Step 5: Start MongoDB

```bash
# Start MongoDB
sudo systemctl start mongod

# Enable on boot
sudo systemctl enable mongod

# Check status
sudo systemctl status mongod
```

#### Step 6: Verify Installation

```bash
mongod --version
```

### Installation on macOS

#### Using Homebrew

```bash
# Install Homebrew (if not installed)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Tap MongoDB repository
brew tap mongodb/brew

# Install MongoDB
brew install mongodb-community@7.0

# Start MongoDB
brew services start mongodb-community@7.0

# Verify
mongod --version
```

### Docker Installation

```bash
# Pull MongoDB image
docker pull mongo:latest

# Run MongoDB container
docker run -d \
  --name mongodb \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=password123 \
  -v mongodb_data:/data/db \
  mongo:latest

# Access MongoDB shell
docker exec -it mongodb mongosh
```

### Docker Compose

```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:7.0
    container_name: mongodb
    restart: always
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password123
      MONGO_INITDB_DATABASE: mydb
    volumes:
      - mongodb_data:/data/db
      - mongodb_config:/data/configdb

volumes:
  mongodb_data:
  mongodb_config:
```

```bash
# Start
docker-compose up -d

# Stop
docker-compose down
```

### Installing MongoDB Shell (mongosh)

#### Windows

1. Download from: https://www.mongodb.com/try/download/shell
2. Extract ZIP file
3. Add to PATH: `C:\mongosh\bin`

#### Linux/macOS

```bash
# Ubuntu/Debian
wget https://downloads.mongodb.com/compass/mongodb-mongosh_2.1.0_amd64.deb
sudo dpkg -i mongodb-mongosh_2.1.0_amd64.deb

# macOS
brew install mongosh
```

#### Verify mongosh

```bash
mongosh --version
```

---

## MongoDB Basics

### Connecting to MongoDB

```bash
# Connect to local MongoDB
mongosh

# Connect to specific host and port
mongosh "mongodb://localhost:27017"

# Connect with authentication
mongosh "mongodb://username:password@localhost:27017/dbname"

# Connect to MongoDB Atlas
mongosh "mongodb+srv://cluster.mongodb.net/mydb" --username myuser
```

### Basic Shell Commands

```javascript
// Show all databases
show dbs

// Switch to database (creates if doesn't exist)
use myDatabase

// Show current database
db

// Show collections in current database
show collections

// Show users
show users

// Show roles
show roles

// Get help
help

// Database-specific help
db.help()

// Collection-specific help
db.collection.help()

// Clear screen
cls  // Windows
clear  // Linux/Mac
```

### Database Operations

```javascript
// Create database (auto-created when you insert data)
use testDB

// Drop current database
db.dropDatabase()

// Get database statistics
db.stats()

// Get database name
db.getName()
```

### Collection Operations

```javascript
// Create collection explicitly
db.createCollection("users")

// Create collection with options
db.createCollection("products", {
  capped: true,
  size: 100000,
  max: 5000
})

// Show all collections
show collections

// Drop collection
db.users.drop()

// Rename collection
db.users.renameCollection("customers")

// Get collection statistics
db.users.stats()
```

---

## Data Types in MongoDB

MongoDB supports various data types in BSON format:

### 1. String

```javascript
{ name: "John Doe" }
```

### 2. Number

```javascript
{
  age: 30,              // Integer (32-bit)
  price: 99.99,         // Double
  views: NumberLong(1000000),  // 64-bit integer
  decimal: NumberDecimal("19.99")  // 128-bit decimal
}
```

### 3. Boolean

```javascript
{ isActive: true, verified: false }
```

### 4. Array

```javascript
{
  tags: ["mongodb", "database", "nosql"],
  scores: [95, 87, 92]
}
```

### 5. Object (Embedded Document)

```javascript
{
  address: {
    street: "123 Main St",
    city: "New York",
    zipcode: "10001"
  }
}
```

### 6. ObjectId

```javascript
{
  _id: ObjectId("507f1f77bcf86cd799439011")
}
```

### 7. Date

```javascript
{
  createdAt: new Date(),
  updatedAt: ISODate("2024-01-15T10:30:00Z")
}
```

### 8. Null

```javascript
{ middleName: null }
```

### 9. Binary Data

```javascript
{ image: BinData(0, "base64encodedstring") }
```

### 10. Regular Expression

```javascript
{ pattern: /^test/i }
```

### 11. JavaScript Code

```javascript
{ func: function() { return 1; } }
```

### 12. Timestamp

```javascript
{ ts: Timestamp(1420070400, 1) }
```

### Data Type Examples

```javascript
// Insert document with various data types
db.samples.insertOne({
  _id: ObjectId(),
  username: "john_doe",                    // String
  age: 30,                                  // Number
  balance: NumberDecimal("1234.56"),        // Decimal
  isActive: true,                           // Boolean
  hobbies: ["reading", "gaming"],           // Array
  address: {                                // Object
    city: "New York",
    zip: "10001"
  },
  createdAt: new Date(),                    // Date
  metadata: null,                           // Null
  tags: ["user", "premium"]                 // Array
})
```

---

## CRUD Operations

CRUD stands for Create, Read, Update, and Delete operations.

### Create Operations

#### insertOne()

Insert a single document into a collection.

```javascript
// Syntax
db.collection.insertOne(document, options)

// Example
db.users.insertOne({
  name: "Alice Johnson",
  email: "[email protected]",
  age: 28,
  city: "San Francisco"
})

// Output
{
  acknowledged: true,
  insertedId: ObjectId("6501234abcdef...")
}
```

#### insertMany()

Insert multiple documents at once.

```javascript
// Syntax
db.collection.insertMany([document1, document2, ...], options)

// Example
db.users.insertMany([
  {
    name: "Bob Smith",
    email: "[email protected]",
    age: 35,
    city: "Seattle"
  },
  {
    name: "Carol Williams",
    email: "[email protected]",
    age: 42,
    city: "Boston"
  },
  {
    name: "David Brown",
    email: "[email protected]",
    age: 29,
    city: "Austin"
  }
])

// Output
{
  acknowledged: true,
  insertedIds: {
    '0': ObjectId("..."),
    '1': ObjectId("..."),
    '2': ObjectId("...")
  }
}
```

#### insert() (Deprecated)

```javascript
// Insert single document
db.users.insert({ name: "Test User" })

// Insert multiple documents
db.users.insert([
  { name: "User1" },
  { name: "User2" }
])
```

### Read Operations

#### find()

Retrieve multiple documents from a collection.

```javascript
// Syntax
db.collection.find(query, projection)

// Find all documents
db.users.find()

// Find with condition
db.users.find({ city: "New York" })

// Find with multiple conditions
db.users.find({ 
  age: { $gte: 25 }, 
  city: "San Francisco" 
})

// Pretty print
db.users.find().pretty()

// Limit results
db.users.find().limit(5)

// Skip results
db.users.find().skip(10)

// Sort results
db.users.find().sort({ age: 1 })  // 1 = ascending, -1 = descending

// Chaining methods
db.users.find({ age: { $gte: 25 } })
  .sort({ name: 1 })
  .limit(10)
  .skip(5)
```

#### findOne()

Retrieve a single document.

```javascript
// Syntax
db.collection.findOne(query, projection)

// Example
db.users.findOne({ email: "[email protected]" })

// Find by _id
db.users.findOne({ _id: ObjectId("650...") })
```

#### Projection

Select specific fields to return.

```javascript
// Include only name and email (exclude _id)
db.users.find({}, { name: 1, email: 1, _id: 0 })

// Exclude specific fields
db.users.find({}, { password: 0, ssn: 0 })

// Example
db.users.find(
  { age: { $gte: 30 } },
  { name: 1, email: 1, age: 1, _id: 0 }
)
```

### Update Operations

#### updateOne()

Update a single document.

```javascript
// Syntax
db.collection.updateOne(filter, update, options)

// Example: Update age
db.users.updateOne(
  { name: "Alice Johnson" },
  { $set: { age: 29 } }
)

// Example: Increment age
db.users.updateOne(
  { name: "Alice Johnson" },
  { $inc: { age: 1 } }
)

// Example: Add to array
db.users.updateOne(
  { name: "Alice Johnson" },
  { $push: { hobbies: "photography" } }
)
```

#### updateMany()

Update multiple documents.

```javascript
// Syntax
db.collection.updateMany(filter, update, options)

// Example: Update all users in New York
db.users.updateMany(
  { city: "New York" },
  { $set: { timezone: "EST" } }
)

// Example: Increment age for all users over 30
db.users.updateMany(
  { age: { $gte: 30 } },
  { $inc: { age: 1 } }
)
```

#### replaceOne()

Replace an entire document.

```javascript
// Syntax
db.collection.replaceOne(filter, replacement, options)

// Example
db.users.replaceOne(
  { name: "Alice Johnson" },
  {
    name: "Alice Johnson",
    email: "[email protected]",
    age: 30,
    city: "Los Angeles",
    phone: "555-0123"
  }
)
```

#### Update Operators

```javascript
// $set: Set field value
db.users.updateOne(
  { name: "Alice" },
  { $set: { email: "[email protected]" } }
)

// $unset: Remove field
db.users.updateOne(
  { name: "Alice" },
  { $unset: { phone: "" } }
)

// $inc: Increment value
db.users.updateOne(
  { name: "Alice" },
  { $inc: { age: 1, loginCount: 1 } }
)

// $mul: Multiply value
db.users.updateOne(
  { name: "Alice" },
  { $mul: { price: 1.1 } }
)

// $rename: Rename field
db.users.updateOne(
  { name: "Alice" },
  { $rename: { "name": "fullName" } }
)

// $min: Update if new value is less
db.users.updateOne(
  { name: "Alice" },
  { $min: { lowestScore: 50 } }
)

// $max: Update if new value is greater
db.users.updateOne(
  { name: "Alice" },
  { $max: { highestScore: 95 } }
)

// $currentDate: Set to current date
db.users.updateOne(
  { name: "Alice" },
  { $currentDate: { lastModified: true } }
)

// Array operators
// $push: Add to array
db.users.updateOne(
  { name: "Alice" },
  { $push: { hobbies: "painting" } }
)

// $pull: Remove from array
db.users.updateOne(
  { name: "Alice" },
  { $pull: { hobbies: "gaming" } }
)

// $addToSet: Add to array if not exists
db.users.updateOne(
  { name: "Alice" },
  { $addToSet: { tags: "premium" } }
)

// $pop: Remove first or last element
db.users.updateOne(
  { name: "Alice" },
  { $pop: { hobbies: 1 } }  // 1 = last, -1 = first
)
```

### Delete Operations

#### deleteOne()

Delete a single document.

```javascript
// Syntax
db.collection.deleteOne(filter, options)

// Example
db.users.deleteOne({ name: "Alice Johnson" })

// Delete by _id
db.users.deleteOne({ _id: ObjectId("650...") })
```

#### deleteMany()

Delete multiple documents.

```javascript
// Syntax
db.collection.deleteMany(filter, options)

// Example: Delete all users from New York
db.users.deleteMany({ city: "New York" })

// Delete all documents in collection
db.users.deleteMany({})
```

#### remove() (Deprecated)

```javascript
// Remove single document
db.users.remove({ name: "Alice" }, { justOne: true })

// Remove multiple documents
db.users.remove({ age: { $lt: 18 } })

// Remove all documents
db.users.remove({})
```

### Practical CRUD Examples

#### Complete User Management System

```javascript
// Create database
use userManagement

// Insert users
db.users.insertMany([
  {
    name: "John Doe",
    email: "[email protected]",
    age: 30,
    role: "admin",
    status: "active",
    createdAt: new Date(),
    address: {
      street: "123 Main St",
      city: "New York",
      zipcode: "10001"
    },
    hobbies: ["reading", "gaming"]
  },
  {
    name: "Jane Smith",
    email: "[email protected]",
    age: 25,
    role: "user",
    status: "active",
    createdAt: new Date(),
    address: {
      street: "456 Oak Ave",
      city: "San Francisco",
      zipcode: "94102"
    },
    hobbies: ["cooking", "yoga"]
  },
  {
    name: "Bob Wilson",
    email: "[email protected]",
    age: 35,
    role: "user",
    status: "inactive",
    createdAt: new Date(),
    address: {
      street: "789 Elm St",
      city: "Chicago",
      zipcode: "60601"
    },
    hobbies: ["sports", "music"]
  }
])

// Find all active users
db.users.find({ status: "active" })

// Find users older than 28
db.users.find({ age: { $gt: 28 } })

// Find users in San Francisco
db.users.find({ "address.city": "San Francisco" })

// Update user status
db.users.updateOne(
  { email: "[email protected]" },
  { $set: { status: "active" } }
)

// Add hobby to user
db.users.updateOne(
  { name: "John Doe" },
  { $push: { hobbies: "photography" } }
)

// Increment age for all users
db.users.updateMany(
  {},
  { $inc: { age: 1 } }
)

// Delete inactive users
db.users.deleteMany({ status: "inactive" })

// Count documents
db.users.countDocuments({ role: "admin" })
```

---

## Query Operations

### Comparison Operators

```javascript
// $eq: Equal to
db.users.find({ age: { $eq: 30 } })
// Shorthand
db.users.find({ age: 30 })

// $ne: Not equal to
db.users.find({ status: { $ne: "inactive" } })

// $gt: Greater than
db.users.find({ age: { $gt: 25 } })

// $gte: Greater than or equal
db.users.find({ age: { $gte: 30 } })

// $lt: Less than
db.users.find({ age: { $lt: 40 } })

// $lte: Less than or equal
db.users.find({ age: { $lte: 35 } })

// $in: In array
db.users.find({ city: { $in: ["New York", "Boston", "Chicago"] } })

// $nin: Not in array
db.users.find({ status: { $nin: ["inactive", "banned"] } })
```

### Logical Operators

```javascript
// $and: All conditions must be true
db.users.find({
  $and: [
    { age: { $gte: 25 } },
    { city: "New York" }
  ]
})

// Shorthand for $and
db.users.find({ age: { $gte: 25 }, city: "New York" })

// $or: At least one condition must be true
db.users.find({
  $or: [
    { city: "New York" },
    { city: "San Francisco" }
  ]
})

// $nor: None of the conditions should be true
db.users.find({
  $nor: [
    { age: { $lt: 18 } },
    { status: "banned" }
  ]
})

// $not: Negates the condition
db.users.find({ age: { $not: { $gte: 30 } } })
```

### Element Operators

```javascript
// $exists: Field exists
db.users.find({ email: { $exists: true } })

// $type: Field type
db.users.find({ age: { $type: "number" } })
db.users.find({ age: { $type: "int" } })

// Multiple types
db.users.find({ age: { $type: ["int", "double"] } })
```

### Array Operators

```javascript
// $all: Array contains all specified elements
db.users.find({ hobbies: { $all: ["reading", "gaming"] } })

// $elemMatch: Array element matches all conditions
db.users.find({
  scores: {
    $elemMatch: {
      $gte: 80,
      $lte: 90
    }
  }
})

// $size: Array has specific length
db.users.find({ hobbies: { $size: 3 } })

// Query array element by index
db.users.find({ "hobbies.0": "reading" })
```

### String Operators

```javascript
// $regex: Regular expression pattern matching
db.users.find({ name: { $regex: /^John/i } })

// Alternative syntax
db.users.find({ 
  email: { 
    $regex: "gmail.com$",
    $options: "i"  // case-insensitive
  }
})

// Text search (requires text index)
db.users.createIndex({ bio: "text" })
db.users.find({ $text: { $search: "mongodb database" } })
```

### Evaluation Operators

```javascript
// $expr: Use aggregation expressions
db.users.find({
  $expr: {
    $gt: ["$age", "$retirementAge"]
  }
})

// $jsonSchema: Validate against JSON schema
db.users.find({
  $jsonSchema: {
    required: ["name", "email"],
    properties: {
      age: {
        bsonType: "int",
        minimum: 0,
        maximum: 150
      }
    }
  }
})

// $mod: Modulo operation
db.users.find({ age: { $mod: [5, 0] } })  // Age divisible by 5

// $where: JavaScript expression (use sparingly)
db.users.find({
  $where: function() {
    return this.age > 25 && this.city === "New York";
  }
})
```

### Querying Embedded Documents

```javascript
// Exact match
db.users.find({
  address: {
    street: "123 Main St",
    city: "New York",
    zipcode: "10001"
  }
})

// Query nested field
db.users.find({ "address.city": "New York" })

// Multiple nested fields
db.users.find({
  "address.city": "New York",
  "address.zipcode": "10001"
})
```

### Advanced Query Examples

```javascript
// Find users aged 25-35 in New York or San Francisco
db.users.find({
  age: { $gte: 25, $lte: 35 },
  city: { $in: ["New York", "San Francisco"] }
})

// Find users with email ending in gmail.com
db.users.find({ email: { $regex: /@gmail\.com$/i } })

// Find users with at least 3 hobbies
db.users.find({
  hobbies: { $exists: true },
  $expr: { $gte: [{ $size: "$hobbies" }, 3] }
})

// Find users where age equals loginCount
db.users.find({
  $expr: { $eq: ["$age", "$loginCount"] }
})

// Complex query with multiple operators
db.users.find({
  $and: [
    { status: "active" },
    {
      $or: [
        { role: "admin" },
        { age: { $gte: 30 } }
      ]
    },
    { email: { $exists: true } }
  ]
})
```

### Cursor Methods

```javascript
// count() - Count documents
db.users.find({ city: "New York" }).count()
db.users.countDocuments({ city: "New York" })

// limit() - Limit results
db.users.find().limit(10)

// skip() - Skip results
db.users.find().skip(20)

// sort() - Sort results
db.users.find().sort({ age: 1 })  // Ascending
db.users.find().sort({ age: -1 }) // Descending
db.users.find().sort({ age: 1, name: 1 })  // Multiple fields

// forEach() - Iterate over results
db.users.find().forEach(function(doc) {
  print(doc.name);
})

// toArray() - Convert to array
db.users.find().toArray()

// hasNext() and next() - Manual iteration
var cursor = db.users.find();
while (cursor.hasNext()) {
  print(cursor.next().name);
}
```

---

## Indexing

Indexes improve query performance by allowing MongoDB to quickly locate documents without scanning the entire collection.

### Why Use Indexes?

**Without Index:**
- MongoDB must scan every document (Collection Scan)
- Slow performance for large collections
- O(n) time complexity

**With Index:**
- MongoDB directly accesses matching documents
- Fast query performance
- O(log n) time complexity

### Creating Indexes

#### Single Field Index

```javascript
// Syntax
db.collection.createIndex({ field: 1 })  // 1 = ascending, -1 = descending

// Example: Create index on age field
db.users.createIndex({ age: 1 })

// Descending index
db.users.createIndex({ age: -1 })

// Example: Create unique index
db.users.createIndex({ email: 1 }, { unique: true })
```

#### Compound Index

Index on multiple fields.

```javascript
// Syntax
db.collection.createIndex({ field1: 1, field2: -1, ... })

// Example: Compound index on city and age
db.users.createIndex({ city: 1, age: -1 })

// Order matters!
// Good for: { city: "NYC", age: 30 }
// Good for: { city: "NYC" }
// NOT efficient for: { age: 30 }
```

#### Text Index

For text search functionality.

```javascript
// Create text index
db.articles.createIndex({ title: "text", content: "text" })

// Search with text index
db.articles.find({ $text: { $search: "mongodb tutorial" } })

// Text search with score
db.articles.find(
  { $text: { $search: "mongodb" } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } })
```

#### Geospatial Index

For location-based queries.

```javascript
// 2dsphere index for GPS coordinates
db.places.createIndex({ location: "2dsphere" })

// Insert document with GeoJSON
db.places.insertOne({
  name: "Coffee Shop",
  location: {
    type: "Point",
    coordinates: [-73.97, 40.77]  // [longitude, latitude]
  }
})

// Find nearby places
db.places.find({
  location: {
    $near: {
      $geometry: {
        type: "Point",
        coordinates: [-73.98, 40.75]
      },
      $maxDistance: 1000  // meters
    }
  }
})
```

#### Hashed Index

For sharding and equality matches.

```javascript
// Create hashed index
db.users.createIndex({ _id: "hashed" })

// Good for: db.users.find({ _id: someValue })
// NOT good for: range queries
```

#### Wildcard Index

Index all fields or fields matching a pattern.

```javascript
// Index all fields
db.products.createIndex({ "$**": 1 })

// Index all fields in a subdocument
db.products.createIndex({ "attributes.$**": 1 })
```

### Index Properties

#### Unique Index

```javascript
// Ensure unique values
db.users.createIndex({ email: 1 }, { unique: true })

// Compound unique index
db.users.createIndex(
  { username: 1, email: 1 },
  { unique: true }
)
```

#### Sparse Index

Only index documents that contain the indexed field.

```javascript
db.users.createIndex(
  { phone: 1 },
  { sparse: true }
)
```

#### TTL Index

Automatically delete documents after a certain time.

```javascript
// Delete documents 24 hours after createdAt
db.sessions.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 86400 }
)
```

#### Partial Index

Index only documents matching a filter.

```javascript
db.users.createIndex(
  { age: 1 },
  {
    partialFilterExpression: {
      age: { $gte: 18 },
      status: "active"
    }
  }
)
```

#### Case-Insensitive Index

```javascript
db.users.createIndex(
  { email: 1 },
  { collation: { locale: "en", strength: 2 } }
)

// Query with same collation
db.users.find({ email: "USER@EXAMPLE.COM" })
  .collation({ locale: "en", strength: 2 })
```

### Index Management

#### List Indexes

```javascript
// Get all indexes on collection
db.users.getIndexes()

// Example output
[
  { v: 2, key: { _id: 1 }, name: "_id_" },
  { v: 2, key: { email: 1 }, name: "email_1", unique: true },
  { v: 2, key: { city: 1, age: -1 }, name: "city_1_age_-1" }
]
```

#### Drop Index

```javascript
// Drop by name
db.users.dropIndex("email_1")

// Drop by key pattern
db.users.dropIndex({ email: 1 })

// Drop all indexes except _id
db.users.dropIndexes()
```

#### Rebuild Indexes

```javascript
// Rebuild all indexes
db.users.reIndex()
```

#### Hide/Unhide Index

```javascript
// Hide index (doesn't use it but keeps it)
db.users.hideIndex("email_1")

// Unhide index
db.users.unhideIndex("email_1")
```

### Explain Query Performance

```javascript
// Execution stats
db.users.find({ age: 30 }).explain("executionStats")

// Query planner
db.users.find({ city: "New York" }).explain()

// All plans
db.users.find({ age: { $gte: 25 } }).explain("allPlansExecution")

// Key fields in explain output
{
  "executionStats": {
    "executionSuccess": true,
    "nReturned": 10,           // Documents returned
    "executionTimeMillis": 2,   // Execution time
    "totalDocsExamined": 10,    // Documents scanned
    "totalKeysExamined": 10,    // Index keys scanned
    "executionStages": {
      "stage": "IXSCAN"         // Used index scan
    }
  }
}
```

### Index Best Practices

1. **Create indexes for frequent queries**
2. **Use compound indexes wisely** (ESR rule: Equality, Sort, Range)
3. **Limit number of indexes** (each index has storage/write cost)
4. **Monitor index usage** with `db.collection.aggregate([{$indexStats: {}}])`
5. **Use covered queries** (query only indexed fields)
6. **Avoid unnecessary indexes on write-heavy collections**
7. **Index fields used in sorts**
8. **Create indexes on filter fields**

### Practical Indexing Examples

```javascript
// E-commerce product search
db.products.createIndex({ 
  category: 1, 
  price: -1, 
  rating: -1 
})

// User search by username (case-insensitive)
db.users.createIndex(
  { username: 1 },
  { 
    unique: true,
    collation: { locale: "en", strength: 2 }
  }
)

// Blog posts with text search
db.posts.createIndex(
  { title: "text", content: "text" },
  { weights: { title: 10, content: 5 } }
)

// Session management with auto-expiry
db.sessions.createIndex(
  { lastAccess: 1 },
  { expireAfterSeconds: 3600 }
)

// Location-based services
db.restaurants.createIndex({ location: "2dsphere" })
```

---

## Aggregation Framework

The aggregation framework processes data and returns computed results through a pipeline of stages.

### Aggregation Pipeline

Data flows through multiple stages, each transforming the documents.

```javascript
db.collection.aggregate([
  { stage1 },
  { stage2 },
  { stage3 }
])
```

### Common Aggregation Stages

#### $match

Filter documents (like find()).

```javascript
// Basic match
db.users.aggregate([
  { $match: { age: { $gte: 25 } } }
])

// Multiple conditions
db.users.aggregate([
  { 
    $match: { 
      age: { $gte: 25 },
      city: "New York"
    }
  }
])
```

#### $group

Group documents and perform aggregations.

```javascript
// Syntax
{
  $group: {
    _id: <expression>,  // Group by field
    <field>: { <accumulator>: <expression> }
  }
}

// Example: Count users by city
db.users.aggregate([
  {
    $group: {
      _id: "$city",
      count: { $sum: 1 }
    }
  }
])

// Example: Average age by city
db.users.aggregate([
  {
    $group: {
      _id: "$city",
      avgAge: { $avg: "$age" },
      totalUsers: { $sum: 1 }
    }
  }
])

// Example: Group by multiple fields
db.orders.aggregate([
  {
    $group: {
      _id: {
        city: "$city",
        status: "$status"
      },
      count: { $sum: 1 },
      totalAmount: { $sum: "$amount" }
    }
  }
])
```

#### $project

Select and transform fields.

```javascript
// Include specific fields
db.users.aggregate([
  {
    $project: {
      name: 1,
      email: 1,
      age: 1,
      _id: 0
    }
  }
])

// Create computed fields
db.users.aggregate([
  {
    $project: {
      name: 1,
      ageInMonths: { $multiply: ["$age", 12] },
      isAdult: { $gte: ["$age", 18] }
    }
  }
])

// String operations
db.users.aggregate([
  {
    $project: {
      fullName: { $concat: ["$firstName", " ", "$lastName"] },
      emailDomain: { $substr: ["$email", { $indexOfBytes: ["$email", "@"] }, -1] }
    }
  }
])
```

#### $sort

Sort documents.

```javascript
// Sort ascending
db.users.aggregate([
  { $sort: { age: 1 } }
])

// Sort descending
db.users.aggregate([
  { $sort: { age: -1 } }
])

// Multiple fields
db.users.aggregate([
  { $sort: { city: 1, age: -1 } }
])
```

#### $limit

Limit number of documents.

```javascript
db.users.aggregate([
  { $sort: { age: -1 } },
  { $limit: 10 }
])
```

#### $skip

Skip documents.

```javascript
db.users.aggregate([
  { $sort: { age: -1 } },
  { $skip: 20 },
  { $limit: 10 }
])
```

#### $unwind

Deconstruct array field.

```javascript
// Example document
{
  _id: 1,
  name: "John",
  hobbies: ["reading", "gaming", "cooking"]
}

// Unwind hobbies
db.users.aggregate([
  { $unwind: "$hobbies" }
])

// Result: 3 documents
{ _id: 1, name: "John", hobbies: "reading" }
{ _id: 1, name: "John", hobbies: "gaming" }
{ _id: 1, name: "John", hobbies: "cooking" }
```

#### $lookup

Perform left outer join.

```javascript
// Example: Join orders with customers
db.orders.aggregate([
  {
    $lookup: {
      from: "customers",           // Collection to join
      localField: "customerId",    // Field from orders
      foreignField: "_id",         // Field from customers
      as: "customerInfo"           // Output array field
    }
  }
])
```

#### $addFields

Add new fields.

```javascript
db.users.aggregate([
  {
    $addFields: {
      fullName: { $concat: ["$firstName", " ", "$lastName"] },
      ageInMonths: { $multiply: ["$age", 12] }
    }
  }
])
```

#### $count

Count documents.

```javascript
db.users.aggregate([
  { $match: { age: { $gte: 25 } } },
  { $count: "totalUsers" }
])

// Output: { totalUsers: 150 }
```

#### $out

Write results to a new collection.

```javascript
db.users.aggregate([
  { $match: { status: "active" } },
  { $out: "activeUsers" }
])
```

### Aggregation Operators

#### Arithmetic Operators

```javascript
{
  $project: {
    total: { $add: ["$price", "$tax"] },
    discounted: { $subtract: ["$price", "$discount"] },
    area: { $multiply: ["$width", "$height"] },
    average: { $divide: ["$total", "$count"] },
    remainder: { $mod: ["$quantity", 10] }
  }
}
```

#### Comparison Operators

```javascript
{
  $project: {
    isEqual: { $eq: ["$a", "$b"] },
    notEqual: { $ne: ["$a", "$b"] },
    greaterThan: { $gt: ["$score", 80] },
    greaterOrEqual: { $gte: ["$age", 18] },
    lessThan: { $lt: ["$price", 100] },
    lessOrEqual: { $lte: ["$rating", 5] }
  }
}
```

#### Logical Operators

```javascript
{
  $project: {
    result: {
      $and: [
        { $gte: ["$age", 18] },
        { $eq: ["$status", "active"] }
      ]
    },
    isValid: {
      $or: [
        { $eq: ["$role", "admin"] },
        { $gte: ["$experience", 5] }
      ]
    },
    isInvalid: {
      $not: { $eq: ["$status", "active"] }
    }
  }
}
```

#### String Operators

```javascript
{
  $project: {
    fullName: { $concat: ["$firstName", " ", "$lastName"] },
    upper: { $toUpper: "$name" },
    lower: { $toLower: "$email" },
    substring: { $substr: ["$text", 0, 10] },
    length: { $strLenCP: "$name" }
  }
}
```

#### Array Operators

```javascript
{
  $project: {
    arraySize: { $size: "$tags" },
    firstElement: { $arrayElemAt: ["$scores", 0] },
    sliced: { $slice: ["$items", 3] },
    concatenated: { $concatArrays: ["$tags", "$categories"] },
    inArray: { $in: ["premium", "$tags"] }
  }
}
```

#### Date Operators

```javascript
{
  $project: {
    year: { $year: "$createdAt" },
    month: { $month: "$createdAt" },
    day: { $dayOfMonth: "$createdAt" },
    hour: { $hour: "$createdAt" },
    dayOfWeek: { $dayOfWeek: "$createdAt" },
    dayOfYear: { $dayOfYear: "$createdAt" }
  }
}
```

### Aggregation Accumulators

```javascript
{
  $group: {
    _id: "$category",
    total: { $sum: "$amount" },        // Sum values
    average: { $avg: "$price" },       // Average
    maximum: { $max: "$score" },       // Maximum value
    minimum: { $min: "$age" },         // Minimum value
    first: { $first: "$name" },        // First value
    last: { $last: "$status" },        // Last value
    values: { $push: "$item" },        // Push to array
    uniqueValues: { $addToSet: "$tag" }, // Unique values
    stdDev: { $stdDevPop: "$values" }  // Standard deviation
  }
}
```

### Complete Aggregation Examples

#### Example 1: Sales Analysis

```javascript
// Calculate total sales by category
db.orders.aggregate([
  // Stage 1: Filter completed orders
  {
    $match: { status: "completed" }
  },
  // Stage 2: Group by category
  {
    $group: {
      _id: "$category",
      totalSales: { $sum: "$amount" },
      avgOrderValue: { $avg: "$amount" },
      orderCount: { $sum: 1 },
      maxOrder: { $max: "$amount" },
      minOrder: { $min: "$amount" }
    }
  },
  // Stage 3: Sort by total sales
  {
    $sort: { totalSales: -1 }
  },
  // Stage 4: Limit to top 10
  {
    $limit: 10
  },
  // Stage 5: Format output
  {
    $project: {
      category: "$_id",
      totalSales: { $round: ["$totalSales", 2] },
      avgOrderValue: { $round: ["$avgOrderValue", 2] },
      orderCount: 1,
      _id: 0
    }
  }
])
```

#### Example 2: User Activity Report

```javascript
db.users.aggregate([
  // Unwind orders array
  { $unwind: "$orders" },
  // Group by user
  {
    $group: {
      _id: "$_id",
      username: { $first: "$username" },
      totalOrders: { $sum: 1 },
      totalSpent: { $sum: "$orders.amount" },
      avgOrderValue: { $avg: "$orders.amount" }
    }
  },
  // Add computed fields
  {
    $addFields: {
      userSegment: {
        $switch: {
          branches: [
            { case: { $gte: ["$totalSpent", 1000] }, then: "Premium" },
            { case: { $gte: ["$totalSpent", 500] }, then: "Gold" },
            { case: { $gte: ["$totalSpent", 100] }, then: "Silver" }
          ],
          default: "Bronze"
        }
      }
    }
  },
  // Sort by total spent
  { $sort: { totalSpent: -1 } }
])
```

#### Example 3: Join Collections

```javascript
// Join orders with customer information
db.orders.aggregate([
  // Match recent orders
  {
    $match: {
      orderDate: { $gte: new Date("2024-01-01") }
    }
  },
  // Lookup customer info
  {
    $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer"
    }
  },
  // Unwind customer array
  { $unwind: "$customer" },
  // Project required fields
  {
    $project: {
      orderNumber: 1,
      customerName: "$customer.name",
      customerEmail: "$customer.email",
      amount: 1,
      orderDate: 1
    }
  }
])
```

---

## Data Modeling

Data modeling in MongoDB requires understanding your application's query patterns and designing schemas accordingly.

### Schema Design Patterns

#### 1. Embedding (Denormalization)

Store related data in a single document.

**When to use:**
- 1:1 relationships
- 1:Few relationships
- Data accessed together
- Read-heavy workloads

```javascript
// User with embedded address
{
  _id: ObjectId("..."),
  name: "John Doe",
  email: "[email protected]",
  address: {
    street: "123 Main St",
    city: "New York",
    zipcode: "10001",
    country: "USA"
  },
  orders: [
    {
      orderId: "ORD001",
      date: ISODate("2024-01-15"),
      total: 99.99
    },
    {
      orderId: "ORD002",
      date: ISODate("2024-02-20"),
      total: 149.50
    }
  ]
}
```

**Advantages:**
- Single query retrieves all data
- Better read performance
- Atomic updates
- Data locality

**Disadvantages:**
- Document size limit (16MB)
- Data duplication
- Complex updates

#### 2. Referencing (Normalization)

Store references to documents in other collections.

**When to use:**
- 1:Many relationships
- Many:Many relationships
- Data updated frequently
- Document size concerns

```javascript
// Users collection
{
  _id: ObjectId("user1"),
  name: "John Doe",
  email: "[email protected]"
}

// Orders collection
{
  _id: ObjectId("order1"),
  userId: ObjectId("user1"),  // Reference
  items: [...],
  total: 99.99,
  date: ISODate("2024-01-15")
}
```

**Advantages:**
- No duplication
- Flexible relationships
- Smaller documents
- Easier updates

**Disadvantages:**
- Multiple queries or $lookup
- No joins (application-level)
- More complex queries

#### 3. Hybrid Approach

Combine embedding and referencing.

```javascript
// User document
{
  _id: ObjectId("user1"),
  name: "John Doe",
  email: "[email protected]",
  // Embed recent orders
  recentOrders: [
    {
      orderId: ObjectId("order1"),
      date: ISODate("2024-01-15"),
      total: 99.99
    }
  ],
  // Reference to all orders
  orderIds: [
    ObjectId("order1"),
    ObjectId("order2"),
    ObjectId("order3")
  ]
}
```

### Design Patterns

#### Extended Reference Pattern

Store frequently accessed data with reference.

```javascript
// Order with embedded customer summary
{
  _id: ObjectId("order1"),
  customerId: ObjectId("customer1"),  // Full reference
  customerSummary: {                  // Frequently accessed data
    name: "John Doe",
    email: "[email protected]"
  },
  items: [...],
  total: 199.99
}
```

#### Bucket Pattern

Group data into "buckets" for time-series or IoT data.

```javascript
// Temperature readings bucketed by hour
{
  _id: ObjectId("..."),
  sensorId: "sensor123",
  date: ISODate("2024-01-15T10:00:00Z"),
  readings: [
    { time: ISODate("2024-01-15T10:00:00Z"), temp: 22.5 },
    { time: ISODate("2024-01-15T10:05:00Z"), temp: 22.7 },
    { time: ISODate("2024-01-15T10:10:00Z"), temp: 23.1 },
    // ... more readings
  ],
  count: 12,
  avgTemp: 22.8
}
```

#### Subset Pattern

Embed only a subset of related data.

```javascript
// Product with subset of reviews
{
  _id: ObjectId("product1"),
  name: "Laptop",
  price: 999,
  // Only recent reviews embedded
  recentReviews: [
    { rating: 5, comment: "Excellent!", date: ISODate("2024-01-20") },
    { rating: 4, comment: "Good value", date: ISODate("2024-01-18") }
  ],
  totalReviews: 247,  // Total count
  avgRating: 4.3
}
```

#### Computed Pattern

Pre-calculate and store derived values.

```javascript
// Order with computed totals
{
  _id: ObjectId("order1"),
  items: [
    { product: "Item1", price: 10, quantity: 2 },
    { product: "Item2", price: 15, quantity: 1 }
  ],
  // Computed values
  subtotal: 35,
  tax: 3.5,
  shipping: 5,
  total: 43.5
}
```

### Schema Design Best Practices

1. **Design for your queries** - Model data based on how it's accessed
2. **Embed when data is accessed together** - Improves read performance
3. **Reference when data is large or changing** - Reduces duplication
4. **Denormalize for read-heavy workloads** - Accept duplication for speed
5. **Keep frequently accessed data together** - Better performance
6. **Use arrays for bounded lists** - Avoid unbounded growth
7. **Consider document size** - 16MB limit
8. **Index appropriately** - Support your query patterns
9. **Use schema validation** - Ensure data quality

### Schema Validation

```javascript
// Create collection with validation
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["name", "email", "age"],
      properties: {
        name: {
          bsonType: "string",
          description: "must be a string and is required"
        },
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
          description: "must be a valid email and is required"
        },
        age: {
          bsonType: "int",
          minimum: 0,
          maximum: 150,
          description: "must be an integer between 0 and 150 and is required"
        },
        status: {
          enum: ["active", "inactive", "banned"],
          description: "can only be one of the enum values"
        }
      }
    }
  },
  validationLevel: "strict",     // or "moderate"
  validationAction: "error"      // or "warn"
})

// Add validation to existing collection
db.runCommand({
  collMod: "users",
  validator: {
    $jsonSchema: { /* schema */ }
  }
})
```

---

## Replication

Replication provides redundancy and high availability by maintaining multiple copies of data across different servers.

### Replica Set

A replica set is a group of MongoDB instances that maintain the same dataset.

**Components:**
- **Primary**: Receives all write operations
- **Secondary**: Replicates data from primary
- **Arbiter**: Participates in elections but doesn't hold data

```
┌─────────────┐
│   Primary   │◄──── Writes
└──────┬──────┘
       │ replicates
       ├─────────────┬─────────────┐
       ↓             ↓             ↓
┌─────────────┐ ┌─────────────┐ ┌──────────┐
│ Secondary 1 │ │ Secondary 2 │ │ Arbiter  │
└─────────────┘ └─────────────┘ └──────────┘
```

### Setting Up Replica Set

#### Step 1: Start MongoDB Instances

```bash
# Start first instance (will become primary)
mongod --replSet rs0 --port 27017 --dbpath /data/rs0-1 --bind_ip localhost

# Start second instance
mongod --replSet rs0 --port 27018 --dbpath /data/rs0-2 --bind_ip localhost

# Start third instance
mongod --replSet rs0 --port 27019 --dbpath /data/rs0-3 --bind_ip localhost
```

#### Step 2: Initialize Replica Set

```javascript
// Connect to first instance
mongosh --port 27017

// Initialize replica set
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "localhost:27017" },
    { _id: 1, host: "localhost:27018" },
    { _id: 2, host: "localhost:27019" }
  ]
})
```

#### Step 3: Verify Replica Set

```javascript
// Check replica set status
rs.status()

// Check configuration
rs.conf()

// Check if primary
rs.isMaster()
```

### Replica Set Operations

```javascript
// Add member
rs.add("localhost:27020")

// Add member with options
rs.add({
  host: "localhost:27020",
  priority: 0,    // Can't become primary
  hidden: true    // Hidden from applications
})

// Remove member
rs.remove("localhost:27020")

// Add arbiter
rs.addArb("localhost:27021")

// Stepdown primary (force election)
rs.stepDown(60)  // Step down for 60 seconds

// Reconfig
var config = rs.conf()
config.members[1].priority = 2
rs.reconfig(config)
```

### Read Preference

Control where reads are directed.

```javascript
// Primary (default)
db.collection.find().readPref("primary")

// Primary preferred
db.collection.find().readPref("primaryPreferred")

// Secondary
db.collection.find().readPref("secondary")

// Secondary preferred
db.collection.find().readPref("secondaryPreferred")

// Nearest (lowest latency)
db.collection.find().readPref("nearest")
```

### Write Concern

Control acknowledgment of writes.

```javascript
// Wait for acknowledgment from primary only (default)
db.users.insertOne(
  { name: "John" },
  { writeConcern: { w: 1 } }
)

// Wait for majority
db.users.insertOne(
  { name: "Jane" },
  { writeConcern: { w: "majority" } }
)

// Wait for specific number of members
db.users.insertOne(
  { name: "Bob" },
  { writeConcern: { w: 2, wtimeout: 5000 } }
)
```

### Replica Set Best Practices

1. **Use odd number of voting members** (3, 5, 7)
2. **Enable authentication** between members
3. **Monitor replication lag**
4. **Use appropriate write concerns**
5. **Configure priority for preferred primary**
6. **Use hidden members for backups**
7. **Regular backups** even with replication

---

## Sharding

Sharding distributes data across multiple machines for horizontal scaling.

### Sharded Cluster Architecture

```
Application
     ↓
┌─────────┐
│ mongos  │  (Query Router)
└────┬────┘
     ├───────────────┬───────────────┐
     ↓               ↓               ↓
┌─────────┐     ┌─────────┐     ┌─────────┐
│ Shard 1 │     │ Shard 2 │     │ Shard 3 │
│  (RS)   │     │  (RS)   │     │  (RS)   │
└─────────┘     └─────────┘     └─────────┘
     ↑               ↑               ↑
     └───────────────┴───────────────┘
                     │
            ┌────────┴────────┐
            │  Config Servers │
            │   (Replica Set) │
            └─────────────────┘
```

**Components:**
- **Shards**: Store subset of data (each is a replica set)
- **Config Servers**: Store metadata and configuration
- **Mongos**: Query routers (interface for applications)

### Setting Up Sharded Cluster

#### Step 1: Start Config Servers

```bash
# Start 3 config servers
mongod --configsvr --replSet configRS --port 27019 --dbpath /data/config1
mongod --configsvr --replSet configRS --port 27020 --dbpath /data/config2
mongod --configsvr --replSet configRS --port 27021 --dbpath /data/config3

# Initialize config replica set
mongosh --port 27019
rs.initiate({
  _id: "configRS",
  configsvr: true,
  members: [
    { _id: 0, host: "localhost:27019" },
    { _id: 1, host: "localhost:27020" },
    { _id: 2, host: "localhost:27021" }
  ]
})
```

#### Step 2: Start Shard Servers

```bash
# Shard 1
mongod --shardsvr --replSet shard1RS --port 27017 --dbpath /data/shard1
# ... more members

# Shard 2
mongod --shardsvr --replSet shard2RS --port 27018 --dbpath /data/shard2
# ... more members

# Initialize each shard
mongosh --port 27017
rs.initiate({
  _id: "shard1RS",
  members: [{ _id: 0, host: "localhost:27017" }]
})
```

#### Step 3: Start Mongos

```bash
mongos --configdb configRS/localhost:27019,localhost:27020,localhost:27021 --port 27016
```

#### Step 4: Add Shards

```javascript
// Connect to mongos
mongosh --port 27016

// Add shards
sh.addShard("shard1RS/localhost:27017")
sh.addShard("shard2RS/localhost:27018")

// Verify
sh.status()
```

### Enable Sharding

```javascript
// Enable sharding on database
sh.enableSharding("myDatabase")

// Shard a collection
sh.shardCollection("myDatabase.users", { userId: "hashed" })

// Or with range-based sharding
sh.shardCollection("myDatabase.orders", { orderDate: 1, customerId: 1 })
```

### Shard Keys

The shard key determines how data is distributed.

#### Hashed Shard Key

```javascript
// Good for even distribution
sh.shardCollection("db.collection", { _id: "hashed" })
```

**Advantages:**
- Even data distribution
- Random write distribution

**Disadvantages:**
- No range queries on shard key
- Can't use unique indexes

#### Range-Based Shard Key

```javascript
// Good for range queries
sh.shardCollection("db.collection", { timestamp: 1 })
```

**Advantages:**
- Efficient range queries
- Ordered data

**Disadvantages:**
- Potential hotspots
- Uneven distribution

#### Compound Shard Key

```javascript
// Best of both worlds
sh.shardCollection("db.collection", { country: 1, userId: 1 })
```

### Choosing a Shard Key

**Good shard key characteristics:**
1. **High Cardinality**: Many distinct values
2. **Uniform Distribution**: Even data spread
3. **Query Isolation**: Queries target single shard
4. **Monotonically Changing**: Avoid hotspots

**Examples:**

```javascript
// ❌ Bad: Low cardinality
{ status: 1 }  // Only few values

// ❌ Bad: Monotonically increasing
{ createdAt: 1 }  // Always writes to last shard

// ✅ Good: High cardinality + distribution
{ userId: "hashed" }

// ✅ Good: Compound key
{ country: 1, userId: 1 }
```

### Sharding Operations

```javascript
// Check sharding status
sh.status()

// Check database sharding
db.printShardingStatus()

// Move chunk manually
sh.moveChunk("db.collection", { shardKey: value }, "targetShard")

// Split chunk
sh.splitAt("db.collection", { shardKey: value })

// Balance shards
sh.startBalancer()
sh.stopBalancer()
sh.isBalancerRunning()

// Add shard tag/zone
sh.addShardTag("shard1RS", "NYC")
sh.addTagRange("db.collection", { zip: 10000 }, { zip: 20000 }, "NYC")
```

### Sharding Best Practices

1. **Choose shard key carefully** (difficult to change)
2. **Plan for growth** (future data distribution)
3. **Monitor chunk distribution**
4. **Use zones for geographic data**
5. **Index shard key**
6. **Avoid monotonically increasing keys**
7. **Test before production**

---

## Security - Authentication & Authorization

### Enable Authentication

#### Step 1: Create Admin User

```javascript
// Connect without auth
mongosh

// Switch to admin database
use admin

// Create admin user
db.createUser({
  user: "admin",
  pwd: "securePassword123",
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" }
  ]
})
```

#### Step 2: Enable Authentication

**mongod.conf:**

```yaml
security:
  authorization: enabled
```

Or start with flag:

```bash
mongod --auth --port 27017 --dbpath /data/db
```

#### Step 3: Restart MongoDB

```bash
sudo systemctl restart mongod
```

#### Step 4: Connect with Authentication

```bash
mongosh -u admin -p securePassword123 --authenticationDatabase admin
```

### User Management

#### Create Users

```javascript
// Create database user
use myDatabase

db.createUser({
  user: "appUser",
  pwd: "appPassword123",
  roles: [
    { role: "readWrite", db: "myDatabase" }
  ]
})

// Create read-only user
db.createUser({
  user: "reportUser",
  pwd: "reportPassword123",
  roles: [
    { role: "read", db: "myDatabase" }
  ]
})
```

#### Built-in Roles

```javascript
// Database roles
read                    // Read data
readWrite              // Read and write data

// Admin roles
dbAdmin                // Database admin
dbOwner                // Database owner
userAdmin              // Manage users

// Cluster roles
clusterAdmin           // Cluster administration
clusterManager         // Manage cluster
clusterMonitor         // Monitor cluster

// Backup roles
backup                 // Backup data
restore                // Restore data

// All-database roles
readAnyDatabase
readWriteAnyDatabase
userAdminAnyDatabase
dbAdminAnyDatabase

// Superuser role
root                   // Full access
```

#### Custom Roles

```javascript
// Create custom role
use admin

db.createRole({
  role: "customAnalyst",
  privileges: [
    {
      resource: { db: "analytics", collection: "" },
      actions: [ "find", "listCollections", "listIndexes" ]
    }
  ],
  roles: []
})

// Grant role to user
db.grantRolesToUser("analyst", [
  { role: "customAnalyst", db: "admin" }
])
```

#### User Operations

```javascript
// List users
use admin
db.getUsers()

// Show user info
db.getUser("appUser")

// Change password
db.changeUserPassword("appUser", "newPassword456")

// Update user roles
db.updateUser("appUser", {
  roles: [
    { role: "readWrite", db: "myDatabase" },
    { role: "read", db: "analytics" }
  ]
})

// Grant roles
db.grantRolesToUser("appUser", [
  { role: "dbAdmin", db: "myDatabase" }
])

// Revoke roles
db.revokeRolesFromUser("appUser", [
  { role: "dbAdmin", db: "myDatabase" }
])

// Drop user
db.dropUser("appUser")
```

### Network Security

#### Bind to Specific IP

```yaml
# mongod.conf
net:
  bindIp: 127.0.0.1,192.168.1.100
  port: 27017
```

#### TLS/SSL Encryption

```yaml
# mongod.conf
net:
  tls:
    mode: requireTLS
    certificateKeyFile: /path/to/mongodb.pem
    CAFile: /path/to/ca.pem
```

#### Firewall Rules

```bash
# Ubuntu/Debian
sudo ufw allow from 192.168.1.0/24 to any port 27017

# CentOS/RHEL
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" port protocol="tcp" port="27017" accept'
```

### Encryption at Rest

```yaml
# mongod.conf
security:
  enableEncryption: true
  encryptionKeyFile: /path/to/keyfile
```

### Connection String with Authentication

```javascript
// MongoDB URI
mongodb://username:password@host:port/database?authSource=admin

// Example
mongodb://appUser:appPassword123@localhost:27017/myDatabase?authSource=myDatabase

// Replica set
mongodb://user:pass@host1:27017,host2:27017,host3:27017/db?replicaSet=rs0

// With options
mongodb://user:pass@localhost:27017/db?authSource=admin&ssl=true&retryWrites=true
```

### Security Best Practices

1. **Enable authentication** always
2. **Use strong passwords** (min 12 characters)
3. **Principle of least privilege** (minimal permissions)
4. **Separate admin and application users**
5. **Use network encryption** (TLS/SSL)
6. **Bind to specific IPs** (not 0.0.0.0)
7. **Enable encryption at rest**
8. **Regular security audits**
9. **Keep MongoDB updated**
10. **Use firewall rules**
11. **Audit logging** for compliance
12. **Backup encryption keys**

---

## Backup and Restore

### mongodump & mongorestore

#### Backup Database

```bash
# Backup single database
mongodump --db myDatabase --out /backup/

# Backup specific collection
mongodump --db myDatabase --collection users --out /backup/

# Backup all databases
mongodump --out /backup/

# With authentication
mongodump --username admin --password pass --authenticationDatabase admin --db myDatabase --out /backup/

# Compressed backup
mongodump --db myDatabase --archive=/backup/mydb.archive --gzip
```

#### Restore Database

```bash
# Restore database
mongorestore --db myDatabase /backup/myDatabase/

# Restore specific collection
mongorestore --db myDatabase --collection users /backup/myDatabase/users.bson

# Restore all databases
mongorestore /backup/

# With authentication
mongorestore --username admin --password pass --authenticationDatabase admin --db myDatabase /backup/myDatabase/

# From compressed archive
mongorestore --gzip --archive=/backup/mydb.archive
```

### Filesystem Snapshot

```bash
# Stop writes (optional but recommended)
db.fsyncLock()

# Create snapshot (varies by system)
# AWS EBS snapshot, LVM snapshot, etc.

# Resume writes
db.fsyncUnlock()
```

### Backup Script Example

```bash
#!/bin/bash

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backup/mongodb"
DB_NAME="myDatabase"

# Create backup directory
mkdir -p $BACKUP_DIR

# Perform backup
mongodump \
  --username admin \
  --password $MONGO_PASSWORD \
  --authenticationDatabase admin \
  --db $DB_NAME \
  --archive=$BACKUP_DIR/${DB_NAME}_${DATE}.archive \
  --gzip

# Delete backups older than 7 days
find $BACKUP_DIR -name "*.archive" -mtime +7 -delete

echo "Backup completed: ${DB_NAME}_${DATE}.archive"
```

### Automated Backup with Cron

```bash
# Edit crontab
crontab -e

# Add daily backup at 2 AM
0 2 * * * /path/to/backup_script.sh
```

---

## Performance Optimization

### Query Optimization

#### Use Indexes

```javascript
// Create indexes for frequently queried fields
db.users.createIndex({ email: 1 })
db.orders.createIndex({ customerId: 1, orderDate: -1 })
```

#### Use explain()

```javascript
// Analyze query performance
db.users.find({ age: { $gte: 25 } }).explain("executionStats")

// Check if index is used
db.users.find({ email: "[email protected]" }).explain()
```

#### Projection

```javascript
// Return only required fields
db.users.find(
  { age: { $gte: 25 } },
  { name: 1, email: 1, _id: 0 }
)
```

#### Covered Queries

```javascript
// Query covered by index (doesn't access documents)
db.users.createIndex({ name: 1, age: 1 })
db.users.find({ name: "John" }, { name: 1, age: 1, _id: 0 })
```

### Connection Pooling

```javascript
// Node.js example
const { MongoClient } = require('mongodb');

const client = new MongoClient(uri, {
  maxPoolSize: 50,           // Max connections
  minPoolSize: 10,           // Min connections
  maxIdleTimeMS: 30000       // Idle timeout
});
```

### Bulk Operations

```javascript
// Bulk insert
db.users.insertMany([
  { name: "User1" },
  { name: "User2" },
  // ... more documents
], { ordered: false })  // Parallel inserts

// Bulk write operations
db.users.bulkWrite([
  {
    insertOne: {
      document: { name: "User1" }
    }
  },
  {
    updateOne: {
      filter: { name: "User2" },
      update: { $set: { age: 30 } }
    }
  },
  {
    deleteOne: {
      filter: { name: "User3" }
    }
  }
])
```

### Profiling

```javascript
// Enable profiling (level 2 = all operations)
db.setProfilingLevel(2)

// Level 1 = slow operations only (>100ms)
db.setProfilingLevel(1, { slowms: 100 })

// View profiling data
db.system.profile.find().limit(10).sort({ ts: -1 }).pretty()

// Disable profiling
db.setProfilingLevel(0)
```

### Performance Tips

1. **Create appropriate indexes**
2. **Use projection to limit returned fields**
3. **Limit result sets**
4. **Use covered queries**
5. **Avoid $where and JavaScript evaluation**
6. **Use aggregation pipeline instead of map-reduce**
7. **Enable compression**
8. **Monitor with mongostat and mongotop**
9. **Optimize schema design**
10. **Use SSD storage**

---

## Monitoring

### mongostat

Real-time statistics.

```bash
# Basic stats
mongostat

# Every 5 seconds
mongostat 5

# With authentication
mongostat --username admin --password pass --authenticationDatabase admin

# Output
insert query update delete getmore command dirty used flushes vsize   res qrw arw net_in net_out conn                time
    *0    *0     *0     *0       0     2|0  0.0% 0.0%       0 1.4G 83.0M 0|0 0|0   158b   49.0k   92 Oct 29 11:42:00.123
```

### mongotop

Track time spent per collection.

```bash
# Basic usage
mongotop

# Every 10 seconds
mongotop 10

# Output shows read/write time per collection
```

### Database Statistics

```javascript
// Database stats
db.stats()

// Collection stats
db.users.stats()

// Current operations
db.currentOp()

// Kill operation
db.killOp(opid)

// Server status
db.serverStatus()
```

### MongoDB Atlas Monitoring

If using MongoDB Atlas:
- Real-time performance metrics
- Query performance insights
- Index suggestions
- Alert configuration
- Backup automation

---

## Advantages and Disadvantages

### Advantages

1. **Flexible Schema**
   - No predefined schema required
   - Easy to evolve data model
   - Different documents can have different fields

2. **Scalability**
   - Horizontal scaling through sharding
   - Handles large volumes of data
   - Distributed architecture

3. **High Performance**
   - Fast read/write operations
   - Indexing support
   - Embedded documents reduce joins

4. **High Availability**
   - Automatic failover with replica sets
   - Data redundancy
   - No single point of failure

5. **Rich Query Language**
   - Powerful query capabilities
   - Aggregation framework
   - Geospatial queries
   - Text search

6. **Document Model**
   - Natural mapping to objects
   - Easier for developers
   - Supports complex hierarchies

7. **Developer Friendly**
   - JSON-like documents
   - Multiple language drivers
   - Easy to learn

8. **Replication**
   - Built-in replication
   - Automatic recovery
   - Read scaling

9. **Aggregation Framework**
   - Powerful data processing
   - Pipeline-based operations
   - Real-time analytics

10. **Large Community**
    - Extensive documentation
    - Active community
    - Third-party tools

### Disadvantages

1. **Memory Usage**
   - High RAM requirements
   - Working set must fit in RAM
   - Can be expensive for large datasets

2. **Data Duplication**
   - Embedded documents cause duplication
   - Increased storage requirements
   - Consistency challenges

3. **No Joins** (Traditional)
   - Application-level joins required
   - $lookup is not as efficient as SQL joins
   - Complex queries can be difficult

4. **Document Size Limit**
   - 16MB per document
   - Limitations for large objects
   - May require GridFS

5. **Transaction Support**
   - Limited compared to RDBMS
   - Multi-document transactions added later
   - Performance overhead

6. **No ACID Guarantees** (Traditionally)
   - Eventual consistency in distributed systems
   - Multi-document transactions have limitations
   - Not suitable for all use cases

7. **Data Consistency**
   - Eventual consistency with replication
   - Read preference affects consistency
   - Requires careful design

8. **Maturity**
   - Less mature than traditional RDBMS
   - Fewer experienced DBAs
   - Limited enterprise support compared to Oracle/SQL Server

9. **Storage**
   - Can use more disk space than RDBMS
   - Indexes add overhead
   - Compression helps but not always sufficient

10. **Learning Curve**
    - Different paradigm from RDBMS
    - Requires understanding of NoSQL concepts
    - Schema design patterns

---

## Best Practices

### Schema Design

1. **Design for your application queries**
2. **Embed data that is accessed together**
3. **Reference data that is large or changes frequently**
4. **Use arrays for bounded lists only**
5. **Denormalize for read-heavy workloads**
6. **Keep frequently accessed data together**
7. **Consider document size limits (16MB)**
8. **Use schema validation for data quality**

### Indexing

1. **Create indexes for frequent queries**
2. **Use compound indexes wisely (ESR rule)**
3. **Limit number of indexes (storage/write cost)**
4. **Monitor index usage regularly**
5. **Drop unused indexes**
6. **Use covered queries when possible**
7. **Index fields used in sorts**

### Query Optimization

1. **Use projection to limit returned fields**
2. **Limit result sets appropriately**
3. **Use explain() to analyze queries**
4. **Avoid $where and JavaScript**
5. **Use aggregation instead of map-reduce**
6. **Batch operations when possible**

### Operations

1. **Enable authentication always**
2. **Use strong passwords**
3. **Implement backup strategy**
4. **Monitor performance metrics**
5. **Keep MongoDB updated**
6. **Use replica sets for production**
7. **Consider sharding for large datasets**
8. **Enable journaling for durability**

### Development

1. **Use connection pooling**
2. **Handle errors gracefully**
3. **Close connections properly**
4. **Use appropriate write concerns**
5. **Test with production-like data**
6. **Version control schema changes**
7. **Document your schema**

### Security

1. **Never expose MongoDB to the internet**
2. **Use VPN or SSH tunneling**
3. **Enable encryption at rest and in transit**
4. **Audit logs regularly**
5. **Principle of least privilege**
6. **Separate admin and application users**
7. **Use firewall rules**

---

## Conclusion

MongoDB is a powerful, flexible NoSQL database perfect for modern applications requiring scalability, flexibility, and high performance. Its document-oriented model, rich query language, and horizontal scaling capabilities make it an excellent choice for a wide range of use cases.

**Key Takeaways:**
- Use MongoDB for flexible, evolving schemas
- Design schema based on query patterns
- Leverage indexing for performance
- Use replication for high availability
- Consider sharding for horizontal scaling
- Implement proper security measures
- Monitor and optimize regularly

**When to Use MongoDB:**
✅ Rapid development with changing requirements  
✅ High-volume data ingestion  
✅ Scalability requirements  
✅ Complex hierarchical data  
✅ Real-time analytics  
✅ Content management systems  
✅ Mobile and IoT applications  

**When NOT to Use MongoDB:**
❌ Complex multi-table transactions  
❌ Strong ACID guarantees required  
❌ Complex joins across many entities  
❌ Existing SQL-heavy infrastructure  

---

## Additional Resources

### Official Documentation
- [MongoDB Manual](https://docs.mongodb.com/manual/)
- [MongoDB University](https://university.mongodb.com/)
- [MongoDB Drivers](https://docs.mongodb.com/drivers/)

### Tools
- **MongoDB Compass**: GUI for MongoDB
- **Studio 3T**: Advanced MongoDB GUI
- **MongoDB Atlas**: Cloud-hosted MongoDB
- **Robo 3T**: Lightweight MongoDB GUI

### Community
- [MongoDB Community Forums](https://www.mongodb.com/community/forums/)
- [MongoDB GitHub](https://github.com/mongodb/mongo)
- [Stack Overflow - MongoDB Tag](https://stackoverflow.com/questions/tagged/mongodb)

### Learning Resources
- MongoDB for Beginners (MongoDB University)
- MongoDB Performance Tuning
- MongoDB Data Modeling Course
- MongoDB Aggregation Framework

---

**Version:** 1.0  
**Last Updated:** October 2025  
**Author:** DevOps Tutorial Series  

For the latest updates, refer to official MongoDB documentation.