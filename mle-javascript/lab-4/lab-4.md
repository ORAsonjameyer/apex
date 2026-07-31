# Complete the image metadata application

## Introduction

In this lab, you will complete the application by connecting the pages to `MLE_DATA`, displaying the extracted EXIF metadata, and using the MLE modules created in Lab 3.

The final application will differ from the scaffold in three important ways:

- All APEX pages use the scaffold's updated and enhanced `MLE_DATA` table.
- Processing is in largest parts performed in MLE/JavaScript
- Page 2 displays
    - EXIF data based on a JSON Source
    - Google Gemini analysis (or your own model's conclusion)
    - A calculated _realness_ score

This lab requires Oracle AI Database 26ai, Oracle APEX 26.1, and the `MLE_EXIF_ENV` environment plus all the MLE modules created in the previous lab.

Estimated time to complete: 20 minutes

### Prerequisites

Complete [Lab 3](../lab-3/lab-3.md) first. The application uses the `MLE_DATA` table, including its `EXIF_DATA` JSON column.

The `EXIF_DATA` column is populated by the EXIF MLE module after an image is uploaded. Keep the table name `MLE_DATA` throughout this lab.

### Objectives

In this lab, you will:

- Finish all APEX pages
- Add logic to extract EXIF data to page 1
- Complete the image detail page by passing it to Gemini for an assessment and displaying the location where the photo was taken

## Task 1: Update Page 1 and embed EXIF Data and the AI Assessment

The scaffold uses `MLE_DATA` as its source table. Keep this table name in all Page 1 components.

Open Page 1: _Photo Metadata_ in Page Designer and make the following changes.

1. **Form region**

    Select the **Post** form region and set its table or view to `MLE_DATA` if not done so already.

1. **Map region**

    Set the map region (**Post Locations**) source to ensure this SQL query is used:

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

1. **Cards region**

    Set the **Timeline** cards region source to this SQL query should it differ:

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

1. **Delete action**

    If the delete action (_Dynamic Actions_ > _Custom_ > _action_delete_ > _DELETE - do database work_) is retained in the application, ensure its server-side code looks as follows:

    ```sql
    delete from mle_data
    where id = :P1_ACTION_ID
    and created_by = :APP_USER;
    ```

1. **EXIF Data Extraction**

    You created a JavaScript module `MLE_EXIF_HELPER_MODULE` in the previous lab. It's sole exported function, `extractAndSaveEXIF` takes an image ID and extracts the EXIF data, validates it, and stores it in `MLE_DATA.EXIF_DATA`. Only a subset of all available EXIF fields is stored to keep the example reasonably simple.

    Switch to the **Processes** pane and add the following process _after_ insert post. Keep the defaults, except for the following:

    - **Name**: Extract validate and store EXIF data
    - **Source**/**Language**: JavaScript (MLE)
    - **Source**/**JavaScript Code**

        ```javascript
        <copy>
        const { extractAndSaveEXIF } = await import ('exifr-helper');
        extractAndSaveEXIF(apex.env.P1_ID);
        </copy>
        ```

    - **Success Message**: EXIF Data successfully stored in the database
    - **Error Message**: Something went wrong extracting/storing the EXIF information

Save Page 1 before continuing.

## Task 2: Update Page 2 and add the EXIF data region

You are going to complete the design and code for the second APEX page in this task. Start by opening Page 2: _Photo Metadata Details_ in Page Designer.

1. **Photo item**

    Ensure the SQL source of `P2_PHOTO` is set to:

    ```sql
    select file_blob
    from mle_data
    where id = :P2_ID
    ```

1. **Map region**

    Confirm the map layer's table attribute is set to `MLE_DATA` and features the following where condition:

    ```sql
    id = :P2_ID
    ```

1. **Realness Score**

    This region displays the "realness score". It is _not_ a forensic number, but rather an approximation whether the AI service (Google Gemini by default) considers the photo AI generated, or not. Define the region as follows:

    - **Identification**
        - Name: Realness Score
        - Type: static content

    Place the region above the existing _Summary_ region

    Add a page item named `P2_REALNESS_SCORE` in the region's body, with the following properties

    - **Type**: `Display Only`
    - **Format**: `Plain Text`
    - **Based On**: `Item Value`
    - **Show Line Breaks**: enabled
    - **Send On Page Submit**: enabled
    - **Region**: `Realness Score`
    - **Sequence**: `10`
    - **Slot**: `Region Body`
    - **Column Span**: `12`

1. **Google AI Analysis**

    Add the **Google AI Analysis** region as a static item.

    Leave the label empty. The MLE realness-score module writes the formatted value, such as `85% likely real` or `50% inconclusive`, into the page item to create next.

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

    Ensure the following hidden page items exist on the summary region's level. All they need is creating, and the type set to `hidden`. Everything else is performed by Dynamic Actions.

    - `P2_AI_ASSESSMENT_JSON`
    - `P2_AI_ASSESSMENT_POST_ID`
    - `P2_AI_REASON`

1. **EXIF Data region**

    Create a sub-region named **EXIF Data** and place it after the previously created AI Analysis region. Change its type to _classic report_ and set the following properties:

    - **Identification**:
        - Name: `EXIF_DATA`
        - Title: EXIF Data
    - **Source**
        - Location: JSON Source
        - JSON Source: `EXIF_SOURCE`
    - **Local Post Processing**:
        - Type: where/order by clause
        - Where clause: `id = :P2_ID`

    Set the region's attributes as follows:

    - **Appearance**
        - Template Type: Theme
        - Template: Value Attribute Pairs - Column
    - **Messages**:
        - when no data found: Photo does not contain EXIF data.

The JSON Source reads `MLE_DATA.EXIF_DATA`. No additional SQL query is required for this region.

## Task 3: Add Dynamic Actions to Page 2

Dynamic Actions breathe life into the page. They are executed whenever a condition such as _page loads_ is satisfied and make embedding JavaScript code much easier.

Start by creating a Dynamic Action named `ai_assessment` **on Page Load** by right clicking it and selecting _Create Dynamic Action_. Add these two actions in this order in the _True_ branch:

1. **Reset AI assessment**

    - **Name**: Reset AI assessment
    - **Action**: Execute JavaScript Code

    It's job is to clear the previous assessment before a new image is analyzed by executing the following JavaScript code:

    ```javascript
    <copy>
    apex.item("P2_AI_ASSESSMENT_POST_ID").setValue("");
    apex.item("P2_AI_ASSESSMENT_JSON").setValue("");
    apex.item("P2_AI_REASON").setValue("");
    apex.item("P2_REALNESS_SCORE").setValue("Assessment pending");
    </copy>
    ```

    The page items modified by the snippet have just been created by you in the previous task.

2. **AI assessment server-side**

    Create another _True_ action like so:

    - **Name**: AI assessment server-side
    - **Action**: Execute Server-Side Code
    - **Language**: JavaScript (MLE)
    - **Items to Submit**: leave empty
    - **Items to Return**: `P2_AI_ASSESSMENT_POST_ID,P2_AI_ASSESSMENT_JSON,P2_AI_REASON,P2_REALNESS_SCORE`
    - **Show Processing**: enabled
    - **Stop Execution on Error**: enabled
    - **Wait for Result**: enabled

    Use the following JavaScript code. It's job is to invoke the MLE JavaScript module you created earlier, sending the photo to Gemini for assessment.

    ```javascript
    <copy>
    const { analyzePhotoWithGemini } = await import('gemini-ai-verify-module');
    const { getAssessmentReason, formatRealnessScore } =
        await import('realness-score-module');

    const postId = apex.env.P2_ID;
    const assessment = analyzePhotoWithGemini(postId);

    apex.env.P2_AI_ASSESSMENT_POST_ID = postId;
    apex.env.P2_AI_ASSESSMENT_JSON = assessment;

    apex.env.P2_AI_REASON = assessment.reason || "assessment error";
    apex.env.P2_REALNESS_SCORE = formatRealnessScore(
        assessment,
        postId,
        postId
    );
    </copy>
    ```

    Return the values to the page items listed above. The Gemini module performs the image analysis; the realness-score module formats the result for the APEX page.

Save Page 2 before continuing.

## Verify the application

Upload an image and confirm that:

- Page 1 stores the image in `MLE_DATA`.
- The EXIF MLE module writes metadata to `MLE_DATA.EXIF_DATA`.
- Page 2 displays the image, EXIF data, map, Gemini analysis, and realness score.

## Learn More

- [Oracle Database JavaScript Developer's Guide: Multilingual Engine](https://docs.oracle.com/en/database/oracle/oracle-database/26/mlejs/index.html)
- [Oracle Database JSON Developer's Guide: JSON_TABLE](https://docs.oracle.com/en/database/oracle/oracle-database/26/adjsn/json_table-sql-function.html)
- [APEX App Builder User's Guide: Page Designer](https://docs.oracle.com/en/database/oracle/apex/26.1/htmdb/page-designer.html)

## Acknowledgements

- **Author** - Martin Bach, Senior Principal Product Manager
- **Contributors** - Sonja Meyer, Consulting Member of Technical Staff
- **Last Updated By/Date** - Martin Bach, Senior Principal Product Manager, July 2026
