# Solution

## Task 1

### Solution

I would run a query that lists all clients in the source calculation and
distribution output tables in the database. The query performs the
following checks:

1.  Compares the expected net amount in the calculation data with the
    distributed amount in the distribution data table. If the values
    match, the `validation_status` column displays `PASS`.
2.  Checks for duplicate distribution records.
3.  Validates whether each client appears in both tables.
4.  Checks that the distribution transaction status is `COMPLETED`.
5.  Identifies clients appearing in the distribution output without a
    matching client in the source calculation table.

### SQL query

``` sql
WITH distribution_summary AS (
    SELECT
        client_id,
        COUNT(*) AS distribution_count,
        SUM(distributed_amount) AS total_distributed,
        GROUP_CONCAT(DISTINCT status) AS transaction_status
    FROM distribution_output
    GROUP BY client_id
)

SELECT
    cf.client_id,
    cf.client_name,
    cf.expected_net_amount,
    COALESCE(ds.total_distributed, 0) AS total_distributed,
    COALESCE(ds.distribution_count, 0) AS distribution_count,
    CASE
        WHEN ds.client_id IS NULL
            THEN 'FAIL - MISSING DISTRIBUTION'
        WHEN ds.distribution_count > 1
            THEN 'FAIL - DUPLICATE DISTRIBUTION'
        WHEN cf.expected_net_amount <> ds.total_distributed
            THEN 'FAIL - AMOUNT MISMATCH'
        WHEN ds.transaction_status <> 'COMPLETED'
            THEN 'FAIL - INVALID STATUS'
        ELSE 'PASS'
    END AS validation_status,
    ROUND(
        cf.expected_net_amount -
        COALESCE(ds.total_distributed, 0), 2
    ) AS amount_difference
FROM client_fees cf
LEFT JOIN distribution_summary ds
    ON cf.client_id = ds.client_id

UNION ALL

SELECT
    d.client_id,
    'UNKNOWN CLIENT',
    0.00,
    ds.total_distributed,
    ds.distribution_count,
    'FAIL - UNEXPECTED CLIENT',
    ROUND(0 - ds.total_distributed, 2)
FROM distribution_summary ds
JOIN (
    SELECT DISTINCT client_id
    FROM distribution_output
) d
    ON d.client_id = ds.client_id
LEFT JOIN client_fees cf
    ON ds.client_id = cf.client_id
WHERE cf.client_id IS NULL

ORDER BY client_id;
```

### Expected output

  --------------------------------------------------------------------------------------------------------------------------------
  client_id   client_name     expected_net_amount   total_distributed   distribution_count validation_status     amount_difference
  ----------- ------------- --------------------- ------------------- -------------------- ------------------- -------------------
  C001        Client A                    9800.00             9800.00                    1 PASS                               0.00

  C002        Client B                    5145.74             5145.73                    1 FAIL - AMOUNT                      0.01
                                                                                           MISMATCH            

  C003        Client C                     857.50              857.50                    1 PASS                               0.00

  C004        Client D                   12093.30            12093.30                    1 PASS                               0.00

  C005        Client E                    2940.00                0.00                    0 FAIL - MISSING                  2940.00
                                                                                           DISTRIBUTION        

  C006        Client F                     979.99              979.99                    1 PASS                               0.00

  C007        Client G                   44100.00            88200.00                    2 FAIL - DUPLICATE              -44100.00
                                                                                           DISTRIBUTION        

  C008        Client H                     147.00              147.00                    1 PASS                               0.00

  C009        Client I                    2450.49             2450.49                    1 PASS                               0.00

  C010        Client J                    7622.21             7622.21                    1 PASS                               0.00

  C011        Client K                    -196.00                0.00                    1 FAIL - AMOUNT                   -196.00
                                                                                           MISMATCH            

  C012        Client L                    5880.00             5880.00                    1 PASS                               0.00

  C013        UNKNOWN                        0.00              320.00                    1 FAIL - UNEXPECTED               -320.00
              CLIENT                                                                       CLIENT              
  --------------------------------------------------------------------------------------------------------------------------------

------------------------------------------------------------------------






Validation Status	# of Clients	Total Expected Amount	Total Distributed Amount	Net Difference (Expected - Distributed)
PASS	8	$39,830.49	$39,830.49	$0.00
FAIL - DUPLICATE DISTRIBUTION	1	$44,100.00	$88,200.00	-$44,100.00
FAIL - MISSING DISTRIBUTION	1	$2,940.00	$0.00	$2,940.00
FAIL - AMOUNT MISMATCH	2	$4,949.74	$5,145.73	-$195.99
FAIL - UNEXPECTED CLIENT	1	$0.00	$320.00	-$320.00
GRAND TOTAL	13	$91,820.23	$133,496.22	
## Task 2

### (i) Identify clients that appear in the distribution output but have no corresponding approved transaction record

#### SQL query

``` sql
SELECT
    d.client_id,
    c.client_name,
    d.distribution_id,
    d.period,
    d.distributed_amount,
    t.status AS transaction_status
FROM Distributions d
JOIN Clients c
    ON d.client_id = c.client_id
LEFT JOIN Transactions t
    ON d.distribution_id = t.distribution_id
    AND d.client_id = t.client_id
    AND t.status = 'APPROVED'
WHERE t.transaction_id IS NULL
ORDER BY d.client_id;
```

