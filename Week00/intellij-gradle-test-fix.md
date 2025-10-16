# IntelliJ IDEA: Fix Gradle Multi-Module Test Runner Issues

## Problem
When trying to run tests in a Gradle multi-module project, IntelliJ shows the error:
```
Cannot locate tasks that match 'order-service:testClasses' as project 'order-service'
not found in root project 'order-service'
```

## Root Cause
IntelliJ imported submodules (like `order-service`) as **separate standalone Gradle projects** instead of recognizing them as submodules of the parent project. This creates duplicate/conflicting Gradle project configurations.

## Solution

### Step 1: Add All Submodules to Parent Settings
Ensure all submodules are included in the parent's `settings.gradle.kts`:

```kotlin
rootProject.name = "microservices-parent"
include("product-service")
include("order-service")
include("inventory-service")
```

### Step 2: Fix gradle.xml Configuration
Edit `.idea/gradle.xml` to have **only ONE** `GradleProjectSettings` block for the parent:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project version="4">
  <component name="GradleMigrationSettings" migrationVersion="1" />
  <component name="GradleSettings">
    <option name="linkedExternalProjectsSettings">
      <GradleProjectSettings>
        <option name="testRunner" value="PLATFORM" />
        <option name="externalProjectPath" value="$PROJECT_DIR$/microservices-parent" />
        <option name="gradleJvm" value="21" />
        <option name="modules">
          <set>
            <option value="$PROJECT_DIR$/microservices-parent" />
            <option value="$PROJECT_DIR$/microservices-parent/inventory-service" />
            <option value="$PROJECT_DIR$/microservices-parent/order-service" />
            <option value="$PROJECT_DIR$/microservices-parent/product-service" />
          </set>
        </option>
      </GradleProjectSettings>
    </option>
  </component>
</project>
```

**Important:** Remove any duplicate `GradleProjectSettings` blocks for individual submodules.

### Step 3: Unlink Duplicate Projects in IntelliJ
1. Open **Gradle tool window** (View → Tool Windows → Gradle)
2. Look for any submodule (e.g., `order-service`) appearing as a **separate root project**
3. Right-click on it → Select **"Unlink Gradle Project"**

### Step 4: Clean Gradle Caches
Close IntelliJ and run these commands:

```bash
rm -rf .idea/modules
rm -rf microservices-parent/.gradle
rm -rf microservices-parent/order-service/.gradle
rm -rf microservices-parent/product-service/.gradle
rm -rf microservices-parent/inventory-service/.gradle
```

### Step 5: Reload Project
1. Reopen IntelliJ IDEA
2. Open **Gradle tool window**
3. Click the **reload icon** (circular arrows) on the parent project
4. Wait for sync to complete

### Step 6: Verify Project Structure
In the Gradle tool window, you should see:
```
microservices-parent
  ├── inventory-service
  ├── order-service
  └── product-service
```

**NOT:**
```
microservices-parent
  ├── order-service
  └── product-service
order-service (separate root)
inventory-service (separate root)
```

### Step 7: Configure Test Runner
**Option A: Via IntelliJ Settings (Recommended)**
1. Go to **Settings/Preferences → Build, Execution, Deployment → Build Tools → Gradle**
2. Find **"Run tests using:"** dropdown
3. Change to **"IntelliJ IDEA"**
4. Click **OK**

**Option B: Via gradle.xml (Manual)**
Already configured if you followed Step 2 above:
```xml
<option name="testRunner" value="PLATFORM" />
```

## Running Tests

### Method 1: From Test Class
1. Navigate to the test file (e.g., `OrderServiceApplicationTests.java`)
2. Right-click on the class name or test method
3. Select **"Run 'OrderServiceApplicationTests'"**

### Method 2: From Gradle Tool Window
1. In Gradle window, expand: `microservices-parent → order-service → Tasks → verification`
2. Double-click **test**

## Alternative: Use Gradle Test Runner
If IntelliJ test runner continues to have issues:

1. Go to **Settings → Build, Execution, Deployment → Build Tools → Gradle**
2. Change **"Run tests using:"** to **"Gradle"**
3. This is slightly slower but more reliable

## Common Issues

### Issue: IntelliJ Re-adds Duplicate Projects After Reload
**Solution:** This happens when IntelliJ auto-detects `build.gradle.kts` files in subdirectories. Always unlink them via the Gradle tool window.

### Issue: Tests Run But Can't Find Dependencies
**Solution:** Ensure `gradleJvm` in `gradle.xml` points to the correct Java version (e.g., "21" or "ms-21").

### Issue: "Project 'X' not found in root project 'X'"
**Solution:** This confirms duplicate project configuration. Re-check `gradle.xml` for multiple `GradleProjectSettings` blocks.

## Prevention
To prevent this issue in future projects:

1. **Always open the parent project directory** in IntelliJ, not individual submodules
2. **Never open submodule directories** as separate IntelliJ projects
3. Let IntelliJ auto-detect the multi-module structure from `settings.gradle.kts`
4. If a submodule gets imported separately, immediately unlink it

## Summary
The key is ensuring IntelliJ treats your multi-module Gradle project as **one parent project with multiple submodules**, not as **multiple independent projects**. The `gradle.xml` file should have exactly ONE `GradleProjectSettings` block listing all modules.
