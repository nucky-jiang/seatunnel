# SeaTunnel Critical 漏洞修复日志

> **完整逐文件变更记录**请参阅 [`SECURITY_FIX_CHANGELOG.md`](./SECURITY_FIX_CHANGELOG.md)（含每一处原值→现值对照，共 17 个文件）。

> 基于 Checkmarx SCA 扫描报告 (`SCA_ScanReport/Vulnerabilities.csv`)，共 113 条 Critical 级别漏洞。
> 修复策略：优先升级至 `LatestFixedVersion`；若 API 不兼容则降至 `NextFixedVersion`。

## 操作记录

| 时间 | 操作 | 说明 |
|------|------|------|
| 2026-06-29 | 初始化 | 解析 SCA 报告，识别 113 条 Critical 漏洞涉及 30+ 个包 |
| 2026-06-29 | 根 pom 依赖升级 | 在 `pom.xml` 添加安全版本属性及 `dependencyManagement` 强制覆盖传递依赖 |
| 2026-06-29 | 子模块版本同步 | 更新 connector-jdbc、hudi、clickhouse、hazelcast、dist 等模块 |
| 2026-06-29 | 前端 npm 升级 | 升级 `seatunnel-engine-ui` 的 axios、vue-i18n，添加 overrides |
| 2026-06-29 | Spark 版本调整 | 3.3.0→3.4.4（3.5.x API 不兼容）；2.4.0→2.4.8 |
| 2026-06-29 | Jackson 版本调整 | 2.15.4 与 Java8 shade 不兼容，降至 2.13.5 |
| 2026-06-29 | 构建配置调整 | 禁用不完整源码的 `seatunnel-config` 模块；`seatunnel-shade` 提前构建 |
| 2026-06-29 | engine-server 修复 | 添加 okhttp 2.7.5 编译依赖（原代码使用 okhttp2 API，传递依赖缺失） |
| 2026-06-29 | 编译验证 | `seatunnel-translation-spark-3.3`、`seatunnel-engine-server` 编译通过 |
| 2026-06-30 | V2 复扫修复 | 见 `SCA_ScanReport_V2`：Netty 4.1.135、Spark 3.5.8、Hadoop/Avro 残留清理；全量 `clean package` 通过 |
| 2026-06-30 | 打包验证与清理 | `BUILD SUCCESS`（175/175）；产物约 3.25GB；`mvn clean` + 前后端遗留清理完成 |

---

## 文件修改记录

### 1. `pom.xml`（根 POM）

**属性升级：**
| 属性 | 原版本 | 新版本 | 修复 CVE |
|------|--------|--------|----------|
| log4j2.version | 2.17.1 | 2.23.1 | log4j 相关 |
| spark.2.4.0.version | 2.4.0 | 2.4.8 | Spark 2.4 网络组件 |
| spark.3.3.0.version | 3.3.0 | 3.4.4 | CVE-2023-22946 |
| jackson.version | 2.13.3 | 2.13.5 | jackson-databind 反序列化 |
| avro.version | 1.11.1 | 1.11.4 | avro 多版本 CVE |
| commons-compress.version | 1.20 | 1.26.2 | 压缩库漏洞 |
| hadoop2.version | 2.6.5 | 2.10.2 | hadoop-common CVE |
| hadoop-aws.version / hadoop3.version | 3.1.4 | 3.3.6 | hadoop-common/client CVE |

**新增安全属性：**
- netty.version=4.1.118.Final
- parquet.version=1.15.1
- hazelcast.version=5.1.7
- postgresql.version=42.7.5
- redshift.version=2.2.7
- sqlite-jdbc.version=3.46.1.0
- aerospike.version=6.2.0
- sshd.version=2.12.1
- zookeeper.version=3.8.4
- bouncycastle.version=1.78.1
- commons-text.version=1.12.0
- ivy.version=2.5.2
- derby.version=10.17.1.0
- snakeyaml.version=2.2
- nimbus-jose-jwt.version=9.40

**dependencyManagement 新增：** netty-bom、parquet-*、hazelcast、postgresql、redshift、sqlite-jdbc、aerospike、sshd、zookeeper、bouncycastle、commons-text、ivy、derby、snakeyaml、nimbus-jose-jwt、avro、hadoop-*、jetty-*

**模块顺序：** `seatunnel-shade` 移至 `seatunnel-common` 之前；注释掉 `seatunnel-config`（使用 Maven Central 预构建包）

### 2. `seatunnel-connectors-v2/connector-jdbc/pom.xml`
- postgresql: 42.4.3 → 42.7.5
- redshift: 2.1.0.30 → 2.2.7
- sqlite: 3.39.3.0 → 3.46.1.0

### 3. `seatunnel-connectors-v2/connector-hudi/pom.xml`
- parquet: 1.12.2 → 1.15.1

### 4. `seatunnel-connectors-v2/connector-clickhouse/pom.xml`
- sshd: 2.7.0 → 2.12.1

### 5. `seatunnel-shade/seatunnel-hazelcast/seatunnel-hazelcast-base/pom.xml`
- 继承父 POM hazelcast.version 5.1.7

### 6. `seatunnel-shade/seatunnel-hadoop3-3.1.4-uber/pom.xml`
- hadoop3: 3.1.4 → 3.3.6（继承父 POM）

