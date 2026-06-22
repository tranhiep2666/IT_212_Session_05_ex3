1. Phân tích các lỗ hổng nghiêm trọng của mã nguồn ban đầu
   Đoạn code của lập trình viên tập sự tuy có thể chạy được trong điều kiện lý tưởng (Happy Path), nhưng tiềm ẩn những rủi ro cực kỳ lớn khi đưa lên môi trường Production của một hệ thống tài chính/ngân hàng:

Lỗ hổng về Transaction (Tính toàn vẹn dữ liệu): * Hàm transfer thực hiện hai thao tác cập nhật độc lập (save(from) và save(to)). Nếu hệ thống bị sập hoặc xảy ra lỗi mạng ngay sau khi trừ tiền tài khoản from nhưng chưa kịp cộng tiền cho tài khoản to, tiền của khách hàng sẽ "biến mất".

Việc thiếu cơ chế quản lý giao dịch (như @Transactional) vi phạm nghiêm trọng tính Atomicity (Tính nguyên tử) trong hệ thống ACID.

Lỗ hổng về Validation (Kiểm tra dữ liệu đầu vào):

Null Pointer Exception (NPE): Hàm orElse(null) trả về null nếu không tìm thấy tài khoản. Ngay sau đó, code gọi trực tiếp from.getBalance() mà không kiểm tra null, chắc chắn gây ra lỗi NullPointerException làm sập luồng xử lý.

Dữ liệu không hợp lệ: Không kiểm tra số tiền chuyển (amount). Người dùng có thể truyền vào số tiền âm (-100,000), dẫn đến việc tài khoản nguồn được cộng thêm tiền và tài khoản đích bị trừ tiền.

Overdraft (Thấu chi vô tội vạ): Không kiểm tra xem số dư tài khoản nguồn có đủ để chuyển hay không (from.getBalance() < amount). Điều này cho phép tài khoản bị âm tiền ngoài ý muốn.

Lỗ hổng về Kiểu dữ liệu (Data Type):

Sử dụng kiểu double cho tiền tệ. Kiểu double sử dụng số thập phân dấu phẩy động (floating-point), dễ dẫn đến sai số làm tròn (rounding errors) trong các phép toán tài chính. Bắt buộc phải dùng BigDecimal.

Lỗ hổng về Bắt Exception & Logging:

Không có bất kỳ cơ chế try-catch hay ném ra các Ngoại lệ tường minh (Custom Exceptions) để tầng Controller xử lý và trả về mã lỗi phù hợp cho Client.

Hoàn toàn không có hệ thống Log (slf4j/logback). Khi xảy ra lỗi hoặc khi cần đối soát giao dịch (Auditing), quản trị viên không có bất kỳ manh mối nào để điều tra vết dòng tiền.

2. Thiết kế chuỗi Prompt Cải tiến đầu ra (3 Vòng)
   Để AI có thể gọt giũa đoạn code này một cách tốt nhất mà không bị "quá tải" dẫn đến sinh code thiếu, chúng ta áp dụng kỹ thuật Prompt Chaining qua 3 vòng tăng dần độ phức tạp.

Vòng 1: Tập trung vào tính Robustness (Độ mạnh mẽ của Code)
Prompt Vòng 1:
"Bạn là một Chuyên gia Đảm bảo Chất lượng Phần mềm (QA/Backend Engineer). Hãy tối ưu hàm transfer dưới đây để tăng tính Robustness.

Yêu cầu cụ thể:

Thay đổi kiểu dữ liệu của amount và balance từ double sang BigDecimal để đảm bảo độ chính xác tài chính.

Bổ sung kiểm tra dữ liệu đầu vào (Validation):

Nếu không tìm thấy fromAccountId hoặc toAccountId, ném ra AccountNotFoundException.

Nếu amount nhỏ hơn hoặc bằng 0, ném ra InvalidArgumentException.

Nếu số dư tài khoản from không đủ để thực hiện giao dịch, ném ra InsufficientBalanceException.

Hãy viết các định nghĩa Custom Exception này (kế thừa từ RuntimeException) và cập nhật lại đoạn code logic của hàm transfer."

