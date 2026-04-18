# Screenshot Cleanup Logic - Test Plan

## Overview

The cleanup logic in `peekaboo see` automatically removes screenshot files older than the retention period (default: 3 days) from:
1. **User save folder** (e.g., `~/Desktop/Screenshots/`)
2. **Internal snapshot folders** (`~/.peekaboo/snapshots/<id>/`)

**Cleanup Trigger:** Runs on every `peekaboo see` command execution

**Implementation:** `SeeCommand.cleanupOldScreenshots(retentionDays:)`

## Test Scenarios

### Scenario 1: Basic Cleanup - User Save Folder

**Objective:** Verify that old screenshot files in user save folder are deleted

**Setup:**
```bash
# Create test screenshots with different ages
mkdir -p ~/Desktop/Screenshots

# Recent file (should be kept)
echo "recent" > ~/Desktop/Screenshots/recent.png
touch -t $(date -v-1d +%Y%m%d%H%M.%S) ~/Desktop/Screenshots/recent.png

# Old file - 5 days (should be deleted with default 3-day retention)
echo "old" > ~/Desktop/Screenshots/old.png
touch -t $(date -v-5d +%Y%m%d%H%M.%S) ~/Desktop/Screenshots/old.png

# Very old file - 10 days (should be deleted)
echo "very old" > ~/Desktop/Screenshots/very_old.png
touch -t $(date -v-10d +%Y%m%d%H%M.%S) ~/Desktop/Screenshots/very_old.png

# Count before
BEFORE=$(ls ~/Desktop/Screenshots/*.png 2>/dev/null | wc -l)
echo "Before cleanup: $BEFORE files"
```

**Test Command:**
```bash
peekaboo see --app "Google Chrome" --json >/dev/null 2>&1
```

**Expected Result:**
```bash
# Count after
AFTER=$(ls ~/Desktop/Screenshots/*.png 2>/dev/null | wc -l)
echo "After cleanup: $AFTER files"

# Verify
ls ~/Desktop/Screenshots/
# Should show: recent.png only
# old.png and very_old.png should be deleted
```

**Validation:**
- ✅ `recent.png` still exists (1 day old < 3 days retention)
- ✅ `old.png` deleted (5 days > 3 days retention)
- ✅ `very_old.png` deleted (10 days > 3 days retention)

---

### Scenario 2: Cleanup with Custom Retention Period

**Objective:** Verify `--retain-days` flag overrides default retention

**Setup:**
```bash
# Create test files
mkdir -p ~/Desktop/Screenshots

# 2-day old file
echo "two days" > ~/Desktop/Screenshots/two_days.png
touch -t $(date -v-2d +%Y%m%d%H%M.%S) ~/Desktop/Screenshots/two_days.png

# 4-day old file
echo "four days" > ~/Desktop/Screenshots/four_days.png
touch -t $(date -v-4d +%Y%m%d%H%M.%S) ~/Desktop/Screenshots/four_days.png
```

**Test Command (1-day retention):**
```bash
peekaboo see --app "Google Chrome" --retain-days 1 --json >/dev/null 2>&1
```

**Expected Result:**
```bash
ls ~/Desktop/Screenshots/
# Both files should be deleted (both > 1 day)
```

**Test Command (3-day retention):**
```bash
# Recreate files
echo "two days" > ~/Desktop/Screenshots/two_days.png
touch -t $(date -v-2d +%Y%m%d%H%M.%S) ~/Desktop/Screenshots/two_days.png

echo "four days" > ~/Desktop/Screenshots/four_days.png
touch -t $(date -v-4d +%Y%m%d%H%M.%S) ~/Desktop/Screenshots/four_days.png

peekaboo see --app "Google Chrome" --retain-days 3 --json >/dev/null 2>&1
```

**Expected Result:**
```bash
ls ~/Desktop/Screenshots/
# two_days.png kept (2 days < 3 days)
# four_days.png deleted (4 days > 3 days)
```