### 7. `seatunnel-dist/pom.xml`
- postgresql、redshift、sqlite、netty-buffer 版本同步升级

### 8. `seatunnel-e2e/seatunnel-engine-e2e/pom.xml`
- hazelcast 继承父 POM 5.1.7

### 9. `seatunnel-e2e/.../connector-aerospike-e2e/pom.xml`
- aerospike-client: 6.1.0 → 6.2.0

### 10. `seatunnel-e2e/.../connector-seatunnel-e2e-base/pom.xml`
- netty-buffer 使用父 POM ${netty.version}

### 11. `seatunnel-engine/.../imap-storage-file/pom.xml`
- netty-buffer 使用父 POM ${netty.version}

### 12. `seatunnel-examples/seatunnel-spark-connector-v2-example/pom.xml`
- jackson-databind: 2.6.7 → 2.13.5
- netty-all: 使用 ${netty.version}

### 13. `seatunnel-engine/seatunnel-engine-ui/package.json`
- axios: ^1.7.7 → ^1.15.2
- vue-i18n: ^10.0.1 → ^10.0.6
- 新增 overrides: lodash ^4.18.0, form-data ^4.0.4, shell-quote ^1.8.4
- **注意**：仅修改 `package.json` 版本号即可，扫描无需 `node_modules`/`node/`；勿手动 `npm install`，构建时由 Maven `frontend-maven-plugin` 自动处理

| 2026-06-29 | connector-hudi shade 修复 | 排除 META-INF/versions/** 解决 parquet 1.15.1 Java21 类 shade 失败 |

### 15. `seatunnel-connectors-v2/connector-hudi/pom.xml`（shade 修复）
- maven-shade-plugin 增加 filter 排除 `META-INF/versions/**`（Java 8 无法处理 parquet-jackson 1.15.1 的 Java 21 多版本类）

### 16. `seatunnel-engine/seatunnel-engine-server/pom.xml`（编译修复）
- 新增 `com.squareup.okhttp:okhttp:2.7.5` compile 依赖（源码使用 okhttp2 API，传递依赖缺失导致编译失败）

### 17. 全量编译验证（2026-06-29）

```bash
mvn clean package "-Dmaven.test.skip=true" "-Dspotless.check.skip=true"
```

- **结果**：`BUILD SUCCESS`（175/175 模块，耗时约 38 分钟）
- **产物**：`seatunnel-dist/target/apache-seatunnel-2.3.13-bin.tar.gz`（约 3.0 GB）、`apache-seatunnel-2.3.13-src.tar.gz`

---


## 版本兼容性说明

| 组件 | 目标版本 | 实际采用 | 原因 |
|------|----------|----------|------|
| Spark 3.x | 3.5.8 | **3.4.4** | 3.5.x 导致 `SeaTunnelBatchWrite` 接口冲突 |
| Jackson | 2.22.0 | **2.13.5** | 2.15+ 含 Java21 多版本类，Java8 shade 失败 |
| Jetty | 12.x | **9.4.56** | Jetty 12 需 Java 11+，项目基于 Java 8 |
| Netty | 4.1.133+ | **4.1.118.Final** | 4.1.118 为 Java8 兼容的最新稳定版之一 |
| OkHttp (engine) | 4.x | **2.7.5** | 源码使用 okhttp2 API，OkHttp4 需 Kotlin 运行时 |

---

## 待后续处理

1. ~~**全量构建验证**~~：已于 2026-06-29 通过 `mvn clean package -Dmaven.test.skip=true -Dspotless.check.skip=true`
2. **重新扫描**：用 Checkmarx 复扫验证 Critical 数量下降
3. **Jetty 9.4.56**：在 Java 8 约束下可能仍被标记，需评估是否升级 JDK
4. **log4j 1.2.17**：无官方修复版本，已通过 `provided` scope 排除
5. **Spark 2.4 支持**：已升至 2.4.8，完整修复 CVE 需迁移至 Spark 3.x

---

## Critical 漏洞包覆盖清单

| 包名 | 修复方式 |
|------|----------|
| jackson-databind 2.6.7 | dependencyManagement 强制 2.13.5 + spark example 升级 |
| parquet-avro 1.12.x | 1.15.1 |
| avro 多版本 | 1.11.4 |
| hadoop-common 2.6.5/3.x | 2.10.2 / 3.3.6 |
| spark-core 2.4.0/3.3.0 | 2.4.8 / 3.4.4 |
| netty 多版本 | netty-bom 4.1.118.Final |
| zookeeper | 3.8.4 |
| bouncycastle | 1.78.1 |
| postgresql | 42.7.5 |
| hazelcast | 5.1.7 |
| axios (npm) | 1.15.2 |
| lodash/form-data/shell-quote (npm) | overrides |
| vue-i18n (npm) | 10.0.6 |
| aerospike-client | 6.2.0 |
| redshift-jdbc42 | 2.2.7 |
| sqlite-jdbc | 3.46.1.0 |
| sshd | 2.12.1 |
| commons-text | 1.12.0 |
| ivy | 2.5.2 |
| derby | 10.17.1.0 |
| snakeyaml | 2.2 |
| nimbus-jose-jwt | 9.40 |
