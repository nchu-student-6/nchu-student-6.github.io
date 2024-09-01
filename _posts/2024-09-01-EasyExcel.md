---
layout:     post
title:      EasyExcel实战
subtitle:   EasyExcel用于处理大文件Excel，相较于传统的 Excel 解析工具（如 Hutool），可以解决内存溢出问题，它通过流式处理数据，有效地降低了内存占用。
date:       2024-09-01
author:     Zheng Yang
header-img: img/post-bg-article.jpg
catalog: true
tags:
    - EasyExcel
---
# EasyExcel解析百万Excel创建批量分发任务

## 业务背景

项目中优惠券的分发：获取到用户信息的 Excel 后，将优惠券写入到用户领券列表中，同时根据配置选择是否通知用户，通知的话有短信、微信公众号、邮件等。

[![image-20240822184743746.png](https://i.postimg.cc/Y0kbvHWV/image-20240822184743746.png)](https://postimg.cc/jwgz1BvQ)
用户信息的 Excel 从哪里来？一般来说，可以通过数据仓库里提取。

> *数据仓库指的是数仓，一个专门设计用于数据存储和分析的系统。它用于集成、存储和管理来自不同来源的数据，并提供对这些数据的高效查询和分析功能。*

例如，如果我们要上线一家高端服装店，为了提升其生意，我们可以**从数据仓库中提取长期浏览高端服装或已经购买过类似品牌或价位的用户信息**，然后将优惠券和通知发送到这些用户的账户。这样可以精准地锁定潜在客户，提高营销效果。

## 数据库表设计

进入 `one_coupon_rebuild_0` 数据库中执行下述 SQL 语句。

```java
CREATE TABLE `t_coupon_task` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'ID',
  `shop_number` bigint(20) DEFAULT NULL COMMENT '店铺编号',
  `batch_id` bigint(20) DEFAULT NULL COMMENT '批次ID',
  `task_name` varchar(128) DEFAULT NULL COMMENT '优惠券批次任务名称',
  `file_address` varchar(512) DEFAULT NULL COMMENT '文件地址',
  `fail_file_address` varchar(512) DEFAULT NULL COMMENT '发放失败用户文件地址',
  `send_num` int(11) DEFAULT NULL COMMENT '发放优惠券数量',
  `notify_type` varchar(32) DEFAULT NULL COMMENT '通知方式，可组合使用 0：站内信 1：弹框推送 2：邮箱 3：短信',
  `coupon_template_id` bigint(20) DEFAULT NULL COMMENT '优惠券模板ID',
  `send_type` tinyint(1) DEFAULT NULL COMMENT '发送类型 0：立即发送 1：定时发送',
  `send_time` datetime DEFAULT NULL COMMENT '发送时间',
  `status` tinyint(1) DEFAULT NULL COMMENT '状态 0：待执行 1：执行中 2：执行失败 3：执行成功 4：取消',
  `completion_time` datetime DEFAULT NULL COMMENT '完成时间',
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  `operator_id` bigint(20) DEFAULT NULL COMMENT '操作人',
  `update_time` datetime DEFAULT NULL COMMENT '修改时间',
  `del_flag` tinyint(1) DEFAULT NULL COMMENT '删除标识 0：未删除 1：已删除',
  PRIMARY KEY (`id`),
  KEY `idx_batch_id` (`batch_id`) USING BTREE,
  KEY `idx_coupon_template_id` (`coupon_template_id`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=1816672964423188483 DEFAULT CHARSET=utf8mb4 COMMENT='优惠券模板发送任务表';
```



我们针对一些核心字段做个讲解：

- file_address`：文件地址，保存分发目标用户的 Excel 文件地址。
- fail_file_address`：发放失败用户文件地址，如果发放执行过程中失败，需要保存错误信息生成一个新的 Excel。
- send_num`：发放优惠券数量，file_address 中共有多少条记录，方便后续记录是否发放完成。

## 生成百万测试 Excel 文件

### 1. Excel 中有哪些字段？

上面的数据库表中有个字段是通知方式，一共有四个值：

- 站内信：需要用户 ID。
- 弹框推送：需要用户 ID。
- 邮箱：需要用户邮箱，这个属于是考虑到了，实际中基本不存在。
- 短信：需要用户手机号，有些公司考虑到用户隐私泄露问题，可能也是记录用户 ID，发送时查询用户接口获取。

那基于上面的描述，我们需要搞三个字段，用户 ID、邮箱、手机号，接下来开始模拟记录。

### 2. 什么是 Faker？

此 Faker 非彼 Faker。咱们这个章节聊的 Faker 是一个开源库，提供了生成伪随机数据的功能。该库可以用来生成各种各样的测试数据，例如姓名、地址、电话号码、电子邮件、公司名、日期等。

那我们先引入，试试效果怎么样。

#### 2.1 引入 Faker Maven 依赖

```java
<!-- Mock 数据相关依赖 -->
<dependency>
    <groupId>com.github.javafaker</groupId>
    <artifactId>javafaker</artifactId>
    <scope>test</scope>
    <version>1.0.2</version>
</dependency>
```

#### 2.2 写个单元测试

通过一个简单的单元测试让大家熟悉下 Faker 怎么使用。

```java
package com.nageoffer.onecoupon.merchant.admin.task;

import com.github.javafaker.Address;
import com.github.javafaker.Faker;
import com.github.javafaker.PhoneNumber;
import org.junit.jupiter.api.Test;

import java.util.Locale;

/**
 * Faker 单元测试类
 */
public class FakerTests {

    @Test
    public void testFaker() {
        // 创建一个 Faker 实例
        Faker faker = new Faker(Locale.CHINA);

        // 生成中文名
        String chineseName = faker.name().fullName();
        System.out.println("中文名: " + chineseName);

        // 生成手机号
        PhoneNumber phoneNumber = faker.phoneNumber();
        String mobileNumber = phoneNumber.cellPhone();
        System.out.println("手机号: " + mobileNumber);

        // 生成电子邮箱
        String email = faker.internet().emailAddress();
        System.out.println("电子邮箱: " + email);
    }
}
```

打印日志如下：

```java
中文名: 沈烨霖
手机号: 15109362990
电子邮箱: 明哲.孙@gmail.com
```

### 3. 什么是 EasyExcel？

EasyExcel 是一个基于 Java 的、快速、简洁、解决大文件内存溢出的 Excel 处理工具。他能让你在不用考虑性能、内存的等因素的情况下，快速完成 Excel 的读、写等功能。

我们在生成 Excel 文件时，刚好使用 EasyExcel 操作，可以看出非常的便捷。

> *官网地址：https://easyexcel.opensource.alibaba.com/*

#### 3.1 引入 EasyExcel Maven 依赖

```java
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>easyexcel</artifactId>
    <version>4.0.1</version>
</dependency>
```

#### 3.2 生成百万用户 Excel

基于 Faker 生成示例数据，将示例数据执行 EasyExcel 数据写入流程，最终保存到项目的 /tmp 文件中。

```java
package com.nageoffer.onecoupon.merchant.admin.task;

import cn.hutool.core.io.FileUtil;
import cn.hutool.core.util.IdUtil;
import com.alibaba.excel.EasyExcel;
import com.alibaba.excel.annotation.ExcelProperty;
import com.alibaba.excel.annotation.write.style.ColumnWidth;
import com.alibaba.excel.util.ListUtils;
import com.github.javafaker.Faker;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.junit.jupiter.api.Test;

import java.nio.file.Paths;
import java.util.List;
import java.util.Locale;

/**
 * 百万 Excel 文件生成单元测试
 * <p>
 * 作者：马丁
 * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
 * 开发时间：2024-07-12
 */
public final class ExcelGenerateTests {

    /**
     * 写入优惠券推送示例 Excel 的数据，自行控制即可
     */
    private final int writeNum = 5000;
    private final Faker faker = new Faker(Locale.CHINA);
    private final String excelPath = Paths.get("").toAbsolutePath().getParent() + "/tmp";

    @Test
    public void testExcelGenerate() {
        if (!FileUtil.exist(excelPath)) {
            FileUtil.mkdir(excelPath);
        }
        String fileName = excelPath + "/oneCoupon任务推送Excel.xlsx";
        EasyExcel.write(fileName, ExcelGenerateDemoData.class).sheet("优惠券推送列表").doWrite(data());
    }

    private List<ExcelGenerateDemoData> data() {
        List<ExcelGenerateDemoData> list = ListUtils.newArrayList();
        for (int i = 0; i < writeNum; i++) {
            ExcelGenerateDemoData data = ExcelGenerateDemoData.builder()
                    .mail(faker.number().digits(10) + "@163.com")
                    .phone(faker.phoneNumber().cellPhone())
                    .userId(IdUtil.getSnowflakeNextIdStr())
                    .build();
            list.add(data);
        }
        return list;
    }


    /**
     * 百万 Excel 生成器示例数据模型
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2024-07-12
     */
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    static class ExcelGenerateDemoData {

        @ColumnWidth(30)
        @ExcelProperty("用户ID")
        private String userId;

        @ColumnWidth(20)
        @ExcelProperty("手机号")
        private String phone;

        @ColumnWidth(30)
        @ExcelProperty("邮箱")
        private String mail;
    }
}
```

执行这个单元测试后会在项目根目录下创建 /tmp 文件夹，文件夹下就是咱们的 Excel 数据文件。

为了避免这种测试数据文件上传到 Git 项目，我们需要在 `.gitignore` 忽略文件中添加 tmp 目录，如下图所示：

[![image-20240822201434250.png](https://i.postimg.cc/P5NKpKCt/image-20240822201434250.png)](https://postimg.cc/zyZT4CwM)
#### 3.3 EasyExcel 注解讲解

- @ColumnWidth(30)：表示当前列占单元格多大宽度。
- @ExcelProperty("用户ID")：写入的表头标题。

## 开发创建优惠券分发任务

### 1. 生成后的 Excel 文件

我们调用上面的生成 Excel 单元测试后，会生成一个 Excel 文件。可以看到，一个 100 万记录的 Excel 在 30M 左右。

[![image-20240822205334855.png](https://i.postimg.cc/Njy7fMnF/image-20240822205334855.png)](https://postimg.cc/LJpZxH2F)### 2. Hutool 获取 Excel 文件行数

为了对比 EasyExcel 提到的内存安全，我们先尝试使用 Hutool 中的 Excel 工具获取下 Excel 行数，看看效果怎么样。

```java
package com.nageoffer.onecoupon.merchant.admin.service.impl;

import cn.hutool.core.bean.BeanUtil;
import cn.hutool.core.util.IdUtil;
import cn.hutool.poi.excel.ExcelReader;
import cn.hutool.poi.excel.ExcelUtil;
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import com.nageoffer.onecoupon.framework.exception.ClientException;
import com.nageoffer.onecoupon.merchant.admin.common.context.UserContext;
import com.nageoffer.onecoupon.merchant.admin.common.enums.CouponTaskSendTypeEnum;
import com.nageoffer.onecoupon.merchant.admin.common.enums.CouponTaskStatusEnum;
import com.nageoffer.onecoupon.merchant.admin.dao.entity.CouponTaskDO;
import com.nageoffer.onecoupon.merchant.admin.dao.mapper.CouponTaskMapper;
import com.nageoffer.onecoupon.merchant.admin.dto.req.CouponTaskCreateReqDTO;
import com.nageoffer.onecoupon.merchant.admin.dto.resp.CouponTemplateQueryRespDTO;
import com.nageoffer.onecoupon.merchant.admin.service.CouponTaskService;
import com.nageoffer.onecoupon.merchant.admin.service.CouponTemplateService;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.Objects;

/**
 * 优惠券推送业务逻辑实现层
 * <p>
 * 作者：马丁
 * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
 * 开发时间：2024-07-12
 */
@Service
@RequiredArgsConstructor
public class CouponTaskServiceImpl extends ServiceImpl<CouponTaskMapper, CouponTaskDO> implements CouponTaskService {

    private final CouponTemplateService couponTemplateService;
    private final CouponTaskMapper couponTaskMapper;

    @Override
    public void createCouponTask(CouponTaskCreateReqDTO requestParam) {
        // 验证非空参数
        // 验证参数是否正确，比如文件地址是否为我们期望的格式等
        // 验证参数依赖关系，比如选择定时发送，发送时间是否不为空等
        CouponTemplateQueryRespDTO couponTemplate = couponTemplateService.findCouponTemplateById(requestParam.getCouponTemplateId());
        if (couponTemplate == null) {
            throw new ClientException("优惠券模板不存在，请检查提交信息是否正确");
        }
        // ......

        // 构建优惠券推送任务数据库持久层实体
        CouponTaskDO couponTaskDO = BeanUtil.copyProperties(requestParam, CouponTaskDO.class);
        couponTaskDO.setBatchId(IdUtil.getSnowflakeNextId());
        couponTaskDO.setOperatorId(Long.parseLong(UserContext.getUserId()));
        couponTaskDO.setShopNumber(UserContext.getShopNumber());
        couponTaskDO.setStatus(
                Objects.equals(requestParam.getSendType(), CouponTaskSendTypeEnum.IMMEDIATE.getType())
                        ? CouponTaskStatusEnum.IN_PROGRESS.getStatus()
                        : CouponTaskStatusEnum.PENDING.getStatus()
        );

        // 读取 Excel 文件
        ExcelReader reader = ExcelUtil.getReader(requestParam.getFileAddress());

        // 获取总行数（包括标题行）
        int rowCount = reader.getRowCount();
        couponTaskDO.setSendNum(rowCount);

        // 保存优惠券推送任务记录到数据库
        couponTaskMapper.insert(couponTaskDO);
    }
}
```

通过 API 管理工具开始发起调用，一些参数说明：

- fileAddress：写上面 Excel 文件的绝对路径即可。
- couponTemplateId：写个之前创建并且存在的优惠券模板 ID。

[![image-20240822205221532.png](https://i.postimg.cc/J7qQpSJr/image-20240822205221532.png)](https://postimg.cc/s1MSX6wq)
我们通过 JDK 自带的 visualvm 监控工具查看下内存变化，可以看到有个非常明显的内存上升。这里有点纳闷，为什么一个不到 30M 的 Excel 能引发这么大的内存占用。

[![image-20240822204908394.png](https://i.postimg.cc/zvJFJkmY/image-20240822204908394.png)](https://postimg.cc/G85yKGhX)
### 3. EasyExcel 获取 Excel 文件行数

创建 EasyExcel 读取监听类，代码很简单，只是用于类似于 i++ 的逻辑。

```java
package com.nageoffer.onecoupon.merchant.admin.service.handler.excel;

import com.alibaba.excel.context.AnalysisContext;
import com.alibaba.excel.event.AnalysisEventListener;
import lombok.Getter;

/**
 * Excel 行数统计监听器
 * <p>
 * 作者：马丁
 * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
 * 开发时间：2024-07-12
 */
public class RowCountListener extends AnalysisEventListener<Object> {

    @Getter
    private int rowCount = 0;

    @Override
    public void invoke(Object data, AnalysisContext context) {
        rowCount++;
    }

    @Override
    public void doAfterAllAnalysed(AnalysisContext context) {
        // No additional actions needed after all data is analyzed
    }
}
```

调整业务代码，切换 Hutool 的统计为 EasyExcel 行数统计。

调整业务代码，切换 Hutool 的统计为 EasyExcel 行数统计。

```java
package com.nageoffer.onecoupon.merchant.admin.service.impl;

import cn.hutool.core.bean.BeanUtil;
import cn.hutool.core.util.IdUtil;
import cn.hutool.poi.excel.ExcelReader;
import cn.hutool.poi.excel.ExcelUtil;
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import com.nageoffer.onecoupon.framework.exception.ClientException;
import com.nageoffer.onecoupon.merchant.admin.common.context.UserContext;
import com.nageoffer.onecoupon.merchant.admin.common.enums.CouponTaskSendTypeEnum;
import com.nageoffer.onecoupon.merchant.admin.common.enums.CouponTaskStatusEnum;
import com.nageoffer.onecoupon.merchant.admin.dao.entity.CouponTaskDO;
import com.nageoffer.onecoupon.merchant.admin.dao.mapper.CouponTaskMapper;
import com.nageoffer.onecoupon.merchant.admin.dto.req.CouponTaskCreateReqDTO;
import com.nageoffer.onecoupon.merchant.admin.dto.resp.CouponTemplateQueryRespDTO;
import com.nageoffer.onecoupon.merchant.admin.service.CouponTaskService;
import com.nageoffer.onecoupon.merchant.admin.service.CouponTemplateService;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.Objects;

/**
 * 优惠券推送业务逻辑实现层
 * <p>
 * 作者：马丁
 * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
 * 开发时间：2024-07-12
 */
@Service
@RequiredArgsConstructor
public class CouponTaskServiceImpl extends ServiceImpl<CouponTaskMapper, CouponTaskDO> implements CouponTaskService {

    private final CouponTemplateService couponTemplateService;
    private final CouponTaskMapper couponTaskMapper;

    @Override
    public void createCouponTask(CouponTaskCreateReqDTO requestParam) {
        // 验证非空参数
        // 验证参数是否正确，比如文件地址是否为我们期望的格式等
        // 验证参数依赖关系，比如选择定时发送，发送时间是否不为空等
        CouponTemplateQueryRespDTO couponTemplate = couponTemplateService.findCouponTemplateById(requestParam.getCouponTemplateId());
        if (couponTemplate == null) {
            throw new ClientException("优惠券模板不存在，请检查提交信息是否正确");
        }
        // ......

        // 构建优惠券推送任务数据库持久层实体
        CouponTaskDO couponTaskDO = BeanUtil.copyProperties(requestParam, CouponTaskDO.class);
        couponTaskDO.setBatchId(IdUtil.getSnowflakeNextId());
        couponTaskDO.setOperatorId(Long.parseLong(UserContext.getUserId()));
        couponTaskDO.setShopNumber(UserContext.getShopNumber());
        couponTaskDO.setStatus(
                Objects.equals(requestParam.getSendType(), CouponTaskSendTypeEnum.IMMEDIATE.getType())
                        ? CouponTaskStatusEnum.IN_PROGRESS.getStatus()
                        : CouponTaskStatusEnum.PENDING.getStatus()
        );

        // 通过 EasyExcel 监听器获取 Excel 中所有行数
        RowCountListener listener = new RowCountListener();
        EasyExcel.read(requestParam.getFileAddress(), listener).sheet().doRead();

        // 为什么需要统计行数？因为发送后需要比对所有优惠券是否都已发放到用户账号
        int totalRows = listener.getRowCount();
        couponTaskDO.setSendNum(totalRows);

        // 保存优惠券推送任务记录到数据库
        couponTaskMapper.insert(couponTaskDO);
    }
}
```

重启项目，再看看内存占用怎么样。

查看 visualvm 堆内存监控得知，虽然还是有内存上升，但是相对来说好很多了。Hutool 的内存占用在 3G 还要多点，EasyExcel 的内存在 250M 多点。

[![image-20240822210648374.png](https://i.postimg.cc/LXm1f6xz/image-20240822210648374.png)](https://postimg.cc/QBnCZsGt)
## 文末总结

在本章节中，我们探讨了使用 EasyExcel 处理大文件 Excel 的方法，特别是在开发批量优惠券分发任务时如何解决内存溢出的问题。传统的 Excel 解析工具（如 Hutool）在处理大规模数据时容易导致高内存消耗，甚至出现内存溢出问题。EasyExcel 通过流式处理数据，有效地降低了内存占用。

完结，撒花 🎉
