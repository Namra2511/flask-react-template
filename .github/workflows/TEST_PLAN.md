# Test Plan for Preview Deployment Conditions

## Overview
This document outlines the test plan for verifying that preview deployments only occur when:
1. The PR is not a draft
2. All required tests pass (npm:coverage, npm:lint, compose:test)

## Changes Made

### 1. Modified `preview_on_pr_update.yml`
- **Before**: `if: github.event.pull_request.state == 'open'`
- **After**: `if: github.event.pull_request.state == 'open' && !github.event.pull_request.draft`
- **Effect**: Prevents deployment for draft PRs

### 2. Enhanced `preview_on_dispatch.yml`
- **Added**: `checks: "['npm:coverage', 'npm:lint', 'compose:test']"`
- **Effect**: Ensures manual deployments also run tests before deploying

## Test Scenarios

### Scenario 1: Draft PR with Passing Tests
- **Setup**: Create a draft PR with code that passes all tests
- **Expected Result**: No preview deployment should be triggered
- **Verification**: Check GitHub Actions tab - workflow should be skipped

### Scenario 2: Draft PR with Failing Tests
- **Setup**: Create a draft PR with code that fails tests
- **Expected Result**: No preview deployment should be triggered
- **Verification**: Check GitHub Actions tab - workflow should be skipped

### Scenario 3: Non-Draft PR with Passing Tests
- **Setup**: Create a regular (non-draft) PR with code that passes all tests
- **Expected Result**: Preview deployment should be triggered and succeed
- **Verification**: 
  - Check GitHub Actions tab - workflow should run
  - Verify all checks pass
  - Confirm deployment succeeds

### Scenario 4: Non-Draft PR with Failing Tests
- **Setup**: Create a regular PR with code that fails one or more tests
- **Expected Result**: Preview deployment should be triggered but fail at the checks stage
- **Verification**: 
  - Check GitHub Actions tab - workflow should run
  - Verify checks fail
  - Confirm deployment does not proceed

### Scenario 5: Converting Draft to Ready
- **Setup**: 
  1. Create a draft PR (should not deploy)
  2. Mark PR as "Ready for review"
- **Expected Result**: 
  1. No deployment while draft
  2. Deployment triggered when marked ready (if tests pass)
- **Verification**: Check workflow runs only after marking as ready

## Manual Testing Steps

### Step 1: Create Test Branch with Intentional Test Failure
```bash
# Create a new branch
git checkout -b test-failing-checks

# Modify a test file to make it fail
# For example, modify a test assertion to be incorrect
echo "// Intentional test failure" >> tests/test_example.js

# Commit and push
git add .
git commit -m "Add intentional test failure for testing"
git push origin test-failing-checks
```

### Step 2: Create Draft PR
1. Go to GitHub repository
2. Create a new PR from the test branch
3. Mark it as "Draft"
4. Verify no deployment workflow runs

### Step 3: Mark PR as Ready
1. Click "Ready for review" on the draft PR
2. Verify deployment workflow starts
3. Verify workflow fails at the checks stage
4. Check that no deployment occurs

### Step 4: Fix Tests and Update PR
```bash
# Fix the test failure
git checkout test-failing-checks
# Remove the intentional failure
git add .
git commit -m "Fix test failure"
git push origin test-failing-checks
```

### Step 5: Verify Successful Deployment
1. Verify workflow runs again after push
2. Verify all checks pass
3. Verify deployment succeeds

## Expected Workflow Behavior

### Draft PR Events
- `opened` (draft): ❌ Workflow skipped
- `synchronize` (draft): ❌ Workflow skipped  
- `ready_for_review`: ✅ Workflow runs (if tests pass)

### Non-Draft PR Events
- `opened`: ✅ Workflow runs (if tests pass)
- `synchronize`: ✅ Workflow runs (if tests pass)
- `reopened`: ✅ Workflow runs (if tests pass)

### Test Failure Scenarios
- Any check failure (`npm:coverage`, `npm:lint`, `compose:test`): ❌ Deployment blocked

## Verification Commands

### Check Workflow Status
```bash
# List recent workflow runs
gh run list --workflow=preview_on_pr_update.yml

# Get details of a specific run
gh run view <run-id>
```

### Check PR Status
```bash
# View PR details including draft status
gh pr view <pr-number>
```

## Success Criteria

✅ Draft PRs do not trigger preview deployments  
✅ Non-draft PRs trigger preview deployments  
✅ Failed tests prevent deployment  
✅ Passing tests allow deployment  
✅ Converting draft to ready triggers deployment (if tests pass)  
✅ Manual dispatch workflow also respects test requirements  

## Notes

- The `jalantechnologies/github-ci` reusable workflow handles the actual test execution
- The `checks` parameter specifies which tests must pass before deployment
- The workflow uses concurrency groups to cancel in-progress runs when new commits are pushed
