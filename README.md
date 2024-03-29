# Lab 1: Processing IoT Data using an Autonomous Transaction Processing Database

## Introduction

This lab walks you through the steps of using the Oracle Autonomous Transaction Processing Database to process data generated by IoT devices.

You will be building on top of the Customer Orders schema. You can use your favorite REST testing tool or the provided curl statements to simulate inventory items moving from shelf to shelf in a smart warehouse. The inventory data will be tracked in a JSON column (PRODUCT_DETAILS) in the PRODUCTS table.

To **log issues**, click [here](https://github.com/oracle/learning-library/issues/new) to go to the github Oracle repository issue submission form.

## Objectives

- Learn how to create an Oracle REST Data API to quickly add to and retrieve data from the Oracle Database.
- Learn how to work with JSON columns to handle data that may change over time.

## Required Artifacts

- The following lab requires an Oracle Public Cloud account. You may use your own cloud account, a cloud account that you obtained through a trial, or a training account whose details were given to you by an Oracle instructor.
- An Autonomous Transaction Processing Database Instance provisioned and running in your cloud account.
- Oracle SQL Developer installed on your computer.
- The Customer Orders schema installed and the connection open in SQL Developer. 

### Get the ORDS URI
Follow these steps to get the beginning of the URI from your ATP instance:
1. Log into your Oracle Cloud Dashboard.
2. From the menu, select ‘Autonomous Transaction Processing’.
![](./images/DashboardToAtp.png)
3. From the instance menu, select ‘Service Console’.
![](./images/serviceConsole1.png)
Or, Click on your Instance Name then Click the ‘Service Console’ button.
![](./images/serviceConsole2.png)
4. Click ‘Administration’.
![](./images/adminTab.png)
5. In the Rest Data Services box, click either button.
![](./images/apexOrSqlDev.png)
6. Copy from the beginning of the URL up to and including ‘/ords/’ Ignore the rest.
Save this to use in place of '\<YourURI>' when testing your API.
Example: https://someIdstuff-blaineords.adb.my-region-1.oraclecloudapps.com/ords

# Create a REST API to track inventory location data

In this section you will be creating an Oracle REST Data Service that will be used to track the current location of inventory items.

## **The JSON Object**

- In the column PRODUCTS.PRODUCT_DETAILS you will see a JSON Object similar to the following.
```
{
    "colour": "green",
    "gender": "Women's",
    "brand": "FLEETMIX",
    "description": "Excepteur anim adipisicing aliqua ad. Ex aliquip ad tempor cupidatat dolore ipsum ex anim Lorem aute amet.",
    "sizes": [
        0,
        2,
        4,
        6,
        8,
        10,
        12,
        14,
        16,
        18,
        20
    ],
    "reviews": [],
    "location": "shelf 2"
}
```

- You will add an inventory object to the existing JSON Object that will be similart to the following.
```
{
    "colour": "green",
    "gender": "Women's",
    "brand": "FLEETMIX",
    "description": "Excepteur anim adipisicing aliqua ad. Ex aliquip ad tempor cupidatat dolore ipsum ex anim Lorem aute amet.",
    "sizes": [
        0,
        2,
        4,
        6,
        8,
        10,
        12,
        14,
        16,
        18,
        20
    ],
    "reviews": [],
    "location": "shelf 2",
    "inventory": {
        "1": {
            "lastScanned": "03-MAY-19 07.07.46.877696 PM UTC",
            "location": "shelf 314"
        },
        "3": {
            "location": "shelf 4",
            "lastScanned": "03-MAY-19 07.15.00.014329 PM UTC"
        },
        "18": {
            "location": "shelf 5",
            "lastScanned": "03-MAY-19 07.42.37.320719 PM UTC"
        }
    }
}
```
The inventory objects will use the item's id as the Key and the value will be a JSON Object tracking the last time and location the item was scanned.

Using the inventory id as the key will allow you to look up an item quickly using just it's id without having to parse an array.
```
inventory.1 = {
            "lastScanned": "03-MAY-19 07.07.46.877696 PM UTC",
            "location": "shelf 314"
        }
```

You may be thinking, "why don't we just create a separate table for this data?". That is a perfectly fine solution as well, this example is intended to demonstrate the power of using JSON in the Oracle Database. You can use ORDS for either scenario.


## Steps

### **STEP 1: REST enable the Customer Orders schema**

- In SQL Developer
1. Right click on the Customer Orders connection.
2. Click on **REST Services**.
3. Click on **Enable REST Services**.

![](./images/RestEnableSchema.png)

- In the **Specify Details** window
1. Check **Enable schema**.
2. Leave **Schema alias** set to **co**.
3. Un-Check **Authorization required**.
4. Click **Next**.

![](./images/RestEnableSchemaDetails.png)

- In the **RESTful Summary** window review the details.
You can also switch to the **SQL** tab to review the SQL that will be executed.

![](./images/RestEnableSchemaSummary.png)
- Click **Finish**.

![](./images/Successful.png)

### **STEP 2: Create a REST Module**

- Expand **REST Data Services**

- Right click on **Modules**

- Click on **New Module**

![](./images/NewModule.png)

- In the **Specify Module** window enter the following:
1. Module Name: ```Products```.
2. URI Prefix: ```products```.
Notice that the **Example:** URI changes to match this entry.
3. Check Publish.
4. Click Next.

![](./images/SpecifyModule.png)

- In the **Specify Template** window enter the following:
1. URI Pattern: {productId}/inventory/{inventoryId}
The curly braces designate those sections of the URI as parameters that will be mapped as bind variables in the following steps.
2. Click **Next**.

![](./images/SpecifyTemplate.png)

- In the **RESTful Summary** window review the details.
You can also switch to the **SQL** tab to review the SQL that will be executed.

![](./images/NewModuleSummary.png)
- Click **Finish**.

![](./images/Successful.png)

### **STEP 3: Create a PUT handler to accept incoming data**

- Right click on the **REST Template**
- Click on **Add Handler**
- Click on **PUT**

![](./images/AddPutHandler.png)

- Click **Apply**

![](./images/ApplyPutHandler.png)

- A SQL Worksheet is opened, add the following PL/SQL.
This will create a JSON object for the incoming inventory item and replace the existing entry or add it to inventory if it's new.

```
BEGIN
  UPDATE products p SET
  p.product_details = json_mergepatch (p.product_details ,
                                       JSON_OBJECT (KEY 'inventory' is
                                                    JSON_OBJECT (KEY to_char(:inventoryId) is
                                                                 JSON_OBJECT (KEY 'location' value nvl(to_char(:location), 'null') ,
                                                                              KEY 'lastScanned' value to_char(current_timestamp)
                                                                             )
                                                                )
                                                   )
                                      )
  WHERE p.product_id = :productId
  RETURNING p.product_details.inventory INTO :inventory;
END;
```

![](./images/PutPlsql.png)

- Click on the **Parameters** tab.
- Add the following parameters.

| Name | Bind Parameter | Access Method | Source Type | Data Type
| ------------- |-------------| -----| -----| -----
| inventoryId | inventoryId | IN | URI | STRING
| productId | productId | IN | URI | INTEGER
| {}inventory | inventory | OUT | RESPONSE | STRING

![](./images/PutParams.png)

- Click on the **Details** tab.
- Note the highlighted portion of the URI in the image. This part of the URI will be used to consume the REST service.

![](./images/PutDetails.png)

- Click the **Save** icon

![](./images/PutSave.png)

- **Test It**

Using the URI you saved from above, Use your favorite REST testing tool, use the following CURL statement. Replace '\<YourURI>' with the URI you saved from above.
```
curl -X PUT \
  <YourURI>/co/products/2/inventory/1x3g \
  -H 'Content-Type: application/json' \
  -d '{"location": "shelf 314"}'
```
You should receive the following response.
```
{
    "inventory": {
        "1x3g": {
            "location": "shelf 314",
            "lastScanned": "03-JUL-19 05.40.21.825698 PM UTC"
        }
    }
}
```
### **STEP 4: Add a helper function**
Create a PL/SQL function that will be used to get an inventory item from the PRODUCTS.PRODUCT_DETAILS JSON object, using a dynamic key value.

- Right click on the connection.
- Click on **Open SQL Worksheet**

![](./images/OpenSqlWorksheet.png)

- Add the following PL/SQL.

```
CREATE OR REPLACE FUNCTION getvalbykey (
   json VARCHAR2,
   key VARCHAR2
) RETURN VARCHAR2 IS
   o json_object_t;
   v VARCHAR2 (4000);
BEGIN
   o := json_object_t(json);
   o := o.get_object(key);
   v := o.to_string ;
   RETURN v;
END getvalbykey;
```
-  Click the **Run Statement** icon.

![](./images/getvalbykeyFunc.png)

### **STEP 5: Add a GET handler**

- Right click on the **REST Template**
- Expand **Add Handler**
- Click on **GET**

![](./images/AddGetHandler.png)

- Click **Apply**

![](./images/ApplyGetHandler.png)

- A SQL Worksheet is opened, add the following SQL.
  Note: The column alias for inventoryItem is prefixed with '{}'. This will cause ORDS to return the value as JSON and not an escaped string.
```
select p.product_id,
       :inventoryId "inventoryId",
       getValByKey (p.product_details.inventory, :inventoryId) "{}inventoryItem"
from products p
where p.product_id = :productId
```

![](./images/GetSql.png)

- Click on the Parameters tab.
- Add the following parameters.

| Name | Bind Parameter | Access Method | Source Type | Data Type
| ------------- |-------------| -----| -----| -----
| inventoryId | inventoryId | IN | URI | STRING
| productId | productId | IN | URI | INTEGER

![](./images/GetParams.png)

- Click on the **Details** tab.
- Note the highlighted portion of the URI in the image. This part of the URI will be used to consume the REST service.

![](./images/GetDetails.png)

- Click the **Save** icon

![](./images/GetSave.png)

- **Test It**

Using the URI you saved from above, Use your favorite REST testing tool, use the following CURL statement. Replace '\<YourURI>' with the URI you saved from above.
```
curl -X GET <YourURI>/co/products/2/inventory/1x3g
```
You should receive the following response.
```
{
    "items": [
        {
            "product_id": 1,
            "inventoryid": "1x3g",
            "inventoryitem": {
                "location": "shelf 314",
                "lastScanned": "03-JUL-19 05.40.21.825698 PM UTC"
            }
        }
    ],
    "hasMore": false,
    "limit": 25,
    "offset": 0,
    "count": 1,
    "links": [
        {
            "rel": "self",
            "href": "<YourURI>/co/products/2/inventory/1x3g"
        },
        {
            "rel": "edit",
            "href": "<YourURI>/co/products/2/inventory/1x3g"
        },
        {
            "rel": "describedby",
            "href": "<YourURI>/co/metadata-catalog/products/2/inventory/item"
        },
        {
            "rel": "first",
            "href": "<YourURI>/co/products/2/inventory/1x3g"
        }
    ]
}
```

### **STEP 5: Add a DELETE handler**

- Right click on the **REST Template**
- Expand **Add Handler**
- Click on **DELETE**

![](./images/AddDeleteHandler.png)

- Click **Apply**

![](./images/ApplyDeleteHandler.png)

- A SQL Worksheet is opened, add the following PL/SQL.
Updating the inventory object to **is null** will remove the item from the JSON object. Setting it to an empty string would leave the item with a value of "".

```
BEGIN
  UPDATE products p SET
  p.product_details = json_mergepatch (p.product_details ,
                                       JSON_OBJECT (KEY 'inventory' is
                                                    JSON_OBJECT (KEY to_char(:inventoryId) is null
                                                                )
                                                   )
                                      )
  WHERE p.product_id = :productId
  RETURNING p.product_details.inventory INTO :inventory;
END;
```

![](./images/DeletePlsql.png)

- Click on the Parameters tab.
- Add the following parameters.

| Name | Bind Parameter | Access Method | Source Type | Data Type
| ---- |----------------| --------------| ------------| ---------
| inventoryId | inventoryId | IN | URI | STRING
| productId | productId | IN | URI | INTEGER
| {}inventory | inventory | OUT | RESPONSE | STRING

![](./images/DeleteParams.png)

- Click on the **Details** tab.
- Note the highlighted portion of the URI in the image. This part of the URI will be used to consume the REST service.

![](./images/DeleteDetails.png)

- Click the **Save** icon

![](./images/DeleteSave.png)

- **Test It**

Using the URI you saved from above, Use your favorite REST testing tool, use the following CURL statement. Replace '\<YourURI>' with the URI you saved from above.
```
curl -X DELETE <YourURI>/pdb1/co/products/2/inventory/1x3g
```
You should receive the following response.
```
{
    "inventory": {}
}
```

-  

## Great Work - All Done!