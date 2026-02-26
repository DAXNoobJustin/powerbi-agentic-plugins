# [Spec name]

**Version**: [Version]
**Date**: [Date]
**Author**: [Author]

## Overview
<!--
  ACTION REQUIRED: Define High-level summary of the project. It should briefly explain the project intent and goal.

  Example: "This document specifies a Microsoft Fabric analytics solution that ingests sample sales data from CSV files into a Lakehouse, creates a semantic model for self-service business intelligence, and provides advanced analytics capabilities through notebooks."
-->

## Requirements

<!--
  ACTION REQUIRED: 
  - Transform vague feature ideas into concrete, measurable requirements
  - Create a shared understanding between stakeholders
  - For each requirement create a user story and acceptance criteria following EARS notation  
  - Use the requirement below as example
  - Each requirement and acceptance criteria must be numbered

  Example of requirements:

    ### Requirement 1: Lakehouse Setup

    **User Story:** As a data engineer, I want to create a lakehouse and upload sample data files, so that I have the infrastructure ready for data ingestion.

    #### Acceptance Criteria

    1. THE System SHALL create a lakehouse in the workspace
    2. THE System SHALL upload all CSV files from the sample-data directory to the lakehouse Files section
    3. WHEN uploading files, THE System SHALL preserve the original CSV file names
    4. WHEN the lakehouse setup completes, THE System SHALL verify that all 6 CSV files are accessible in the Files section

    ### Requirement 2: Lakehouse Tables

    **User Story:** As a data engineer, I want to load all CSV files into the lakehouse as tables, so that the data is available for analysis and reporting.

    #### Acceptance Criteria

    1. THE System SHALL create a corresponding Delta table in the Tables section for each CSV file
    2. THE System SHALL name each table based on the CSV filename (without the .csv extension) using business friendly names (e.g. sales instead of fact_sales)
    3. THE System SHALL preserve all columns from the source files
    4. THE System SHALL infer and apply appropriate data types for each column
    6. WHEN all CSV files are processed, THE System SHALL have created 6 Delta tables (5 dimension tables and 1 fact table)

-->

## Design

<!--
  ACTION REQUIRED: Summarized description of the project design with main components. This section document the technical architecture, sequence diagrams, and implementation considerations. Captures the big picture of how the project will work, including the components and their interactions.

  Example:
    This design implements a Microsoft Fabric lakehouse analytics solution following a star schema data warehouse pattern. The solution consists of four main components:

      - **Lakehouse Infrastructure**: A Fabric lakehouse that stores raw CSV files and Delta tables
      - **Data Loading**: Automated loading of CSV files into Delta tables using Fabric's Load table APIs
      - **Semantic Model**: A Power BI semantic model with relationships, measures, and time intelligence
      - **Analytics Notebook**: A PySpark notebook for advanced analytics and exploratory data analysis
-->

### Architecture

<!--
  ACTION REQUIRED: Draw a simple architecture diagram using mermaid syntax.

  - Focus on the big picture, major components and connections between them
  - Try to display the data flow of the solution from data sources (upstream) to consumption reports (downstream)
  - Use appropriate shapes and colors to distinguish between layers
-->

### Components and Interfaces

