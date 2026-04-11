+++
title = '接口 + 抽象模板组合模式：让子类只写核心逻辑'
date = 2026-04-11T10:00:00+08:00
draft = false
description = '通过"接口定义契约 + 抽象类控制流程 + 子类专注业务"的组合设计，消除重复代码，以 JdbcTemplate 为例深入理解该模式的工程实践'

categories = ['java']
tags = ['java', 'design-pattern', 'template-method', 'spring']
+++

## 🤔 你是否在子类里写了大量重复的"流程代码"？

假设你有三个业务 Service：订单服务、支付服务、库存服务。它们各自的 `apply()` 方法看起来是这样的：

```java
public class OrderService {
    public Result apply(Request request) {
        validate(request);          // 参数校验
        initContext();              // 初始化上下文
        try {
            Result result = doOrder(request);  // ← 唯一不同的部分
            postProcess(result);    // 后置处理
            return result;
        } catch (Exception e) {
            handleError(e);         // 异常处理
            throw e;
        } finally {
            cleanUp();              // 资源清理
        }
    }
}
```

把上面代码里的 `doOrder` 换成 `doPayment`、`doInventory`，你会发现**流程完全一样**，只有核心业务逻辑那一行不同。

⚠️️这就是问题所在：**相同的流程控制代码在每个子类中重复出现**。如果某天需要在所有服务中增加一个"日志记录"步骤，你得挨个修改每一个 Service。

本文将介绍一种**接口 + 抽象模板的组合模式**，让通用流程只写一次，子类只需关注自己的核心逻辑。

读完本文，你将了解：

- "接口定义契约 + 抽象模板控制流程"的组合设计思想
- Spring 的 `JdbcTemplate` 是如何运用这一思想的

## 🏗️ 核心思想：三层分工

这个模式的核心可以用一句话概括：

> **接口定义契约，抽象模板控制流程，子类专注实现具体逻辑。**

三个角色各有明确的职责边界：

| 角色 | 职责 | 关键手段 |
|------|------|----------|
| **接口** | 定义"做什么"——对外暴露的业务契约 | 声明 `apply()` 方法 |
| **抽象模板类** | 定义"怎么做"——控制不可变的通用流程 | 实现 `apply()` 为 `final`，内部调用 `doApply()` |
| **具体子类** | 实现"做哪样"——只填入核心业务逻辑 | 重写 `protected doApply()` |

为什么不能只用接口或只用抽象类？

- **只用接口**：无法复用流程代码，每个实现类都要重复写一遍 `validate → try → do → catch → finally`。
- **只用抽象类**：可以实现模板方法，但一个类只能单继承，限制了灵活性。而且调用方无法面向接口编程。
- **接口 + 抽象类组合**：接口提供了多态和面向接口编程的能力，抽象类提供了流程复用的能力。两者互补。

## 🛠️ 一步步实现组合模式

### 第一步：接口定义业务契约

接口只关心"能做什么"，不关心"怎么做"：

```java
public interface BusinessService {
    Result apply(Request request);
}
```

这个接口是对外的契约。调用方只需要知道 `BusinessService` 有一个 `apply()` 方法，不需要知道背后是哪个具体实现。

### 第二步：抽象模板封装通用流程

这是整个模式的核心。抽象类实现接口，把不可变的流程写死在 `apply()` 中，把可变的部分留给子类：

```java
public abstract class AbstractBusinessTemplate implements BusinessService {

    // 声明为 final，子类不能覆盖流程
    @Override
    public final Result apply(Request request) {
        validateRequest(request);
        initializeContext();

        try {
            // 核心：调用子类实现的具体逻辑
            Result result = doApply(request);

            postProcess(result);
            return result;
        } catch (Exception e) {
            handleException(e);
            throw e;
        } finally {
            cleanUp();
        }
    }

    // 子类必须实现的抽象方法
    protected abstract Result doApply(Request request);

    // 以下通用方法提供默认实现，子类可选择性覆盖
    protected void validateRequest(Request request) {
        if (request == null) {
            throw new IllegalArgumentException("Request cannot be null");
        }
    }

    protected void initializeContext() { /* 默认实现 */ }

    protected void postProcess(Result result) { /* 默认实现 */ }

    protected void handleException(Exception e) {
        // 默认异常处理，比如记录日志
    }

    protected void cleanUp() { /* 默认清理 */ }
}
```

注意几个关键设计决策：

1. **`apply()` 声明为 `final`**：防止子类覆盖流程。流程是"不变的骨架"，不应该被篡改。
2. **`doApply()` 声明为 `protected abstract`**：子类只能实现这个方法。`protected` 确保只有子类能看到它，外部调用方不会直接调用。
3. **通用方法提供默认实现**：`validateRequest`、`postProcess` 等方法提供了钩子（Hook），子类可以按需覆盖，但不是必须的。

### 第三步：子类只写核心逻辑

