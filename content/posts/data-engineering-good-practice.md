+++
title = 'Data Engineering Good Practice'
date = 2023-11-27T02:21:28+01:00
draft = true
+++

# Guidelines to setup a modern Data System on Databricks and Azure 

## I. Introduction

Data engineering plays a crucial role in the success of any data-driven organization. It involves the design, development, testing, and maintenance of the infrastructure, data pipelines, and data processing systems that enable the organization to turn raw data into actionable insights. As data engineering has grown in importance, it has become increasingly intertwined with software engineering practices. The result is a set of best practices and tools that can help ensure the success of data engineering projects.

This document is a knowledge base for starting a new data engineering project on your company data platform. It emphasizes the importance of using practices inherited from software engineering, such as Git, infrastructure as code, testable code, and continuous integration/continuous delivery (CI/CD). By using these practices, we can ensure that our data engineering projects are well-structured, maintainable, and scalable.

In this document, we will cover best practices for your project. By following these best practices, we can ensure that our data engineering projects are successful and contribute to the growth of a data driven mindset at your company.

## II. Raw Data Storage in the Bronze Layer

*The lakehouse architecture is described in annex A.*

The bronze layer is the first layer in the data platform where raw data is stored. This layer is dedicated to storing the raw data as it is ingested from external sources. The raw data can be in different formats such as CSV, JSON, or Parquet. It is important to organize this layer in a way that makes it easy to access and process the data.

One way to organize the data in the bronze layer is to partition it by ingest timestamp. When data is ingested into the data platform, a timestamp indicating the time of ingestion should be added to each record. This timestamp can then be used to partition the data in the bronze layer, making it easier to manage and process. By partitioning the data by ingest timestamp, we can easily query and retrieve the data for a specific time period or replay a data processing at a previous point in time.

For example, let's say we ingest data every day at 1am. We can partition the data by the ingest timestamp to create a new partition for each day's data. The partition name could be in the format ISO 8601 to indicate the date of ingestion. This makes it easy to query the data for a specific day or range of days, and also allows us to easily identify the most recent data for a specific dataset. This format is used in my projects.

Additionally, it's important to maintain the raw data in the bronze layer as-is, without any modifications or transformations. This allows us to ensure data lineage and traceability, which is important for compliance and auditing purposes.

```
bronze/project_1
├── ingest_timestamp=2022-03-16T12:00:00.000Z/
│   ├── customer_data_0.parquet
│   ├── product_data_0.parquet
│   └── sales_data_0.parquet
├── ingest_timestamp=2022-03-16T13:00:00.000Z/
│   ├── product_data_0.parquet
│   └── sales_data_0.parquet
└── ingest_timestamp=2022-03-16T14:00:00.000Z/
    ├── customer_data_0.parquet
    ├── product_data_0.parquet
    └── sales_data_0.parquet

```

Of course, it can be adapted but it is advised to keep a partition by timestamp.

How the data is written to the bronze layer is not covered in this guide but Azure Data Factory allows to leverage lots of different connectors (SAP BW, SQL Server...). It is our go-to solution.

## III. Data Ingestion

The first step in any data engineering project is to ingest the data. Ingesting data involves loading data from external sources into our data platform for further processing and analysis. There are several options for ingesting data into Databricks, including using external tools and libraries or building custom components from scratch. However, the best option is to use what Databricks offers out of the box, as it avoids the risk of bugs and the burden of maintenance that comes with building components from scratch.

One key tool that Databricks offers for data ingestion is Databricks Autoloader. Autoloader is a managed service that enables easy and efficient loading of data from cloud storage platforms such as AWS S3 or Azure Blob Storage. With Autoloader, data ingestion can be automated with just a few lines of code, and the service automatically scales to handle large volumes of data. Autoloader also offers built-in error handling and schema inference, making it easy to ensure the data is properly formatted and cleaned before it is loaded into Databricks.

Another benefit of using Autoloader is that it integrates seamlessly with other Databricks tools such as Delta Lake, allowing for efficient data processing and analysis. With Delta Lake, data can be versioned, optimized for performance, and easily queried using SQL or Python. Additionally, Delta Lake provides ACID transactions, making it easy to ensure data consistency and integrity.

**Try using Autoloader as the ingestion engine and DeltaLake as the storage format.**

No business transformation should occur at this stage and the result tables should be stored in the silver layer. For instance we can choose to name these tables `ingest_*` to be able to easily recognize them.

The partition scheme is the same for all `ingest_*` tables and also uses `ingest_timestamp`.

