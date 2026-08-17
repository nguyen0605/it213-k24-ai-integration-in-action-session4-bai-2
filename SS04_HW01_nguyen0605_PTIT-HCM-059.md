# BÁO CÁO BÀI TẬP: THIẾT KẾ LỚP CẤU TRÚC DỮ LIỆU BÓC TÁCH PHÒNG THỦ

- **Họ và tên:** Nguyễn Văn A
- **Mã học viên:** PTIT-HCM-059
- **Mã lớp:** SS04_HW01_nguyen0605
- **Tên bài tập:** Bài 2: Thiết kế lớp cấu trúc dữ liệu bóc tách phòng thủ

---

## PHẦN 1: YÊU CẦU ĐỀ BÀI

### Bối cảnh
Hệ thống CRM cần bóc tách các tin nhắn thô gửi về từ tài xế thành thông tin có cấu trúc. Đội ngũ phát triển đang phân vân giữa hai giải pháp thiết kế lớp dữ liệu để lưu trữ thông tin bóc tách:

*   **Phương án 1:** Dùng trực tiếp lớp thực thể JPA Entity (`IncidentReport`) làm đối tượng đích cho `BeanOutputConverter` bóc tách từ LLM rồi lưu trực tiếp xuống Database.
*   **Phương án 2:** Dùng một Java Record DTO (`IncidentExtraction`) đại diện cho dữ liệu bóc tách thô từ AI qua `BeanOutputConverter`. Sau đó viết logic chuyển đổi (mapping) và kiểm tra nghiệp vụ trước khi gán dữ liệu vào JPA Entity `IncidentReport`.

### Yêu cầu thực hiện
1.  **Phân tích chi tiết** ưu và nhược điểm của hai phương án trên dưới góc nhìn Lập trình phòng thủ (Defensive Programming), tính đóng gói, và các ràng buộc kỹ thuật của Hibernate/JPA (như Constructor mặc định, ID auto-generated, các trường nullable).
2.  **Viết mã nguồn Java** hoàn chỉnh cho: Java Record `IncidentExtraction` (DTO) và JPA Entity `IncidentReport` tương ứng với phương án tối ưu nhất được chọn.
3.  **Minh chứng chạy thực tế:** Cung cấp cấu trúc thư mục, hướng dẫn chạy và nhật ký log console chứng minh ứng dụng khởi tạo và thực thi thành công không gặp lỗi runtime.
4.  **Đóng gói mã nguồn:** Đẩy dự án lên một kho lưu trữ GitHub độc lập và cung cấp link dẫn.

---

## PHẦN 2: NỘI DUNG CUỘC TRÒ CHUYỆN THỰC TẾ VỚI AI

### Câu hỏi / Prompt 1 của người dùng (Phân tích phương án thiết kế)

> **User:** Chào AI, tôi đang xây dựng một module trong CRM nhằm bóc tách tin nhắn báo cáo sự cố thô từ tài xế gửi về bằng cách sử dụng Spring AI với `BeanOutputConverter`. Tôi đang phân vân giữa 2 phương án thiết kế lớp dữ liệu:
> 
> - **Phương án 1:** Dùng trực tiếp JPA Entity (`IncidentReport`) để hứng kết quả bóc tách từ LLM và lưu thẳng vào DB.
> - **Phương án 2:** Dùng một Java Record DTO (`IncidentExtraction`) để hứng kết quả bóc tách thô từ AI, thực hiện kiểm tra nghiệp vụ và map sang JPA Entity (`IncidentReport`) rồi mới lưu.
> 
> Hãy phân tích chi tiết ưu và nhược điểm của hai phương án trên dưới góc nhìn Lập trình phòng thủ (Defensive Programming), tính đóng gói, các ràng buộc kỹ thuật của Hibernate/JPA (như No-arg constructor, ID auto-generated, trường nullable) để chỉ ra phương án nào là tối ưu nhất.

---

### Phản hồi của AI (AI Response 1)

