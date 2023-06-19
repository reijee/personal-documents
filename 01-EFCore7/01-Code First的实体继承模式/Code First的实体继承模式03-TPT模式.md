# Table per Type (TPT)

在上一篇文章中，你看到有三种不同的方法来表示继承层次结构，我将每个层次结构的表 （TPH） 解释为 EF Code First 中的默认映射策略。我们认为TPH的缺点对于我们的设计来说可能太严重了，因为它会导致非规范化的模式，从长远来看，这可能会成为主要负担。在今天的博客文章中，我们将了解每种类型的表（TPT）作为另一种继承映射策略，我们将看到TPT不会使我们面临此问题。

使用数据表的外键来表示实体类的继承关系。为每一个实体（**包含基类**）创建一个数据表，表中只包含实体直接定义的属性。它们之间使用一个共同的主键来维护实体类的继承关系。

## 创建上下文

与上一节中 **Table per Hierarchy (TPH)** 一样，也需要在DBContext上下文中定义实体映射方式。

与上一章节中主要区别如下：

- 不需要配置鉴别器列
- 只需要把基类（Picture）添加到DbSet，实体类（LandscapePicture、PeoplePicture）由于是继承于基类可以不用添加到DbSet
- 在OnModelCreating中把实体类以及基类都添加ToTable配置

上下文代码如下：

~~~csharp
public class DbContextForTPT: DbContext
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

    // public DbSet<LandscapePicture> LandscapePictures { get; set; }

    // public DbSet<PeoplePicture> PeoplePictures { get; set; }

    /// <summary>
    /// 创建模型
    /// </summary>
    /// <param name="modelBuilder"></param>
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Picture>(b =>
        {
            b.ToTable("tpt_picthure");
            b.Property(t => t.Width).HasColumnType("decimal(18,2)");
            b.Property(t => t.Height).HasColumnType("decimal(18,2)");
        });

        modelBuilder.Entity<LandscapePicture>(b =>
        {
            b.ToTable("tpt_landscape_picthure");
            b.Property(t => t.LandscapeFlag).HasColumnType("decimal(18,4)");
        });

        modelBuilder.Entity<PeoplePicture>(b =>
        {
            b.ToTable("tpt_people_picthure");
        });
    }
}
~~~

## 生成数据表

在项目上点右键，然后选择 “在终端中打开” 然后在终端中执行以下脚本生成迁移并更新到数据库

~~~shell
# 生成迁移，--context 参数用于指定上下文，如果项目中只有一个，可以不用指定
dotnet ef migrations add init-tpt --context DbContextForTPT

# 把迁移更新到数据库
dotnet ef database update --context DbContextForTPT
~~~

数据库中生成出来的表结构如下所示：

~~~sql
CREATE TABLE [tpt_picthure] (
    [PictureId] int NOT NULL IDENTITY,
    [Name] nvarchar(max) NOT NULL,
    [Description] nvarchar(max) NULL,
    [Url] nvarchar(max) NOT NULL,
    [Width] decimal(18,2) NOT NULL,
    [Height] decimal(18,2) NOT NULL,
    CONSTRAINT [PK_tpt_picthure] PRIMARY KEY ([PictureId])
);
GO

CREATE TABLE [tpt_landscape_picthure] (
    [PictureId] int NOT NULL,
    [Address] nvarchar(max) NULL,
    [LandscapeFlag] decimal(18,4) NOT NULL,
    CONSTRAINT [PK_tpt_landscape_picthure] PRIMARY KEY ([PictureId]),
    CONSTRAINT [FK_tpt_landscape_picthure_tpt_picthure_PictureId] FOREIGN KEY ([PictureId]) REFERENCES [tpt_picthure] ([PictureId]) ON DELETE CASCADE
);
GO

CREATE TABLE [tpt_people_picthure] (
    [PictureId] int NOT NULL,
    [PeopleIntro] nvarchar(max) NULL,
    CONSTRAINT [PK_tpt_people_picthure] PRIMARY KEY ([PictureId]),
    CONSTRAINT [FK_tpt_people_picthure_tpt_picthure_PictureId] FOREIGN KEY ([PictureId]) REFERENCES [tpt_picthure] ([PictureId]) ON DELETE CASCADE
);
GO
~~~

从生成的脚本中来看

1. EF为每一个类（包含实体类及基类）都生成一个数据表，且每个类表只包含本身直接声明的属性。
2. 实体类生成的数据表共享基类表的主键（PictureId），并建立外键关系，用于表示实体类的继承关系。

数据表更新完成之后，我们就要可以向表中添加一些数据

~~~csharp
using (var dbContext = new DbContextForTPT())
{
    dbContext.Set<LandscapePicture>().Add(new LandscapePicture()
    {
        Name = "黄帝故里",
        Description = "黄帝故里为汉籍史书中记载有熊氏的族居地，故有熊国之墟。",
        Url = "https://youimg1.c-ctrip.com/target/100u0q000000g8vrp9E7F.jpg",
        Width = 1219,
        Height = 1050,
        LandscapeFlag = 2.2m,
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

然后执行以下查询语句，看一看数据库中数据存储情况：

~~~sql
SELECT PictureId, [Name] FROM dbo.tpt_picthure
SELECT PictureId, [Address], LandscapeFlag FROM dbo.tpt_landscape_picthure
SELECT PictureId,PeopleIntro FROM dbo.tpt_people_picthure
~~~

~~~txt

PictureId   Name
----------- -----------------
1           黄帝故里
2           毛泽东


PictureId   Address                          LandscapeFlag
----------- -------------------------------- ---------------------------------------
1           河南省郑州市新郑市轩辕路1号        2.2


PictureId   PeopleIntro
----------- ----------------------------------
2           毛泽东，字润之，湖南湘潭人。...
~~~

在查询实体类时，EF会自动生成 Inner Join 查询关联实体表及基类对应的表

~~~csharp
using (var dbContext = new DbContextForTPT())
{
    var landscapePictureSet = dbContext.Set<LandscapePicture>();  
    var firstPicture = landscapePictureSet.FirstOrDefault();
}
~~~

~~~sql
SELECT TOP(1) [t].[PictureId], [t].[Description], [t].[Height], [t].[Name], [t].[Url], [t].[Width], [t0].[Address], [t0].[LandscapeFlag]
FROM [tpt_picthure] AS [t]
INNER JOIN [tpt_landscape_picthure] AS [t0] ON [t].[PictureId] = [t0].[PictureId]
~~~