**Try using generic code for ingestion.** This approach involves defining all datasets' metadata in a catalog.toml file, including the location, pattern, format, and sink table details. The code for ingestion, IngestionJob, then reads the metadata from the catalog.toml file and ingests all datasets without any modification to the code. The ingestion code reads raw files in a given directory location and applies the defined schema if available, or infers it if not. After processing, the resulting data frames are returned with an additional input_file column. Finally, the code writes the processed data frames to the defined sink table in Delta format, partitioned by the specified partitions. This approach enables efficient and scalable ingestion of new datasets while reducing code maintenance and increasing code reuse.

  *Databricks Autoloader is described in annex B.*

## IV. Data Processing

In a typical data engineering pipeline, the silver layer stores raw data that has been ingested from various sources, and the gold layer contains business-ready data that can be consumed by the end-users or applications. However, it is not uncommon for intermediate tables to be stored in the silver layer. The gold layer is the final destination for data that has been processed and transformed into a model that meets the requirements of the business.

In our project, we have opted to create views in the gold layer to allow stakeholders to easily transform data. These views are defined as SQL statements and are stored in our code repository, as versioning is a critical requirement for a modern data project. While this approach has its benefits, our preferred method is to write Spark code using Python. Writing code in Python enables easy unit testing and allows us to lint, format, and organize the code to facilitate maintenance.

We could define an abstract class `SparkJob` that acts like a blueprint for all our Spark Jobs.
The advantage of implementing an abstract class is that it provides a structure and framework for building and maintaining data processing applications. By defining a standardized interface, the class can be extended and reused for different use cases and scenarios, without having to rewrite the core processing logic.

This approach also enables separation of concerns, as the implementation details are encapsulated in the methods of the abstract class, making the code easier to read, understand, and modify. In addition, it allows for testing and debugging of individual components, as well as optimizing and scaling the application as necessary.

Below is an example of how a Spark job can be implemented (extract of the file `spark_job.py`):
```python
class SparkJob(ABC):
    """An Abstract Class to describe how a Spark Job is composed."""

    def __init__(self, spark: SparkSession, parameters: dict[str, str], logger: Log4j):
        """Construct a new Spark Job."""
        self.spark = spark
        self.logger = logger
        self.parameters = parameters
        self.empty_dataframe: DataFrame = self.spark.createDataFrame([], StructType([]))
        self.datasets: dict[str, dict[str, str]] = {}
        super().__init__()

    @abstractmethod
    def start(self) -> dict[str, DataFrame]:
        """Run job initialization steps and reads the data from storage."""

    @abstractmethod
    def run(self, dataframes: dict[str, DataFrame]) -> dict[str, DataFrame]:
        """Run core processing logic on the data."""

    @abstractmethod
    def end(self, dataframes: dict[str, DataFrame]) -> None:
        """End the Spark job by running the data to storage."""

    def run_job(self) -> None:
        """Run the Spark Job."""
        start_result = self.start()
        processing_result = self.run(start_result)
        self.end(processing_result)


class DummyJob(SparkJob):
    """A Dummy Job to test Spark runs on the developper machine."""

    def start(self) -> dict[str, DataFrame]:
        """Create a small dataframe."""
        dataframe = self.spark.createDataFrame([("Scala", 25000), ("Spark", 35000), ("PHP", 21000)])
        return {"x": dataframe}

    def run(self, dataframes: dict[str, DataFrame]) -> dict[str, DataFrame]:
        """Do nothing."""
        return dataframes

    def end(self, dataframes: dict[str, DataFrame]) -> None:
        """Print the dataframe to the console."""
        dataframes["x"].show()
```

`DummyJob` implements the `SparkJob` abstract class. This abstract class gives a pattern to follow to write a Spark Job where IO operation (side effects) are located in the `start` and `end` methods. The `run` method contains the application of the core business logic. Moreover, as `run` takes DataFrames as inputs and outputs Dataframes, it should be easy to unit test. As `start` only reads data and `write` only write data, unit tests are less important on these methods.

## V. Git Repository and CI/CD

In order to maintain the source code of a project, it is typically kept in a git repository that is hosted on a platform such as Azure DevOps. This allows developers to collaborate on the project and keep track of changes made to the codebase over time.

One common development workflow is to create a new branch for each feature or bug fix that needs to be implemented. This allows developers to work on their changes independently without affecting the main/master branch until they are ready to merge their changes back into it.

When a developer has completed their work on a branch, they will typically create a pull request to ask for their changes to be reviewed and merged into the main branch. This allows other developers to review the code changes and provide feedback before they are merged in.

To keep the commit history on the main branch as clean as possible, it is recommended to squash commits before merging. This means that all of the individual commits made on the branch will be combined into a single, cohesive commit that describes the feature or logical change being made. Azure DevOps offers a convenient option for squashing commits during the merge process. This approach makes it easier to understand the history of changes made to the codebase and to roll back changes if necessary.

