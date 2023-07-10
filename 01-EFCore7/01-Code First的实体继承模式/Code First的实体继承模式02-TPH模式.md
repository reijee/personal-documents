# Table per Hierarchy (TPH)

通过非规范化 SQL 架构来启用多态性，并利用保存类型信息的类型鉴别器列。通俗讲就是只建立一个数据表，把基类和子类中的所有属性都映射为表中的列，并在表中添加一个数据列用于鉴别实体来源。

此为EFCore的默认继承策略。

## 创建上下文

实体的映射方式在DBContext上下文中定义。

在上下文中需要把实体类添加到DbSet（LandscapePicture、PeoplePicture），而基类（Picture）则无硬性要求，需要根据项目的实际情况而定。

在OnModelCreating方法中 通过 Fluent API 配置模型时，需要注意以下事项：

- 只需要在配置基类时调用 **ToTable** 方法，因为按照TPH模式我们只需要生成一张表。
- 对于实体类，可以调用 **Entity** 方法配置字段属性，但是不能调用 **ToTable** 方法。如果实体类调用  **ToTable** 方法，那么EF会按照TPT模式生成映射表。

上下文代码如下：

~~~csharp
public class DbContextForTPH : DbContext
{
    /// <summary>
    /// 配置数据库，这里只是为了方便演示
    /// 实际项目中建议放到 Program 类中注入
    /// </summary>
    /// <param name="options"></param>
    protected override void OnConfiguring(DbContextOptionsBuilder options)
    {
        if (!options.IsConfigured)
        {
            options.UseSqlServer("server=.;database=rick_demo_efcoremap;uid=sa;pwd=123456; Encrypt=False;");
        }
    }

    //public DbSet<Picture> Pictures { get; set; }

    public DbSet<LandscapePicture> LandscapePictures { get; set; }

    public DbSet<PeoplePicture> PeoplePictures { get; set; }

    /// <summary>
    /// 创建模型
    /// </summary>
    /// <param name="modelBuilder"></param>
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Picture>(b =>
        {
            b.ToTable("tph_picthure");
            // 为Width指定数据类型为decimal(18,2)
            b.Property(t => t.Width).HasColumnType("decimal(18,2)"); 
            // 为Height指定数据类型为decimal(18,2)
            b.Property(t => t.Height).HasColumnType("decimal(18,2)"); 
        });

        // 可以配置字段属性，但不可以调用 ToTable 方法
        modelBuilder.Entity<LandscapePicture>(b =>
        {
            b.Property(t => t.LandscapeFlag).HasColumnType("decimal(18,4)");
        });
    }
}
~~~

## 生成数据表

在项目上点右键，然后选择 “在终端中打开” 然后在终端中执行以下脚本生成迁移并更新到数据库

~~~shell
# 生成迁移，--context 参数用于指定上下文，如果项目中只有一个，可以不用指定
dotnet ef migrations add init-tph --context DbContextForTPH

# 把迁移更新到数据库
dotnet ef database update --context DbContextForTPH
~~~

数据库中生成出来的表结构如下所示：

~~~sql
CREATE TABLE [tph_picthure] (
    [PictureId] int NOT NULL IDENTITY,
    [Name] nvarchar(max) NOT NULL,
    [Description] nvarchar(max) NULL,
    [Url] nvarchar(max) NOT NULL,
    [Width] decimal(18,2) NOT NULL,
    [Height] decimal(18,2) NOT NULL,
    [Discriminator] nvarchar(max) NOT NULL,
    [Address] nvarchar(max) NULL,
    [LandscapeFlag] decimal(18,4) NULL,
    [PeopleIntro] nvarchar(max) NULL,
    CONSTRAINT [PK_tph_picthure] PRIMARY KEY ([PictureId])
)
~~~

数据表更新完成之后，我们就要可以向表中添加一些数据

~~~csharp
using (var dbContext = new DbContextForTPH())
{
    dbContext.Set<LandscapePicture>().Add(new LandscapePicture()
    {
        Name = "黄帝故里",
        Description = "黄帝故里为汉籍史书中记载有熊氏的族居地，故有熊国之墟。",
        Url = "https://youimg1.c-ctrip.com/target/100u0q000000g8vrp9E7F.jpg",
        Width = 1219,
        Height = 1050,
        Address = "河南省郑州市新郑市轩辕路1号"
    });
    dbContext.Set<PeoplePicture>().Add(new PeoplePicture()
    {
        Name = "毛泽东",
        Description = "伟大的马克思主义者，无产阶级革命家、战略家和理论家，中国共产党、中国人民解放军和中华人民共和国的主要缔造者和领导人。",
        Url = "https://www.rmzxb.com.cn/upload/resources/image/2014/09/17/31766.jpg",
        Width = 158,
        Height = 218,
        PeopleIntro = "毛泽东，字润之，湖南湘潭人。"
    });

    dbContext.SaveChanges();
}
~~~

### 鉴别器列

从数据库中生成的数据表结构中可以看到，EF在表结构中自动添加一列 **Discriminator**，此数据列用于记录数据来源于哪个实体。

我们执行一个脚本查询一下数据库中列 **Discriminator** 的数据存储情况：

~~~sql
SELECT PictureId, Name, Discriminator FROM [tph_picthure]
~~~

