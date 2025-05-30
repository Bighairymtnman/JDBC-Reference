# JDBC Advanced Concepts

## Table of Contents
- [Connection Pooling](#connection-pooling)
- [Transaction Management](#transaction-management)
- [Batch Operations](#batch-operations)
- [Stored Procedures](#stored-procedures)
- [Result Set Processing](#result-set-processing)
- [Performance Optimization](#performance-optimization)

## Connection Pooling

```java
// HikariCP Connection Pool Setup
public class ConnectionPoolUtil {
    private static HikariDataSource dataSource;
    
    static {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:h2:./h2/db");
        config.setUsername("sa");
        config.setPassword("sa");
        config.setMaximumPoolSize(20);              // Max connections
        config.setMinimumIdle(5);                   // Min idle connections
        config.setConnectionTimeout(30000);         // 30 seconds timeout
        config.setIdleTimeout(600000);              // 10 minutes idle timeout
        config.setMaxLifetime(1800000);             // 30 minutes max lifetime
        
        dataSource = new HikariDataSource(config);
    }
    
    public static Connection getConnection() throws SQLException {
        return dataSource.getConnection();          // Get pooled connection
    }
    
    public static void closePool() {
        if (dataSource != null) {
            dataSource.close();                     // Close all connections
        }
    }
}

// Custom Connection Pool
public class SimpleConnectionPool {
    private final Queue<Connection> pool = new ConcurrentLinkedQueue<>();
    private final int maxSize;
    
    public SimpleConnectionPool(int maxSize) {
        this.maxSize = maxSize;
        initializePool();
    }
    
    private void initializePool() {
        for (int i = 0; i < maxSize; i++) {
            try {
                Connection conn = DriverManager.getConnection("jdbc:h2:./h2/db", "sa", "sa");
                pool.offer(conn);                   // Add to pool
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
    }
    
    public Connection getConnection() {
        return pool.poll();                         // Get connection from pool
    }
    
    public void returnConnection(Connection conn) {
        if (conn != null && pool.size() < maxSize) {
            pool.offer(conn);                       // Return to pool
        }
    }
}
```

## Transaction Management

```java
// Manual Transaction Control
public class TransactionManager {
    
    public boolean transferMoney(int fromAccount, int toAccount, double amount) {
        Connection conn = null;
        try {
            conn = ConnectionPoolUtil.getConnection();
            conn.setAutoCommit(false);              // Start transaction
            
            // Debit from source account
            String debitSql = "UPDATE accounts SET balance = balance - ? WHERE id = ?";
            try (PreparedStatement debitStmt = conn.prepareStatement(debitSql)) {
                debitStmt.setDouble(1, amount);
                debitStmt.setInt(2, fromAccount);
                debitStmt.executeUpdate();
            }
            
            // Credit to destination account
            String creditSql = "UPDATE accounts SET balance = balance + ? WHERE id = ?";
            try (PreparedStatement creditStmt = conn.prepareStatement(creditSql)) {
                creditStmt.setDouble(1, amount);
                creditStmt.setInt(2, toAccount);
                creditStmt.executeUpdate();
            }
            
            conn.commit();                          // Commit transaction
            return true;
            
        } catch (SQLException e) {
            if (conn != null) {
                try {
                    conn.rollback();                // Rollback on error
                } catch (SQLException rollbackEx) {
                    rollbackEx.printStackTrace();
                }
            }
            e.printStackTrace();
            return false;
        } finally {
            if (conn != null) {
                try {
                    conn.setAutoCommit(true);       // Reset auto-commit
                    conn.close();
                } catch (SQLException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}

// Savepoint Usage
public void complexTransaction() {
    Connection conn = null;
    Savepoint savepoint = null;
    
    try {
        conn = ConnectionPoolUtil.getConnection();
        conn.setAutoCommit(false);
        
        // First operation
        executeOperation1(conn);
        
        savepoint = conn.setSavepoint("checkpoint1");  // Create savepoint
        
        try {
            // Risky operation
            executeRiskyOperation(conn);
        } catch (SQLException e) {
            conn.rollback(savepoint);               // Rollback to savepoint
            executeAlternativeOperation(conn);
        }
        
        conn.commit();                              // Commit all changes
        
    } catch (SQLException e) {
        if (conn != null) {
            try {
                conn.rollback();                    // Full rollback
            } catch (SQLException rollbackEx) {
                rollbackEx.printStackTrace();
            }
        }
    }
}
```

## Batch Operations

```java
// Batch Insert Operations
public class BatchProcessor {
    
    public int[] batchInsertUsers(List<User> users) {
        String sql = "INSERT INTO users (username, email) VALUES (?, ?)";
        
        try (Connection conn = ConnectionPoolUtil.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql)) {
            
            for (User user : users) {
                pstmt.setString(1, user.getUsername());
                pstmt.setString(2, user.getEmail());
                pstmt.addBatch();                   // Add to batch
                
                // Execute batch every 1000 records
                if (users.indexOf(user) % 1000 == 0) {
                    pstmt.executeBatch();           // Execute current batch
                    pstmt.clearBatch();             // Clear for next batch
                }
            }
            
            return pstmt.executeBatch();            // Execute remaining records
            
        } catch (SQLException e) {
            e.printStackTrace();
            return new int[0];
        }
    }
    
    // Batch Update Operations
    public void batchUpdatePrices(Map<Integer, Double> priceUpdates) {
        String sql = "UPDATE products SET price = ? WHERE id = ?";
        
        try (Connection conn = ConnectionPoolUtil.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql)) {
            
            conn.setAutoCommit(false);              // Use transaction for batch
            
            for (Map.Entry<Integer, Double> entry : priceUpdates.entrySet()) {
                pstmt.setDouble(1, entry.getValue());
                pstmt.setInt(2, entry.getKey());
                pstmt.addBatch();
            }
            
            int[] results = pstmt.executeBatch();   // Execute all updates
            conn.commit();
            
            System.out.println("Updated " + results.length + " records");
            
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

## Stored Procedures

```java
// Calling Stored Procedures
public class StoredProcedureDAO {
    
    // Call procedure with IN parameters
    public void callSimpleProcedure(int userId, String status) {
        String sql = "{CALL update_user_status(?, ?)}";
        
        try (Connection conn = ConnectionPoolUtil.getConnection();
             CallableStatement cstmt = conn.prepareCall(sql)) {
            
            cstmt.setInt(1, userId);                // Set IN parameter
            cstmt.setString(2, status);             // Set IN parameter
            cstmt.execute();                        // Execute procedure
            
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
    
    // Call procedure with OUT parameters
    public String getUserReport(int userId) {
        String sql = "{CALL get_user_report(?, ?)}";
        
        try (Connection conn = ConnectionPoolUtil.getConnection();
             CallableStatement cstmt = conn.prepareCall(sql)) {
            
            cstmt.setInt(1, userId);                // IN parameter
            cstmt.registerOutParameter(2, Types.VARCHAR);  // OUT parameter
            
            cstmt.execute();
            return cstmt.getString(2);              // Get OUT parameter value
            
        } catch (SQLException e) {
            e.printStackTrace();
            return null;
        }
    }
    
    // Call function (returns value)
    public double calculateDiscount(int customerId, double amount) {
        String sql = "{? = CALL calculate_discount(?, ?)}";
        
        try (Connection conn = ConnectionPoolUtil.getConnection();
             CallableStatement cstmt = conn.prepareCall(sql)) {
            
            cstmt.registerOutParameter(1, Types.DOUBLE);   // Function return value
            cstmt.setInt(2, customerId);
            cstmt.setDouble(3, amount);
            
            cstmt.execute();
            return cstmt.getDouble(1);              // Get function result
            
        } catch (SQLException e) {
            e.printStackTrace();
            return 0.0;
        }
    }
}
```

## Result Set Processing

```java
// Advanced ResultSet Handling
public class AdvancedResultSetProcessor {
    
    // Scrollable ResultSet
    public void processScrollableResults() {
        String sql = "SELECT * FROM users ORDER BY id";
        
        try (Connection conn = ConnectionPoolUtil.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql, 
                 ResultSet.TYPE_SCROLL_INSENSITIVE, 
                 ResultSet.CONCUR_READ_ONLY);
             ResultSet rs = pstmt.executeQuery()) {
            
            // Move to last row to get count
            rs.last();
            int totalRows = rs.getRow();
            System.out.println("Total rows: " + totalRows);
            
            // Move to first row
            rs.first();
            System.out.println("First user: " + rs.getString("username"));
            
            // Move to specific row
            rs.absolute(5);                         // Go to 5th row
            System.out.println("5th user: " + rs.getString("username"));
            
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
    
    // Large ResultSet Processing
    public void processLargeResultSet() {
        String sql = "SELECT * FROM large_table";
        
        try (Connection conn = ConnectionPoolUtil.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql)) {
            
            pstmt.setFetchSize(1000);               // Process in chunks of 1000
            
            try (ResultSet rs = pstmt.executeQuery()) {
                int count = 0;
                while (rs.next()) {
                    processRow(rs);                 // Process each row
                    count++;
                    
                    if (count % 10000 == 0) {
                        System.out.println("Processed " + count + " rows");
                    }
                }
            }
            
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
    
    // ResultSet Metadata
    public void analyzeResultSet(ResultSet rs) throws SQLException {
        ResultSetMetaData metaData = rs.getMetaData();
        int columnCount = metaData.getColumnCount();
        
        for (int i = 1; i <= columnCount; i++) {
            String columnName = metaData.getColumnName(i);
            String columnType = metaData.getColumnTypeName(i);
            int columnSize = metaData.getColumnDisplaySize(i);
            
            System.out.println("Column " + i + ": " + columnName + 
                             " (" + columnType + ", size: " + columnSize + ")");
        }
    }
}
```

## Performance Optimization

```java
// Prepared Statement Caching
public class PreparedStatementCache {
    private final Map<String, PreparedStatement> cache = new ConcurrentHashMap<>();
    private final Connection connection;
    
    public PreparedStatementCache(Connection connection) {
        this.connection = connection;
    }
    
    public PreparedStatement getPreparedStatement(String sql) throws SQLException {
        return cache.computeIfAbsent(sql, key -> {
            try {
                return connection.prepareStatement(key);
            } catch (SQLException e) {
                throw new RuntimeException(e);
            }
        });
    }
    
    public void clearCache() throws SQLException {
        for (PreparedStatement pstmt : cache.values()) {
            pstmt.close();                          // Close all cached statements
        }
        cache.clear();
    }
}

// Pagination for Large Datasets
public class PaginationDAO {
    
    public List<User> getUsersWithPagination(int page, int pageSize) {
        String sql = "SELECT * FROM users ORDER BY id LIMIT ? OFFSET ?";
        List<User> users = new ArrayList<>();
        
        try (Connection conn = ConnectionPoolUtil.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql)) {
            
            pstmt.setInt(1, pageSize);              // LIMIT
            pstmt.setInt(2, page * pageSize);       // OFFSET
            
            try (ResultSet rs = pstmt.executeQuery()) {
                while (rs.next()) {
                    users.add(mapToUser(rs));
                }
            }
            
        } catch (SQLException e) {
            e.printStackTrace();
        }
        
        return users;
    }
    
    // Count total records for pagination
    public int getTotalUserCount() {
        String sql = "SELECT COUNT(*) FROM users";
        
        try (Connection conn = ConnectionPoolUtil.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql);
             ResultSet rs = pstmt.executeQuery()) {
            
            if (rs.next()) {
                return rs.getInt(1);                // Return count
            }
            
        } catch (SQLException e) {
            e.printStackTrace();
        }
        
        return 0;
    }
}

// Database Monitoring
public class DatabaseMonitor {
    
    public void logSlowQueries(String sql, long executionTime) {
        if (executionTime > 1000) {                 // Log queries > 1 second
            System.out.println("SLOW QUERY (" + executionTime + "ms): " + sql);
        }
    }
    
    public void monitorConnectionPool() {
        if (dataSource instanceof HikariDataSource) {
            HikariDataSource hikari = (HikariDataSource) dataSource;
            HikariPoolMXBean poolBean = hikari.getHikariPoolMXBean();
            
            System.out.println("Active connections: " + poolBean.getActiveConnections());
            System.out.println("Idle connections: " + poolBean.getIdleConnections());
            System.out.println("Total connections: " + poolBean.getTotalConnections());
        }
    }
}
```