For code deployment, a CI/CD is used. On this project the pipeline has three stages: build, publish and deploy_dev. The build stage has one job that installs the required Python packages listed in requirements-dev.txt and requirements.txt files, runs static code analysis using **flake8, mypy, black, isort,** and generates test coverage using **coverage and pytest**. If the job is successful, the coverage results are published in a dedicated tab using Cobertura and xml.

The publish stage has one job creates a **Python wheel** using pip and publishes the wheel as an artifact. The job also authenticates twine to access the artifact feed and uploads the wheel to **Azure Artifacts**.

The deploy stage of the CI/CD pipeline is responsible for deploying the same code to each Databricks environment. This is achieved by using the version of the build, which is determined by either SemVer for the master branch or x.y.z.dev.n for a feature branch. The pipeline generates Databricks workflows by utilizing the code defined in the databricks_workflow folder, and replaces the current version number to be used. This allows developers to quickly test their work on the dev environment. It's worth noting that all Databricks workflows are defined as code, which means that no manual operations on Databricks should happen during the deployment process.


{{< mermaid >}}
graph TD
  subgraph Source Code Management
    A[Git] --> B[Commit to a feature branch or to master]
  end

  subgraph Build Stage
    B --> C[Install Python Dependencies]
    C --> D[Static analysis using flake8, black, mypy and isort]
    D --> EE[Run Pytest and publish coverage]
  end

subgraph Publish Stage
    EE --> FF[Build the Python wheel]
    FF --> GG[Upload to Azure Artifacts]
end

subgraph Deploy Stage
    GG --> E[Deploy to Dev]
    E --> F[Deploy to Qal]
    F --> G[Deploy to Prod]
end
{{< /mermaid >}}

## VI. Conclusion

Here is a recap list of the good practice worth following:

 - Organize the data in the bronze layer by partitioning it by ingest timestamp, using a format such as ISO 8601 to indicate the date of ingestion.

 - Maintain the raw data in the bronze layer as-is, without any modifications or transformations.

 - Use Databricks Autoloader as the ingestion engine and Delta Lake as the storage format.

 - Store the result tables in the silver layer, and name these tables to easily recognize them.

 - Use generic code for ingestion, which involves defining all datasets' metadata in a catalog.toml file, including the location, pattern, format, and sink table details. The code for ingestion then reads the metadata from the catalog.toml file and ingests all datasets without any modification to the code. This approach enables efficient and scalable ingestion of new datasets while reducing code maintenance and increasing code reuse.

 - Write Spark code using Python to enable easy unit testing and facilitate maintenance.
 - Use some kind of pattern in Spark jobs to facilitate jobs creation, unit tests and maintenance.

 - Host the source code in a git repository.
 - Create a new branch for each feature or bug fix that needs to be implemented. It allows developers to work on their changes independently without affecting the main/master branch.
 - Use CI/CD for code deployment.
 - In the CI/CD:
   - Use tools to enforce code standard (flake8, black, mypy, isort...)
   - Package your code and host it on Azure Artifact
   - Give your package a version number (preferably SemVer)
   - Let the CI/CD deploy your packaged code on the target environments.

## VII. Annex

### A. The Lakehouse architecture

As a reminder, the Lakehouse architecture is used on your company Dataplatform:

{{< mermaid >}}
graph TD
A[Data Sources] -->|Copy| B[Bronze]
B -->|Ingest| C[Silver]
C -->|Transform| C
C -->|Aggregate| D[Gold]

classDef bronze fill:#F5DEB3,stroke:#000,stroke-width:1px;
classDef silver fill:#C0C0C0,stroke:#000,stroke-width:1px;
classDef gold fill:#FFD700,stroke:#000,stroke-width:1px;

class B bronze;
class C silver;
class D gold;
{{< /mermaid >}}

 - Bronze: This layer represents the raw data that has been ingested from various data sources. The data in this layer is typically stored in its original format and may contain duplicates and errors. The goal of this layer is to collect and store all the data for further processing.

 - Silver: This layer represents the cleaned and transformed data. The data in this layer has been standardized, de-duplicated, and cleaned of any errors. The goal of this layer is to provide a reliable and consistent view of the data that can be used for analysis.

 - Gold: This layer represents the aggregated and enriched data that has been transformed into a format suitable for analysis. The data in this layer has been aggregated and enriched with additional information to support analysis and decision-making. The goal of this layer is to provide a comprehensive and accurate view of the data that can be used to drive business insights.

### B. Databricks Autoloader

One of the benefits of using Autoloader is that it automatically tracks which files have already been ingested using the same mechanism as Spark Streaming. This means that you can add new files in the bronze layer at any time, and Autoloader will only ingest the new files, without having to keep track of which files have already been processed. This makes it easy to keep your data up-to-date, and avoids the need for complex bookkeeping to manage ingested files.

To use the autoloader, simply use `self.spark.readStream.format("cloudFiles")` when reading raw data files. An example of how to implement to use the autoloader can be found in ingestion.py.

