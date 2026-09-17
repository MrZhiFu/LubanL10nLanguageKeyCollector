# LubanL10nLanguageKeyCollector

## 功能说明

本工具用于将Luban配置表中的多语言Key自动收集并导出为一个C#静态类，方便在代码中直接引用多语言Key，避免硬编码字符串，提高代码可维护性。

支持通过多语言表的 `is_code`（bool）列进行**选择性导出**：仅 `is_code = true` 的 Key 会导出到静态类，`false` 的 Key 只保留在配置数据中（供策划表引用）。所有本地化表必须声明 `is_code` 列，缺失时生成会直接报错。

<img width="500" height="420" alt="QQ_1775133213097" src="https://github.com/user-attachments/assets/6ebf9b08-9169-4f73-bf96-0b6e188fc952" />

## 使用方法

### 1. 安装模板文件

将本项目下的文件复制到Luban源码对应目录下：

#### **注意：** 代码模板文件 `l10n-keys.sbn` 的属性请在IDE中设置为 **"始终复制"**。参考Luban文档：[https://www.datable.cn/docs/manual/template](https://www.datable.cn/docs/manual/template)

```
1.1 Luban.CSharp\Templates\cs-l10n-key 文件夹复制到 => Luban/src/Luban.CSharp/Templates
1.2 Luban.CSharp\CodeTarget\CsharpL10NKeyCodeTarget.cs 文件复制到 => Luban/src/Luban.CSharp/CodeTarget
1.3 Luban.CSharp\TemplateExtensions\CsharpL10NKeyTemplateExtension.cs 文件复制到 => Luban/src/Luban.CSharp/TemplateExtensions
```

### 2. 编译Luban

使用Luban编译工具build-luban.bat进行编译，生成最新的Luban.dll。

### 3. 配置导出命令

在导出配置的命令行中加入以下参数(示例是Win平台，XOS和Linux平台更改换行符" ^ " 为 " \ " 即可)：

```batch
-c cs-l10n-key ^
-x cs-l10n-key.outputCodeDir=自己工程配置表导出目录的上一级目录/L10nKey ^
```

完整示例：

```batch
dotnet ./Tools/Luban/Luban.dll ^
    -t client ^
    -d bin ^
    -c cs-bin ^
    -c cs-l10n-key ^
    -x outputDataDir=../Unity/Assets/Bundles/Config ^
    -x cs-bin.outputCodeDir=../Unity/Assets/Scripts/Hotfix/Config/Generate ^
    -x cs-l10n-key.outputCodeDir=../Unity/Assets/Scripts/Hotfix/Config/L10nKey ^
    --conf ./Luban.conf
```

#### **注意：**

##### 1.多语言Key静态类输出目录 `cs-l10n-key.outputCodeDir` **不可**指定在其他数据配置表导出的目录之下，否则会覆盖导出的配置表代码，建议和配置表导出目录同级即可。

##### 2.多语言配置表名称固定为 Localization。(如果需要自定，更改源码CsharpL10NKeyCodeTarget的CollectKeys方法，如下图：)

```csharp
    /// <summary>
    /// 从多语言表中收集Key信息
    /// </summary>
    /// <param name="tables">所有被导出的表</param>
    /// <returns>多语言表key结果列表</returns>
    private List<L10NKeyInfo> CollectKeys(List<DefTable> tables)
    {
         var keys   = new List<L10NKeyInfo>();
        var keySet = new HashSet<string>();

        // 筛选出多语言表
        var l10NTables = tables.Where(t => t.Name == "TbLocalization").ToList();

        foreach (var table in l10NTables)
        {
            CollectKeysFromTable(table, keys, keySet);
        }

        s_logger.Info("收集导出到代码中的多语言key: 总共收集到 {Count} 个多语言Key", keys.Count);

        // 按 Key 按照字母进行 排序
        return keys.OrderBy(k => k.Key, StringComparer.Ordinal).ToList();
    }
```

##### 3.生成的目标多语言静态类的字段注释固定取"简体中文"字段下的值："ChineseSimplified" or "chineseSimplified" or "chinese_simplified"。(如果需要自定，更改源码CsharpL10NKeyCodeTarget的CollectKeysFromTable方法，如下图：)

```csharp
    /// <summary>
    /// 从单个表中收集Key
    /// </summary>
    /// <param name="table">多语言表</param>
    /// <param name="resultKeys">多语言表key结果列表</param>
    /// <param name="keySet">用于去重的集合</param>
    private void CollectKeysFromTable(DefTable table, List<L10NKeyInfo> resultKeys, HashSet<string> keySet)
    {
       // 获取 key 字段（通常是第一个字段或名为 key 的字段）
        var keyField = table.ValueTType.DefBean.ExportFields.FirstOrDefault(f => f.Name == "key") ?? table.ValueTType.DefBean.ExportFields.FirstOrDefault();
        if (keyField == null) return;

        // 获取 ChineseSimplified(简体中文) 字段作为注释
        var commentField = table.ValueTType.DefBean.ExportFields.FirstOrDefault(f => f.Name is "ChineseSimplified" or "chineseSimplified" or "chinese_simplified");

        // 获取 is_code 字段（标记该 key 是否导出到代码），所有本地化表必须声明该列
        var isCodeField = table.ValueTType.DefBean.ExportFields.FirstOrDefault(f => f.Name == "is_code");
        if (isCodeField == null)
        {
            throw new Exception($"本地化表:'{table.Name}' 缺少 'is_code' 列！所有本地化表必须声明 is_code(bool) 列，用于标记该 key 是否导出到 L10nKey 类");
        }

        // 获取表数据
        var tableDataInfo = GenerationContext.Current.GetTableDataInfo(table);
        if (tableDataInfo == null) return;

        // 遍历表记录，收集 Key 信息
        foreach (var record in tableDataInfo.FinalRecords)
        {
            ProcessRecord(record, keyField, commentField, isCodeField, resultKeys, keySet);
        }
    }
```

## 生成的代码示例

工具会生成类似以下的C#静态类：

```csharp
namespace YourNamespace
{
    /// <summary>
    /// 本地化多语言Key列表
    /// </summary>
    public static class L10nKey
    {
        /// <summary>
        /// 欢迎文本
        /// </summary>
        public const string WELCOME_TEXT = "welcome_text";

        /// <summary>
        /// 提交
        /// </summary>
        public const string BTN_CONFIRM = "btn_confirm";

        // ... 更多Key
    }
}
```

## 依赖要求

- Luban版本：支持 `cs-l10n-key` 代码生成目标
- .NET SDK：用于编译Luban工具

## 相关链接

- [Luban官方文档](https://www.datable.cn/)
- [Luban GitHub](https://github.com/focus-creative-games/luban)
