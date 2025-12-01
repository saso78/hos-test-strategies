# TEST STRATEGY: Hours of Service (HoS) - Sleeper Berth Provisioning

## Feature Owner: *Sasho Trpovski*
Version: 1.0

Last Updated: [11/24/2025]

### 1. EXECUTIVE SUMMARY
## *Feature:*
    Hours of Service (HoS) Sleeper Berth Provisioning

## *Criticality:*  
    HIGH - Regulatory compliance feature (FMCSA ELD mandate)

## Scope: 
    API validation, database integrity, UI verification
## Test Strategy Approach:
    Focus on API-level validation (70%) with database verification and minimal UI coverage (10%) for critical user journeys. This aligns with testing pyramid best practices and provides fast, reliable feedback on regulatory compliance logic.

## 2. FEATURE OVERVIEW
*What is Sleeper Berth (SB) Provisioning?*
#### Sleeper Berth Provisioning is a regulatory feature mandated by FMCSA for ELD compliance that allows commercial drivers to split their required 10-hour rest period into two segments (e.g., 7/3 or 8/2 hours) while maintaining compliance with Hours of Service regulations.
Business Rules:
  *Standard HoS Rules:*

* 14-hour shift limit (driving window)
* 11-hour driving limit within 14-hour shift
* 10-hour rest requirement (Off Duty or Sleeper Berth)
* 70-hour/8-day or 80-hour/7-day cycle limit
* 36-hour reset required to restart cycle

#### Sleeper Berth Split Rules:

    Split must be 7/3, 8/2, or similar combinations totaling 10 hours
    First period pauses 14-hour clock
    Second period (minimum 7 hours) resets drive and shift timers
    Both periods must be in Sleeper Berth status

#### Example Scenario:

    Driver drives 4 hours
    Takes 3-hour SB rest (pauses 14-hour clock)
    Drives another 4 hours (total: 8 hours driven, 11 hours elapsed)
    Takes 7-hour SB rest (resets timers)
    Can start fresh 14-hour shift with full 11-hour drive time


1. RISK ASSESSMENT
## Business Impact: CRITICAL
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

    Standard 7/3 split
    Standard 8/2 split
    Complete shift with SB split
    Multiple SB splits in same shift
    SB split at shift boundary

##### B. Negative Scenarios:

    Invalid splits (4/6 - both under 7 hours)
    Insufficient rest (1:59/8:00 - first period too short)
    Incorrect duty status (Off Duty instead of SB)
    Exceeded drive time before split
    Violated shift time before split

##### C. Edge Cases:

    SB split crossing midnight
    Retroactive event insertion
    SB split at cycle limit boundary
    Multiple drivers, same vehicle
    Timezone changes during split

Test Flow Example:

   ```python
    # Pseudocode
def test_sleeper_berth_7_3_split():
    # 1. Authenticate
    token = authenticate(driver_credentials)
    
    # 2. Insert initial event (ON DUTY)
    response = post_event(token, event_type="ON", timestamp=start_time)
    assert response.status_code == 201
    
    # 3. Validate initial timers
    timers = get_timers(token)
    assert timers["shift_remaining"] == 14 * 3600  # 14 hours in seconds
    assert timers["drive_remaining"] == 11 * 3600  # 11 hours
    
    # 4. Insert DRIVING event (4 hours)
    post_event(token, event_type="D", duration=4*3600)
    
    # 5. Validate timers after driving
    timers = get_timers(token)
    assert timers["drive_remaining"] == 7 * 3600  # 11 - 4 = 7 hours
    assert timers["shift_remaining"] == 10 * 3600  # 14 - 4 = 10 hours
    
    # 6. Insert SLEEPER BERTH (3 hours)
    post_event(token, event_type="SB", duration=3*3600)
    
    # 7. Validate timers (should pause, not reset)
    timers = get_timers(token)
    assert timers["drive_remaining"] == 7 * 3600  # Unchanged
    assert timers["shift_remaining"] == 10 * 3600  # Paused
    
    # 8. Insert DRIVING (4 more hours)
    post_event(token, event_type="D", duration=4*3600)
    
    # 9. Insert SLEEPER BERTH (7 hours) - This resets timers
    post_event(token, event_type="SB", duration=7*3600)
    
    # 10. Validate timers RESET
    timers = get_timers(token)
    assert timers["drive_remaining"] == 11 * 3600  # RESET to full
    assert timers["shift_remaining"] == 14 * 3600  # RESET to full
    
    # 11. Validate NO VIOLATIONS
    violations = get_violations(token)
    assert len(violations) == 0
   ```

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

Validation Queries:
Example: Verify Timer State
```sql
    SELECT 
    user_id,
    shift_remaining,
    drive_remaining,
    cycle_remaining,
    last_updated
FROM timers
WHERE user_id = {test_driver_id}
AND shift_remaining = 50400  -- 14 hours after reset
AND drive_remaining = 39600  -- 11 hours after reset
```
Example: Verify No False Violations
```sql
SELECT COUNT(*) as violation_count
FROM violations
WHERE user_id = {test_driver_id}
AND violation_type = 'DRIVE_TIME'
AND created_at > {test_start_time}
```
#### 5.3 UI Testing (Minimal Coverage)
Platforms: Mobile app + Web portal

Test Scenarios (ONLY):

    Timer display matches API/DB state
    Violation display when present
    SB split visually represented in logs
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

Data Cleanup:
```python
     def cleanup_test_data(driver_id):
    """Remove all test data for fresh start"""
    delete_events(driver_id)
    reset_timers(driver_id)
    clear_violations(driver_id)
    reset_cycle(driver_id)
```
### 7. QUALITY GATES
#### Gate 1: API Tests (BLOCKING)
    
✅ 100% of API happy path tests pass

✅ 100% of negative scenario tests pass

✅ 95%+ of edge case tests pass

❌ If API tests fail, DO NOT proceed to UI testing

#### Gate 2: Database Validation (BLOCKING)

✅ All timer states match expected calculations

✅ No false positive violations logged

✅ Event log integrity maintained


#### Gate 3: UI Validation (NON-BLOCKING for deployment)

✅ Timer display matches API state

⚠️ UI test failures are logged but don't block deployment

🔄 UI failures trigger bug tickets, not deployment rollback

___
### Rationale: *If API/DB are correct, UI issues are display bugs, not regulatory compliance failures.*
___

### 8. TEST EXECUTION PLAN

#### Phase 1: API Test Development (Week 1-2)

    Set up test framework (pytest + requests)
    Implement authentication helper
    Create test data factory
    Write 10 happy path tests
    Write 10 negative tests
    Write 5 edge case tests

#### Phase 2: Database Validation (Week 3)

    Write SQL validation queries
    Write MongoDB validation queries
    Integrate DB checks into API tests
    Create data cleanup utilities

#### Phase 3: UI Validation (Week 4)

    Identify 5 critical display scenarios
    Write Playwright/Maestro tests
    Integrate with CI/CD

#### Phase 4: CI/CD Integration (Week 5)

    Add tests to pipeline
    Configure quality gates
    Set up test reporting
    Schedule regression runs


### 9. SUCCESS METRICS

**Test Coverage:**

✅ 25+ API test scenarios

✅ 15+ database validation queries

✅ 5 critical UI paths


**Quality Indicators:**

✅ Zero production violations related to SB provisioning

✅ API test execution time < 5 minutes

✅ Test failure rate < 5% (excluding legitimate bugs)

✅ Bug detection before production: 95%+


**Business Impact:**

✅ Zero DOT citations due to SB calculation errors

✅ Zero customer escalations related to SB timers

✅ Regulatory compliance maintained

