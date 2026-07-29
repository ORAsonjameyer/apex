# Import JavaScript modules into the database

## Introduction

In this lab, you will import JavaScript modules into Oracle AI Database 26ai and make them available through an MLE environment. Multilingual Engine (MLE) allows JavaScript to run close to the data inside the database. The modules can query database tables, process files, and call database functionality while the application uses familiar JavaScript imports.

You will create three modules for the image analysis application:

- `MLE_EXIFR_MODULE` loads the third-party `exifr` library. It extracts EXIF metadata from an uploaded image.
- `MLE_GEMINI_AI_VERIFY_MODULE` reads an image from `SM_POSTS` and calls the configured APEX AI service.
- `MLE_REALNESS_SCORE_MODULE` converts the structured AI response into text for the APEX user interface.

The modules are kept separate so that each one has a clear responsibility. The MLE environment then gives them the import names that the APEX application uses.

This lab requires Oracle AI Database 26ai and Oracle APEX 26.1.

Estimated time to complete: 15 minutes

### Objectives

In this lab, you will:

- Allow the browser to provide location information for the map
- Create the three JavaScript MLE modules
- Create the `MLE_EXIF_ENV` environment
- Add the modules to the environment with the required import names

### Prerequisites

This lab assumes that you completed [Lab 2](../lab-2/lab-2.md) and created the `google gemini` Generative AI service and the `EXIF_SOURCE` JSON Source.

## Task 1: Allow location access

The application includes a map that can use the browser's location service. When the browser displays a message such as **Allow localhost:7005 to access your location?**, click **Allow**. You can select **Remember this decision** if you want the browser to remember the permission.

This permission belongs to the browser and is separate from the MLE environment. It allows the map to use the current location; it does not grant the application access to your OCI credentials or database account. If you select **Block** by mistake, open the browser's site permissions for the APEX URL and enable **Location** before continuing.

## Task 2: Open the MLE module area

1. Open **Object Browser**.
2. In the object tree, expand **MLE Modules - JavaScript**.
3. Use the **Create** or **+** action to create a JavaScript MLE module.

MLE modules are database objects. They are not APEX page components, so create them in the database schema that owns `SM_POSTS` and the MLE environment. In the workshop environment, this is the `ORACLE` schema.

## Task 3: Create the EXIF module

Create a JavaScript module with the following values:

- **Module Name**: `MLE_EXIFR_MODULE`
- **Language**: JavaScript

Paste the bundled `exifr` module source provided with this lab into the code editor. The bundled file is the browser-independent ESM build of `exifr` (for example, `exifr@7.1.3/dist/full.esm.mjs`). It is imported under the name `exifr-module` in the MLE environment.

The module is responsible for reading metadata from the image BLOB. It does not call Gemini and does not calculate a score. Keeping this work in a separate module makes the EXIF extraction reusable and easier to understand.

Click **Save and Compile**. Continue only after the module compiles successfully.

## Task 4: Create the Gemini verification module

Create a second JavaScript module:

- **Module Name**: `MLE_GEMINI_AI_VERIFY_MODULE`
- **Language**: JavaScript

Paste the following source into the code editor and click **Save and Compile**:

```javascript
export function analyzePhotoWithGemini(id) {
    if (!id) throw new Error("Please provide an image record ID.");

    const photo = session.execute(
        `select 1 from sm_posts where id = :id and file_blob is not null`,
        { id }
    );

    if (photo.rows.length !== 1) {
        return JSON.stringify({
            status: "no-image",
            classification: "inconclusive",
            confidence: null,
            reason: "No image available for analysis."
        });
    }

    try {
        const result = session.execute(
            `
            declare
                l_blob blob;
                l_mime varchar2(255);
                l_name varchar2(255);
                l_response clob;
                l_attachments apex_ai.t_attachments := apex_ai.t_attachments();
            begin
                select file_blob, nvl(file_mime, 'image/jpeg'),
                       nvl(file_name, 'photo.jpg')
                  into l_blob, l_mime, l_name
                  from sm_posts
                 where id = :id;

                l_attachments.extend;
                l_attachments(1).mime_type := l_mime;
                l_attachments(1).content_blob := l_blob;
                l_attachments(1).file_name := l_name;
                l_attachments(1).detail_level := apex_ai.c_detail_level_high;

                l_response := apex_ai.generate(
                    p_service_static_id => 'google-gemini',
                    p_system_prompt => 'Assess whether this photo is camera-captured or AI-generated. Return only valid JSON.',
                    p_prompt => 'Return status, classification, confidence from 0 to 100, and one short reason.',
                    p_temperature => 0.2,
                    p_attachments => l_attachments
                );

                :assessment := dbms_lob.substr(l_response, 4000, 1);
            end;`,
            {
                id,
                assessment: {
                    dir: oracledb.BIND_OUT,
                    type: oracledb.STRING,
                    maxSize: 4000
                }
            }
        );
        return result.outBinds.assessment;
    } catch (err) {
        return JSON.stringify({
            status: "error",
            classification: "inconclusive",
            confidence: null,
            reason: `AI assessment unavailable: ${err}`
        });
    }
}
```

This module first checks whether the requested `SM_POSTS` record contains an image. It then uses `session.execute` to load the BLOB and calls `APEX_AI.GENERATE` with the native APEX attachment type. JavaScript controls the overall flow, while PL/SQL is used only inside the module where APEX AI requires native database types.

