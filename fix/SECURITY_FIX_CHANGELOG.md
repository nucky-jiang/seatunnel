# SeaTunnel Critical 漏洞修复 — 完整变更记录

> **基准源码**：Apache SeaTunnel **2.3.13**（与 [官方 tag 2.3.13](https://github.com/apache/seatunnel/tree/2.3.13) 对比）  
> **扫描依据**：`SCA_ScanReport/Vulnerabilities.csv`（初始 **113** 条 Critical）  
> **修复策略**：优先升级至 `LatestFixedVersion`；API / Java 8 / shade 不兼容时降至可编译的稳定版本  
> **文档日期**：2026-06-29  

---

## 一、修改文件总览（共 17 个文件）

| 序号 | 文件路径 | 变更类型 |
|------|----------|----------|
| 1 | `pom.xml` | 依赖版本、模块顺序、`dependencyManagement` |
| 2 | `seatunnel-connectors-v2/connector-jdbc/pom.xml` | JDBC 驱动版本 |
| 3 | `seatunnel-connectors-v2/connector-hudi/pom.xml` | Parquet 版本 + shade 配置 |
| 4 | `seatunnel-connectors-v2/connector-clickhouse/pom.xml` | SSHD 版本 |
| 5 | `seatunnel-shade/seatunnel-hazelcast/seatunnel-hazelcast-base/pom.xml` | Hazelcast 版本继承 |
| 6 | `seatunnel-shade/seatunnel-hadoop3-3.1.4-uber/pom.xml` | Hadoop3 版本继承 |
| 7 | `seatunnel-dist/pom.xml` | 打包用 JDBC / Netty 版本 |
| 8 | `seatunnel-e2e/seatunnel-engine-e2e/pom.xml` | Hazelcast 版本继承 |
| 9 | `seatunnel-e2e/seatunnel-connector-v2-e2e/connector-aerospike-e2e/pom.xml` | Aerospike 版本 |
| 10 | `seatunnel-e2e/seatunnel-engine-e2e/connector-seatunnel-e2e-base/pom.xml` | Netty 版本 |
| 11 | `seatunnel-engine/seatunnel-engine-storage/imap-storage-plugins/imap-storage-file/pom.xml` | Netty 版本 |
| 12 | `seatunnel-examples/seatunnel-spark-connector-v2-example/pom.xml` | Jackson / Netty 版本 |
| 13 | `seatunnel-engine/seatunnel-engine-ui/package.json` | 前端依赖与 overrides |
| 14 | `seatunnel-engine/seatunnel-engine-server/pom.xml` | 新增 OkHttp2 编译依赖 |
| 15 | `seatunnel-engine/seatunnel-engine-ui/pom.xml` | `mvn clean` 清理 `node/` |
| 16 | `.gitignore` | 忽略 `node_modules/` |
| 17 | `SECURITY_FIX_LOG.md` | 修复过程摘要日志（本详细文档之姊妹文件） |

**说明**：`JobEventHttpReportHandler.java` 曾短暂改为 OkHttp3，最终已**恢复为原始 OkHttp2 代码**，与官方源码一致，**不计入最终变更**。

---

## 二、逐文件变更明细

### 1. `pom.xml`（根 POM）

#### 1.1 `<properties>` — 已有属性修改

| 属性名 | 原值 | 现值 | 备注 |
|--------|------|------|------|
| `log4j2.version` | `2.17.1` | `2.23.1` | Log4j2 安全升级 |
| `log4j-core.version` | `2.17.1` | `2.23.1` | 与 log4j2 同步 |
| `spark.2.4.0.version` | `2.4.0` | `2.4.8` | Spark 2.4 安全补丁 |
| `spark.3.3.0.version` | `3.3.0` | `3.4.4` | 曾试 `3.5.4`，因 `SeaTunnelBatchWrite` API 冲突回退 |
| `jackson.version` | `2.13.3` | `2.13.5` | 曾试 `2.15.4`，因 Java 21 多版本类 shade 失败回退 |
| `commons-compress.version` | `1.20` | `1.26.2` | 压缩库 CVE |
| `avro.version` | `1.11.1` | `1.11.4` | Avro CVE |
| `hadoop2.version` | `2.6.5` | `2.10.2` | Hadoop 2.x CVE |
| `hadoop-aws.version` | `3.1.4` | `3.3.6` | Hadoop 3.x AWS 模块 |
| `okhttp.version` | `4.12.0` | `3.14.9` | **编译兼容**（OkHttp4 需 Kotlin；非 SCA 主目标，见 §五） |

#### 1.2 `<properties>` — 新增属性（原仓库不存在）

```xml
<!-- Security fix: Critical CVE dependency versions (Checkmarx SCA) -->
<netty.version>4.1.118.Final</netty.version>
<parquet.version>1.15.1</parquet.version>
<hazelcast.version>5.1.7</hazelcast.version>
<postgresql.version>42.7.5</postgresql.version>
<redshift.version>2.2.7</redshift.version>
<sqlite-jdbc.version>3.46.1.0</sqlite-jdbc.version>
<aerospike.version>6.2.0</aerospike.version>
<sshd.version>2.12.1</sshd.version>
<zookeeper.version>3.8.4</zookeeper.version>
<bouncycastle.version>1.78.1</bouncycastle.version>
<commons-text.version>1.12.0</commons-text.version>
<ivy.version>2.5.2</ivy.version>
<derby.version>10.17.1.0</derby.version>
<snakeyaml.version>2.2</snakeyaml.version>
<nimbus-jose-jwt.version>9.40</nimbus-jose-jwt.version>
```

另新增（配合 Hadoop shade 模块）：

| 属性名 | 原值 | 现值 |
|--------|------|------|
| `hadoop3.version` | *不存在* | `3.3.6` |

> **内部调整记录**（未保留在最终代码中）：`hazelcast.version` 曾设为 `5.3.8` 后改为 `5.1.7`；`aerospike.version` 曾设为 `7.2.0` 后改为 `6.2.0`。

#### 1.3 `<modules>` — 构建模块顺序

**原顺序（官方 2.3.13）：**

```
seatunnel-config          ← 启用
seatunnel-common
seatunnel-core
…（中间模块不变）…
seatunnel-e2e
seatunnel-shade           ← 在 e2e 之后
seatunnel-ci-tools
```

**现顺序：**

```
<!-- <module>seatunnel-config</module> -->   ← 已注释禁用
seatunnel-shade                           ← 提前至第二位
seatunnel-common
…（中间模块不变）…
seatunnel-e2e
seatunnel-ci-tools                        ← seatunnel-shade 不再在末尾
```

**禁用 `seatunnel-config` 原因**：本地 `seatunnel-config-shade` 源码不完整（缺少 `Token` 等类），按上游设计应使用 Maven Central 预构建的 `seatunnel-config-shade:2.3.13`。

**`seatunnel-shade` 提前原因**：`seatunnel-common` 等模块依赖 shade 产物，须先构建。

#### 1.4 `<dependencyManagement>` — 新增依赖强制版本

在 `hugegraph-client` 依赖之后、`</dependencies>` 之前，新增注释块：

`<!-- Security fix: force safe versions for transitive dependencies (Checkmarx Critical CVEs) -->`

| groupId | artifactId | 版本属性 | 版本值 |
|---------|------------|----------|--------|
| `io.netty` | `netty-bom` | `${netty.version}` | `4.1.118.Final`（`type=pom`, `scope=import`） |
| `org.apache.parquet` | `parquet-avro` | `${parquet.version}` | `1.15.1` |
| `org.apache.parquet` | `parquet-hadoop` | `${parquet.version}` | `1.15.1` |
| `org.apache.parquet` | `parquet-hadoop-bundle` | `${parquet.version}` | `1.15.1` |
| `org.apache.parquet` | `parquet-common` | `${parquet.version}` | `1.15.1` |
| `org.apache.parquet` | `parquet-column` | `${parquet.version}` | `1.15.1` |
| `org.apache.parquet` | `parquet-encoding` | `${parquet.version}` | `1.15.1` |
| `com.hazelcast` | `hazelcast` | `${hazelcast.version}` | `5.1.7` |
| `org.postgresql` | `postgresql` | `${postgresql.version}` | `42.7.5` |
| `com.amazon.redshift` | `redshift-jdbc42` | `${redshift.version}` | `2.2.7` |
| `org.xerial` | `sqlite-jdbc` | `${sqlite-jdbc.version}` | `3.46.1.0` |
| `com.aerospike` | `aerospike-client` | `${aerospike.version}` | `6.2.0` |
| `org.apache.sshd` | `sshd-core` | `${sshd.version}` | `2.12.1` |
| `org.apache.sshd` | `sshd-common` | `${sshd.version}` | `2.12.1` |
| `org.apache.sshd` | `sshd-scp` | `${sshd.version}` | `2.12.1` |
| `org.apache.zookeeper` | `zookeeper` | `${zookeeper.version}` | `3.8.4` |
| `org.bouncycastle` | `bcprov-jdk18on` | `${bouncycastle.version}` | `1.78.1` |
| `org.bouncycastle` | `bcprov-jdk15on` | 固定 | `1.70` |
| `org.apache.commons` | `commons-text` | `${commons-text.version}` | `1.12.0` |
| `org.apache.ivy` | `ivy` | `${ivy.version}` | `2.5.2` |
| `org.apache.derby` | `derby` | `${derby.version}` | `10.17.1.0` |
| `org.yaml` | `snakeyaml` | `${snakeyaml.version}` | `2.2` |
| `com.nimbusds` | `nimbus-jose-jwt` | `${nimbus-jose-jwt.version}` | `9.40` |
| `org.apache.avro` | `avro` | `${avro.version}` | `1.11.4` |
| `org.apache.hadoop` | `hadoop-common` | `${hadoop3.version}` | `3.3.6` |
| `org.apache.hadoop` | `hadoop-client-api` | `${hadoop3.version}` | `3.3.6` |
| `org.apache.hadoop` | `hadoop-client` | `${hadoop3.version}` | `3.3.6` |
| `org.eclipse.jetty` | `jetty-http` | `${jetty.version}` | `9.4.56.v20240826` |
| `org.eclipse.jetty` | `jetty-server` | `${jetty.version}` | `9.4.56.v20240826` |
| `org.eclipse.jetty` | `jetty-servlet` | `${jetty.version}` | `9.4.56.v20240826` |
| `org.eclipse.jetty` | `jetty-util` | `${jetty.version}` | `9.4.56.v20240826` |
| `org.eclipse.jetty` | `jetty-io` | `${jetty.version}` | `9.4.56.v20240826` |
| `org.eclipse.jetty` | `jetty-security` | `${jetty.version}` | `9.4.56.v20240826` |
| `org.eclipse.jetty` | `jetty-webapp` | `${jetty.version}` | `9.4.56.v20240826` |

> `jetty.version` 属性在官方源码中已为 `9.4.56.v20240826`，**未改属性值**；新增的是 `dependencyManagement` 中对 Jetty 各组件的**显式锁定**。

---

### 2. `seatunnel-connectors-v2/connector-jdbc/pom.xml`

`<properties>` 内驱动版本：

| 属性名 | 原值 | 现值 |
|--------|------|------|
| `postgresql.version` | `42.4.3` | `42.7.5` |
| `sqlite.version`（两处重复定义均已改） | `3.39.3.0` | `3.46.1.0` |
| `redshift.version` | `2.1.0.30` | `2.2.7` |

---

### 3. `seatunnel-connectors-v2/connector-hudi/pom.xml`

#### 3.1 版本

| 属性名 | 原值 | 现值 |
|--------|------|------|
| `parquet.version` | `1.12.2` | `1.15.1` |

#### 3.2 `maven-shade-plugin` 配置（新增）

在 `<configuration>` 内、`<relocations>` **之前**新增：

```xml
<filters>
    <filter>
        <artifact>*:*</artifact>
        <excludes>
            <exclude>META-INF/versions/**</exclude>
        </excludes>
    </filter>
</filters>
```

**原因**：`parquet-jackson 1.15.1` 含 Java 21 多版本类（class file major version 65），Java 8 + shade 插件无法处理。

---

### 4. `seatunnel-connectors-v2/connector-clickhouse/pom.xml`

| 属性名 | 原值 | 现值 |
|--------|------|------|
| `sshd.scp.version` | `2.7.0` | `${sshd.version}`（解析为 `2.12.1`） |

---

### 5. `seatunnel-shade/seatunnel-hazelcast/seatunnel-hazelcast-base/pom.xml`

**原：**

```xml
<properties>
    <!--  SeaTunnel Engine use     -->
    <hazelcast.version>5.1</hazelcast.version>
</properties>
```

**现：**

```xml
<properties>
    <!--  SeaTunnel Engine use     -->
    <!-- inherits hazelcast.version from parent -->
</properties>
```

依赖仍使用 `${hazelcast.version}`，实际版本由根 POM 解析为 **5.1.7**（原 `5.1`）。

---

### 6. `seatunnel-shade/seatunnel-hadoop3-3.1.4-uber/pom.xml`

**原：**

```xml
<properties>
    <hadoop3.version>3.1.4</hadoop3.version>
    <guava.version>27.0-jre</guava.version>
</properties>
```

**现：**

```xml
<properties>
    <!-- inherits hadoop3.version from parent -->
    <guava.version>27.0-jre</guava.version>
</properties>
```

`hadoop-client` 等依赖的 `${hadoop3.version}` 由根 POM 解析为 **3.3.6**（原 `3.1.4`）。

---

### 7. `seatunnel-dist/pom.xml`

在 `seatunnel` profile 的 `<properties>` 中：

| 属性名 | 原值 | 现值 |
|--------|------|------|
| `postgresql.version` | `42.4.3` | `42.7.5` |
| `sqlite.version`（两处） | `3.39.3.0` | `3.46.1.0` |
| `redshift.version` | `2.1.0.30` | `2.2.7` |
| `netty-buffer.version` | `4.1.89.Final` | `4.1.118.Final` |

---

### 8. `seatunnel-e2e/seatunnel-engine-e2e/pom.xml`

**原：**

```xml
<properties>
    <!--  SeaTunnel Engine use     -->
    <hazelcast.version>5.1</hazelcast.version>
</properties>
```

**现：**

```xml
<properties>
    <!--  SeaTunnel Engine use     -->
    <!-- inherits hazelcast.version from parent -->
</properties>
```

---

### 9. `seatunnel-e2e/seatunnel-connector-v2-e2e/connector-aerospike-e2e/pom.xml`

`aerospike-client` 依赖：

| 字段 | 原值 | 现值 |
|------|------|------|
| `<version>` | `6.1.0`（硬编码） | `${aerospike.version}`（解析为 `6.2.0`） |

---

### 10. `seatunnel-e2e/seatunnel-engine-e2e/connector-seatunnel-e2e-base/pom.xml`

| 属性名 | 原值 | 现值 |
|--------|------|------|
| `netty-buffer.version` | `4.1.89.Final` | `${netty.version}`（解析为 `4.1.118.Final`） |

---

### 11. `seatunnel-engine/seatunnel-engine-storage/imap-storage-plugins/imap-storage-file/pom.xml`

| 属性名 | 原值 | 现值 |
|--------|------|------|
| `netty-buffer.version` | `4.1.60.Final` | `${netty.version}`（解析为 `4.1.118.Final`） |

---

### 12. `seatunnel-examples/seatunnel-spark-connector-v2-example/pom.xml`

| 位置 | 原值 | 现值 |
|------|------|------|
| `spark.2.4.0.jackson.version` | `2.6.7` | `2.13.5` |
| `jackson-databind` 依赖版本 | `${spark.2.4.0.jackson.version}` → `2.6.7` | → `2.13.5` |
| `netty-all` 依赖 `<version>` | `4.1.104.Final`（硬编码） | `${netty.version}` → `4.1.118.Final` |

---

### 13. `seatunnel-engine/seatunnel-engine-ui/package.json`

#### 13.1 `dependencies` 版本变更

| 包名 | 原值 | 现值 |
|------|------|------|
| `axios` | `^1.7.7` | `^1.15.2` |
| `vue-i18n` | `^10.0.1` | `^10.0.6` |

其余 `dependencies` / `devDependencies` **未改**。

#### 13.2 新增 `overrides` 段（原文件不存在）

```json
"overrides": {
  "lodash": "^4.18.0",
  "form-data": "^4.0.4",
  "shell-quote": "^1.8.4"
}
```

> **扫描说明**：仅需保留 `package.json`（及可选 `package-lock.json`），**无需**提交 `node_modules/` 或 Maven 下载的 `node/` 目录。

---

### 14. `seatunnel-engine/seatunnel-engine-server/pom.xml`

**新增**编译依赖（原文件不存在该依赖）：

```xml
<dependency>
    <groupId>com.squareup.okhttp</groupId>
    <artifactId>okhttp</artifactId>
    <version>2.7.5</version>
</dependency>
```

**原因**：`JobEventHttpReportHandler.java` 使用 OkHttp **2.x** API（`com.squareup.okhttp.*`），升级传递依赖后该包不再自动出现在编译 classpath。

---

### 15. `seatunnel-engine/seatunnel-engine-ui/pom.xml`

#### 15.1 新增属性

```xml
<node.dir>node</node.dir>
```

#### 15.2 `exec-maven-plugin` clean 阶段

| 配置项 | 原值 | 现值 |
|--------|------|------|
| `commandlineArgs` | `${args.rm.clean} ${dist.dir} ${nodemodules.dir} ${deployed.dir}` | 增加 `${node.dir}` |
| Windows `args.rm.clean` | 仅删 `dist` / `node_modules` / `.deployed` | 增加 `if exist "${node.dir}" rmdir /S /Q "${node.dir}"` |
| Linux `args.rm.clean` | `-rf ${dist.dir} ${nodemodules.dir} ${deployed.dir}` | 增加 `${node.dir}` |

**原因**：`frontend-maven-plugin` 下载的 Node 运行时目录（约 70MB）原先不会被 `mvn clean` 清理，影响扫描包体积。

---

### 16. `.gitignore`

**原：**

```
node/

dist/
```

**现：**

```
node/
node_modules/

dist/
```

---

### 17. `SECURITY_FIX_LOG.md`

新建并持续更新的修复摘要日志（非业务代码，记录修复过程与验证结果）。

---

## 三、版本调整过程（中间态，未保留在最终代码）

| 组件 | 尝试版本 | 最终版本 | 回退原因 |
|------|----------|----------|----------|
| Spark 3.x | `3.5.4` | `3.4.4` | `SeaTunnelBatchWrite` 与 Spark 3.5 DataSource V2 API 冲突 |
| Jackson | `2.15.4` | `2.13.5` | `META-INF/versions/21/` 多版本类导致 shade 失败 |
| Hazelcast | `5.3.8` | `5.1.7` | 兼容性收敛 |
| Aerospike | `7.2.0` | `6.2.0` | 兼容性收敛 |
| OkHttp (根 POM) | `4.12.0` | `3.14.9` | OkHttp4 基于 Kotlin，与 Java 8 编译环境冲突 |

---

## 四、编译验证

```bash
mvn clean package "-Dmaven.test.skip=true" "-Dspotless.check.skip=true"
```

| 项目 | 结果 |
|------|------|
| 状态 | **BUILD SUCCESS** |
| 模块 | 175 / 175 |
| 耗时 | 约 38 分钟 |
| 产物 | `seatunnel-dist/target/apache-seatunnel-2.3.13-bin.tar.gz` |

---

## 五、已知限制与未修复项

| 项目 | 说明 |
|------|------|
| Jetty 12.x | 需 Java 11+，项目基于 Java 8，保持 `9.4.56` |
| Netty `4.1.133+` | 扫描建议更高版本；当前 `4.1.118.Final` 为 Java 8 兼容选择 |
| log4j `1.2.17` | 无官方修复版，根 POM 中仍为 `provided` scope 排除 |
| Spark 2.4.8 | 已升至 2.4 线最新补丁，彻底修复需迁移 Spark 3.x |
| `okhttp.version` `3.14.9` | 相对官方 `4.12.0` 降级，仅为保证 Java 8 下全量编译；engine 使用独立 OkHttp2 `2.7.5` |

---

## 六、Critical 漏洞包与修复映射

| 漏洞相关包 | 修复方式 | 生效版本 |
|------------|----------|----------|
| jackson-databind | 根 POM + spark example | `2.13.5` |
| parquet-* | 根 POM + connector-hudi | `1.15.1` |
| avro | 根 POM `avro.version` | `1.11.4` |
| hadoop-common / client | 根 POM `hadoop2` / `hadoop3` | `2.10.2` / `3.3.6` |
| spark-core | 根 POM spark 属性 | `2.4.8` / `3.4.4` |
| netty-* | 根 POM `netty-bom` + 子模块 | `4.1.118.Final` |
| zookeeper | 根 POM dependencyManagement | `3.8.4` |
| bouncycastle | 根 POM dependencyManagement | `1.78.1` |
| postgresql | jdbc + dist | `42.7.5` |
| hazelcast | shade-base + 根 POM | `5.1.7` |
| aerospike-client | e2e + 根 POM | `6.2.0` |
| redshift-jdbc42 | jdbc + dist | `2.2.7` |
| sqlite-jdbc | jdbc + dist | `3.46.1.0` |
| sshd | clickhouse + 根 POM | `2.12.1` |
| commons-text | 根 POM | `1.12.0` |
| ivy | 根 POM | `2.5.2` |
| derby | 根 POM | `10.17.1.0` |
| snakeyaml | 根 POM | `2.2` |
| nimbus-jose-jwt | 根 POM | `9.40` |
| log4j-core / log4j-api | 根 POM `log4j2.version` | `2.23.1` |
| commons-compress | 根 POM | `1.26.2` |
| axios (npm) | package.json | `^1.15.2` |
| vue-i18n (npm) | package.json | `^10.0.6` |
| lodash / form-data / shell-quote (npm) | package.json overrides | 见 §二.13.2 |

---

## 八、第二轮扫描修复（`SCA_ScanReport_V2`，2026-06-30）

> 第一轮修复后复扫剩余 **25** 条 Critical（去重后约 **9** 类组件）。本轮在保持 Java 8 与全量编译前提下继续收敛。

### 8.1 根 `pom.xml` 追加/调整

| 属性/依赖 | 原值 | 现值 | 针对 CVE |
|-----------|------|------|----------|
| `netty.version` | `4.1.118.Final` | **`4.1.135.Final`** | CVE-2026-42581 / CVE-2026-44249 / CVE-2026-42584 等 |
| `spark.3.3.0.version` | `3.4.4` | **`3.5.8`** | CVE-2018-17190（Spark 3.x） |
| `aws-java-sdk.version` | *不存在* | **`1.12.779`** | 消除 `aws-java-sdk-bundle` 内嵌旧 Netty |
| `dependencyManagement` | — | 新增 `netty-all`、`hadoop-aws`、`hadoop-aliyun`、`aws-java-sdk-bundle` 版本锁定 | 传递依赖收敛 |

### 8.2 子模块 POM 修复（消除残留的 3.1.4 / 1.11.1）

| 文件 | 修改 |
|------|------|
| `seatunnel-formats/seatunnel-format-avro/pom.xml` | 删除本地 `avro.version=1.11.1`，继承父 POM `1.11.4` |
| `seatunnel-core/seatunnel-starter/pom.xml` | `hadoop3.version` 由 `3.1.4` 改为继承父 POM `3.3.6` |
| `seatunnel-dist/pom.xml` | `hadoop-aliyun` `3.1.4`→`${hadoop3.version}`；`netty-buffer`→`${netty.version}` |
| `checkpoint-storage-hdfs/pom.xml` | `hadoop-aws` `3.1.4`→`${hadoop-aws.version}`；`aws-java-sdk-bundle`→`${aws-java-sdk.version}` |
| `connector-iceberg-s3-e2e/pom.xml` | 删除本地 `hadoop3.version=3.1.4` |
| `connector-jdbc/pom.xml` | `hive-jdbc` 排除 `jackson-mapper-asl`、`jackson-core-asl`、`netty-all`、`log4j:log4j` |

### 8.3 源码适配 Spark 3.5.8

| 文件 | 修改 |
|------|------|
| `seatunnel-translation-spark-3.3/.../SeaTunnelBatchWrite.java` | 新增 `useCommitCoordinator()` 显式实现，解决 Spark 3.5 `BatchWrite`/`StreamingWrite` 默认方法冲突 |
| `seatunnel-spark-starter-common/.../TransformExecuteProcessor.java` | `RowEncoder.apply(schema)` 改为 `Encoders.row(schema)`（Spark 3.5 公开 API） |

### 8.4 编译验证

```bash
mvn clean package "-Dmaven.test.skip=true" "-Dspotless.check.skip=true"
```

- **结果**：`BUILD SUCCESS`（175/175，2026-06-30，约 41 分钟）

### 8.5 仍可能出现在扫描中的项（已知限制）

| 组件 | 原因 |
|------|------|
| `spark-core_2.12` **2.4.8** | Spark 2.4 线无 CVE-2018-17190 / CVE-2023-22946 官方修复，需迁移 Spark 3.x 或弃用 Spark2 模块 |
| `log4j:log4j` **1.2.17** | EOL，无修复版；已通过 `provided`/排除降低运行时暴露，扫描仍可能标记传递引用 |
| `jetty-http` **9.4.56** | CVE-2026-2332 需 **9.4.60+**；当前 Maven 镜像尚未提供 9.4.60 构件 |
| `aws-java-sdk-bundle` 内嵌 Netty | 超 fat JAR 内嵌依赖，SCA 可能仍解析出旧版本类路径 |

---

*本文档与 `SECURITY_FIX_LOG.md` 配套：本文档为「逐文件、逐字段」完整记录；摘要日志记录时间线与验证结果。*
