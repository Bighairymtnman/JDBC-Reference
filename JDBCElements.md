# JDBCElements.md

```markdown
# JDBC Elements Quick Reference

## Table of Contents
- [Database Connection](#database-connection)
- [Statement Types](#statement-types)
- [CRUD Operations](#crud-operations)
- [Result Set Operations](#result-set-operations)
- [Transaction Management](#transaction-management)
- [Connection Utilities](#connection-utilities)

## Database Connection

```java
// Basic Connection Setup
String url = "jdbc:mysql://localhost:3306/mydb";
String username = "user";
String password = "pass";

Connection conn = DriverManager.getConnection(url, username, password);

// Connection with Properties
Properties props = new Properties();
props.setProperty("user", username);
props.setProperty("password", password);
Connection conn = DriverManager.getConnection(url, props);

// Connection Pool Setup (H2 Example)
JdbcDataSource dataSource = new JdbcDataSource();
dataSource.setURL("jdbc:h2:./h2/db");
dataSource.setUser("sa");
dataSource.setPassword("sa");
Connection conn = dataSource.getConnection();

// Connection Validation
boolean isValid = conn.isValid(5);  // Check if connection is valid
boolean isClosed = conn.isClosed(); // Check if connection is closed
```

## Statement Types

```java
// Statement (Basic)
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery("SELECT * FROM users");

// PreparedStatement (Recommended)
String sql = "SELECT * FROM users WHERE id = ?";
PreparedStatement pstmt = conn.prepareStatement(sql);
pstmt.setInt(1, userId);           // Set parameter
ResultSet rs = pstmt.executeQuery();

// PreparedStatement with Multiple Parameters
String sql = "INSERT INTO users (name, email, age) VALUES (?, ?, ?)";
PreparedStatement pstmt = conn.prepareStatement(sql);
pstmt.setString(1, "John");        // Set string parameter
pstmt.setString(2, "john@email.com"); // Set string parameter
pstmt.setInt(3, 25);              // Set integer parameter

// CallableStatement (Stored Procedures)
CallableStatement cstmt = conn.prepareCall("{call getUserById(?)}");
cstmt.setInt(1, userId);
ResultSet rs = cstmt.executeQuery();
```

## CRUD Operations

```java
// CREATE (Insert)
String insertSQL = "INSERT INTO users (name, email) VALUES (?, ?)";
PreparedStatement pstmt = conn.prepareStatement(insertSQL, Statement.RETURN_GENERATED_KEYS);
pstmt.setString(1, "John Doe");
pstmt.setString(2, "john@email.com");
int rowsAffected = pstmt.executeUpdate();

// Get Generated Keys
ResultSet generatedKeys = pstmt.getGeneratedKeys();
if (generatedKeys.next()) {
    int generatedId = generatedKeys.getInt(1);
}

// READ (Select)
String selectSQL = "SELECT id, name, email FROM users WHERE id = ?";
PreparedStatement pstmt = conn.prepareStatement(selectSQL);
pstmt.setInt(1, userId);
ResultSet rs = pstmt.executeQuery();

// UPDATE
String updateSQL = "UPDATE users SET email = ? WHERE id = ?";
PreparedStatement pstmt = conn.prepareStatement(updateSQL);
pstmt.setString(1, "newemail@email.com");
pstmt.setInt(2, userId);
int rowsUpdated = pstmt.executeUpdate();

// DELETE
String deleteSQL = "DELETE FROM users WHERE id = ?";
PreparedStatement pstmt = conn.prepareStatement(deleteSQL);
pstmt.setInt(1, userId);
int rowsDeleted = pstmt.executeUpdate();
```

## Result Set Operations

```java
// Basic ResultSet Processing
while (rs.next()) {                 // Iterate through results
    int id = rs.getInt("id");       // Get integer column
    String name = rs.getString("name"); // Get string column
    String email = rs.getString("email");
    Date created = rs.getDate("created_date");
}

// ResultSet Data Type Methods
int intValue = rs.getInt(1);        // By column index
String stringValue = rs.getString("column_name"); // By column name
double doubleValue = rs.getDouble("price");
boolean boolValue = rs.getBoolean("active");
Date dateValue = rs.getDate("created_date");
Timestamp timestampValue = rs.getTimestamp("updated_at");

// ResultSet Navigation
rs.first();                        // Move to first row
rs.last();                         // Move to last row
rs.next();                         // Move to next row
rs.previous();                     // Move to previous row
rs.absolute(5);                    // Move to specific row

// ResultSet Metadata
ResultSetMetaData metaData = rs.getMetaData();
int columnCount = metaData.getColumnCount();
String columnName = metaData.getColumnName(1);
String columnType = metaData.getColumnTypeName(1);
```

## Transaction Management

```java
// Basic Transaction
try {
    conn.setAutoCommit(false);      // Start transaction
    
    // Execute multiple statements
    PreparedStatement stmt1 = conn.prepareStatement("INSERT INTO users...");
    stmt1.executeUpdate();
    
    PreparedStatement stmt2 = conn.prepareStatement("UPDATE accounts...");
    stmt2.executeUpdate();
    
    conn.commit();                  // Commit transaction
} catch (SQLException e) {
    conn.rollback();               // Rollback on error
} finally {
    conn.setAutoCommit(true);      // Reset auto-commit
}

// Savepoint Management
Savepoint savepoint = conn.setSavepoint("sp1");
try {
    // Some operations
    conn.commit();
} catch (SQLException e) {
    conn.rollback(savepoint);      // Rollback to savepoint
}

// Transaction Isolation Levels
conn.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);
conn.setTransactionIsolation(Connection.TRANSACTION_SERIALIZABLE);
```

## Connection Utilities

```java
// Connection Utility Class Pattern
public class ConnectionUtil {
    private static final String URL = "jdbc:h2:./h2/db";
    private static final String USERNAME = "sa";
    private static final String PASSWORD = "sa";
    
    private static JdbcDataSource dataSource = new JdbcDataSource();
    
    static {
        dataSource.setURL(URL);
        dataSource.setUser(USERNAME);
        dataSource.setPassword(PASSWORD);
    }
    
    public static Connection getConnection() {
        try {
            return dataSource.getConnection();
        } catch (SQLException e) {
            e.printStackTrace();
            return null;
        }
    }
}

// Resource Management (Important!)
// Always close resources in reverse order
try {
    Connection conn = ConnectionUtil.getConnection();
    PreparedStatement pstmt = conn.prepareStatement(sql);
    ResultSet rs = pstmt.executeQuery();
    
    // Process results
    
} catch (SQLException e) {
    e.printStackTrace();
} finally {
    // Close resources (or use try-with-resources)
    if (rs != null) rs.close();
    if (pstmt != null) pstmt.close();
    if (conn != null) conn.close();
}

// Batch Operations
PreparedStatement pstmt = conn.prepareStatement("INSERT INTO users (name) VALUES (?)");
for (String name : names) {
    pstmt.setString(1, name);
    pstmt.addBatch();              // Add to batch
}
int[] results = pstmt.executeBatch(); // Execute all at once
```