The module calls the service using the Static ID `google-gemini`. This is why the service created in Lab 2 must use the exact Static ID. The module returns structured JSON containing a status, classification, confidence, and reason so the APEX page can handle successful and failed AI calls consistently.

## Task 5: Create the realness score module

Create a third JavaScript module:

- **Module Name**: `MLE_REALNESS_SCORE_MODULE`
- **Language**: JavaScript

Paste the following source into the code editor and click **Save and Compile**:

```javascript
export function getAssessmentReason(json) {
    return parse(json).reason || json || "";
}

export function formatRealnessScore(json, imageId, assessmentImageId) {
    if (!imageId) return "";
    if (!assessmentImageId || String(assessmentImageId) !== String(imageId)) {
        return "Assessment pending";
    }
    if (!json) return "Assessment pending";

    const assessment = parse(json);
    if ((assessment.status || "error").toLowerCase() !== "ok") {
        return "Score unavailable";
    }

    const confidence = clamp(Number(assessment.confidence) || 50);
    const classification = (assessment.classification || "inconclusive").toLowerCase();
    let score = 50;
    let label = "inconclusive";

    if (classification === "real") {
        score = confidence;
        label = "likely real";
    } else if (classification === "ai-generated") {
        score = 100 - confidence;
        label = "likely AI";
    }

    return Math.round(score) + "% " + label;
}

function parse(json) {
    try {
        return JSON.parse(json || "{}");
    } catch (error) {
        return { status: "error", reason: json || "Score unavailable" };
    }
}

function clamp(value) {
    return Math.min(Math.max(value, 0), 100);
}
```

This module contains only presentation helpers. It parses the JSON returned by the Gemini module, extracts the short reason shown in the application, and formats the score. It does not call an AI service and does not access the image table. Keeping this logic separate prevents the APEX page from having to calculate scores itself.

## Task 6: Create the MLE environment

1. In **Object Browser**, expand **MLE Environments**.
2. Click **Create** or **+**.
3. Enter `MLE_EXIF_ENV` as the environment name.
4. Save the environment and open its **Imports** tab.
5. Click **Add Import** for each of the three modules.

An MLE environment maps the module names used by JavaScript `import` statements to database MLE modules. The import name is therefore part of the application contract and must be entered exactly as shown below:

| Module Owner | Module Name | Import Name |
| --- | --- | --- |
| `ORACLE` | `MLE_EXIFR_MODULE` | `exifr-module` |
| `ORACLE` | `MLE_GEMINI_AI_VERIFY_MODULE` | `gemini-ai-verify-module` |
| `ORACLE` | `MLE_REALNESS_SCORE_MODULE` | `realness-score-module` |

Do not change the capitalization of the module names or the spelling of the import names. The APEX page uses these import names when it loads the modules. The module owner must match the schema in which you created the modules.

Save the environment and refresh the **Imports** tab. Confirm that all three imports are listed without errors. The environment is resolved when the APEX page runs the MLE code; there is no separate JavaScript compilation step for the import mappings.

## Task 7: Assign the MLE environment to the application

The MLE environment must also be assigned to the APEX application. Creating the environment in the database alone is not enough; the application needs to know which environment resolves imports such as `exifr-module` and `gemini-ai-verify-module`.

1. Return to **App Builder** and open the workshop application.
2. Open the application definition or application settings.
3. Select the **Security** tab.
4. In the **Database Session** section, keep the correct **Parsing Schema** selected.
5. For **MLE Environment**, select `MLE_EXIF_ENV`.
6. Save the application settings.

The application is now associated with `MLE_EXIF_ENV`. When an APEX page executes MLE JavaScript, its `import` statements are resolved against this environment and its three module mappings.

## Verify the configuration

Confirm that the following objects are available in the database:

- `MLE_EXIFR_MODULE`
- `MLE_GEMINI_AI_VERIFY_MODULE`
- `MLE_REALNESS_SCORE_MODULE`
- `MLE_EXIF_ENV`

Confirm that `MLE_EXIF_ENV` contains these exact import mappings:

- `exifr-module` → `MLE_EXIFR_MODULE`
- `gemini-ai-verify-module` → `MLE_GEMINI_AI_VERIFY_MODULE`
- `realness-score-module` → `MLE_REALNESS_SCORE_MODULE`

The environment is now ready for the APEX application. In the next lab, you will use the modules to extract EXIF metadata, call Gemini, and display the image assessment.

## Learn More

- [Oracle Database JavaScript Developer's Guide: Multilingual Engine](https://docs.oracle.com/en/database/oracle/oracle-database/26/mlejs/index.html)
- [Oracle Database JavaScript Developer's Guide: JavaScript Modules](https://docs.oracle.com/en/database/oracle/oracle-database/26/mlejs/using-javascript-modules.html)
- [Oracle Database JavaScript Developer's Guide: MLE Environments](https://docs.oracle.com/en/database/oracle/oracle-database/26/mlejs/using-mle-environments.html)

## Acknowledgements

- **Author** - Sonja Meyer, Consulting Member of Technical Staff 
- **Contributors** - Martin Bach, Senior Principal Product Manager
- **Last Updated By/Date** - Sonja Meyer, Consulting Member of Technical Staff, July 2026
