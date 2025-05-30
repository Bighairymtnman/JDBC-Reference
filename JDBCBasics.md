# JDBC Basics Quick Reference

## Table of Contents
- [DAO Pattern](#dao-pattern)
- [Entity Classes](#entity-classes)
- [Service Layer](#service-layer)
- [Connection Management](#connection-management)
- [Result Set Mapping](#result-set-mapping)
- [Exception Handling](#exception-handling)

## DAO Pattern

```java
// DAO (Data Access Object) - handles all database operations for User table
public class UserDAO {
    
    // Create new user in database
    public User createUser(User user) {
        // SQL INSERT statement with placeholders (?)
        String sql = "INSERT INTO users (username, email) VALUES (?, ?)";
        
        // Try-with-resources automatically closes connections
        try (Connection conn = ConnectionUtil.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
            
            // Set values for placeholders (1-indexed)
            pstmt.setString(1, user.getUsername());  // First ? = username
            pstmt.setString(2, user.getEmail());     // Second ? = email
            pstmt.executeUpdate();                   // Execute the INSERT
            
            // Get the auto-generated ID from database
            ResultSet rs = pstmt.getGeneratedKeys();
            if (rs.next()) {
                user.setId(rs.getInt(1));           // Set the new ID on user object
            }
            return user;                            // Return user with ID set
        } catch (SQLException e) {
            e.printStackTrace();                    // Print error details
            return null;                           // Return null if failed
        }
    }
    
    // Find single user by their ID
    public User getUserById(int id) {
        // SQL SELECT with WHERE clause
        String sql = "SELECT * FROM users WHERE id = ?";
        
        try (Connection conn = ConnectionUtil.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql)) {
            
            pstmt.setInt(1, id);                    // Set the ID parameter
            ResultSet rs = pstmt.executeQuery();   // Execute SELECT query
            
            // Check if we found a user
            if (rs.next()) {
                return mapToUser(rs);               // Convert database row to User object
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return null;                               // No user found or error occurred
    }
    
    // Get all users from database
    public List<User> getAllUsers() {
        String sql = "SELECT * FROM users";       // Get all rows from users table
        List<User> users = new ArrayList<>();    // List to store all users
        
        try (Connection conn = ConnectionUtil.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql);
             ResultSet rs = pstmt.executeQuery()) {
            
            // Loop through all rows in result set
            while (rs.next()) {                     
                users.add(mapToUser(rs));           // Convert each row to User and add to list
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return users;                              // Return list of all users
    }
    
    // Update existing user in database
    public User updateUser(User user) {
        // SQL UPDATE statement
        String sql = "UPDATE users SET username = ?, email = ? WHERE id = ?";
        
        try (Connection conn = ConnectionUtil.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql)) {
            
            // Set new values and specify which user to update
            pstmt.setString(1, user.getUsername()); // New username
            pstmt.setString(2, user.getEmail());    // New email
            pstmt.setInt(3, user.getId());          // Which user to update (by ID)
            
            int rowsUpdated = pstmt.executeUpdate(); // Execute UPDATE
            if (rowsUpdated > 0) {
                return user;                        // Return updated user
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return null;                               // Update failed
    }
}
```

## Entity Classes

```java
// User Entity - represents a user record from the database
public class User {
    // Private fields match database columns
    private int id;                                // Primary key from database
    private String username;                       // Username column
    private String email;                          // Email column
    
    // Default constructor - needed for creating empty User objects
    public User() {}
    
    // Constructor for new users (no ID yet)
    public User(String username, String email) {
        this.username = username;
        this.email = email;
        // ID will be set by database when user is created
    }
    
    // Constructor for existing users (with ID from database)
    public User(int id, String username, String email) {
        this.id = id;
        this.username = username;
        this.email = email;
    }
    
    // Getter methods - allow access to private fields
    public int getId() { return id; }
    public String getUsername() { return username; }
    public String getEmail() { return email; }
    
    // Setter methods - allow modification of private fields
    public void setId(int id) { this.id = id; }
    public void setUsername(String username) { this.username = username; }
    public void setEmail(String email) { this.email = email; }
    
    // toString method - useful for debugging and logging
    @Override
    public String toString() {
        return "User{id=" + id + ", username='" + username + "', email='" + email + "'}";
    }
}
```

## Service Layer

```java
// Service Layer - contains business logic and validation
public class UserService {
    private UserDAO userDAO;                       // DAO for database operations
    
    // Constructor - initialize the DAO
    public UserService() {
        this.userDAO = new UserDAO();
    }
    
    // Create user with business logic validation
    public User createUser(User user) {
        // Validate username is not empty
        if (user.getUsername() == null || user.getUsername().trim().isEmpty()) {
            System.out.println("Error: Username cannot be empty");
            return null;
        }
        
        // Validate email format (basic check)
        if (user.getEmail() == null || !user.getEmail().contains("@")) {
            System.out.println("Error: Invalid email format");
            return null;
        }
        
        // If validation passes, create user in database
        return userDAO.createUser(user);
    }
    
    // Get user by ID with validation
    public User getUserById(int id) {
        // Validate ID is positive
        if (id <= 0) {
            System.out.println("Error: User ID must be positive");
            return null;
        }
        
        // Get user from database
        return userDAO.getUserById(id);
    }
    
    // Get all users - no validation needed
    public List<User> getAllUsers() {
        return userDAO.getAllUsers();
    }
    
    // Update user with validation
    public User updateUser(User user) {
        // Check if user has valid ID
        if (user.getId() <= 0) {
            System.out.println("Error: Cannot update user without valid ID");
            return null;
        }
        
        // Check if user exists before updating
        User existingUser = userDAO.getUserById(user.getId());
        if (existingUser == null) {
            System.out.println("Error: User not found");
            return null;
        }
        
        // Update user in database
        return userDAO.updateUser(user);
    }
}
```

## Connection Management

```java
// Utility class to manage database connections
public class ConnectionUtil {
    // Database connection details
    private static final String URL = "jdbc:h2:./h2/db";  // H2 database file location
    private static final String USERNAME = "sa";           // Database username
    private static final String PASSWORD = "sa";           // Database password
    
    private static JdbcDataSource dataSource;             // Connection pool
    
    // Static block - runs once when class is first loaded
    static {
        dataSource = new JdbcDataSource();                // Create data source
        dataSource.setURL(URL);                           // Set database URL
        dataSource.setUser(USERNAME);                     // Set username
        dataSource.setPassword(PASSWORD);                 // Set password
    }
    
    // Get a connection to the database
    public static Connection getConnection() throws SQLException {
        return dataSource.getConnection();                // Return new connection
    }
    
    // Alternative method with error handling
    public static Connection getConnectionSafe() {
        try {
            return dataSource.getConnection();
        } catch (SQLException e) {
            System.err.println("Failed to get database connection: " + e.getMessage());
            return null;                                  // Return null if connection fails
        }
    }
}

// Example of proper resource management
public void exampleMethod() {
    // Try-with-resources automatically closes connection, statement, and result set
    try (Connection conn = ConnectionUtil.getConnection();
         PreparedStatement pstmt = conn.prepareStatement("SELECT * FROM users");
         ResultSet rs = pstmt.executeQuery()) {
        
        // Use the connection here
        // Resources are automatically closed when try block ends
        
    } catch (SQLException e) {
        e.printStackTrace();                              // Handle any database errors
    }
}
```

## Result Set Mapping

```java
// Helper method to convert database row to User object
private User mapToUser(ResultSet rs) throws SQLException {
    User user = new User();                               // Create new User object
    
    // Get data from current row and set on User object
    user.setId(rs.getInt("id"));                         // Get ID column as integer
    user.setUsername(rs.getString("username"));          // Get username column as string
    user.setEmail(rs.getString("email"));                // Get email column as string
    
    return user;                                         // Return populated User object
}

// Safer mapping with null checks
private User mapToUserSafe(ResultSet rs) throws SQLException {
    User user = new User();
    
    // Get ID (should never be null for primary key)
    user.setId(rs.getInt("id"));
    
    // Get username with null check
    String username = rs.getString("username");
    user.setUsername(username != null ? username : "");  // Use empty string if null
    
    // Get email with null check
    String email = rs.getString("email");
    user.setEmail(email != null ? email : "");           // Use empty string if null
    
    return user;
}

// Example mapping for different data types
private Post mapToPost(ResultSet rs) throws SQLException {
    Post post = new Post();
    
    // Different data type examples
    post.setId(rs.getInt("id"));                         // Integer
    post.setTitle(rs.getString("title"));                // String
    post.setContent(rs.getString("content"));            // String (can be long text)
    post.setCreatedAt(rs.getTimestamp("created_at"));    // Timestamp/DateTime
    post.setActive(rs.getBoolean("is_active"));          // Boolean (true/false)
    post.setViewCount(rs.getLong("view_count"));         // Long (big numbers)
    post.setRating(rs.getDouble("rating"));              // Double (decimal numbers)
    
    return post;
}
```

## Exception Handling

```java
// DAO method with comprehensive error handling
public User createUser(User user) {
    String sql = "INSERT INTO users (username, email) VALUES (?, ?)";
    
    try (Connection conn = ConnectionUtil.getConnection();
         PreparedStatement pstmt = conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
        
        // Set parameters
        pstmt.setString(1, user.getUsername());
        pstmt.setString(2, user.getEmail());
        
        // Execute insert and check if it worked
        int rowsAffected = pstmt.executeUpdate();
        if (rowsAffected == 0) {
            throw new SQLException("Creating user failed, no rows affected.");
        }
        
        // Get the generated ID
        ResultSet generatedKeys = pstmt.getGeneratedKeys();
        if (generatedKeys.next()) {
            user.setId(generatedKeys.getInt(1));          // Set the new ID
            return user;                                  // Success!
        } else {
            throw new SQLException("Creating user failed, no ID obtained.");
        }
        
    } catch (SQLException e) {
        // Log the error with details
        System.err.println("Database error creating user: " + e.getMessage());
        e.printStackTrace();                              // Print full error trace
        return null;                                     // Return null to indicate failure
    }
}

// Service layer error handling
public User createUser(User user) {
    try {
        // Validate input first
        if (user == null) {
            System.err.println("Error: User object is null");
            return null;
        }
        
        // Call DAO to create user
        User createdUser = userDAO.createUser(user);
        
        if (createdUser != null) {
            System.out.println("User created successfully: " + createdUser);
        }
        
        return createdUser;
        
    } catch (Exception e) {
        // Catch any unexpected errors
        System.err.println("Unexpected error creating user: " + e.getMessage());
        e.printStackTrace();
        return null;                                     // Handle gracefully
    }
}
```
