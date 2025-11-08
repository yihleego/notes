# 排序规则命名约定

MySQL 排序规则名称遵循以下约定：

- 排序规则名称以与其关联的字符集名称开头，通常后跟一个或多个表示其他排序规则特征的后缀。 例如，`utf8mb4_0900_ai_ci`和`latin1_swedish_ci`分别是`utf8mb4`和`latin1`字符集的排序规则。`binary`字符集具有单一排序规则，也称为`binary`，没有后缀。

- 特定于语言的排序规则包括区域设置代码或语言名称。例如，`utf8mb4_tr_0900_ai_ci`和`utf8mb4_hu_0900_ai_ci`分别使用土耳其语和匈牙利语的规则对`utf8mb4`字符集的字符进行排序。`utf8mb4_turkish_ci`和`utf8mb4_hungarian_ci`类似，但基于`Unicode`排序算法的较新版本。

- 排序规则后缀指示排序规则是区分大小写、区分重音、区分假名（或其某种组合）还是二进制。下表显示了用于指示这些特征的后缀。

| Suffix | Meaning            |
|:-------|:-------------------|
| _ai    | Accent-insensitive |
| _as    | Accent-sensitive   |
| _ci    | Case-insensitive   |
| _cs    | Case-sensitive     |
| _ks    | Kana-sensitive     |
| _bin   | Binary             |

```sql
create table test (
    id   int,
    name varchar(255)
) engine = innodb
  character set utf8mb4
  collate utf8mb4_0900_ai_cs;
```

✅ 推荐排序规则：utf8mb4_0900_ai_ci

在 MySQL 8.0 中，新引入了基于 Unicode 9.0 的排序规则系列，推荐使用 utf8mb4_0900_ai_ci：

- “0900” 表示 Unicode 版本 9.0；
- “ai” 表示 accent-insensitive（不区分音调）；
- “ci” 表示 case-insensitive（不区分大小写）。

此排序规则性能好、排序更符合现代语言规则，并支持多语言比较。

| 使用场景      | 推荐排序规则                                         | 说明                                        |
|-----------|------------------------------------------------|-------------------------------------------|
| 区分大小写     | `utf8mb4_0900_as_cs`                           | as = accent-sensitive，cs = case-sensitive |
| 对性能敏感、仅英文 | `utf8mb4_general_ci`                           | 老规则，速度略快但不完全符合 Unicode 排序标准               |
| 特定语言优化    | 例如 `utf8mb4_0900_ai_ci` / `utf8mb4_0900_as_cs` | 也可用 `utf8mb4_0900_ai_ci` 作通用方案            |

