# EFCore查询跟踪小记

- [EFCore查询跟踪小记](#efcore查询跟踪小记)
  - [示例数据](#示例数据)
  - [永远不会被跟踪的实体](#永远不会被跟踪的实体)
  - [如何查看已跟踪的实体](#如何查看已跟踪的实体)
  - [什么是标识解析](#什么是标识解析)
  - [如何执行非跟踪查询](#如何执行非跟踪查询)
  - [非跟踪查询的性能一定就高吗](#非跟踪查询的性能一定就高吗)


查询跟踪算是EF/EFCore特有的核心功能，本文章的内容主要记录一些查询跟踪比较细节的东西。

## 示例数据

为了方便说明问题，需要添加一些实体及示例数据。包含两个实体 Product（商品）以及Property（商品属性）。

代码如下所示：

~~~csharp
/// <summary>
/// 商品
/// </summary>
public class Product
{
    public int Id { get; set; }

    public string Name { get; set; }

    public string? Description { get; set; }

    public decimal Price { get; set; }

    public List<Property>? Properties { get; set; }
}

/// <summary>
/// 商品属性
/// </summary>
public class Property
{
    public int Id { get; set; }

    public string Title { get; set; }

    public string? Value { get; set; }

    public int ProductId { get; set; }

    public Product Product { get; set; }
}
~~~

DbContext文件

~~~csharp
public class TrackingDbContext: DbContext
{
    public DbSet<Product> Products { get; set; }
    public DbSet<Property> Propertys { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        if (!optionsBuilder.IsConfigured)
        {
            optionsBuilder.UseSqlServer("server=.;database=demo_tracking;uid=sa;pwd=********; Encrypt=False;");
        }
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>(builder =>
        {
            builder.Property(t=>t.Name).IsRequired(true).HasMaxLength(50);
            builder.Property(t => t.Price).HasColumnType("decimal(18,2)");
        });

        modelBuilder.Entity<Property>(builder =>
        {
            builder.Property(t => t.Title).IsRequired(true).HasMaxLength(50);
            builder.Property(t=>t.Value).HasMaxLength(50);
        });
    }
}
~~~

## 永远不会被跟踪的实体

在EFCore中 **无键实体** 是不执行实体跟踪的，无键实体通俗来讲就是没有定义主键的实体，具体请参考：https://learn.microsoft.com/zh-cn/ef/core/modeling/keyless-entity-types?tabs=data-annotations


## 如何查看已跟踪的实体

EFCore是通过 DBContext.ChangeTracker 管理跟踪查询的。要查看已跟踪的实体有以下两个方法：

- 调用 **ChangeTracker.DebugView** 属性输出或打印已跟踪的实体信息
- 调用 **ChangeTracker.Entries()** 方法遍历已跟踪的实体信息

~~~
注意：
正常来说在调用 SaveChanges 时，会进行更改检测，以确保在将更新发送到数据库之前检测到所有更改的值。但是也可以调用context.ChangeTracker.DetectChanges()方法强制执行更改检测。
所以在调用以上两个属性之前，需要调用context.ChangeTracker.DetectChanges()方法强制执行更改检测。
~~~

示例代码如下：

~~~csharp
using(TrackingDbContext context = new TrackingDbContext())
{
    var product = context.Products.Include(t=>t.Properties).First();
    product.Description = "..." + DateTime.Now.ToString();

    // 强制执行更改检查
    context.ChangeTracker.DetectChanges();

    Console.WriteLine(">> ChangeTracker.DebugView");
    Console.WriteLine(context.ChangeTracker.DebugView.LongView);

    Console.WriteLine(">> ChangeTracker.Entries()");
    foreach (var item in context.ChangeTracker.Entries())
    {
        Console.WriteLine($"* entry item：{item.Entity}, State = {item.State}");
    }
    Console.WriteLine();

    context.SaveChanges();
}
~~~

打印输出的内容：

~~~text
>> ChangeTracker.DebugView
Product {Id: 1} Modified
  Id: 1 PK
  Description: '...2023/07/29 17:29:21' Modified Originally '...2023/07/29 17:28:29'
  Name: '21天学会计'
  Price: 9.99
  Properties: [{Id: 1}, {Id: 2}]
Property {Id: 1} Unchanged
  Id: 1 PK
  ProductId: 1 FK
  Title: '讲师'
  Value: 'Value2023/07/29 14:35:43'
  Product: {Id: 1}
Property {Id: 2} Unchanged
  Id: 2 PK
  ProductId: 1 FK
  Title: '来源'
  Value: 'Value2023/07/29 14:35:43'
  Product: {Id: 1}

>> ChangeTracker.Entries()
* entry item：rick.demo.tracking.Entitys.Product, State = Modified
* entry item：rick.demo.tracking.Entitys.Property, State = Unchanged
* entry item：rick.demo.tracking.Entitys.Property, State = Unchanged
~~~

ChangeTracker还提供了几个事件来管理实体跟踪：

- ChangeTracker.Tracked：有实体被跟踪时会激活此事件
- ChangeTracker.StateChanged：被跟踪实体状态发生变化进激活此事件

## 什么是标识解析

默认情况下，EF 跟踪实体实例，以便在调用 时 SaveChanges 检测并保留这些实例上的更改。 跟踪查询的另一个作用是：EF 检测是否已为你的数据加载了实例，并将自动返回跟踪的实例，而不是返回新实例。这种做法称为标识解析。 

也就是说已经被跟踪的实体，再次被查询时，不会从数据库中返回新的实例，而是直接返回已被跟踪的实例。

~~~csharp
using(TrackingDbContext context = new TrackingDbContext())
{
    // 第一次查询
    var product = context.Products.Include(t=>t.Properties).First();
    product.Description = "..." + DateTime.Now.ToString();

    // 第二次查询
    var product2 = context.Products.First(t => t.Id == 1);
                
    Console.WriteLine(Object.ReferenceEquals(product, product2)); // 输出 True

    context.SaveChanges();
}
~~~

## 如何执行非跟踪查询

EFCore会默认跟踪所有查询出来的实体（无键实体除外），那么怎么样执行非跟踪查询呢？

> 一、针对单个语句的非跟踪查询

在查询语句中调用 **AsNoTracking()** 方法，执行非跟踪查询

~~~csharp
using(TrackingDbContext context = new TrackingDbContext())
{
    // 非跟踪查询
    var product = context.Products.AsNoTracking().First(t => t.Id == 1);

}
~~~

> 二、局部非跟踪查询

修改DbContext的属性值，执行非跟踪查询

~~~csharp
using(TrackingDbContext context = new TrackingDbContext())
{
    // 修改属性，设置当前实例范围内为非跟踪查询
    context.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;

}
~~~

> 三、修改全局配置

通过 DbContextOptionsBuilder.UseQueryTrackingBehavior 配置默认跟踪行为

~~~csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    if (!optionsBuilder.IsConfigured)
    {
        // 默认执行非跟踪查询
        optionsBuilder.UseSqlServer("server=.;database=rick_demo_tracking;uid=sa;pwd=123456; Encrypt=False;")
            .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
    }
}
~~~

如果需要执行跟踪查询可以调用 AsTracking() 或者配置 ChangeTracker.QueryTrackingBehavior 属性实现

~~~csharp
using(TrackingDbContext context = new TrackingDbContext())
{
    // 配置跟踪查询
    context.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.TrackAll;

    // 跟踪查询
    var product = context.Products.AsTracking().First(t=>t.Id == 1);
}
~~~

## 非跟踪查询的性能一定就高吗

从上述的标识解析中可以确定，EF 在内部维护跟踪实例的字典。 加载新数据时，EF 会查阅字典，以了解是否已为该实体的键跟踪了实例（标识解析）。

- 跟踪查询会占用更多的内存，而且查询字典也要消耗一定的性能。
- 非跟踪查询会执行数据库查询，返回一个新的实例

也就是说不能单纯的确定哪一个性能好，而是要跟踪实际应用情况来定。