~~~text
PictureId   Name                             Discriminator
----------- -------------------------------- --------------------------------
1           黄帝故里                          LandscapePicture
2           毛泽东                            PeoplePicture
~~~

从查询的结果上来看 **Discriminator** 存储实体类的名称用于区别实体来源。

我们再通过一个Linq查询看一看EF生成的查询脚本是什么样的

~~~csharp
using (var dbContext = new DbContextForTPH())
{
    var peoplePictureSet = dbContext.Set<PeoplePicture>();
    var firstPicture = peoplePictureSet.FirstOrDefault();
}
~~~

~~~sql
SELECT TOP(1) [t].[PictureId], [t].[Description], [t].[Discriminator], [t].[Height], [t].[Name], [t].[Url], [t].[Width], [t].[PeopleIntro]
FROM [tph_picthure] AS [t]
WHERE [t].[Discriminator] = N'PeoplePicture'
~~~

从上面的结果来看，在通过实体查询时，EF会生成一个针对Discriminator的Where条件。由于EF自动生成的 **Discriminator** 列是 nvarchar(MAX) 类型的，不能针对此列创建索引， 所以查询的性能比较差。 不用担心，EF为我们提供自定义识别器列的API用于解决这个问题。

修改 OnModelCreating 方法，调用 HasDiscriminator 及 HasValue 方法配置鉴别器列，代码如下所示：

~~~csharp
/// <summary>
/// 创建模型
/// </summary>
/// <param name="modelBuilder"></param>
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Picture>(b =>
    {
        b.ToTable("tph_picthure");
        b.Property(t => t.Width).HasColumnType("decimal(18,2)");
        b.Property(t => t.Height).HasColumnType("decimal(18,2)");

        // 配置鉴别器列
        b.HasDiscriminator<int>("PicthureType") // 指定鉴别器列的名称及类型
            .HasValue<LandscapePicture>(1) // 指定实体LandscapePicture的值为1
            .HasValue<PeoplePicture>(2); // 指定实体PeoplePicture的值为2
    });

    // 可以配置字段属性，但不可以调用 ToTable 方法
    modelBuilder.Entity<LandscapePicture>(b =>
    {
        b.Property(t => t.LandscapeFlag).HasColumnType("decimal(18,4)");
    });
}
~~~

在上面的示例中，EF 在层次结构的基本实体上隐式添加了鉴别器作为影子属性。 鉴别器也可以映射到实体中的常规 .NET 属性：

1. 修改基类 Picture 添加 属性 PicthureType

~~~csharp
/// <summary>
/// 图片类
/// </summary>
public abstract class Picture
{
    public int PictureId { get; set; }

    public string Name { get; set; }

    public string? Description { get; set; }

    public string Url { get; set; }

    public decimal Width { get; set; }

    public decimal Height { get; set; }

    /// <summary>
    /// 图片类型，鉴别器列
    /// </summary>
    public int PicthureType { get; set; }
}
~~~

2. 修改 OnModelCreating 方法，配置鉴别器列

~~~csharp
/// <summary>
/// 创建模型
/// </summary>
/// <param name="modelBuilder"></param>
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Picture>(b =>
    {
        b.ToTable("tph_picthure");
        b.Property(t => t.Width).HasColumnType("decimal(18,2)");
        b.Property(t => t.Height).HasColumnType("decimal(18,2)");

        // 配置鉴别器列
        b.HasDiscriminator(t=>t.PicthureType) // 使用 Picture 类的 PicthureType 属性作为鉴别器列
            .HasValue<LandscapePicture>(1) // 指定实体LandscapePicture的值为1
            .HasValue<PeoplePicture>(2); // 指定实体PeoplePicture的值为2
    });

    // 可以配置字段属性，但不可以调用 ToTable 方法
    modelBuilder.Entity<LandscapePicture>(b =>
    {
        b.Property(t => t.LandscapeFlag).HasColumnType("decimal(18,4)");
    });
}
~~~

## 个人总结

TPH实体映射模式把所有的属性都放在一张表中，它在查询性能以及简单化上面有很大的优势。但是它的问题也比较明显，首先这种设计方式不符合数据的第三范式，另外因为TPH中很多子类属性对应的列是可为空的，就为数据验证增加了复杂性，也浪费了很多存储空间。


## 索引目录

- [Code First的实体继承模式01-概述](./Code%20First%E7%9A%84%E5%AE%9E%E4%BD%93%E7%BB%A7%E6%89%BF%E6%A8%A1%E5%BC%8F01-%E6%A6%82%E8%BF%B0.md)
- [Code First的实体继承模式02-TPH模式](./Code%20First%E7%9A%84%E5%AE%9E%E4%BD%93%E7%BB%A7%E6%89%BF%E6%A8%A1%E5%BC%8F02-TPH%E6%A8%A1%E5%BC%8F.md)
- [Code First的实体继承模式03-TPT模式](./Code%20First%E7%9A%84%E5%AE%9E%E4%BD%93%E7%BB%A7%E6%89%BF%E6%A8%A1%E5%BC%8F03-TPT%E6%A8%A1%E5%BC%8F.md)
- [Code First的实体继承模式04-TPC模式](./Code%20First%E7%9A%84%E5%AE%9E%E4%BD%93%E7%BB%A7%E6%89%BF%E6%A8%A1%E5%BC%8F04-TPC%E6%A8%A1%E5%BC%8F.md)