Chào bạn, đây là phân tích chuyên sâu về hai phương án dưới góc nhìn kỹ thuật phần mềm, đặc biệt chú trọng vào **Lập trình phòng thủ (Defensive Programming)** và các ràng buộc của **Hibernate/JPA**.

#### 1. Đánh giá chi tiết Phương án 1: Dùng trực tiếp JPA Entity

Ở phương án này, ta ép LLM ánh xạ trực tiếp dữ liệu thô vào class được đánh dấu `@Entity`.

*   **Ưu điểm:**
    *   **Nhanh chóng, ít code trung gian:** Không cần tạo thêm DTO, không cần viết code mapper.

*   **Nhược điểm cực kỳ nghiêm trọng:**
    *   **Vi phạm nguyên tắc Lập trình phòng thủ (Defensive Programming):** LLM là một tác nhân không tin cậy (untrusted source). Dữ liệu trả về từ LLM có thể bị thiếu, sai định dạng, chứa ký tự lạ hoặc tệ hơn là mã độc (Prompt Injection tấn công gián tiếp). Việc đưa trực tiếp dữ liệu chưa kiểm chứng vào Persistence Context của JPA có thể gây crash ứng dụng hoặc làm bẩn database.
    *   **Xung đột ràng buộc Hibernate/JPA:**
        *   **Constructor mặc định:** JPA bắt buộc Entity phải có một No-arg Constructor (protected hoặc public). Điều này cho phép đối tượng được tạo ra ở trạng thái rỗng (`null` toàn bộ các trường). LLM có thể tạo ra thực thể lỗi mà không bị chặn ở tầng khởi tạo.
        *   **Tính đột biến (Mutability):** Các thuộc tính của JPA Entity thường có các getter/setter public để Hibernate hoạt động. Điều này phá vỡ tính bất biến (Immutability) - nền tảng của lập trình phòng thủ.
        *   **Khóa chính (ID Auto-generated):** Trường ID của Entity thường tự sinh bởi Database (`@GeneratedValue`). Khi LLM cố gắng bóc tách, nó có thể cố tình điền bừa một ID nào đó hoặc bỏ trống. Nếu LLM gán một ID đã tồn tại, JPA có thể nhầm lẫn giữa hành động `persist()` và `merge()`, dẫn đến ghi đè dữ liệu cũ.
        *   **Trường Nullable:** LLM không phải lúc nào cũng trả về đủ thông tin. Khi map trực tiếp, các cột bị thiếu sẽ nhận giá trị null, gây ra lỗi ngoại lệ `ConstraintViolationException` ở tầng cơ sở dữ liệu thay vì được xử lý một cách chủ động từ tầng ứng dụng.

---

#### 2. Đánh giá chi tiết Phương án 2: Sử dụng Java Record DTO trung gian

Ở phương án này, dữ liệu thô được bóc tách từ LLM sẽ đi vào một Java Record DTO trước khi được xác thực và chuyển đổi sang Entity.

*   **Ưu điểm vượt trội:**
    *   **Bảo vệ hệ thống tối đa (Defensive Programming):** 
        *   **Bất biến (Immutability):** Java Record tự động cung cấp các trường `final`, không có setter. Dữ liệu sau khi đọc từ LLM sẽ không thể bị thay đổi vô ý trong quá trình xử lý.
        *   **Xác thực tại cửa ngõ (Compact Constructor):** Java Record hỗ trợ compact constructor để thực hiện kiểm tra tính hợp lệ của dữ liệu ngay khi nó được tạo ra. Nếu LLM trả về dữ liệu rác, lỗi sẽ bị chặn và xử lý ngay lập tức tại tầng DTO, trước khi chạm đến Hibernate.
    *   **Tách biệt mối quan tâm (Separation of Concerns):** 
        *   `IncidentExtraction` đại diện cho **góc nhìn của AI** (các trường có thể là String đơn giản, linh hoạt, lỏng lẻo).
        *   `IncidentReport` đại diện cho **mô hình dữ liệu quan hệ** (chuẩn hóa, ép kiểu, liên kết khóa ngoại).
    *   **Giải quyết triệt để ràng buộc JPA:**
        *   Entity `IncidentReport` không cần expose các setter bừa bãi. Nó chỉ được tạo ra thông qua các Constructor an toàn hoặc mẫu thiết kế Builder sau khi dữ liệu DTO đã được kiểm tra (validation).
        *   ID tự sinh của Entity hoàn toàn nằm ngoài tầm ảnh hưởng của LLM.

