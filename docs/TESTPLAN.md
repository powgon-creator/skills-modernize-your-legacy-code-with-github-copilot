# COBOL Student Account Test Plan

This test plan captures the current behavior of the COBOL application before
the migration to Node.js. It is intended for business stakeholder review and
as a behavioral baseline for future unit and integration tests.

The application starts each run with a single in-memory balance of `1000.00`.
`Actual Result` and `Status` are intentionally left as `-` until a test is
executed. Record `Pass` or `Fail` in the `Status` column after execution.

| Test Case ID | Test Case Description | Pre-conditions | Test Steps | Expected Result | Actual Result | Status (Pass/Fail) | Comments |
|---|---|---|---|---|---|---|---|
| TC-001 | Start a new application session | The application is compiled and not running. | 1. Start the application. | The account balance is initialized to `1000.00` in memory. The account management menu is displayed. | - | - | Confirms the initial state for every independent run. |
| TC-002 | Display the account menu | The application is running. | 1. Observe the first screen. 2. Complete any operation and return to the main menu. | The menu displays View Balance, Credit Account, Debit Account, and Exit, and prompts for a choice from 1 to 4. | - | - | The menu is displayed once per loop iteration. |
| TC-003 | View the initial balance | A new session is running with the default balance. | 1. Select `1`. | The application calls the balance read operation and displays `Current balance: 1000.00`. The balance is unchanged. | - | - | Validates the `TOTAL ` operation and `READ` data flow. |
| TC-004 | Credit the account with a positive amount | The current balance is `1000.00`. | 1. Select `2`. 2. Enter `250.50`. | The application adds the amount, writes the new balance, and displays `Amount credited. New balance: 1250.50`. | - | - | Validates a standard credit. |
| TC-005 | Retain a credited balance for a later operation | The account has been credited to `1250.50` in the same session. | 1. Select `1`. | The application displays `Current balance: 1250.50`. | - | - | Confirms that a successful credit is persisted in memory. |
| TC-006 | Debit the account with sufficient funds | The current balance is `1000.00`. | 1. Select `3`. 2. Enter `250.50`. | The application subtracts the amount, writes the new balance, and displays `Amount debited. New balance: 749.50`. | - | - | Validates a standard approved debit. |
| TC-007 | Debit the exact available balance | The current balance is `1000.00`. | 1. Select `3`. 2. Enter `1000.00`. | The debit is approved and the application displays a new balance of `0.00`. | - | - | Confirms the implemented `balance >= amount` boundary rule. |
| TC-008 | Reject a debit greater than the balance | The current balance is `1000.00`. | 1. Select `3`. 2. Enter `1000.01`. 3. Select `1`. | The application displays `Insufficient funds for this debit.` and does not write a new balance. The later balance inquiry still displays `1000.00`. | - | - | Confirms that the debit path does not allow an overdraft. |
| TC-009 | Perform multiple operations in sequence | A new session is running with balance `1000.00`. | 1. Credit `100.00`. 2. Debit `40.00`. 3. Select `1`. | The final displayed balance is `1060.00`. Each successful operation uses the balance produced by the previous operation. | - | - | Integration test for repeated read, calculate, and write operations. |
| TC-010 | Handle an invalid menu choice | The application is running. | 1. Enter `0`, `5`, or another value outside `1` through `4`. | The application displays `Invalid choice, please select 1-4.` and returns to the menu without changing the balance. | - | - | Repeat with a representative invalid value. |
| TC-011 | Exit the application | The application is running. | 1. Select `4`. | The loop ends and the application displays `Exiting the program. Goodbye!` before stopping. | - | - | Confirms the `CONTINUE-FLAG` exit behavior. |
| TC-012 | Reset the balance between application runs | A previous run ended after changing the balance. | 1. Start the application again. 2. Select `1`. | The new run displays `1000.00`, not the balance from the previous run. | - | - | Confirms that storage is in memory only and is not persistent. |
| TC-013 | Credit zero amount | The current balance is `1000.00`. | 1. Select `2`. 2. Enter `0.00`. 3. Select `1`. | Characterize the current implementation: the amount is accepted and the balance remains `1000.00`. Confirm with stakeholders whether a zero credit should instead be rejected in the Node.js application. | - | - | No explicit zero-amount validation exists in the COBOL code. |
| TC-014 | Debit zero amount | The current balance is `1000.00`. | 1. Select `3`. 2. Enter `0.00`. 3. Select `1`. | Characterize the current implementation: the debit passes the funds check, writes the unchanged balance, and the balance remains `1000.00`. Confirm whether zero debits should be rejected after migration. | - | - | No explicit zero-amount validation exists in the COBOL code. |
| TC-015 | Enter a negative amount | The application is running with a known balance. | 1. Select `2` or `3`. 2. Enter a negative amount. | Record the actual runtime behavior. The current data fields are unsigned numeric COBOL fields and the application has no explicit negative-amount validation. Agree on the required Node.js behavior with stakeholders before migration. | - | - | Characterization and business-rule decision case. |
| TC-016 | Use cents in an amount | The current balance is `1000.00`. | 1. Select `2`. 2. Enter `12.34`. 3. Select `1`. | The balance is increased to `1012.34` and the value is displayed with two decimal places. | - | - | Confirms the implied two-decimal monetary precision. |
| TC-017 | Test the maximum representable account value | The application is running. | 1. Use credits or a test fixture to approach the field limit. 2. Attempt an amount that would exceed `999999.99`. | Record the actual runtime behavior. `PIC 9(6)V99` can represent at most `999999.99`; the current program does not explicitly handle overflow. Define the Node.js validation rule with stakeholders. | - | - | Implementation boundary case for migration. |
| TC-018 | Verify an unrecognized operation is ignored at the data layer | A unit-test harness or direct program call can invoke `DataProgram`. | 1. Call `DataProgram` with an operation other than `READ` or `WRITE`. 2. Inspect the balance. | No read or write occurs because the COBOL data component has no branch for an unrecognized operation. The caller should define how such input is reported in the Node.js implementation. | - | - | Unit-level characterization of `data.cob`; the main menu never sends an unknown operation. |
| TC-019 | Verify a successful data write and read | A unit-test harness or direct program call can invoke `DataProgram`. | 1. Call `DataProgram` with `WRITE` and a known balance such as `1234.56`. 2. Call it with `READ`. | The subsequent `READ` returns `1234.56` during the same process execution. | - | - | Unit-level validation of in-memory storage. |

## Coverage Notes

- `main.cob` is covered by menu display, valid choices, invalid choices, the
  repeat loop, and exit cases.
- `operations.cob` is covered by total, credit, debit approval, debit refusal,
  sequential operations, and amount edge cases.
- `data.cob` is covered by initialization, `READ`, `WRITE`, in-memory
  persistence, and the unrecognized-operation characterization case.
- Cases TC-013 through TC-018 identify behavior that is currently
  implementation-defined or lacks validation. Stakeholders should decide
  whether the Node.js application preserves that behavior or introduces a
  validation rule.