Vòng 2: Tập trung vào Maintainability & Clean Code
Prompt Vòng 2:
"Dựa trên mã nguồn đã cải tiến ở Vòng 1, hãy tiếp tục tối ưu hóa cấu trúc code để tăng tính dễ bảo trì (Maintainability) và tích hợp các chuẩn Framework.

Yêu cầu cụ thể:

Sử dụng Spring Boot framework: Thêm annotation @Transactional(rollbackFor = Exception.class) vào tầng Service/Hàm để đảm bảo tính nguyên tử (Atomicity), tự động rollback nếu có bất kỳ Exception nào xảy ra trong quá trình chuyển tiền.

Sử dụng thư viện Lombok: Áp dụng @RequiredArgsConstructor cho class AccountService để thay thế cho việc inject repository thủ công.

Tích hợp Logging: Sử dụng annotation @Slf4j của Lombok. Thực hiện log thông tin ở các cấp độ:

info: Khi bắt đầu giao dịch và khi giao dịch thành công.

warn/error: Khi xảy ra các trường hợp ngoại lệ (không tìm thấy tài khoản, không đủ số dư, số tiền không hợp lệ).

Hãy xuất ra đoạn code hoàn chỉnh."

Vòng 3: Context-specific Tuning (Đầu ra chuẩn hóa & Viết Unit Test)
Prompt Vòng 3:
"Đây là bước hoàn thiện cuối cùng. Hãy tinh chỉnh đầu ra của hàm để phù hợp với môi trường Production thực tế và viết bộ test case để kiểm chứng.

Yêu cầu cụ thể:

Thay vì kiểu trả về void, hãy thiết kế và trả về một đối tượng TransactionResult chứa các thông tin: transactionId (String UUID tự sinh), status (SUCCESS/FAILED), message (String), và timestamp (LocalDateTime).

Cập nhật lại hàm transfer để trả về đối tượng TransactionResult này khi thành công.

Viết một class kiểm thử Mockito JUnit 5 tên là AccountServiceTest. Trong đó, viết riêng một test case cho trường hợp: Chuyển tiền thất bại do tài khoản nguồn không đủ số dư (InsufficientBalanceException). Đảm bảo có sử dụng @Mock cho AccountRepository và verify để chắc chắn rằng hàm save KHÔNG được gọi khi có lỗi xảy ra."

3. Mã nguồn Java và Test Case JUnit tối ưu từ AI
   Dưới đây là toàn bộ mã nguồn hoàn chỉnh sau khi đi qua 3 vòng cải tiến bằng chuỗi Prompt trên.

3.1. Các định nghĩa DTO và Custom Exceptions
Java
package com.bank.dto;

import lombok.Builder;
import lombok.Data;
import java.time.LocalDateTime;

@Data
@Builder
public class TransactionResult {
private String transactionId;
private String status;
private String message;
private LocalDateTime timestamp;
}
Java
package com.bank.exception;

public class AccountNotFoundException extends RuntimeException {
public AccountNotFoundException(String message) { super(message); }
}

public class InvalidArgumentException extends RuntimeException {
public InvalidArgumentException(String message) { super(message); }
}

public class InsufficientBalanceException extends RuntimeException {
public InsufficientBalanceException(String message) { super(message); }
}
3.2. Mã nguồn tầng Service tối ưu
Java
package com.bank.service;

import com.bank.dto.TransactionResult;
import com.bank.exception.AccountNotFoundException;
import com.bank.exception.InvalidArgumentException;
import com.bank.exception.InsufficientBalanceException;
import com.bank.model.Account;
import com.bank.repository.AccountRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

@Service
@RequiredArgsConstructor
@Slf4j
public class AccountService {

    private final AccountRepository accountRepository;

