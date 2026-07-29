# Complete the image metadata application

## Introduction

In this lab, you will complete the application by connecting the pages to `MLE_DATA`, displaying the extracted EXIF metadata, and using the MLE modules created in Lab 3.

The final application differs from the scaffold in three important ways:

- The pages use the scaffold's `MLE_DATA` table.
- Page 2 displays EXIF data, Google Gemini analysis, and the realness score.
- Page 3 uses `JSON_TABLE` to turn the JSON document in `MLE_DATA.EXIF_DATA` into report columns.

This lab requires Oracle AI Database 26ai, Oracle APEX 26.1, and the `MLE_EXIF_ENV` environment from Lab 3.

Estimated time to complete: 20 minutes

### Prerequisites

Complete [Lab 3](../lab-3/lab-3.md) first. The application uses the `MLE_DATA` table, including its `EXIF_DATA` JSON column:

```sql
select id, file_blob, exif_data
from mle_data;
```

The `EXIF_DATA` column is populated by the EXIF MLE module after an image is uploaded. Keep the table name `MLE_DATA` throughout this lab.

## Task 1: Use `MLE_DATA` on Page 1

The scaffold uses `MLE_DATA` as its source table. Keep this table name in all Page 1 components.

Open **Page 1: Photo Metadata** in Page Designer and make the following changes.

### Form region

Select the **Post** form region and set its table or view to `MLE_DATA`.

### Map region

Set the map region source to this SQL query:

```sql
select
    lat,
    lon,
    created_by as who,
    apex_util.get_since(created) as since
from mle_data
where lat is not null
  and lon is not null
```

### Cards region

Set the **Timeline** cards region source to this SQL query:

```sql
select
    p.id,
    p.created_by as user_name,
    p.post_comment as comment_text,
    p.file_blob,
    p.file_mime,
    apex_util.get_since(p.created) as post_date
from mle_data p
order by p.created desc
```

### Delete action

If the delete action is retained in the application, update its server-side code as follows:

```sql
delete from mle_data
 where id = :P1_ACTION_ID
   and created_by = :APP_USER;
```

Save Page 1 before continuing.

## Task 2: Update Page 2 and add the EXIF data region

Open **Page 2: Photo Metadata Details** in Page Designer.

### Photo item

Set the SQL source of `P2_PHOTO` to:

```sql
select file_blob
from mle_data
where id = :P2_ID
```

### Map region

Set the map layer table to `MLE_DATA` and keep the condition:

```sql
id = :P2_ID
```

### EXIF Data region

Create a report region named **EXIF Data**. Use the `EXIF_SOURCE` JSON Source created in Lab 2, filter it with `P2_ID`, and set the no-data message to:

```text
Photo does not contain EXIF data.
```

The JSON Source reads `MLE_DATA.EXIF_DATA`. No additional SQL query is required for this region.

### Google AI Analysis and Realness Score

Add the **Google AI Analysis** and **Realness Score** regions with the hidden items used by the application:

- `P2_AI_ASSESSMENT_JSON`
- `P2_AI_ASSESSMENT_POST_ID`
- `P2_AI_REASON`
- `P2_REALNESS_SCORE`

Configure `P2_REALNESS_SCORE` as follows:

- **Type**: `Display Only`
- **Format**: `Plain Text`
- **Based On**: `Item Value`
- **Show Line Breaks**: enabled
- **Send On Page Submit**: enabled
- **Region**: `Realness Score`
- **Sequence**: `10`
- **Slot**: `Region Body`
- **Column Span**: `12`

Leave the label empty. The MLE realness-score module writes the formatted value, such as `85% likely real` or `50% inconclusive`, into this item.

Configure `P2_AI_REASON` as follows:

- **Type**: `Display Only`
- **Format**: `HTML`
- **Send On Page Submit**: enabled
- **Region**: `Google Gemini AI Analysis`
- **Sequence**: `10`
- **Slot**: `Region Body`
- **Start New Row**: enabled
- **Column Span**: `12`

Leave the label empty. The MLE Gemini module writes the concise assessment reason into this item.

Create a Dynamic Action named `ai_assessment` on **Page Load**. The event must fire when the page loads and its client-side condition must be **None**, so the assessment runs for every image.

Add these two true actions in this order:

1. **Reset AI assessment**
   - Action: **Execute JavaScript Code**
   - Sequence: `5`
   - Clear the previous assessment before a new image is analyzed:

