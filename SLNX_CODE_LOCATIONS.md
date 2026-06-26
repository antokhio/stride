# .slnx Compatibility Issues - Code Locations Reference

## Critical Issue: .slnx Format Parser Not Supported

### 🔴 PRIMARY ISSUE: Hard-coded .sln Text Parser

**Problem**: The solution parser only understands old `.sln` text format, fails on `.slnx` XML format

---

## Call Stack with Line Numbers

| #   | Component     | File                               | Line | Method                                             | Purpose                    |
| --- | ------------- | ---------------------------------- | ---- | -------------------------------------------------- | -------------------------- |
| 1   | **VS Event**  | (Native VS)                        | -    | `IVsSolutionEvents.AfterOpenSolution`              | User opens solution        |
| 2   | **Extension** | `StridePackage.cs`                 | 346  | `solutionEventsListener_AfterSolutionOpened()`     | Event handler triggered    |
| 3   | **Extension** | `StridePackage.cs`                 | 338  | `InitializeCommandProxy()`                         | Initializes command proxy  |
| 4   | **Extension** | `StridePackage.cs`                 | 340  | `dte.Solution.FullName`                            | Gets solution path         |
| 5   | **Commands**  | `StrideCommandsProxy.cs`           | 287  | `SetSolution(solutionPath)`                        | Stores solution path       |
| 6   | **Commands**  | `StrideCommandsProxy.cs`           | 341  | `FindStrideSdkDir(solutionPath)`                   | Finds SDK                  |
| 7   | **Commands**  | `StrideCommandsProxy.cs`           | 174  | `PackageSessionHelper.GetPackageVersion(solution)` | Detects Stride version     |
| 8   | **Parser**    | `PackageSessionHelper.Solution.cs` | 21   | `Solution.FromFile(fullPath)`                      | ⚠️ Tries to parse solution |
| 9   | **Parser**    | `Solution.cs`                      | 177  | `new SolutionReader(solutionFullPath)`             | ⚠️ Creates text parser     |
| 10  | **Parser**    | `SolutionReader.cs`                | 56   | `ReadSolutionFile()`                               | ❌ **FAILS HERE**          |

---

## Code That Needs Fixing

### 1️⃣ Solution.FromFile() - Add Format Detection

**File**: `sources/core/Stride.Core.Design/Solutions/Solution.cs`  
**Line**: 177  
**Current Code**:

```csharp
public static Solution FromFile(string solutionFullPath)
{
    using var reader = new SolutionReader(solutionFullPath);
    var solution = reader.ReadSolutionFile();
    solution.FullPath = solutionFullPath;
    return solution;
}
```

**Issue**: No format detection - assumes `.sln` text format

**Needs**: Check file extension and route to appropriate parser

---

### 2️⃣ SolutionReader.ReadSolutionFile() - Hard-coded Format

**File**: `sources/core/Stride.Core.Design/Solutions/SolutionReader.cs`  
**Lines**: 56-100 (ReadSolutionFile method)

**Current Code Logic**:

```csharp
for (var line = ReadLine(); line != null; line = ReadLine())
{
    if (string.IsNullOrWhiteSpace(line)) continue;

    if (line.StartsWith("Project(", StringComparison.InvariantCultureIgnoreCase))
    {
        solution.Projects.Add(ReadProject(line));
    }
    else if (string.Equals(line, "Global", StringComparison.InvariantCultureIgnoreCase))
    {
        ReadGlobal();
        break;
    }
    else if (RegexParsePropertyLine.Match(line).Success)
    {
        solution.Properties.Add(ReadPropertyLine(line));
    }
    else
    {
        throw new SolutionFileException(
            $"Invalid line read on line #{currentLineNumber}.\n" +
            $"Found: {line}\n" +
            $"Expected: A line beginning with 'Project(' or 'Global'."
        );
    }
}
```

**Issue**:

- Line 64: Expects `Project(` keyword
- Line 68: Expects `Global` keyword
- Line 78: Throws exception for anything else (like XML tags)

**When it fails**:

