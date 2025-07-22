# 帝可得后台管理系统技术文档

## 1. 项目概述

### 1.1 系统角色与功能

- **管理员**：管理基础数据（区域、点位、设备、货道、商品等），创建工单，查看订单和统计报表
- **运维人员**：处理设备投放、撤除和维修工单
- **运营人员**：处理商品补货工单
- **合作商**：提供点位资源
- **消费者**：在小程序或屏幕端购买商品

### 1.2 业务流程

下载

管理员登录创建区域创建合作商创建点位创建设备类型创建设备创建货道创建商品创建工单运维/运营处理工单

## 2. 项目搭建

### 2.1 后端项目搭建

1. **克隆仓库**：

   ```bash
   git clone https://gitee.com/ys-gitee/dkd-parent.git
   ```

2. **Maven构建**：等待依赖下载

3. **数据库配置**：

   - 创建数据库：`create schema dkd;`
   - 导入SQL脚本
   - 修改`application-druid.yml`中的数据库连接信息

4. **Redis配置**：

   - 修改`redis.windows.conf`设置密码
   - 启动Redis：`redis-server.exe redis.windows.conf`
   - 修改`application.yml`中的Redis配置

5. **启动项目**：运行`com.ruoyi.DkdApplication`

### 2.2 前端项目搭建

1. **克隆仓库**：

   ```bash
   git clone https://gitee.com/ys-gitee/dkd-vue.git
   ```

2. **安装依赖**：

   ```bash
   npm install
   ```

3. **启动项目**：

   ```bash
   npm run dev
   ```

4. **访问地址**：`http://localhost:80` (默认账号：admin/admin123)

## 3. 功能模块实现

### 3.1 点位管理

#### 3.1.1 数据库设计

```sql
-- 区域表
CREATE TABLE `tb_region` (
  `id` INT AUTO_INCREMENT PRIMARY KEY COMMENT '主键id',
  `region_name` VARCHAR(255) NOT NULL COMMENT '区域名称',
  ...
);

-- 合作商表
CREATE TABLE `tb_partner` (
  `id` INT AUTO_INCREMENT PRIMARY KEY COMMENT '主键id',
  `partner_name` VARCHAR(255) NOT NULL COMMENT '合作商名称',
  `profit_ratio` INT COMMENT '分成比例',
  ...
);

-- 点位表
CREATE TABLE `tb_node` (
  `id` INT AUTO_INCREMENT PRIMARY KEY COMMENT '主键id',
  `node_name` VARCHAR(255) NOT NULL COMMENT '点位名称',
  `business_type` INT COMMENT '商圈类型',
  `region_id` INT COMMENT '区域ID',
  `partner_id` INT COMMENT '合作商ID',
  ...
  FOREIGN KEY (`region_id`) REFERENCES `tb_region`(`id`),
  FOREIGN KEY (`partner_id`) REFERENCES `tb_partner`(`id`)
);
```

#### 3.1.2 关键实现

- **区域点位数统计**：

  ```sql
  SELECT r.*, COUNT(n.id) AS node_count 
  FROM tb_region r 
  LEFT JOIN tb_node n ON r.id = n.region_id 
  GROUP BY r.id;
  ```

- **合作商点位数统计**：

  ```sql
  SELECT p.*, COUNT(n.id) AS node_count 
  FROM tb_partner p 
  LEFT JOIN tb_node n ON p.id = n.partner_id 
  GROUP BY p.id;
  ```

### 3.2 人员管理

#### 3.2.1 数据库设计

```sql
CREATE TABLE `tb_emp` (
  `id` BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '主键',
  `user_name` VARCHAR(64) NOT NULL COMMENT '员工名称',
  `region_id` BIGINT COMMENT '区域id',
  `region_name` VARCHAR(64) COMMENT '区域名称',
  `role_id` BIGINT COMMENT '角色id',
  `role_name` VARCHAR(64) COMMENT '角色名称',
  `role_code` VARCHAR(64) COMMENT '角色编码',
  ...
);
```

#### 3.2.2 关键功能