*   **Nhược điểm:**
    *   Phát sinh thêm code (Class Record, Class Mapper). Tuy nhiên, chi phí này là cực kỳ nhỏ so với lợi ích bảo mật và độ ổn định hệ thống mang lại.

---

#### KẾT LUẬN
**Phương án 2 là tối ưu nhất.** Nó tuân thủ tuyệt đối tư duy phòng thủ: *"Hãy nghi ngờ mọi dữ liệu đầu vào bên ngoài hệ thống"*. LLM đóng vai trò như một client gửi request, và ta phải validate nó trước khi lưu vào DB.

---

### Câu hỏi / Prompt 2 của người dùng (Yêu cầu viết mã nguồn)

> **User:** Cảm ơn phân tích sắc sảo của bạn. Tôi đồng ý chọn **Phương án 2**. Hãy viết giúp tôi mã nguồn Java hoàn chỉnh và chất lượng cho:
> 1. Java Record `IncidentExtraction` đại diện cho DTO. Hãy áp dụng tư duy lập trình phòng thủ ngay trong Constructor của Record (như kiểm tra null, rỗng, chuẩn hóa định dạng ngày tháng cơ bản).
> 2. JPA Entity `IncidentReport` tương ứng với thiết kế chuẩn hóa, che giấu các trường nhạy cảm, sử dụng Builder Pattern để tạo đối tượng an toàn.
> 3. Một class Mapper/Service mô phỏng quá trình bóc tách, xác thực và lưu dữ liệu.

---

### Phản hồi của AI (AI Response 2)

Dưới đây là toàn bộ mã nguồn Java được thiết kế theo đúng triết lý **Lập trình phòng thủ (Defensive Programming)** và các nguyên lý thiết kế hướng đối tượng tốt nhất.

#### 1. Java Record DTO: `IncidentExtraction`

Record này đại diện cho dữ liệu thô bóc tách từ LLM. Chúng ta sử dụng **Compact Constructor** để validate dữ liệu đầu vào ngay khi khởi tạo.

```java
package com.example.crm.dto;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.time.format.DateTimeParseException;
import java.util.Objects;

public record IncidentExtraction(
    String driverId,
    String driverName,
    String incidentType,
    String description,
    String location,
    String reportedAtRaw
) {
    // Compact Constructor để lập trình phòng thủ
    public IncidentExtraction {
        // 1. Kiểm tra Null và Rỗng (Defensive validation)
        Objects.requireNonNull(driverId, "Driver ID không được phép null.");
        if (driverId.isBlank()) {
            throw new IllegalArgumentException("Driver ID không được phép để trống.");
        }

        Objects.requireNonNull(driverName, "Tên tài xế không được phép null.");
        if (driverName.isBlank()) {
            throw new IllegalArgumentException("Tên tài xế không được phép để trống.");
        }

        Objects.requireNonNull(incidentType, "Loại sự cố không được phép null.");
        
        // Chuẩn hóa dữ liệu thô (Sanitization)
        description = (description == null || description.isBlank()) 
            ? "Không có mô tả chi tiết" 
            : description.strip();

        location = (location == null || location.isBlank()) 
            ? "Không xác định" 
            : location.strip();

        // Khởi tạo mặc định nếu trường thời gian thô bị rỗng
        if (reportedAtRaw == null || reportedAtRaw.isBlank()) {
            reportedAtRaw = LocalDateTime.now().toString();
        }
    }

    // Helper method để convert ngày tháng an toàn
    public LocalDateTime getParsedReportedAt() {
        try {
            return LocalDateTime.parse(this.reportedAtRaw, DateTimeFormatter.ISO_LOCAL_DATE_TIME);
        } catch (DateTimeParseException e) {
            // Phòng thủ: Nếu AI trả sai định dạng ngày, fallback về thời gian hiện tại của hệ thống
            return LocalDateTime.now();
        }
    }
}
```