```javascript
apex.item("P2_AI_ASSESSMENT_POST_ID").setValue("");
apex.item("P2_AI_ASSESSMENT_JSON").setValue("");
apex.item("P2_AI_REASON").setValue("");
apex.item("P2_REALNESS_SCORE").setValue("Assessment pending");
```

2. **AI assessment server-side**
   - Action: **Execute Server-Side Code**
   - Sequence: `10`
   - Language: **JavaScript (MLE)**
   - Items to Submit: leave empty
   - Items to Return: `P2_AI_ASSESSMENT_POST_ID,P2_AI_ASSESSMENT_JSON,P2_AI_REASON,P2_REALNESS_SCORE`
   - **Show Processing**: enabled
   - **Stop Execution on Error**: enabled
   - **Wait for Result**: enabled

Use the following JavaScript code:

```javascript
const { analyzePhotoWithGemini } = await import('gemini-ai-verify-module');
const { getAssessmentReason, formatRealnessScore } =
    await import('realness-score-module');

const postId = apex.env.P2_ID;
const assessment = analyzePhotoWithGemini(postId);

apex.env.P2_AI_ASSESSMENT_POST_ID = postId;
apex.env.P2_AI_ASSESSMENT_JSON = assessment;
apex.env.P2_AI_REASON = getAssessmentReason(assessment);
apex.env.P2_REALNESS_SCORE = formatRealnessScore(
    assessment,
    postId,
    postId
);
```

Return the values to the page items listed above. The Gemini module performs the image analysis; the realness-score module formats the result for the APEX page.

Save Page 2 before continuing.

## Task 3: Create the metadata explorer page

Create or open **Page 3: Photo Metadata Explorer**. The scaffold does not contain the final metadata report. Add a **Photo Metadata Results** classic report with source type **SQL Query**.

Use `MLE_DATA` as the source table:

```sql
select
    id,
    make,
    model,
    lens_make,
    lens_model,
    focal_length,
    case
        when exposure_time is null then null
        when exposure_time >= 1 then to_char(round(exposure_time, 2)) || ' s'
        else '1/' || to_char(round(1 / exposure_time)) || ' s'
    end as exposure_time,
    case
        when aperture is not null then 'f/' || to_char(round(aperture, 1))
    end as aperture,
    iso,
    flash,
    latitude,
    longitude
from
    mle_data,
    json_table(
        mle_data.exif_data, '$'
        columns (
            make          varchar2(150) path '$.Make',
            model         varchar2(150) path '$.Model',
            lens_make     varchar2(150) path '$.LensMake',
            lens_model    varchar2(150) path '$.LensModel',
            focal_length  varchar2(150) path '$.FocalLength',
            exposure_time number        path '$.ExposureTime',
            aperture      number        path '$.FNumber',
            iso           number        path '$.ISO',
            flash         varchar2(150) path '$.Flash',
            latitude      number        path '$.latitude',
            longitude     number        path '$.longitude'
        )
    ) jt
```

`JSON_TABLE` converts the JSON document in `EXIF_DATA` into relational columns that APEX can display and filter. The `jt` identifier is only an alias for the `JSON_TABLE` result; it is not a table that needs to be created.

Add faceted search items for the columns used by the report, such as `MAKE`, `MODEL`, `LENS_MAKE`, `LENS_MODEL`, `FOCAL_LENGTH`, and `FLASH`. Save the page and run the application.

## Verify the application

Upload an image and confirm that:

- Page 1 stores the image in `MLE_DATA`.
- The EXIF MLE module writes metadata to `MLE_DATA.EXIF_DATA`.
- Page 2 displays the image, EXIF data, map, Gemini analysis, and realness score.
- Page 3 displays the EXIF attributes through `JSON_TABLE` and allows them to be filtered.

## Learn More

- [Oracle Database JavaScript Developer's Guide: Multilingual Engine](https://docs.oracle.com/en/database/oracle/oracle-database/26/mlejs/index.html)
- [Oracle Database JSON Developer's Guide: JSON_TABLE](https://docs.oracle.com/en/database/oracle/oracle-database/26/adjsn/json_table-sql-function.html)
- [APEX App Builder User's Guide: Page Designer](https://docs.oracle.com/en/database/oracle/apex/26.1/htmdb/page-designer.html)

## Acknowledgements

- **Author** - Sonja Meyer, Consulting Member of Technical Staff
- **Contributors** - Martin Bach, Senior Principal Product Manager
- **Last Updated By/Date** - Sonja Meyer, Consulting Member of Technical Staff, July 2026