```java
public class OrderService extends AbstractBusinessTemplate {

    @Override
    protected Result doApply(Request request) {
        // 只需要关注核心业务逻辑
        Order order = createOrder(request);
        orderRepository.save(order);
        return Result.success(order);
    }
}

public class PaymentService extends AbstractBusinessTemplate {

    @Override
    protected Result doApply(Request request) {
        // 每个子类只写自己的核心逻辑
        Payment payment = processPayment(request);
        paymentGateway.charge(payment);
        return Result.success(payment);
    }
}
```

对比重构前后：原来每个 Service 要写 15-20 行流程代码，现在只需要 3-5 行核心逻辑。

### 调用方式

调用方完全面向接口编程，不知道也不需要知道模板的存在：

```java
BusinessService service = new OrderService();
Result result = service.apply(request);
```

## 📦 实战：数据导出服务

上一节讲了模式的骨架，现在来看一个后端开发中常见的真实场景——**数据导出**。

假设系统需要支持多种格式的数据导出：CSV、Excel、PDF。不管导出哪种格式，流程都是一样的：

```
查询数据 → 生成文件 → 上传到存储 → 发送通知
```

唯一不同的是"生成文件"这一步。这正好是"接口 + 抽象模板"模式的用武之地。

### 第一步：定义领域模型

```java
// 导出请求
public class ExportRequest {
    private String dataType;       // 导出数据类型，如 "order"、"user"
    private LocalDate startDate;
    private LocalDate endDate;
    private String operatorId;     // 操作人
    // getter / setter 省略
}

// 导出结果
public class ExportResult {
    private String fileUrl;        // 文件下载地址
    private String fileName;
    private long file_size;        // 文件大小（字节）
    // getter / setter 省略
}
```

### 第二步：接口定义导出契约

```java
public interface DataExporter {
    ExportResult export(ExportRequest request);
}
```

### 第三步：抽象模板封装导出流程

```java
public abstract class AbstractDataExporter implements DataExporter {

    @Override
    public final ExportResult export(ExportRequest request) {
        // 1. 参数校验
        validateRequest(request);

        // 2. 查询数据
        List<Map<String, Object>> data = queryData(request);

        // 3. 生成文件（由子类实现）
        byte[] fileContent = generateFile(data);
        String fileName = buildFileName(request);

        // 4. 上传文件
        String fileUrl = uploadFile(fileName, fileContent);

        // 5. 发送通知
        notifyOperator(request.getOperatorId(), fileUrl);

        // 6. 构造结果
        ExportResult result = new ExportResult();
        result.setFileUrl(fileUrl);
        result.setFileName(fileName);
        result.setFileSize(fileContent.length);
        return result;
    }

    // 子类必须实现：按特定格式生成文件
    protected abstract byte[] generateFile(List<Map<String, Object>> data);

    // 子类必须实现：声明文件格式后缀
    protected abstract String getFileExtension();

    // 通用校验逻辑
    protected void validateRequest(ExportRequest request) {
        if (request.getStartDate() == null || request.getEndDate() == null) {
            throw new IllegalArgumentException("导出时间范围不能为空");
        }
        if (request.getStartDate().isAfter(request.getEndDate())) {
            throw new IllegalArgumentException("开始时间不能晚于结束时间");
        }
    }

    // 通用数据查询（实际项目中注入 Repository）
    protected List<Map<String, Object>> queryData(ExportRequest request) {
        // 根据 request.getDataType() 和时间范围查询数据
        return dataRepository.queryByDateRange(
            request.getDataType(),
            request.getStartDate(),
            request.getEndDate()
        );
    }

    // 通用文件上传
    private String uploadFile(String fileName, byte[] content) {
        return storageService.upload(fileName, content);
    }

    // 通用通知
    private void notifyOperator(String operatorId, String fileUrl) {
        notificationService.send(operatorId, "导出完成：" + fileUrl);
    }

    // 构建文件名
    private String buildFileName(ExportRequest request) {
        return request.getDataType() + "_"
            + request.getStartDate() + "_"
            + request.getEndDate()
            + "." + getFileExtension();
    }
}
```

### 第四步：具体子类只写文件生成逻辑

```java
// CSV 导出
public class CsvDataExporter extends AbstractDataExporter {

    @Override
    protected byte[] generateFile(List<Map<String, Object>> data) {
        StringBuilder sb = new StringBuilder();

        // 写表头
        if (!data.isEmpty()) {
            String header = String.join(",", data.get(0).keySet());
            sb.append(header).append("\n");
        }

        // 写数据行
        for (Map<String, Object> row : data) {
            String line = row.values().stream()
                .map(v -> String.valueOf(v))
                .collect(Collectors.joining(","));
            sb.append(line).append("\n");
        }

        return sb.toString().getBytes(StandardCharsets.UTF_8);
    }

    @Override
    protected String getFileExtension() {
        return "csv";
    }
}

// Excel 导出（假设使用 Apache POI）
public class ExcelDataExporter extends AbstractDataExporter {

    @Override
    protected byte[] generateFile(List<Map<String, Object>> data) {
        try (Workbook workbook = new XSSFWorkbook()) {
            Sheet sheet = workbook.createSheet("数据导出");

            // 写表头
            Row headerRow = sheet.createRow(0);
            if (!data.isEmpty()) {
                int col = 0;
                for (String key : data.get(0).keySet()) {
                    headerRow.createCell(col++).setCellValue(key);
                }
            }

            // 写数据行
            int rowIdx = 1;
            for (Map<String, Object> rowData : data) {
                Row row = sheet.createRow(rowIdx++);
                int col = 0;
                for (Object value : rowData.values()) {
                    row.createCell(col++).setCellValue(String.valueOf(value));
                }
            }

            ByteArrayOutputStream out = new ByteArrayOutputStream();
            workbook.write(out);
            return out.toByteArray();
        } catch (IOException e) {
            throw new RuntimeException("Excel 生成失败", e);
        }
    }

    @Override
    protected String getFileExtension() {
        return "xlsx";
    }
}
```