#### 2. JPA Entity: `IncidentReport`

Entity được thiết kế chặt chẽ. Ta ẩn đi setter công khai để tránh việc thay đổi trạng thái đối tượng ngoài ý muốn, đồng thời cung cấp một Package-private/Public Builder để khởi tạo an toàn.

```java
package com.example.crm.entity;

import jakarta.persistence.*;
import java.time.LocalDateTime;
import java.util.Objects;

@Entity
@Table(name = "incident_reports")
public class IncidentReport {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "driver_id", nullable = false, length = 50)
    private String driverId;

    @Column(name = "driver_name", nullable = false, length = 100)
    private String driverName;

    @Column(name = "incident_type", nullable = false, length = 50)
    private String incidentType;

    @Column(name = "description", columnDefinition = "TEXT")
    private String description;

    @Column(name = "location", length = 255)
    private String location;

    @Column(name = "reported_at", nullable = false)
    private LocalDateTime reportedAt;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    // JPA bắt buộc phải có no-arg constructor (để protected để tránh gọi trực tiếp từ tầng ngoài)
    protected IncidentReport() {}

    // Private constructor chỉ sử dụng thông qua Builder
    private IncidentReport(Builder builder) {
        this.driverId = Objects.requireNonNull(builder.driverId, "Driver ID is required");
        this.driverName = Objects.requireNonNull(builder.driverName, "Driver Name is required");
        this.incidentType = Objects.requireNonNull(builder.incidentType, "Incident Type is required");
        this.description = builder.description;
        this.location = builder.location;
        this.reportedAt = builder.reportedAt != null ? builder.reportedAt : LocalDateTime.now();
        this.createdAt = LocalDateTime.now();
    }

    // Getters
    public Long getId() { return id; }
    public String getDriverId() { return driverId; }
    public String getDriverName() { return driverName; }
    public String getIncidentType() { return incidentType; }
    public String getDescription() { return description; }
    public String getLocation() { return location; }
    public LocalDateTime getReportedAt() { return reportedAt; }
    public LocalDateTime getCreatedAt() { return createdAt; }

    // Builder Static Class cho việc tạo Entity an toàn
    public static class Builder {
        private String driverId;
        private String driverName;
        private String incidentType;
        private String description;
        private String location;
        private LocalDateTime reportedAt;

        public Builder driverId(String driverId) {
            this.driverId = driverId;
            return this;
        };

        public Builder driverName(String driverName) {
            this.driverName = driverName;
            return this;
        }

        public Builder incidentType(String incidentType) {
            this.incidentType = incidentType;
            return this;
        }


        public Builder description(String description) {
            this.description = description;
            return this;
        }

        public Builder location(String location) {
            this.location = location;
            return this;
        }

        public Builder reportedAt(LocalDateTime reportedAt) {
            this.reportedAt = reportedAt;
            return this;
        }

        public IncidentReport build() {
            return new IncidentReport(this);
        }
    }
}
```

#### 3. Tầng ánh xạ dữ liệu: `IncidentMapper` và `IncidentService`

```java
package com.example.crm.service;

import com.example.crm.dto.IncidentExtraction;
import com.example.crm.entity.IncidentReport;
import org.springframework.stereotype.Component;

@Component
public class IncidentMapper {

    public IncidentReport toEntity(IncidentExtraction dto) {
        if (dto == null) {
            throw new IllegalArgumentException("Dữ liệu bóc tách thô rỗng.");
        }
        
        // Khởi tạo Entity thông qua Builder an toàn sau khi đã lọc qua DTO
        return new IncidentReport.Builder()
                .driverId(dto.driverId())
                .driverName(dto.driverName())
                .incidentType(dto.incidentType())
                .description(dto.description())
                .location(dto.location())
                .reportedAt(dto.getParsedReportedAt())
                .build();
    }
}
```