**Validation:**
- ✅ `--retain-days 1` deletes files > 1 day old
- ✅ `--retain-days 3` deletes files > 3 days old, keeps newer

---

### Scenario 3: Internal Snapshot Folder Cleanup

**Objective:** Verify old image files in `~/.peekaboo/snapshots/` are cleaned

**Setup:**
```bash
# List current snapshots
ls ~/.peekaboo/snapshots/

# Create test snapshot directories
mkdir -p ~/.peekaboo/snapshots/test_old
mkdir -p ~/.peekaboo/snapshots/test_recent

# Create old raw.png (5 days old)
echo "fake image" > ~/.peekaboo/snapshots/test_old/raw.png
touch -t $(date -v-5d +%Y%m%d%H%M.%S) ~/.peekaboo/snapshots/test_old/raw.png

# Create recent raw.png (1 day old)
echo "fake image" > ~/.peekaboo/snapshots/test_recent/raw.png
touch -t $(date -v-1d +%Y%m%d%H%M.%S) ~/.peekaboo/snapshots/test_recent/raw.png

# Verify before
ls -la ~/.peekaboo/snapshots/test_old/
ls -la ~/.peekaboo/snapshots/test_recent/
```

**Test Command:**
```bash
peekaboo see --app "Google Chrome" --json >/dev/null 2>&1
```

**Expected Result:**
```bash
# Check old snapshot - raw.png should be deleted
ls ~/.peekaboo/snapshots/test_old/
# Directory exists but raw.png deleted

# Check recent snapshot - raw.png should be kept
ls ~/.peekaboo/snapshots/test_recent/
# raw.png still exists
```

**Validation:**
- ✅ `test_old/raw.png` deleted (5 days > 3 days)
- ✅ `test_recent/raw.png` kept (1 day < 3 days)
- ✅ Snapshot directories NOT deleted (only image files inside)

---

### Scenario 4: Non-Image Files Not Affected

**Objective:** Verify that non-image files are not deleted

**Setup:**
```bash
mkdir -p ~/Desktop/Screenshots

# Create various file types with old timestamps
echo "text" > ~/Desktop/Screenshots/readme.txt
touch -t $(date -v-10d +%Y%m%d%H%M.%S) ~/Desktop/Screenshots/readme.txt

echo '{"json": true}' > ~/Desktop/Screenshots/data.json
touch -t $(date -v-10d +%Y%m%d%H%M.%S) ~/Desktop/Screenshots/data.json

echo "pdf fake" > ~/Desktop/Screenshots/document.pdf
touch -t $(date -v-10d +%Y%m%d%H%M.%S) ~/Desktop/Screenshots/document.pdf
```

**Test Command:**
```bash
peekaboo see --app "Google Chrome" --json >/dev/null 2>&1
```

**Expected Result:**
```bash
ls ~/Desktop/Screenshots/
# All non-image files still present:
# - readme.txt
# - data.json
# - document.pdf
```

**Validation:**
- ✅ `.txt` files not deleted
- ✅ `.json` files not deleted
- ✅ `.pdf` files not deleted
- ✅ Only `.png`, `.jpg`, `.jpeg` are cleaned

---

### Scenario 5: Cleanup with Empty/Non-existent Folder

**Objective:** Verify cleanup handles missing folders gracefully

**Setup:**
```bash
# Temporarily rename user save folder
mv ~/Desktop/Screenshots ~/Desktop/Screenshots.backup
```

**Test Command:**
```bash
peekaboo see --app "Google Chrome" --json >/dev/null 2>&1
echo "Exit code: $?"
```

**Expected Result:**
```bash
Exit code: 0
# No crash, no error, cleanup silently skips non-existent folder
```

**Restore:**
```bash
mv ~/Desktop/Screenshots.backup ~/Desktop/Screenshots
```

**Validation:**
- ✅ No error when folder doesn't exist
- ✅ Command completes successfully
- ✅ Cleanup silently skips missing directories

---

### Scenario 6: Cleanup on Every `peekaboo see` Execution

**Objective:** Verify cleanup runs on each execution, not just first