    @Transactional(rollbackFor = Exception.class)
    public TransactionResult transfer(Long fromAccountId, Long toAccountId, BigDecimal amount) {
        String txId = UUID.randomUUID().toString();
        log.info("[TxID: {}] Bắt đầu giao dịch chuyển tiền từ tài khoản {} sang {} với số tiền: {}", 
                txId, fromAccountId, toAccountId, amount);

        // 1. Validation: Kiểm tra số tiền hợp lệ
        if (amount == null || amount.compareTo(BigDecimal.ZERO) <= 0) {
            log.warn("[TxID: {}] Giao dịch thất bại: Số tiền chuyển không hợp lệ ({})", txId, amount);
            throw new InvalidArgumentException("Số tiền chuyển phải lớn hơn 0.");
        }

        // 2. Validation: Kiểm tra sự tồn tại của tài khoản
        Account fromAccount = accountRepository.findById(fromAccountId)
                .orElseThrow(() -> {
                    log.warn("[TxID: {}] Giao dịch thất bại: Không tìm thấy tài khoản nguồn {}", txId, fromAccountId);
                    return new AccountNotFoundException("Không tìm thấy tài khoản nguồn: " + fromAccountId);
                });

        Account toAccount = accountRepository.findById(toAccountId)
                .orElseThrow(() -> {
                    log.warn("[TxID: {}] Giao dịch thất bại: Không tìm thấy tài khoản đích {}", txId, toAccountId);
                    return new AccountNotFoundException("Không tìm thấy tài khoản đích: " + toAccountId);
                });

        // 3. Validation: Kiểm tra số dư tài khoản nguồn
        if (fromAccount.getBalance().compareTo(amount) < 0) {
            log.warn("[TxID: {}] Giao dịch thất bại: Tài khoản {} không đủ số dư. Hiện tại: {}, Yêu cầu: {}", 
                    txId, fromAccountId, fromAccount.getBalance(), amount);
            throw new InsufficientBalanceException("Số dư tài khoản không đủ để thực hiện giao dịch.");
        }

        // 4. Thực hiện trừ/cộng tiền an toàn với BigDecimal
        fromAccount.setBalance(fromAccount.getBalance().subtract(amount));
        toAccount.setBalance(toAccount.getBalance().add(amount));

        // 5. Lưu vào Database
        accountRepository.save(fromAccount);
        accountRepository.save(toAccount);

        log.info("[TxID: {}] Giao dịch chuyển tiền thành công.", txId);

        return TransactionResult.builder()
                .transactionId(txId)
                .status("SUCCESS")
                .message("Chuyển tiền thành công.")
                .timestamp(LocalDateTime.now())
                .build();
    }
}
3.3. JUnit 5 Test Case (Trường hợp lỗi số dư)
Java
package com.bank.service;

import com.bank.exception.InsufficientBalanceException;
import com.bank.model.Account;
import com.bank.repository.AccountRepository;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class AccountServiceTest {

    @Mock
    private AccountRepository accountRepository;

    @InjectMocks
    private AccountService accountService;

    @Test
    @DisplayName("Ném InsufficientBalanceException khi số dư tài khoản nguồn không đủ")
    void transfer_ShouldThrowInsufficientBalanceException_WhenBalanceIsLessThanAmount() {
        // Arrange (Chuẩn bị dữ liệu giả lập)
        Long fromId = 1L;
        Long toId = 2L;
        BigDecimal transferAmount = new BigDecimal("500000"); // Cần chuyển 500k

        Account fromAccount = new Account();
        fromAccount.setId(fromId);
        fromAccount.setBalance(new BigDecimal("200000")); // Nhưng chỉ có 200k

        Account toAccount = new Account();
        toAccount.setId(toId);
        toAccount.setBalance(new BigDecimal("100000"));

        when(accountRepository.findById(fromId)).thenReturn(Optional.of(fromAccount));
        when(accountRepository.findById(toId)).thenReturn(Optional.of(toAccount));

        // Act & Assert (Thực hiện hành động và Kiểm tra ngoại lệ)
        assertThrows(InsufficientBalanceException.class, () -> {
            accountService.transfer(fromId, toId, transferAmount);
        });

        // Verify: Đảm bảo dữ liệu CHƯA hề bị thay đổi hay lưu vào database (Tính nguyên tử được giữ vững)
        verify(accountRepository, never()).save(any(Account.class));
    }
}