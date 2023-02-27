# Java 导出Excel 文件

### 引入maven 依赖

```xml
<dependency>
    <groupId>org.apache.poi</groupId>
    <artifactId>poi</artifactId>
    <version>4.1.2</version>
</dependency>
```

### 通用工具类

```java
package cn.yizhoucp.lanling.api.project.biz.util;

import org.apache.commons.lang3.StringUtils;
import org.apache.poi.hssf.usermodel.*;
import org.apache.poi.ss.usermodel.HorizontalAlignment;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.Objects;

/**
 * @Author 六月
 * @Date 2022/12/13 15:01
 * @Version 1.0
 * 导出excel 工具类
 */
public class ExcelExportUtils {

        protected static final Logger logger = LoggerFactory.getLogger(ExcelExportUtils.class);

    /**
     * @param sheetName
     * @param title 表头
     * @param values 数据
     * @param wb 工作簿
     */
    public static HSSFWorkbook getHSSFWorkbook(String sheetName, String[] title, String[][] values, HSSFWorkbook wb) {
        if (Objects.isNull(title) || title.length == 0) {
            return wb;
        }
        if (Objects.isNull(wb)) {
            wb = new HSSFWorkbook();
        }
        HSSFSheet sheet = wb.createSheet(StringUtils.isBlank(sheetName) ? System.currentTimeMillis() + "" : sheetName);
        HSSFRow row = sheet.createRow(0);
        HSSFCellStyle cellStyle = wb.createCellStyle();
        cellStyle.setAlignment(HorizontalAlignment.CENTER);
        HSSFCell cell;
        // 每一列的标题
        for (int i = 0; i < title.length; i++) {
            cell = row.createCell(i);
            cell.setCellStyle(cellStyle);
            cell.setCellValue(title[i]);
        }
        // 填充每一行的内容
        if (Objects.isNull(values) || values.length == 0) {
            return wb;
        }
        for (int i = 0; i < values.length; i++) {
            row = sheet.createRow(i + 1);
            for (int j = 0; j < values[i].length; j++) {
                row.createCell(j).setCellValue(values[i][j]);
            }
        }
        return wb;
    }


    /**
     * 设置 http 响应头
     *
     * @param response
     * @param fileName
     */
    public static void setResponseHeader(HttpServletResponse response, String fileName) {
        try {
            // 设置响应头
            response.reset();
            //设置响应头
            response.setContentType("application/x-download");
            fileName = "活动榜单 " + System.currentTimeMillis() + ".xls";
            fileName = new String(fileName.getBytes(), "ISO-8859-1");
            response.setHeader("Content-Disposition", "attachment;filename=" + fileName);
        } catch (Exception e) {
            logger.error("setResponseHeader e {}", e);
        }
    }

}

```

### 调整响应格式

```java
    /**
     * 设置 http 响应头
     * @param response
     * @param fileName
     */
    private void setResponseHeader(HttpServletResponse response, String fileName) {
        try {
            fileName = new String(fileName.getBytes(StandardCharsets.UTF_8));
            response.setContentType("application/octet-stream;charset=UTF-8");
            response.setHeader("Content-Disposition", "attachment;filename=" + fileName);
            response.addHeader("Pargam","no-cache");
            response.addHeader("Cache-Control", "no-cache");
        } catch (Exception e) {
            log.error("setResponseHeader e {}", e);
        }
    }
```

### 使用案例
```java
    private void exportToExcel(String fileName, List<AssetsVO> assetsList, HttpServletResponse response) {
        try(HSSFWorkbook workbook =
                    ExcelExportUtils.getHSSFWorkbook("资产统计", getExcelTitle(), getContent(assetsList), null);
            OutputStream outputStream = response.getOutputStream()
        ) {
            // 响应到客户端
            this.setResponseHeader(response, fileName);
            workbook.write(outputStream);
            outputStream.flush();
        } catch (Exception e) {
            log.error("exportExcel error {}", e);
        }
    }
```