**Setup:**
```bash
mkdir -p ~/Desktop/Screenshots

# Create old file
echo "old" > ~/Desktop/Screenshots/should_delete.png
touch -t $(date -v-5d +%Y%m%d%H%M.%S) ~/Desktop/Screenshots/should_delete.png
```

**Test Sequence:**
```bash
# First execution
echo "=== First run ==="
peekaboo see --app "Google Chrome" --json >/dev/null 2>&1
ls ~/Desktop/Screenshots/*.png 2>/dev/null | wc -l

# Create another old file
echo "old2" > ~/Desktop/Screenshots/should_delete2.png
touch -t $(date -v-5d +%Y%m%d%H%M.%S) ~/Desktop/Screenshots/should_delete2.png

# Second execution
echo "=== Second run ==="
peekaboo see --app "Google Chrome" --json >/dev/null 2>&1
ls ~/Desktop/Screenshots/*.png 2>/dev/null | wc -l

# Third execution
echo "=== Third run ==="
peekaboo see --app "Google Chrome" --json >/dev/null 2>&1
ls ~/Desktop/Screenshots/*.png 2>/dev/null | wc -l
```

**Expected Result:**
```bash
=== First run ===
0  # old file deleted
=== Second run ===
0  # second old file deleted
=== Third run ===
0  # no old files to delete
```

**Validation:**
- ✅ Cleanup runs on every `peekaboo see` execution
- ✅ Old files consistently removed
- ✅ No accumulation over multiple runs

---

### Scenario 7: Config-Based Retention Period

**Objective:** Verify `defaults.screenshotRetentionDays` in config.json is respected

**Setup:**
```bash
# Backup current config
cp ~/.peekaboo/config.json ~/.peekaboo/config.json.backup

# Set retention to 7 days
cat > /tmp/config_update.json << 'EOF'
{
  "defaults": {
    "savePath": "~/Desktop/Screenshots",
    "screenshotRetentionDays": 7
  }
}
EOF

# Merge with existing config (using jq if available, or manual edit)
jq -s '.[0] * .[1]' ~/.peekaboo/config.json /tmp/config_update.json > /tmp/merged.json
mv /tmp/merged.json ~/.peekaboo/config.json

# Verify config
cat ~/.peekaboo/config.json | jq '.defaults'
```

**Create Test Files:**
```bash
# 5-day old file (should be kept with 7-day retention)
echo "five days" > ~/Desktop/Screenshots/five_days.png
touch -t $(date -v-5d +%Y%m%d%H%M.%S) ~/Desktop/Screenshots/five_days.png

# 10-day old file (should be deleted with 7-day retention)
echo "ten days" > ~/Desktop/Screenshots/ten_days.png
touch -t $(date -v-10d +%Y%m%d%H%M.%S) ~/Desktop/Screenshots/ten_days.png
```

**Test Command:**
```bash
peekaboo see --app "Google Chrome" --json >/dev/null 2>&1
```

**Expected Result:**
```bash
ls ~/Desktop/Screenshots/
# five_days.png kept (5 days < 7 days)
# ten_days.png deleted (10 days > 7 days)
```

**Restore Config:**
```bash
mv ~/.peekaboo/config.json.backup ~/.peekaboo/config.json
```

**Validation:**
- ✅ Config `screenshotRetentionDays: 7` respected
- ✅ 5-day file kept (< 7 days)
- ✅ 10-day file deleted (> 7 days)

---

## Test Automation Script

Create a comprehensive test script:

```bash
#!/bin/bash
# test_cleanup.sh - Automated cleanup logic testing

set -e

echo "🧪 Testing Screenshot Cleanup Logic"
echo "===================================="
echo ""

PEEKABOO="Apps/CLI/.build/debug/peekaboo"
TEST_DIR="$HOME/Desktop/Screenshots"
SNAPSHOT_DIR="$HOME/.peekaboo/snapshots"

# Helper functions
create_old_file() {
    local file=$1
    local days_old=$2
    echo "fake content" > "$file"
    touch -t $(date -v-${days_old}d +%Y%m%d%H%M.%S) "$file"
}

count_png_files() {
    local dir=$1
    ls "$dir"/*.png 2>/dev/null | wc -l | tr -d ' '
}

# Test 1: Basic cleanup
echo "Test 1: Basic cleanup (3-day default retention)"
mkdir -p "$TEST_DIR"
create_old_file "$TEST_DIR/recent.png" 1
create_old_file "$TEST_DIR/old.png" 5
create_old_file "$TEST_DIR/very_old.png" 10

BEFORE=$(count_png_files "$TEST_DIR")
$PEEKABOO see --app "Google Chrome" --json >/dev/null 2>&1
AFTER=$(count_png_files "$TEST_DIR")

if [ "$AFTER" = "1" ]; then
    echo "✅ PASS: Only recent.png kept ($BEFORE -> $AFTER files)"
else
    echo "❌ FAIL: Expected 1 file, found $AFTER"
    ls "$TEST_DIR"
fi
echo ""

# Test 2: Custom retention
echo "Test 2: Custom retention (--retain-days 1)"
create_old_file "$TEST_DIR/two_days.png" 2
create_old_file "$TEST_DIR/four_days.png" 4

$PEEKABOO see --app "Google Chrome" --retain-days 1 --json >/dev/null 2>&1
AFTER=$(count_png_files "$TEST_DIR")

if [ "$AFTER" = "1" ]; then
    echo "✅ PASS: Only recent.png kept with 1-day retention"
else
    echo "❌ FAIL: Expected 1 file, found $AFTER"
fi
echo ""

# Test 3: Non-image files preserved
echo "Test 3: Non-image files preserved"
create_old_file "$TEST_DIR/readme.txt" 10
create_old_file "$TEST_DIR/data.json" 10

$PEEKABOO see --app "Google Chrome" --json >/dev/null 2>&1

if [ -f "$TEST_DIR/readme.txt" ] && [ -f "$TEST_DIR/data.json" ]; then
    echo "✅ PASS: Non-image files preserved"
else
    echo "❌ FAIL: Non-image files were deleted"
fi
echo ""

# Cleanup test files
rm -rf "$TEST_DIR"/*.png "$TEST_DIR"/*.txt "$TEST_DIR"/*.json 2>/dev/null

echo "===================================="
echo "✅ Cleanup logic tests completed!"
```

---

## Manual Test Checklist

Run these commands manually to verify each scenario:

- [ ] **Test 1:** Basic cleanup - old files deleted, recent kept
- [ ] **Test 2:** Custom retention - `--retain-days 1` works
- [ ] **Test 3:** Custom retention - `--retain-days 7` works
- [ ] **Test 4:** Internal snapshot cleanup - old `raw.png` deleted
- [ ] **Test 5:** Non-image files - `.txt`, `.json` preserved
- [ ] **Test 6:** Missing folder - no crash, graceful handling
- [ ] **Test 7:** Multiple runs - cleanup runs every time
- [ ] **Test 8:** Config retention - `screenshotRetentionDays` respected
- [ ] **Test 9:** Both folders - user + internal cleaned
- [ ] **Test 10:** Image formats - `.png`, `.jpg`, `.jpeg` all cleaned

---

## Success Criteria

All tests pass if:

1. ✅ Files older than retention period are deleted
2. ✅ Files newer than retention period are kept
3. ✅ `--retain-days` flag overrides default/config
4. ✅ Config `screenshotRetentionDays` is respected
5. ✅ Non-image files are never deleted
6. ✅ Missing folders don't cause errors
7. ✅ Cleanup runs on every `peekaboo see` execution
8. ✅ Both user and internal folders are cleaned
9. ✅ Only image files (`.png`, `.jpg`, `.jpeg`) are affected
10. ✅ Snapshot directories preserved (only files inside deleted)

---

**Test Status:** Ready to execute
**Estimated Time:** 15-20 minutes for full test suite
**Risk Level:** Low (test files only, no production data)
