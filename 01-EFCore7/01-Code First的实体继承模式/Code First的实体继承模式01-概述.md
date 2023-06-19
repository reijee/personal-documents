# Code First的实体继承模式

将实体类映射到数据库表的简单策略可能是“每个实体持久类一个表”。 如果实体类之间有继承关系，那么就会比较麻烦。继承是面向对象的与关系类的数据库的结构不匹配。

为了解决这个问题，EF提供了三种不同的实体继承模式：

- Table per Hierarchy (TPH)
  通过非规范化 SQL 架构来启用多态性，并利用保存类型信息的类型鉴别器列。通俗讲就是只建立一个数据表，把基类和子类中的所有属性都映射为表中的列，并在表中添加一个数据列用于鉴别实体来源。
- Table per Type (TPT)
  使用数据表的外键来表示实体类的继承关系。为每一个实体（**包含基类**）创建一个数据表，表中只包含实体直接定义的属性。它们之间使用一个共同的主键来维护实体类的继承关系。
- Table per Concrete Type (TPC)
  从 SQL 架构中完全丢弃多态性和继承关系。为每一个实体（***不包含基类***）创建一个数据表，表中包含实体直接定义的属性以及从基类继承过来属性。

## 定义实体类

在进行真正的内容之前，需要把使用到的实体类定义好。

假如我们创建一个图片管理的小工具，把图片分为风景图片及人物图片两大类，它们分别有一些特殊的属性。创建实体类如下所示：

~~~ csharp
/// <summary>
/// 图片类
/// </summary>
public abstract class Picture
{
    public Guid Id { get; set; }

    public string Name { get; set; }

    public string? Description { get; set; }

    public string Url { get; set; }

    public decimal Width { get; set; }

    public decimal Height { get; set; }
}

/// <summary>
/// 人物图片
/// </summary>
public class PeoplePicture: Picture
{
    /// <summary>
    /// 人物介绍
    /// </summary>
    public string? PeopleIntro { get; set; }
}

/// <summary>
/// 风景图片
/// </summary>
public class LandscapePicture: Picture
{
    /// <summary>
    /// 地址
    /// </summary>
    public string? Address { get; set; }

    /// <summary>
    /// 风景标记，不可以为空
    /// </summary>
    public decimal LandscapeFlag { get; set; }
}
~~~

## 索引目录

- [Code First的实体继承模式01-概述](./Code%20First%E7%9A%84%E5%AE%9E%E4%BD%93%E7%BB%A7%E6%89%BF%E6%A8%A1%E5%BC%8F01-%E6%A6%82%E8%BF%B0.md)
- [Code First的实体继承模式02-TPH模式](./Code%20First%E7%9A%84%E5%AE%9E%E4%BD%93%E7%BB%A7%E6%89%BF%E6%A8%A1%E5%BC%8F02-TPH%E6%A8%A1%E5%BC%8F.md)
- [Code First的实体继承模式03-TPT模式](./Code%20First%E7%9A%84%E5%AE%9E%E4%BD%93%E7%BB%A7%E6%89%BF%E6%A8%A1%E5%BC%8F03-TPT%E6%A8%A1%E5%BC%8F.md)
- [Code First的实体继承模式04-TPC模式](./Code%20First%E7%9A%84%E5%AE%9E%E4%BD%93%E7%BB%A7%E6%89%BF%E6%A8%A1%E5%BC%8F04-TPC%E6%A8%A1%E5%BC%8F.md)