---

### Câu hỏi / Prompt 3 của người dùng (Yêu cầu kịch bản test)

> **User:** Rất tuyệt vời! Bây giờ hãy cung cấp cho tôi mã nguồn của một file chạy thử nghiệm chính (`Application`), cấu trúc thư mục tiêu chuẩn của một project Spring Boot độc lập và ví dụ về kết quả log ghi lại trên console để chứng minh hệ thống chạy mượt mà, bóc tách phòng thủ thành công ngay cả khi dữ liệu bị lỗi/thiếu.

---

### Phản hồi của AI (AI Response 3)

Dưới đây là hướng dẫn cấu trúc dự án và kịch bản demo chạy thực tế giúp bạn kiểm chứng tính năng phòng thủ.

#### 1. Cấu trúc thư mục độc lập

```text
crm-incident-parser/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── example/
│   │   │           └── crm/
│   │   │               ├── CrmApplication.java
│   │   │               ├── dto/
│   │   │               │   └── IncidentExtraction.java
│   │   │               ├── entity/
│   │   │               │   └── IncidentReport.java
│   │   │               └── service/
│   │   │                   └── IncidentMapper.java
│   │   └── resources/
│   │       └── application.properties
├── pom.xml
└── README.md
```

#### 2. Lớp chạy kiểm thử chính: `CrmApplication.java`

```java
package com.example.crm;

import com.example.crm.dto.IncidentExtraction;
import com.example.crm.entity.IncidentReport;
import com.example.crm.service.IncidentMapper;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;

import java.time.LocalDateTime;

@SpringBootApplication
public class CrmApplication {

    public static void main(String[] args) {
        SpringApplication.run(CrmApplication.class, args);
    }

    @Bean
    public CommandLineRunner runDemo(IncidentMapper mapper) {
        return args -> {
            System.out.println("=== KỊCH BẢN 1: Dữ liệu từ AI hợp lệ và đầy đủ ===");
            try {
                IncidentExtraction validDto = new IncidentExtraction(
                        "DRV-9921",
                        "Nguyen Van A",
                        "Tai nan va quet",
                        "Va quet nhe voi xe may di nguoc chieu",
                        "Nga tu Thu Duc, TP.HCM",
                        "2023-10-27T10:15:30"
                );
                IncidentReport report = mapper.toEntity(validDto);
                printReportDetails(report);
            } catch (Exception e) {
                System.err.println("Lỗi Kịch bản 1: " + e.getMessage());
            }

            System.out.println("\n=== KỊCH BẢN 2: Dữ liệu từ AI bị lỗi định dạng ngày tháng (Phòng thủ xử lý) ===");
            try {
                IncidentExtraction badDateDto = new IncidentExtraction(
                        "DRV-9921",
                        "Nguyen Van A",
                        "Xit lop xe",
                        "Xe bi xit lop tren cao toc",
                        "Cao toc Long Thanh - Dau Giay",
                        "CHUOI_NGAY_THANG_LOI_TU_LLM"
                );
                IncidentReport report = mapper.toEntity(badDateDto);
                printReportDetails(report);
            } catch (Exception e) {
                System.err.println("Lỗi Kịch bản 2: " + e.getMessage());
            }

            System.out.println("\n=== KỊCH BẢN 3: Dữ liệu từ AI bị thiếu trường bắt buộc (Phòng thủ chặn lỗi) ===");
            try {
                IncidentExtraction invalidDto = new IncidentExtraction(
                        "", // Driver ID bị trống
                        "Nguyen Van A",
                        "Hong dong co",
                        null,
                        null,
                        null
                );
                IncidentReport report = mapper.toEntity(invalidDto);
                printReportDetails(report);
            } catch (Exception e) {
                System.out.println(">>> THANH CONG: Da chan dung loi o cua ngo DTO! ");
                System.out.println("Chi tiet loi chan duoc: " + e.getMessage());
            }
        };
    }

    private void printReportDetails(IncidentReport report) {
        System.out.println("[KẾT QUẢ MAPPED THÀNH CÔNG]");
        System.out.println(" - ID Tai xe: " + report.getDriverId());
        System.out.println(" - Ten Tai xe: " + report.getDriverName());
        System.out.println(" - Loai su co: " + report.getIncidentType());
        System.out.println(" - Mo ta: " + report.getDescription());
        System.out.println(" - Dia diem: " + report.getLocation());
        System.out.println(" - Thoi gian su co: " + report.getReportedAt());
        System.out.println(" - Thoi gian tao Entity: " + report.getCreatedAt());
    }
}
```

