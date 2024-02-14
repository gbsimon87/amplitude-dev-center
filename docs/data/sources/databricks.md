---
title: Databricks Import
---

Amplitude's Databricks import source enables you to import data from Databricks to your Amplitude account. Databricks import uses the Databricks Change Data Feed feature to securely access and extract live data from your Databricks workspace.

!!! alpha
    Amplitude provides this feature as a preview, and may change or update the feature without prior notice.

## Features

- Import all data types, including events, user properties, and group properties.
- Support for delta sync, to ensure Amplitude imports only new or changed data.
  - For **event** data, Amplitude imports rows with an `insert` operation.
  - For **user properties** and **group properties**, Amplitude imports rows with `insert` or `update` operations.
  - Amplitude ignores rows with `delete` operations for all data types.
- Create joins to import data from multiple tables.

## Restrictions

- Databricks import doesn't support data well...
- User stream view isn't available for data ingested with this feature.
- End-to-end latency (from when Amplitude fetches the data, to when you can query it) is between 30 and 60 minutes. Actual latency can vary depending on the workload.

## Configure Databricks

Before you start to configure the Databricks source in Amplitude, complete the following tasks in Databricks.

### Create a service principal

1. Follow the Databricks instructions to create a service principal in [Databricks | OAuth machine-to-machine (M2M) authentication](https://docs.databricks.com/en/dev-tools/auth/oauth-m2m.html). Copy the **UUID** for use in a later step.
2. Generate a **client id** and **secret** and copy both values to use in a later step.

### Find or create an all-purpose compute cluster

Amplitude creates workflows in this cluster on your behalf to start sync jobs. When complete, copy the **Server hostname** and **Http path** values to use in a later step. Find both values on the **Configuration -> JDBC/ODBC** tab. For more information, see [Databricks | Create a cluster](https://docs.databricks.com/en/dev-tools/auth/oauth-m2m.html)

### Grant permissions to the service principal

The service principal you created above requires the following permissions in Databricks:

| Permission | Reason                                                                               | Location in Databricks                                         |
| ---------- | ------------------------------------------------------------------------------------ | -------------------------------------------------------------- |
| Workspace  | Grants access to your Databricks workspace.                                          | *Workspace → <workspace_name> → Permissions → Add permissions* |
| Table      | Grants access to list tables and read data.                                          | *Catalog → pick the catalog→ Permissions → Grant*              |
| Cluster    | Grants access to connect to the cluster and run workflows on your behalf             | *Compute → All-purpose compute → Edit Permission*              |
| Export     | Enables the service principal to unload your data through spark and export it to S3. | Run the SQL commands below in any notebook.                                    |

```sql title="Databricks Export permission commands"
GRANT MODIFY ON ANY FILE TO `<service_principal_uuid>`;
GRANT SELECT ON ANY FILE TO `<service_principal_uuid>`;
```

### Enable CDF on your table(s)

Amplitude uses Databricks' Change Data Feed to continuously import data. To enable CDF on a Databricks table, see [Databricks | Enable change data feed](https://docs.databricks.com/en/delta/delta-change-data-feed.html#enable-change-data-feed)

## Configure the Amplitude Databricks source

To add Databricks as a source in Amplitude, complete the following steps.

### Connect to Databricks

1. In Amplitude Data, click **Catalog** and navigate to the **Sources** tab.
2. Search for **Databricks**.
3. On the **Credentials** tab of the Connect Databricks screen, enter the credentials you configured during the Databricks configuration:
    - Server Hostname
    - HTTP Path
    - Service Principal Client ID
    - Service Principal Secret
4. Click **Next** to verify access.

### Select data to import

1. Select the Databricks table(s) from which Amplitude pulls data.
2. Select the version of the table(s) that Amplitude imports from. Select **First** or **Latest**.
3. Select the data type to import from the selected table(s). The Databricks source supports three data types.
    - Event
    - User properties
    - Group properties
    
    For the **Event** data type, optionally select **Sync User Properties** or **Sync Group Properties** to sync the selected data types *within* an event.

4. Set the sync frequency. This frequency determines the interval at which Amplitude pulls data from Databricks.
5. Configure the SQL command that transforms data in Databricks before Amplitude imports it.
    - Amplitude treats each record in the SQL execution as an event to be import. See the Example body in the [Batch Event Upload API](/analytics/apis/batch-event-upload-api/#example-body) documentation to ensure each record you import complies.
    - Amplitude can transform / import from only the tables you specify in step 1 above.
    - The table names you reference in the SQL command must match exactly the name of the table you select in step 1. For example, if you select `catalog.schema.table1`, use that exact value in the SQL.

    ```sql title="Sample SQL command"
    select 
        unix_millis(current_timestamp())                                       as time,
        id                                                                     as user_id,
        "demo"                                                                 as event_type,
        named_struct('name', name, 'home', home, 'age', age, 'income', income) as user_properties,
        named_struct('group_type1', ARRAY("group_A", "group_B"))               as groups,
        named_struct('group_property', "group_property_value")                 as group_properties
    from catalog.schema.table1;
    ```

6. After you add the SQL, click **Test SQL**. Amplitude runs a test against your Databricks instance to ensure the SQL is valid. Click **Next**.
7. Enter a descriptive name for this instance of the source.
8. The source appears in your workspace's Sources list.

## Verify data import

Events that Amplitude imports assume the name you assign in your SQL statement. In the example above, the events have the name `demo`

To verify the data coming into Amplitude:

- Navigate to the **Performance** tab of the source instance
- View the Events page of your Tracking Plan
- Create a Segmentation chart that filters on the event name you specify. 