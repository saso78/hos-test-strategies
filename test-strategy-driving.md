# TEST STRATEGY: Hours of Service (HoS) - Driving

## Feature Owner: *Sasho Trpovski*
Version: 1.0

Last Updated: [12/01/2025]

### 1. EXECUTIVE SUMMARY
## *Feature:*
    Hours of Service (HoS) Driving

## *Criticality:*  
    HIGH - Regulatory compliance feature (FMCSA ELD mandate)

## Scope: 
    API validation, database integrity, UI verification
## Test Strategy Approach:
    Focus on API-level validation (60%) with database verification (30%)and minimal UI coverage (10%) for critical user journeys. This aligns with testing pyramid best practices and provides fast, reliable feedback on regulatory compliance logic.

## 2. FEATURE OVERVIEW
*What is Driving event in terms of Hours of Service?*
#### A Driving event is an automatic duty status change recorded by the ELD when the commercial motor vehicle is in motion, as required by the FMCSA for ELD compliance. Driving time counts toward the driver’s 11-hour driving limit and must occur within the 14-hour on-duty shift window.A driver cannot drive more than 11 hours before taking the required rest, and cannot drive beyond the 14-hour shift limit without first obtaining a qualifying reset. Additionally, the driver must take at least one uninterrupted 30-minute break after 8 cumulative hours of driving without a 30-minute interruption. This break can be taken in any non-driving status: Off Duty, On Duty (Not Driving), or Sleeper Berth.To reset the shift and driving limits, the driver must obtain at least 10 consecutive hours Off Duty, 10 consecutive hours in the Sleeper Berth, or a compliant split-sleeper combination (e.g., 7/3 or 8/2 splits) that meets FMCSA rules.
### Business Rules:
  *Standard HoS Rules:*

* 14-hour shift limit (driving window)
* 11-hour driving limit within 14-hour shift
* 10-hour rest requirement (Off Duty or Sleeper Berth)
* 70-hour/8-day or 80-hour/7-day cycle limit
* 36-hour reset required to restart cycle

#### Example Scenario:


| Time | Event | Duration | Drive Total | Shift Total | Violation? |
|------|-------|----------|-------------|-------------|------------|
| 00:00 | ON DUTY | 20 min | 0h | 0h 20m | No |
| 00:20 | DRIVING | 4h | 4h | 4h 20m | No |
| 04:20 | ON DUTY | 10 min | 4h | 4h 30m | No |
| 04:30 | DRIVING | 4h | 8h | 8h 30m | No |
| 08:30 | OFF DUTY | 30 min | 8h | 9h | No (30-min break taken) |
| 09:00 | DRIVING | 3h | 11h | 12h | No |

Final State:
- Driving timer: 0h 00m (11h limit reached)
- Shift timer: 2h 00m remaining (12h of 14h used)
- Violations: NONE ✅

___
## 1. RISK ASSESSMENT
### Business Impact: CRITICAL
### User Risk:

    Drivers receive DOT citations ($1,000+ fines)
    Out of Service (OOS) orders issued
    Driver CSA scores negatively impacted
    Potential license suspension

### Company Risk:

    Carrier liability for violations
    Loss of customer trust
    Regulatory audit failures
    Platform reputation damage
    Potential loss of all drivers using the platform

### Regulatory Risk:

    FMCSA non-compliance
    Platform decertification possible
    Legal liability

### Estimated Impact: *If this feature fails, 100% of active users are at risk of violations.*

#### Technical Risk: HIGH
#### Complexity Factors:

 Timer Precision Requirements:

    Timers must be calculated in seconds, not minutes
    10h 59m 59s = COMPLIANT (39,599 seconds)
    11h 00m 00s = COMPLIANT (39,600 seconds)
    11h 00m 01s = VIOLATION (39,601 seconds)
    Single-second errors result in regulatory violations

    Timer calculations across multiple duty status changes
    State management (cycle timer, shift timer, drive timer)
    Edge cases (midnight crossing, timezone changes, retroactive edits)
    Multi-table database synchronization (events, timers, violations)
    Real-time vs. historical calculations

#### Integration Points:

    Event log system
    Timer calculation engine
    Violation detection system
    Database (SQL + MongoDB)
    Mobile app sync
    Web portal display


### 4. TEST STRATEGY (PYRAMID APPROACH)
Testing Pyramid Distribution:
```
         /\
        /UI\ ← 10% - Mobile/Web critical path validation
       /----\
      / API  \ ← 60% - Core business logic validation
     /--------\
    /   DB     \ ← 30% - Data integrity & state validation
   /------------\
```
##### Why This Distribution?

    API Layer (60% of effort):

    Business logic is in the API
    Fastest feedback loop
    Most stable tests
    Easy to validate calculations
    Can test all edge cases efficiently

    Database Layer (30% of effort):

    Validates data persistence
    Checks referential integrity
    Confirms violation logging
    Verifies timer state across tables

    UI Layer (10% of effort):

    Validates display accuracy
    Confirms mobile/web parity
    Tests critical user workflows only
    Reserved for paths that MUST use interface


