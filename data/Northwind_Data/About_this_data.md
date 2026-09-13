# About the Northwind Data

Using this data **use the Northwind database as a learning dataset** as a leanring tool only.

Microsoft created Northwind as a fictitious trading company and sample database for teaching database design and development. Microsoft still distributes Northwind examples, including the current Northwind 2.0 Access templates, and describes them explicitly as learning/sample databases. 

### What this means for your project

The original **database scripts, forms, reports, documentation, graphics, and other Microsoft-created artifacts can be copyrighted works**. Copyright doesn't disappear simply because something is distributed as a sample.

However, Microsoft intentionally provides Northwind for developers to download, run, study, customize, and learn from. Microsoft also publishes the SQL Server Northwind database through its `sql-server-samples` repository. 

For the **Northwind-based data engineering lab you've been building**, I'd distinguish these cases:

| Use                                                    | Practical assessment           |
| ------------------------------------------------------ | ------------------------------ |
| Load Northwind data into PostgreSQL for learning       | Generally appropriate          |
| Create your own PostgreSQL DDL based on the schema     | Generally appropriate          |
| Build a dimensional model from Northwind               | Generally appropriate          |
| Create Python/dbt/Airflow pipelines around it          | Generally appropriate          |
| Publish your own transformations/code on GitHub        | Generally appropriate          |
| Explain Northwind in YouTube tutorials                 | Generally appropriate          |
| Claim you created Northwind or that it is your dataset | Don't                          |
| Copy Microsoft's documentation/graphics verbatim       | Copyright concerns apply       |
| Redistribute Microsoft's original files                | Check the license/source terms |
| Use Microsoft logos/branding as though affiliated      | Avoid                          |

The **database structure itself** is also different from the particular implementation. Concepts such as `Customers`, `Orders`, `Products`, `Suppliers`, primary keys, foreign keys, and their relationships aren't protected in the same way that Microsoft's particular code, documentation, artwork, or expressive content may be.

### For your GitHub portfolio

For your DE Portfolio, a clean approach would be to say something like:

> **Dataset Attribution:** This project uses the Northwind sample dataset originally developed by Microsoft for database education and demonstration. The PostgreSQL schema, dimensional models, ETL/ELT pipelines, Python code, dbt models, Docker configuration, and other implementation components in this repository were developed independently for educational and portfolio purposes.

That's especially appropriate for what you're doing: taking the familiar Northwind relational model and using it to demonstrate **PostgreSQL → Python → dimensional modeling → dbt → Airflow → Docker → cloud/data-platform engineering**. It makes clear that Northwind is the source dataset while the engineering implementation is yours.