### 调用方式与扩展

```java
// 根据用户选择的格式，创建对应的导出器
DataExporter exporter = exporterFactory.getExporter("csv");
ExportResult result = exporter.export(request);
// result.getFileUrl() → "https://storage.example.com/order_2026-01-01_2026-03-31.csv"
```

如果将来需要支持 PDF 导出，只需新增一个 `PdfDataExporter`，实现 `generateFile()` 和 `getFileExtension()`，其他流程代码一行都不用改。

这正体现了**开闭原则**：对扩展开放（新增导出格式），对修改关闭（不动已有的导出器和模板）。

## 从 JdbcTemplate 看真实框架的应用

Spring 的 `JdbcTemplate` 并没有严格使用继承式模板方法，而是用**回调接口**实现了同样的思想。理解它有助于你看到这个模式的另一种变体。

### JdbcTemplate 的"接口 + 模板"结构

```java
// 1. 回调接口 —— 相当于 doApply()
interface StatementCallback<T> {
    T doInStatement(Statement stmt) throws SQLException;
}
```

`StatementCallback` 就是那个"可变的部分"，用户通过实现它来提供具体的 SQL 操作逻辑。

```java
// 2. 模板类 —— 相当于 apply() 的流程控制
public class JdbcTemplate {

    public <T> T execute(StatementCallback<T> action) throws DataAccessException {
        Connection con = DataSourceUtils.getConnection(obtainDataSource());
        Statement stmt = null;
        try {
            stmt = con.createStatement();
            // 调用用户提供的具体逻辑
            T result = action.doInStatement(stmt);
            return result;
        } catch (SQLException ex) {
            // 统一异常转换
            throw translateException("StatementCallback", getSql(action), ex);
        } finally {
            // 统一资源清理
            JdbcUtils.closeStatement(stmt);
            DataSourceUtils.releaseConnection(con, getDataSource());
        }
    }
}
```

注意 `execute()` 方法里的流程：获取连接 → 创建 Statement → 执行回调 → 异常处理 → 资源清理。这和我们的 `apply()` 流程如出一辙。

```java
// 3. 用户代码 —— 只关注 SQL 操作本身
jdbcTemplate.execute(new StatementCallback<Integer>() {
    @Override
    public Integer doInStatement(Statement stmt) throws SQLException {
        stmt.execute("SELECT COUNT(*) FROM users");
        ResultSet rs = stmt.getResultSet();
        rs.next();
        return rs.getInt(1);
    }
});
```

### 两种实现的对比

| 维度 | 继承式（AbstractBusinessTemplate） | 回调式（JdbcTemplate） |
|------|------|------|
| 扩展方式 | 继承抽象类 | 实现回调接口 |
| 灵活性 | 单继承限制 | 可组合多个回调 |
| 适用场景 | 业务逻辑有明确的父子关系 | 一次性操作、策略型逻辑 |
| 核心思想 | 一样：固定流程 + 可变逻辑 | 一样：固定流程 + 可变逻辑 |

### ⚡ 容易踩的坑

1. **忘记把 `apply()` 声明为 `final`**：子类可能不小心覆盖了流程，导致通用逻辑被跳过。如果你使用继承式实现，务必加 `final`。

2. **在 `doApply()` 中写流程控制代码**：`doApply()` 应该只包含纯业务逻辑。如果你发现自己在这里做参数校验或资源清理，说明抽象模板的流程设计有问题。

3. **通用方法过度设计**：不需要一开始就抽出 10 个 Hook 方法。先写 2-3 个最常用的，等有实际需求时再扩展。

## ✅ 总结

"接口 + 抽象模板"组合模式的核心思想是**分离不变和可变**：

- **不变的部分**（流程控制）放在抽象类的 `final` 方法里
- **可变的部分**（业务逻辑）交给子类的 `doApply()` 实现
- **契约**（对外暴露什么）由接口定义

这个模式在框架设计中非常常见。无论是 Spring 的 `JdbcTemplate`、`TransactionTemplate`，还是业务系统中的订单处理、审批流程，只要存在"相同流程、不同业务"的场景，都可以用这个思想来消除重复代码。

下次你发现自己在多个 Service 中写着相似的 `try-catch-finally` 结构时，不妨试试这个模式——通用流程放在抽象模板类，让子类只写核心逻辑。