- **区域名称同步更新**：

  ```java
  @Transactional
  public int updateRegion(Region region) {
      int result = regionMapper.updateRegion(region);
      empMapper.updateByRegionId(region.getRegionName(), region.getId());
      return result;
  }
  ```

### 3.3 设备管理

#### 3.3.1 数据库设计

```sql
-- 设备类型表
CREATE TABLE `tb_vm_type` (
  `id` INT AUTO_INCREMENT PRIMARY KEY COMMENT '主键',
  `name` VARCHAR(255) COMMENT '型号名称',
  `vm_row` INT COMMENT '货道行',
  `vm_col` INT COMMENT '货道列',
  `channel_max_capacity` INT COMMENT '货道最大容量',
  ...
);

-- 设备表
CREATE TABLE `tb_vending_machine` (
  `id` BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '主键',
  `inner_code` VARCHAR(15) COMMENT '设备编号',
  `node_id` INT NOT NULL COMMENT '点位Id',
  `vm_type_id` INT NOT NULL COMMENT '设备型号',
  `vm_status` INT DEFAULT 0 NOT NULL COMMENT '设备状态',
  ...
);
```

#### 3.3.2 关键实现

- **创建设备同时生成货道**：

  ```java
  public int insertVendingMachine(VendingMachine vendingMachine) {
      // 创建设备
      int result = vendingMachineMapper.insertVendingMachine(vendingMachine);
      
      // 生成货道
      List<Channel> channelList = new ArrayList<>();
      for (int i = 1; i <= vmType.getVmRow(); i++) {
          for (int j = 1; j <= vmType.getVmCol(); j++) {
              Channel channel = new Channel();
              channel.setChannelCode(i + "-" + j);
              channelList.add(channel);
          }
      }
      channelService.batchInsertChannel(channelList);
      return result;
  }
  ```

### 3.4 商品管理

#### 3.4.1 数据库设计

```java
-- 商品类型表
CREATE TABLE `tb_sku_class` (
  `class_id` BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '类型主键',
  `class_name` VARCHAR(64) COMMENT '类型名称',
  ...
);

-- 商品表
CREATE TABLE `tb_sku` (
  `sku_id` BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '主键',
  `sku_name` VARCHAR(255) COMMENT '商品名称',
  `price` BIGINT COMMENT '商品价格(分)',
  `class_id` BIGINT COMMENT '商品类型Id',
  ...
);
```

#### 3.4.2 关键功能

- **商品批量导入**：

  ```java
  @PostMapping("/import")
  public AjaxResult excelImport(MultipartFile file) throws Exception {
      List<Sku> skuList = util.importEasyExcel(file.getInputStream());
      return toAjax(skuService.insertSkus(skuList));
  }
  ```

### 3.5 工单管理

#### 3.5.1 数据库设计

```sql
CREATE TABLE `tb_task` (
  `task_id` BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '工单id',
  `task_code` VARCHAR(255) COMMENT '工单编号',
  `task_status` INT COMMENT '工单状态',
  `product_type_id` INT COMMENT '工单类型id',
  ...
);
```

#### 3.5.2 工单状态

1. 待处理
2. 进行中
3. 已取消
4. 已完成

## 4. 关键技术集成

### 4.1 EasyExcel集成

#### 4.1.1 使用步骤

1. **添加依赖**：

   ```java
   <dependency>
       <groupId>com.alibaba</groupId>
       <artifactId>easyexcel</artifactId>
   </dependency>
   ```

2. **实体类注解**：

   ```java
   @ExcelProperty("商品名称")
   private String skuName;
   ```

3. **导入实现**：

   ```java
   public List<Sku> importEasyExcel(InputStream is) {
       return EasyExcel.read(is).head(clazz).sheet().doReadSync();
   }
   ```

## 5. 系统部署

### 5.1 服务器要求

- JDK 1.8+
- MySQL 5.7+
- Redis 5.0+
- Node.js 14+

### 5.2 部署步骤

1. **后端部署**：

   运行即可

2. **前端部署**：

   ```bash
   npm run build
   # 将dist目录部署到Nginx
   ```

