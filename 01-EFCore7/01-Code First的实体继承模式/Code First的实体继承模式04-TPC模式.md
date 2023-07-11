# Table per Concrete Class (TPC)

从 SQL 架构中完全丢弃多态性和继承关系。为每一个实体（***不包含基类***）创建一个数据表，表中包含实体直接定义的属性以及从基类继承过来属性。与前两种模式是主要的区别就是在数据库中不体现实体之是的继承关系，每个实体类都有自己的表，但是不生成基类的表，基类的属性包含在各个实体类表中。

## 创建上下文

TPC模式与TPT的模式差不多，主要有两点区别：

- 由于不生成基类对应的表，所以在配置基类时不需要调用 **ToTable** 指定数据表
- 在配置基类时需要手工调用 **UseTpcMappingStrategy** 方法旨定继承方式为TPC

基代码如下所示：

~~~csharp
public class DbContextForTPC: DbContext
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

    public DbSet<Picture> Pictures { get; set; }

    //public DbSet<LandscapePicture> LandscapePictures { get; set; }

    //public DbSet<PeoplePicture> PeoplePictures { get; set; }

    /// <summary>
    /// 创建模型
    /// </summary>
    /// <param name="modelBuilder"></param>
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Picture>(b =>
        {
            b.Property(t => t.Width).HasColumnType("decimal(18,2)");
            b.Property(t => t.Height).HasColumnType("decimal(18,2)");
                
            // 指定使用TPC模式
            b.UseTpcMappingStrategy();
        });

        modelBuilder.Entity<LandscapePicture>(b =>
        {
            b.ToTable("tpc_landscape_picthure");
            b.Property(t => t.LandscapeFlag).HasColumnType("decimal(18,4)");
        });

        modelBuilder.Entity<PeoplePicture>(b =>
        {
            b.ToTable("tpc_people_picthure");
        });
    }
}
~~~

## 生成数据表

在项目上点右键，然后选择 “在终端中打开” 然后在终端中执行以下脚本生成迁移并更新到数据库

~~~shell
# 生成迁移，--context 参数用于指定上下文，如果项目中只有一个，可以不用指定
dotnet ef migrations add init-tpc --context DbContextForTPC

# 把迁移更新到数据库
dotnet ef database update --context DbContextForTPC
~~~

数据库中生成出来的表结构如下所示：

~~~sql
CREATE SEQUENCE [PictureSequence] START WITH 1 INCREMENT BY 1 NO MINVALUE NO MAXVALUE NO CYCLE;
GO

CREATE TABLE [tpc_landscape_picthure] (
    [PictureId] int NOT NULL DEFAULT (NEXT VALUE FOR [PictureSequence]),
    [Name] nvarchar(max) NOT NULL,
    [Description] nvarchar(max) NULL,
    [Url] nvarchar(max) NOT NULL,
    [Width] decimal(18,2) NOT NULL,
    [Height] decimal(18,2) NOT NULL,
    [Address] nvarchar(max) NULL,
    [LandscapeFlag] decimal(18,4) NOT NULL,
    CONSTRAINT [PK_tpc_landscape_picthure] PRIMARY KEY ([PictureId])
);
GO

CREATE TABLE [tpc_people_picthure] (
    [PictureId] int NOT NULL DEFAULT (NEXT VALUE FOR [PictureSequence]),
    [Name] nvarchar(max) NOT NULL,
    [Description] nvarchar(max) NULL,
    [Url] nvarchar(max) NOT NULL,
    [Width] decimal(18,2) NOT NULL,
    [Height] decimal(18,2) NOT NULL,
    [PeopleIntro] nvarchar(max) NULL,
    CONSTRAINT [PK_tpc_people_picthure] PRIMARY KEY ([PictureId])
);
GO
~~~

从上面的数据库脚本可以看出只生成实体类的表，没有基类对应的表，且基类的属性都包含在实体类表中。另外一点需要注意的是由于表主键使用 **自增型** 的，为了防止两个表中主键重复，所以EF生成一个名为 **PictureSequence** 的序列，而表的主键依次从此序列中获取。

当然如果使用GUID作为主键，那么就不存在重复，也就不需要创建序列。


## 索引目录

- [Code First的实体继承模式01-概述](./Code%20First%E7%9A%84%E5%AE%9E%E4%BD%93%E7%BB%A7%E6%89%BF%E6%A8%A1%E5%BC%8F01-%E6%A6%82%E8%BF%B0.md)
- [Code First的实体继承模式02-TPH模式](./Code%20First%E7%9A%84%E5%AE%9E%E4%BD%93%E7%BB%A7%E6%89%BF%E6%A8%A1%E5%BC%8F02-TPH%E6%A8%A1%E5%BC%8F.md)
- [Code First的实体继承模式03-TPT模式](./Code%20First%E7%9A%84%E5%AE%9E%E4%BD%93%E7%BB%A7%E6%89%BF%E6%A8%A1%E5%BC%8F03-TPT%E6%A8%A1%E5%BC%8F.md)
- [Code First的实体继承模式04-TPC模式](./Code%20First%E7%9A%84%E5%AE%9E%E4%BD%93%E7%BB%A7%E6%89%BF%E6%A8%A1%E5%BC%8F04-TPC%E6%A8%A1%E5%BC%8F.md)
