

--Using a CTE to pick the earliest Credentials for Staffs with Multiple Credentials - RCN 11/12/2024
WITH EarliestCredential AS (
    SELECT 
        sc.STAFF_ID,
        sc.BILLING_CATEGORY,
        sc.CREDENTIAL,
        ROW_NUMBER() OVER (PARTITION BY sc.STAFF_ID ORDER BY sc.DATE_RECEIVED) AS rn
    FROM 
        "RPT_STAFF_CREDENTIAL" sc
    WHERE 
        sc.EXPIRATION_DATE IS NULL OR sc.EXPIRATION_DATE > TRUNC(SYSDATE)
)
SELECT DISTINCT
    hist.end_date,
    st."EMPLOYEE_ID",
    st."FULL_NAME",
    LOGIN."USER_NAME",
    hist."JOB_CODE",
    priv."GROUP_NAME",
    login.last_login_datetime,
    hist."PRIMARY_ORGANIZATION",
    ORG2."ORGANIZATION_NAME" AS "PARENT ORGANIZATION",
    hist."ADDITIONAL_ORGANIZATIONS",
    st.CURRENT_CLINICAL_SUPERVISORS AS clinsup,
    st.CURRENT_ADMIN_SUPERVISORS AS admsup,
     -- Added ON OVERFLOW TRUNCATE clause with the LISTAGG function to the primary credential column  12/11/23--RCN--Row 15
    -- Included DISTINCT keyword within the LISTAGG -- RCN 3/5/2024
    -- Added a WHEN clause to check for EMPLOYEE_ID 123605 --RCN 11/5/2024
    -- Added a WhEN clause to check EMPLOYEE_ID 108655 info within the Credential and billing category column - RCN 11/11/2024


    LISTAGG(DISTINCT
        CASE
            WHEN st."EMPLOYEE_ID" = '123605' THEN '' 
            WHEN ec.BILLING_CATEGORY IS NOT NULL AND ec.CREDENTIAL IS NOT NULL 
                THEN ec.BILLING_CATEGORY || ', ' || ec.CREDENTIAL
            WHEN ec.BILLING_CATEGORY IS NOT NULL THEN ec.BILLING_CATEGORY
            ELSE ''
        END, '; ' ON OVERFLOW TRUNCATE
    ) WITHIN GROUP (ORDER BY ec.CREDENTIAL) OVER (PARTITION BY st."EMPLOYEE_ID") AS Primary_Credential, -- Added the CASE statement to properly reflect the conditions for the primary credentials - RCN 8/13/2024



      -- Added ON OVERFLOW TRUNCATE clause with the LISTAGG function to the billing category and the credential columns 12/11/23--RCN--Row 19
    -- Included DISTINCT keyword within the LISTAGG -- RCN 3/5/2024
    LISTAGG(DISTINCT
        CASE
            WHEN sc.CREDENTIAL IS NOT NULL AND sc.BILLING_CATEGORY IS NOT NULL THEN sc.BILLING_CATEGORY || ', ' || sc.CREDENTIAL
            WHEN sc.CREDENTIAL IS NOT NULL THEN sc.CREDENTIAL
            ELSE sc.BILLING_CATEGORY
        END, '; ' ON OVERFLOW TRUNCATE
    ) WITHIN GROUP (ORDER BY st."EMPLOYEE_ID") OVER (PARTITION BY st."EMPLOYEE_ID") AS credentials,

     -- Added LISTAGG to the Group_name column  12/11/23--RCN--Row 28
    -- Included DISTINCT keyword within the LISTAGG -- RCN 3/5/2024
    LISTAGG(DISTINCT
        ss.GROUP_NAME, ', ' ON OVERFLOW TRUNCATE
    ) WITHIN GROUP (ORDER BY ss.GROUP_NAME) OVER (PARTITION BY st."EMPLOYEE_ID") AS supv_grp,
    --SA.PRIMARY_EMAIL_ADDRESS -- ADDED 4/29/24 EAH
     COALESCE(SA.PRIMARY_EMAIL_ADDRESS, PC.EMAIL1) AS PRIMARY_EMAIL_ADDRESS



FROM
    "RPT_STAFF" st
    LEFT JOIN "RPT_EMPLOYMENT_HISTORY" hist 
        ON st."STAFF_ID" = hist."STAFF_ID"
    INNER JOIN "RPT_ADMIN_ORGANIZATION" ORG1 
        ON hist."PRIMARY_ORGANIZATION_ID" = ORG1."ORGANIZATION_ID"
    LEFT JOIN "RPT_ADMIN_ORGANIZATION" ORG2 
        ON ORG1."PARENT_ORG_ID" = ORG2."ORGANIZATION_ID"
    INNER JOIN "RPT_STAFF_PRIVILEGE" priv 
        ON st."STAFF_ID" = priv."STAFF_ID"
    INNER JOIN "RPT_STAFF_LOGIN" LOGIN 
        ON st."STAFF_ID" = LOGIN."STAFF_ID"
    LEFT JOIN EarliestCredential ec 
        ON st.STAFF_ID = ec.STAFF_ID AND ec.rn = 1
     --Changed the inner join to a left join so all employees' names will be included in the result set--RCN 1/11/24
    LEFT JOIN "RPT_STAFF_CREDENTIAL" sc 
        ON st.staff_id = sc.staff_id
    LEFT JOIN RPT_STAFF_SUPERVISOR ss 
        ON st.STAFF_ID = ss.STAFF_ID 
        AND ss.GROUP_NAME IS NOT NULL
    LEFT JOIN "RPT_STAFF_PROGRAM" sp 
        ON st."STAFF_ID" = sp.STAFF_ID
    LEFT JOIN RPT_STAFF_ADDRESSES SA -- ADDED 4/29/2024 EAH 
        ON st."STAFF_ID" = SA.STAFF_ID 
    LEFT JOIN PERSON_CONTACT PC -- ADDED 12/04/2024 RCN
        ON ST.STAFF_ID = PC.PERSON_ID

WHERE
    (hist.end_date IS NULL OR hist.END_DATE > TRUNC(SYSDATE))
    AND (st.full_name NOT LIKE '%TRAINING%' AND st.FULL_NAME NOT LIKE '%TEST%' AND st.FULL_NAME NOT LIKE '%Test%')

-- Removed the group_by function from the table-- RCN 12/11/23
ORDER BY
    st.full_name, login.last_login_datetime ASC