### 5. TEST APPROACH BY LEVEL
#### 5.1 API Testing (Primary Focus)
##### Tools: Python + pytest + requests + Swagger API
___
#### Test Categories:
#####  A. Happy Path Scenarios:

    Pre-reset OFF 10h00min
    ON  1h00min
    DRV 5h00min
    OFF 1h00min
    DRV 3h00min
    ON  2h00min
    DRV 2h00min
    OFF 10h00min

##### B. Negative Scenarios:

    ON  1h00min
    DRV 4h00min
    OFF 1h00min 
    DRV 4h00min 
    OFF 1h00min
    DRV 4h00min 
    Violation of the 11- and 14-hour rules

#### 5.2 Database Validation (Secondary Focus)
##### Databases: SQL (relational) + MongoDB (document store)

SQL Tables to Validate:

    events - Event log entries
    timers - Current timer states
    violations - Detected violations
    users - Driver information

MongoDB Collections:

    Driver state documents
    Historical timer snapshots

#### 5.3 UI Testing (Minimal Coverage)
Platforms: Mobile app + Web portal

Test Scenarios (ONLY):

    Timer display matches API/DB state
    Violation display when present
    Timer countdown updates in real-time

## *Why Minimal UI Testing?*

    UI is a display layer only
    Business logic validated at API level
    UI tests are slow and flaky
    Focus on critical display validation


#### 6. TEST DATA STRATEGY
#### Test Environment:

    Development: Initial test development and debugging
    Staging: Full regression before production
    Production: Smoke tests post-deployment

    Test Data Approach:
    Fresh Test Company:

    Create dedicated test company: "QA_Test_HoS_Company"
    Isolated from production data
    Can be reset between test runs

    Test Driver Profiles:

    driver_standard - Clean logs, no violations
    driver_near_limit - Close to cycle limit (tests boundary conditions)
    driver_with_violations - Existing violations (tests cascading effects)

    Data Requirements:

    Ability to insert backdated events (up to 7 days for cycle testing)
    Ability to reset driver logs between tests
    Ability to set specific cycle types (60/7 vs 70/8)

Data Cleanup Strategy:
```python
def cleanup_test_data(driver_id):
    """
    Remove all test data for fresh start between test runs
    """
    # 1. Delete all events for test driver
    delete_events(driver_id)
    
    # 2. Reset timers to default state
    reset_timers(driver_id, 
                 drive_remaining=39600,    # 11h in seconds
                 shift_remaining=50400,     # 14h in seconds
                 cycle_remaining=252000)    # 70h in seconds
    
    # 3. Clear all violations
    clear_violations(driver_id)
    
    # 4. Reset cycle type to default (70/8)
    reset_cycle(driver_id, cycle_type='70/8')
    
    # 5. Verify clean state
    verify_clean_state(driver_id)
```

#### Cleanup runs:
- Before each test (ensures clean starting state)
- After test suite completion (prevents data pollution)

### 7. TEST EXECUTION PLAN

#### Phase 1: API Test Development (Week 1-2)

☐ Set up pytest framework with fixtures

☐ Implement authentication helper

☐ Create test data factory (driver profiles, event sequences)

☐ Write 15 happy path scenarios

☐ Write 10 negative scenarios

☐ Write 8 edge case scenarios

#### Phase 2: Database Validation (Week 3)

☐ Write SQL validation queries (5 core queries)

☐ Write MongoDB validation queries (3 state checks)

☐ Integrate DB checks into API tests

☐ Create data cleanup utilities


#### Phase 3: UI Validation (Week 4)
☐ Identify 5 critical display scenarios

☐ Write Playwright/Maestro tests

☐ Integrate with CI/CD

#### Phase 4: CI/CD Integration (Week 5)

☐ Add tests to pipeline (GitHub Actions / Jenkins)

☐ Configure quality gates (100% API pass before UI)

☐ Set up test reporting (Allure / pytest-html)

☐ Schedule nightly regression runs

### 8. QUALITY GATES

#### Gate 1: API Tests (BLOCKING)

✅ 100% of API happy path tests pass

✅ 100% of negative scenario tests pass

✅ 95%+ of edge case tests pass

❌ If API tests fail, DO NOT proceed to UI testing


#### Gate 2: Database Validation (BLOCKING)

✅ All timer states match expected calculations

✅ No false positive violations logged

✅ Event log integrity maintained (no overlaps, gaps)


#### Gate 3: UI Validation (NON-BLOCKING for deployment)

✅ Timer display matches API state

⚠️ UI test failures logged but don't block deployment

🔄 UI failures trigger bug tickets, not deployment rollback


## *Rationale:*

    If API/DB are correct, UI issues are display bugs, 
    not regulatory compliance failures.

### 9. SUCCESS METRICS

#### Test Coverage:

✅ 30+ API test scenarios (happy, negative, edge)

✅ 15+ database validation queries

✅ 5 critical UI paths


#### Quality Indicators:

✅ Zero production violations related to driving limits

✅ API test execution time < 5 minutes (fast feedback)

✅ Test failure rate < 5% (excluding legitimate bugs)

✅ Bug detection before production: 95%+


#### Business Impact:

✅ Zero DOT citations due to driving timer errors

✅ Zero customer escalations related to driving limits

✅ Regulatory compliance maintained (100% FMCSA compliant)