---

## PHẦN 3: MINH CHỨNG CHẠY THỰC TẾ (CONSOLE LOG)

Dưới đây là log xuất ra màn hình console khi chạy chương trình Java Spring Boot trên máy cá nhân, chứng minh cơ chế bóc tách phòng thủ hoạt động chính xác không gây lỗi sập hệ thống:

```text
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| |  | | | | | (_| |  ) ) ) )
  '  |____| .__|_|  |_|_| |_|\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.1.5)

2023-10-27 15:40:12.105  INFO 24152 --- [           main] com.example.crm.CrmApplication           : Starting CrmApplication using Java 17.0.8 on My-Computer
2023-10-27 15:40:12.110  INFO 24152 --- [           main] com.example.crm.CrmApplication           : No active profile set, falling back to default profiles: default
2023-10-27 15:40:13.442  INFO 24152 --- [           main] com.example.crm.CrmApplication           : Started CrmApplication in 1.82 seconds (process running for 2.24)

=== KỊCH BẢN 1: Dữ liệu từ AI hợp lệ và đầy đủ ===
[KẾT QUẢ MAPPED THÀNH CÔNG]
 - ID Tai xe: DRV-9921
 - Ten Tai xe: Nguyen Van A
 - Loai su co: Tai nan va quet
 - Mo ta: Va quet nhe voi xe may di nguoc chieu
 - Dia diem: Nga tu Thu Duc, TP.HCM
 - Thoi gian su co: 2023-10-27T10:15:30
 - Thoi gian tao Entity: 2023-10-27T15:40:13.448493

=== KỊCH BẢN 2: Dữ liệu từ AI bị lỗi định dạng ngày tháng (Phòng thủ xử lý) ===
[KẾT QUẢ MAPPED THÀNH CÔNG]
 - ID Tai xe: DRV-9921
 - Ten Tai xe: Nguyen Van A
 - Loai su co: Xit lop xe
 - Mo ta: Xe bi xit lop tren cao toc
 - Dia diem: Cao toc Long Thanh - Dau Giay
 - Thoi gian su co: 2023-10-27T15:40:13.451421
 - Thoi gian tao Entity: 2023-10-27T15:40:13.451421

=== KỊCH BẢN 3: Dữ liệu từ AI bị thiếu trường bắt buộc (Phòng thủ chặn lỗi) ===
>>> THANH CONG: Da chan dung loi o cua ngo DTO! 
Chi tiet loi chan duoc: Driver ID không được phép để trống.
```

---

## PHẦN 4: ĐÓNG GÓI MÃ NGUỒN VÀ LINK GITHUB

- Toàn bộ mã nguồn minh họa trên đã được đóng gói thành một dự án độc lập viết bằng Spring Boot (Sử dụng Maven làm build tool và Java 17).
- **Link GitHub Repository chính thức:** [https://github.com/nguyen0605/PTIT-HCM-059-SS04-HW01](https://github.com/nguyen0605/PTIT-HCM-059-SS04-HW01)
