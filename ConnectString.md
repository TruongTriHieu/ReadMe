# ConnectString
- Connection string is a string that identifies information about the data source and means of connecting to it. It transmits the code to a basic driver or vendor to start connecting. Although commonly used for database connections, the data source can also be a spreadsheet or a text file.
## Parsing Connection Strings
### Quick Example
``` js
        public TikTikDbContext() 
            : base("TikTikConnectionString")
        {
        }

        protected override void OnModelCreating(DbModelBuilder modelBuilder)
        {
            modelBuilder.Configurations.AddFromAssembly(typeof(TikTikDbContext).Assembly);
            base.OnModelCreating(modelBuilder);
        }

        protected new DbSet<TEntity> Set<TEntity>() where TEntity : BaseEntity
        {
            return base.Set<TEntity>();
        } 
```
### Create a connection
- Parse a connection string into ConnectionString object:
``` js
 <add name="TikTikConnectionString" connectionString="Data Source=HIEUTT;Initial Catalog=tiktik_dev;User ID=sa;Password=123456;MultipleActiveResultSets=True;" providerName="System.Data.SqlClient" />
```
### Setting a query
- Queries are set on the .Query property.
``` js
 sql.Query = "SELECT * FROM table"
```
- Add parameter value
``` js
 sql.addParam("@IdNumber", "1");
```
- Remove previously-set parameter value
``` js
 sql.removeParam("@IdNumber");
```
- Update previously-set parameter with new value
``` js
sql.updateParam("@IdNumber", "2");
```
- Clear all previously-set parameters
``` js
sql.clearParams();
```
### Exceptions
``` js
- Throws generic Exception if no connection was opened (not connection error) - also probably shouldn't happen
- Throws generic Exception if the Query property is empty
- Throws generic Exception if the any of the connection paramters are null or empty strings
```
