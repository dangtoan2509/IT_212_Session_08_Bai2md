# Bài 2: Thực hành Gỡ lỗi từ Stack Trace

## 1. Nội dung Prompt gỡ lỗi do tôi thiết kế

```text
Bạn là Java Debugger chuyên nghiệp, có kinh nghiệm xử lý lỗi dữ liệu và ràng buộc khóa ngoại trong hệ thống Java + JDBC.

Hãy phân tích đoạn mã Java và stack trace dưới đây, giải thích nguyên nhân gốc rễ (root cause) và tối ưu hàm saveTransaction sao cho an toàn hơn.

Mã nguồn gây lỗi:
```java
import java.sql.Connection;
import java.sql.PreparedStatement;

public class TransactionRepository {

    private Connection connection;

    public void saveTransaction(String transactionId, Long userId, double amount) throws Exception {
        String sql = "INSERT INTO transactions (id, user_id, amount) VALUES (?, ?, ?)";
        PreparedStatement ps = connection.prepareStatement(sql);
        ps.setString(1, transactionId);
        ps.setLong(2, userId);
        ps.setDouble(3, amount);
        ps.executeUpdate();
    }
}
```

Stack trace thực tế:
```text
org.h2.jdbc.JdbcSQLIntegrityConstraintViolationException: Referential integrity constraint violation: "CONSTRAINT_FOREIGN_KEY_USER: PUBLIC.TRANSACTIONS FOREIGN KEY(USER_ID) REFERENCES PUBLIC.USERS(ID) (99)"; SQL statement:
INSERT INTO transactions (id, user_id, amount) VALUES (?, ?, ?) [23506-214] at org.h2.message.DbException.getJdbcSQLException(DbException.java:502)
at TransactionRepository.saveTransaction(TransactionRepository.java:13) at Main.main(Main.java:10)
```

Yêu cầu của bạn:
1. Giải thích rõ nguyên nhân lỗi và root cause.
2. Chỉ ra rằng lỗi xảy ra vì không kiểm tra `userId` trước khi insert.
3. Tối ưu lại hàm `saveTransaction` để:
   - kiểm tra xem `userId` có tồn tại trong bảng `users` hay không trước khi insert;
   - nếu không tồn tại, ném ngoại lệ tùy chỉnh `UserNotFoundException`;
   - không để database tự ném lỗi và làm sập hệ thống;
   - sử dụng try-with-resources và chuẩn hóa xử lý ngoại lệ.
4. Đưa ra mã nguồn Java đã sửa đổi đầy đủ, sạch và production-ready.
```

---

## 2. Minh chứng chạy thực tế

### Log phản hồi AI (mô phỏng)

```text
Nguyên nhân gốc rễ (Root Cause):
- Hệ thống gọi `INSERT INTO transactions` mà không kiểm tra trước xem `user_id` có tồn tại trong bảng `users` hay không.
- Khi `userId` không tồn tại, DB báo lỗi khóa ngoại `FOREIGN KEY`, dẫn đến `JdbcSQLIntegrityConstraintViolationException`.
- Đây là lỗi nghiệp vụ chứ không phải lỗi SQL syntax.

Phân tích:
- `TransactionRepository` đang phụ thuộc vào `Connection` và thực hiện insert trực tiếp.
- Không có kiểm tra tồn tại người dùng trước khi ghi giao dịch.
- Vì vậy, error được ném từ tầng database và làm gián đoạn luồng xử lý.

Mã nguồn Java đã sửa:
```java
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;

public class TransactionRepository {

    private final Connection connection;

    public TransactionRepository(Connection connection) {
        this.connection = connection;
    }

    public void saveTransaction(String transactionId, Long userId, double amount) throws Exception {
        if (!userExists(userId)) {
            throw new UserNotFoundException("User not found with id: " + userId);
        }

        String sql = "INSERT INTO transactions (id, user_id, amount) VALUES (?, ?, ?)";

        try (PreparedStatement preparedStatement = connection.prepareStatement(sql)) {
            preparedStatement.setString(1, transactionId);
            preparedStatement.setLong(2, userId);
            preparedStatement.setDouble(3, amount);
            preparedStatement.executeUpdate();
        } catch (SQLException ex) {
            throw new SQLException("Failed to save transaction", ex);
        }
    }

    private boolean userExists(Long userId) throws SQLException {
        String sql = "SELECT 1 FROM users WHERE id = ?";
        try (PreparedStatement preparedStatement = connection.prepareStatement(sql)) {
            preparedStatement.setLong(1, userId);
            try (ResultSet resultSet = preparedStatement.executeQuery()) {
                return resultSet.next();
            }
        }
    }
}

class UserNotFoundException extends Exception {
    public UserNotFoundException(String message) {
        super(message);
    }
}
```

Kết luận:
- Việc kiểm tra trước dữ liệu ở tầng service/repository giúp hệ thống phản hồi lỗi một cách kiểm soát và dễ hiểu hơn.
- Thay vì để DB ném lỗi ngắt luồng, ta nên ném `UserNotFoundException` để tầng gọi xử lý phù hợp.
```