- `.slnx` starts with `<?xml>` → Exception
- `.slnx` has `<Project>` tags → Exception (different format)
- `.slnx` has `<ItemGroup>` → Exception

---

### 3️⃣ PackageSessionHelper.GetPackageVersion() - Calls Parser

**File**: `sources/assets/Stride.Core.Assets/PackageSessionHelper.Solution.cs`  
**Line**: 21

**Current Code**:

```csharp
public static async Task<PackageVersion?> GetPackageVersion(string fullPath)
{
    try
    {
        // Solution file: extract projects
        var solutionDirectory = Path.GetDirectoryName(fullPath) ?? "";
        var solution = Solution.FromFile(fullPath);  // ← LINE 21: CALLS FAILING PARSER

        foreach (var project in solution.Projects)
        {
            // Process projects...
        }
    }
    catch (Exception e)
    {
        e.Ignore();
    }
    return null;
}
```

**Issue**: Calls `Solution.FromFile()` which fails on `.slnx`

---

## Exception That Gets Thrown

**Type**: `SolutionFileException`  
**Location**: Thrown from `SolutionReader.cs` line ~78

**Message**:

```
Invalid line read on line #3.
Found: <Project Format="0.1" ...>
Expected: A line beginning with 'Project(' or 'Global'.
```

**Result**:

- Exception is caught in `PackageSessionHelper.GetPackageVersion()` (line 16: `catch (Exception e)`)
- Returns `null` for version detection
- Extension treats it as non-Stride solution
- All Stride menus remain hidden/disabled

---

## File Extension Comparison

| File        | Extension | Format           | Supported? | Parser            |
| ----------- | --------- | ---------------- | ---------- | ----------------- |
| Stride.sln  | `.sln`    | Text (key-value) | ✅ YES     | `SolutionReader`  |
| Stride.slnx | `.slnx`   | XML              | ❌ NO      | _(doesn't exist)_ |

---

## Detection Logic Needed

To fix this, `Solution.FromFile()` needs:

```csharp
// Pseudo-code for fix
public static Solution FromFile(string solutionFullPath)
{
    var extension = Path.GetExtension(solutionFullPath);

    if (extension.Equals(".slnx", StringComparison.OrdinalIgnoreCase))
    {
        // Use XML parser for .slnx
        return SlnxReader.FromFile(solutionFullPath);
    }
    else if (extension.Equals(".sln", StringComparison.OrdinalIgnoreCase))
    {
        // Use text parser for .sln (existing code)
        using var reader = new SolutionReader(solutionFullPath);
        var solution = reader.ReadSolutionFile();
        solution.FullPath = solutionFullPath;
        return solution;
    }
    else
    {
        throw new ArgumentException($"Unknown solution format: {extension}");
    }
}
```

---

## Why Build System Already Works

The build system (`build/GenerateDevPackages.cs` line 64) already has awareness:

```csharp
solution = Directory.GetFiles(Path.Combine(strideRoot, "build"), "Stride.slnx")
    .FirstOrDefault()
    ?? Path.Combine(strideRoot, "build", "Stride.sln");
```

This just picks the file but doesn't parse it, so it doesn't hit the parser issue. The VS extension, however, **must** parse the solution to extract Stride version info.

---

## Summary Table

| Aspect             | Details                                                            |
| ------------------ | ------------------------------------------------------------------ |
| **Root Cause**     | Hard-coded `.sln` text parser doesn't support `.slnx` XML format   |
| **Failure Point**  | `SolutionReader.ReadSolutionFile()` regex patterns fail on XML     |
| **Trigger**        | User opens a `.slnx` solution in VS                                |
| **Result**         | Extension can't detect Stride version → Commands disabled          |
| **Affected Files** | 3 files in solution parsing chain                                  |
| **Severity**       | 🔴 **CRITICAL** - Extension completely non-functional with `.slnx` |
| **Workaround**     | Keep solutions in `.sln` format until fixed                        |
| **Fix Complexity** | Medium (requires XML parser for `.slnx`)                           |