<!--
  ACTION REQUIRED: Considering the requirements and data sources plan for which Fabric items must be created or modified.

  For each component, provide structured subsections:
  - Key Features: Bulleted list of capabilities
  - Tables/Objects to Create: Specific naming with source mapping
  - Relationships: Explicit definitions with cardinality
  - Measures/Calculations: Formula specifications
  - Code Sections: Structured outline for notebooks/pipelines
  - Formats to be used, for example use PySpark for notebook cells
  - Do not include extensive implementation details. Keep it high-level the implementation agent should handle the details according to its own development guidelines.

  Example of a component section:

    #### Semantic Model Component

    **Purpose**: Defines business logic, relationships, and measures for Ad-hoc reporting.

    **Responsibilities**:
    - Create a semantic model connected to the lakehouse tables
    - Use Direct Lake storage mode
    - Define star schema relationships between fact and dimension tables
    - Create calculated measures for business metrics
    - Configure time intelligence using the date dimension

    **Relationships**:
    The semantic model implements a star schema with the following relationships:

    ```
    fact_sale (many) → dimension_city (one) via CityKey
    fact_sale (many) → dimension_customer (one) via CustomerKey
    fact_sale (many) → dimension_date (one) via InvoiceDateKey
    fact_sale (many) → dimension_employee (one) via SalespersonKey
    fact_sale (many) → dimension_stock_item (one) via StockItemKey
    ```
    **Measures**:

    1. **Total Sales**:
      ```dax
      Sales = SUMX(fact_sale, fact_sale[Quantity] * fact_sale[UnitPrice])
      ```
      Calculates the sum of all sales amounts across any dimension slice.

    2. **Profit Margin**:
      ```dax
      Profit = SUM(fact_sale[Profit])

      Profit Margin = DIVIDE([Profit], [Total Sales])
      ```
      Calculates profit as a percentage of total sales, handling division by zero.

    **Date Dimension Configuration**:
    - Mark `dimension_date` as a date table using the `Date` column
    - This enables time intelligence functions like SAMEPERIODLASTYEAR, DATEADD, etc.
-->

### Data Sources

<!--
  ACTION REQUIRED: Analyze the project data sources and detail each one as individual sections.
  - When access to data source is not possible, stop and ask user for guidance. 
  - Do not guess data sources or schemas
  - For each data source section:
    - Include expected schema information: column names, data types, constraints
    - Document record counts if known
    - Identify primary keys, foreign keys, and relationships
    - Note any special attributes (SCD2, nullable columns, default values)
  
  Example of data source section:

    #### CSV files

    Based on typical star schema sales data, the expected schemas are:

    **dimension_city**:
    - CityKey (int) - Primary key
    - City (string)
    - StateProvince (string)
    - Country (string)
    - Region (string)

    **dimension_customer**:
    - CustomerKey (int) - Primary key
    - CustomerName (string)
    - BillToCustomer (string)
    - Category (string)
    - BuyingGroup (string)
    - PrimaryContact (string)

    **dimension_date**:
    - DateKey (int) - Primary key (format: YYYYMMDD)
    - Date (date)
    - Day (int)
    - Month (int)
    - MonthName (string)
    - Quarter (int)
    - Year (int)
    - DayOfWeek (int)
    - DayName (string)   

    **fact_sale**:
    - SaleKey (int) - Primary key
    - CityKey (int) - Foreign key to dimension_city    
    - InvoiceDateKey (int) - Foreign key to dimension_date
    - SalespersonKey (int) - Foreign key to dimension_employee    
    - Quantity (int)
    - UnitPrice (decimal)
    - TotalAmount (decimal)
    - Profit (decimal)
-->


## Tasks

<!--
  ACTION REQUIRED: Elaborate an implementation task plan of the project and break it down into multiple sequential phases. 
  - Each task should have its own task section with sub-tasks numbered
  - Focus on concrete activities
  - Include a "**Notes**" section before the each task section with list of important notes related to the tasks such as variables for workspace names and item names
  - Each task should produce working, testable output
  - Tasks should build incrementally on previous work
  - Each task references specific requirements for traceability
  - Avoid tasks that can't be completed by a coding agent 

    Example of Task section:

    ### 1. Setup lakehouse and upload CSV files

    - [ ] 1.1 Create Lakehouse
      - Create a new Lakehouse named `Lakehouse_Name` in the workspace
      - Requirements: 1.1, 1.2

    - [ ] 1.1 Upload CSV files
      - Upload all 6 CSV files from sample-data directory to the lakehouse Files section
      - Verify all files are accessible in the Files section
      - Requirements: 1.3, 1.4, 1.5
-->


