# Welcome to the Timestamp Readme!

# Background

Adding time variables to all relations will help long-term data management, especially beneficial when we will be having data updated. Time stamp attribute on relations will enable us to think of further analysis such as time series. 

## Why we use standardized columns?

Many raw columns often have different names even though they have shared meanings. Therefore, we decide to create time standardized columns such that we can group raw columns that have standard meanings across data source into time standardized columns and have a clear understanding of each column. To achieve this, we use user-defined-functions(UDFs) to have shared data type(4-digit year information). All standardized columns live [here](https://docs.google.com/spreadsheets/d/10CFfUEibINVj2KMbe7iqWzDyLXVX2am1g2usCgZ2ZfY/edit#gid=0).

## Time Standardized Columns
We can find `record_year`, `source_load_year`, `bl_collect_year`, `termination_year` often regardless of the source category. However, some standardized columns can be found in  particular categories: `reg_date`, `signed_date` in lobbyists, `file_year`, `transaction_date` in contributions. 

- **record_year** : 
Time stamp of each row regarding the main purpose of the data. Aka, recorded year of the data. Therefore, each row can be different. This always come from raw columns (Bluelabs does not create record_year)

- **source_load_year** : 
Year when they(authorities) load data. It is very common to see that the same year is applied to the entire rows of data since the authority is likely to load the whole dataset at once. 

- **bl_collect_year** : 
Year when we(Bluelabs) acquire data. This information is obtained via previous Asana task and date last pulled/refreshed section of each source on [Wikipage](https://github.com/bluelabsio/knowledge/wiki/Influencers)

- **reg_date** : 
When lobbyists register their lobby activity or themselves as lobbyists in lobbyists table 

- **signed_date** : 
Year when lobby is signed in lobbyists table 

- **file_year** : 
Year when individuals file transaction (to state) in contribution tables

- **transaction_date** : 
Transaction year of contribution in contribution tables

- **reappointed_year** : 
Year when an individual gets reappointed to serve on organization such as committee.

- **graduation_year** : 
Year when an individual graduate from school. 

- **termination_year** : 
Year when individuals terminate their activities. Raw columns with clear representation of termination will be standardized as termination_year.

Example to understand the difference among record_year, source_load_year and bl_collect_year.
Let’s take a look at `ar_staffers` as an example.

| Time Standardized Column  |  record_year |  source_load_year |  bl_collect_year | 
| ------------- | ------------- | ------------- | ------------- |
| Easy Definition | Time stamp of each row of the data  | When they(authority) loaded data | When BL collected data |
| Data will be the same? different?  | Each row can be different  | All data will likely be the same | All data will be the same |
| Which raw column to take? | year  | load_date | 2018::float |

## Update Yaml Configuration File

Now, we update `ar_staffers` configuration file by adding the following time standardized columns under transform_columns as follows. 

```yaml
tranform_columns:
  record_year: year
  source_load_year: load_date
  bl_collect_year: 2018::float
```

# Work

Identify the start and end year of employment between person and organization relation. 
The bottom line is to select the “best” possible start year whereas to leave the end year as `null` unless it clearly represents the termination of the relationship. We would rather leave end year as `null`, because we do not want to provide wrong/biased information on termination of employment. 

### Work Relations
- Person_work_other_house_office
- Person_work_other_senate_office
- Person_work_rep_office (federal, state)
- Person_work_senator_office (federal, state)
- Person_work_employer
- Person_work_department
- Person_work_governor_office

## Source Category

We note that sources have different priorities when selecting the start and end year of work relation. Within the same category, source tables have the same priority list of columns, and the logic is generated by the characteristics of source as follows. 

- `staffers` : Source ending with _staffers _e.g., ar_staffers_
- `lobbyists` : Source ending with _lobbyists _e.g., ak_lobbyists_
- `contributions` : Source ending with _contributions  _e.g., ca_contributions_
- `others` : Source not ending with staffers, lobbyists, contributions are others. e.g., `forty_under_forty` is in business, `nva_transportation_officials` in officials, `union_employees` in others -- but we don’t specify others’ category.

We have 2 comprehensive functions(`work_t1` and `work_t2`) that select start and end year of staffers, lobbyists, contributions, business/officials/others accordingly. 

## Priority Column List by Source Category

1. **Staffers**
    - Start Year
      1. `hire_year`
      2. `record_year`
      3. `source_load_year`
      4. `bl_collect_year`
    - End Year
      1. `termination_year`
      
      If `retire` = 1, `record_year` will be the closest indicator of end year.  
      If `still_employed` is `null`, `record_year` will be the closest indicator of end year.

2. **Lobbyist**
    - Start Year
      1. `hire_year`
      2. `MAX(report_year, reg_date, signed_date)` if two or more columns exist
      3. `report_year`
      4. `reg_date`
      5. `signed_date`
      6. `record_year`
      7. `source_load_year`
      8. `bl_collect_year`
    - End Year is `null` since we don't have any columns that represent the termination of employment
      

3. **Contributions**
    - Start Year
      1. `hire_year`
      2. `record_year` case except `retire` = 1 
      3. `MIN(file_year, transaction_date)` case except `retire` = 1
      4. `file_year` case except `retire` = 1
      5. `transaction_date` case except `retire` = 1
      6. `source_load_year` case except `retire` = 1
      7. `bl_collect_year` case except `retire` = 1
    - End Year
      1. `coalesce(termination_year, record_year, transaction_date)` case when `retire` = 1
      
      If `retire` != 1, end year is `null`
      
4. **Others**
    - Start Year
      1. `hire_year`
      2. `record_year`
      3. `source_load_year`
      4. `bl_collect_year`
    - End Year is `null`

# Serves On

Identify the start and end year of `serves_on` relationship between person and organization. Unlike `work`, `serves_on` does not use source category, because we have fewer sources, and each has different relations. Therefore, `serves_on` seems relatively simple but needs to incorporate special conditions by individual source. Let’s take a look at `whitehouse_staffers`. Joshua Friedman, who served Obama as a deputy associate counsel, has a termination year at least in 2017 since we clearly know that Obama’s presidency ended in 2017. However, we are going to leave `null` for Trump staffers unless there is any clear indication of termination. 

### Serves On Relations
- Cabinet_secretary_serves_on_cabinet
- Person_serves_on_advisory_board (federal, state)
- Person_serves_on_department
- Person_serves_on_whitehouse
- Person_serves_on_cabinet

# Alumni

Unlike other relations, we do not have termination year(T2) of alumni, so this is going to be more like a single time label. You will become an alumnus once you graduate and this relation stays forever. If John Smith graduated from Georgetown University in 2018, then John became an alumnus of GU from 2018. 

We use *graduation_year* column, which is already in our column standardization list. However, we often see individuals who hold multiple degrees, and a single timestamp would not be able to handle them. To solve this, we need to standardize `degree` column such that we take control of multiple degree holders. 

Example
```yaml
  transform_columns:
    graduation_year: grad_year
    degree: CASE WHEN degree_type = 'Associate' THEN 'associate' WHEN degree_type = 'Bachelor' THEN 'bachelor' WHEN degree_type = 'Business' OR degree_type = 'Graduate' THEN 'graduate' WHEN degree_type = 'PhD' THEN 'phd' WHEN degree_type = 'Medical' THEN 'md' WHEN degree_type = 'Law' THEN 'law' WHEN degree_type = 'Other' THEN 'other' END
```

## Applying functions
To apply the right, corresponding function to a source table, we have multiple validation steps before applying functions to add start and end year of person_work_employer relation. 
Let’s suppose we are trying to add start and end year of person_work_employer relation to `ar_staffers`. 
- First, we get the full source table list of person_work_employer relation by running the query. 
- Second, check if `ar_staffers` is in person_work_employer relation list. If this returns `True`, proceed to step 3. 
- In step 3, find which category `ar_staffers` belongs to among staffers/lobbyists/contributions/business/officials/others. `ar_staffers` is in staffers category as the source name ends with _staffers. 
- In step 4, function for staffers’ start year and one for staffers’ end year that we generated earlier will be applied to `ar_staffers`, return the corresponding 4-digit numeric start and end year as strings. 
- In step 5, generate a **local temporary table** that includes start and end year and replace the original `ar_staffers` table with the temporary table. Now, we will have updated `ar_staffers` which contains start_year_person_work_employer and end_year_person_work_employer on top of everything!