#### Output

  ------------------------------------------------------------------------------------------------
  client_id   client_name     distribution_id period       distributed_amount transaction_status
  ----------- ------------- ----------------- ---------- -------------------- --------------------
  C106        Client                       17 2026-05                 2205.74 
              Foxtrot                                                         

  C109        Client India                 27 2026-06                 4998.00 

  C113        Client Mike                  37 2026-04                 3234.00 
  ------------------------------------------------------------------------------------------------

### (ii) Calculate the total distributed amount per month for the period covered by the data

#### SQL query

``` sql
SELECT
    period,
    SUM(distributed_amount) AS total_distributed
FROM Distributions
GROUP BY period
ORDER BY period;
```

#### Output

  period      total_distributed
  --------- -------------------
  2026-04             102909.69
  2026-05             102411.23
  2026-06             104412.72

### (iii) Identify clients whose distributed amount differs from their calculated source amount by more than \$0.01

#### SQL query

``` sql
SELECT
    s.client_id,
    c.client_name,
    s.period,
    s.source_amount,
    s.fee_pct,
    ROUND(s.source_amount * (1 - s.fee_pct), 2)
        AS expected_amount,
    ROUND(d.distributed_amount, 2)
        AS distributed_amount,
    ROUND(
        d.distributed_amount -
        (s.source_amount * (1 - s.fee_pct)), 2
    ) AS difference
FROM SourceCalculations s
JOIN Distributions d
    ON s.client_id = d.client_id
    AND s.period = d.period
JOIN Clients c
    ON s.client_id = c.client_id
WHERE ABS(
    ROUND(d.distributed_amount, 2) -
    ROUND(s.source_amount * (1 - s.fee_pct), 2)
) > 0.01
ORDER BY s.period, s.client_id;
```

#### Output

  -----------------------------------------------------------------------------------------------------------------
  client_id   client_name   period      source_amount   fee_pct   expected_amount   distributed_amount   difference
  ----------- ------------- --------- --------------- --------- ----------------- -------------------- ------------
  C112        Client Lima   2026-04          20000.00      0.02          19600.00             19612.30        12.30

  C103        Client        2026-05          11800.00      0.02          11564.00             11569.00         5.00
              Charlie                                                                                  

  C110        Client Juliet 2026-06           7200.00      0.02           7056.00              7055.25        -0.75
  -----------------------------------------------------------------------------------------------------------------

### (iv) Likely root cause of the error

On 14 June 2026, a batch file was run. A total of 4,812 records were
submitted for processing. Client record `C0453` failed and was skipped
because the `fee_pct` field was `NULL` in the table instead of
containing an integer or decimal value. The other 4,811 records were
processed successfully.

------------------------------------------------------------------------

## Task 3

### Solution: API status validation using Postman

My solution uses Postman to automate the checking of distribution
statuses.

I created an API request to check the distribution API. The API endpoint
is:

`http://127.0.0.1:5000/api/distributions/status`

This API runs on my local machine. I added a Postman test script that
checks the API response and validates that every distribution has a
status of `COMPLETED`.

Using Postman, the test can be run manually or scheduled. If any
distribution has a status other than `COMPLETED`, the test will fail and
the affected records can be investigated.

### Postman test script

``` javascript
const response = pm.response.json();

// Check API response
pm.test("API returns HTTP 200", function () {
    pm.response.to.have.status(200);
});

// Check API execution status
pm.test("API execution successful", function () {
    pm.expect(response.status).to.eql("success");
});

// Check database records exist
pm.test("Distribution records returned", function () {
    pm.expect(response.total_records).to.be.above(0);
});

// Check every record has a status
pm.test("All records have a status", function () {
    response.data.forEach(function (record) {
        pm.expect(record.status).to.not.be.empty;
    });
});

// Check distribution IDs exist
pm.test("All records have distribution IDs", function () {
    response.data.forEach(function (record) {
        pm.expect(record.distribution_id).to.exist;
    });
});

// Check that all distributions have COMPLETED status
pm.test("All distributions must have COMPLETED status", function () {
    const failedDistributions = response.data.filter(function (record) {
        return record.status !== "COMPLETED";
    });

    if (failedDistributions.length > 0) {
        const details = failedDistributions.map(function (record) {
            return "Client: " + record.client_id +
                   ", Name: " + record.client_name +
                   ", Distribution ID: " + record.distribution_id +
                   ", Status: " + record.status;
        });

        pm.expect(
            failedDistributions.length,
            "Distributions with incorrect status:\n" + details.join("\n")
        ).to.eql(0);
    } else {
        pm.expect(failedDistributions.length).to.eql(0);
    }
});
```

### Tests covered

1.  Check that the database returns at least one record.
2.  Validate that every record in the database has a status.
3.  Check that every distribution has a status of `COMPLETED`.
4.  Validate that the API execution status is `success`.
5.  Check that every record has a distribution ID.

### Regression strategy

After a new batch has been processed, I will run the Postman script to
verify that all records added to the database are valid and contain the
required fields and a valid status.

If any required fields are missing, or if any distribution has a status
other than `COMPLETED`, the API test will fail. The failed records can
then be investigated and corrected before the batch is considered
successful.



The drawbacks of this solution is that it is running locally on my machine and everytime that there is a change, I have to manually run my scripts on Jenkins.




AI Tool usage:

I used AI to assist with creating sql queries and  also my javascript  tests on postman.
