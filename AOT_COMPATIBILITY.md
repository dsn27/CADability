# AOT (Ahead-of-Time) Compilation Compatibility

## Overview

This document outlines AOT compilation compatibility considerations for CADability and related tooling choices.

## What is AOT?

AOT (Ahead-of-Time) compilation is a feature in .NET 6+ that compiles your application to native code before runtime, providing:
- Faster startup times
- Reduced memory footprint
- Smaller deployment size (with trimming)
- No JIT compilation at runtime

Enable with: `<PublishAot>true</PublishAot>` in your .csproj file.

## Current CADability Status

### Target Frameworks
- **CADability.dll**: `netstandard2.0` - No AOT support (pre-.NET 6)
- **CADability.App**: `net48` - No AOT support (.NET Framework)
- **CADability.Tests**: `net6.0-windows` - AOT capable but not enabled

### ILRepack Status
- **Currently NOT implemented** - only documented as an option in `STRONG_NAME_RUNTIME_ERROR.md`
- If implemented, would be incompatible with AOT

## AOT Compatibility Issues

### 1. ILRepack and AOT

**Problem**: ILRepack is **NOT compatible** with AOT compilation.

**Why**:
- ILRepack performs post-build IL (Intermediate Language) manipulation
- Merges multiple assemblies by rewriting IL code and metadata
- AOT compiler requires static analysis of original IL
- Post-build manipulation breaks AOT's static analysis
- Can cause runtime failures or compilation errors

**Microsoft's Position**:
- Post-build IL manipulation tools are not supported with PublishAot
- Includes ILMerge, ILRepack, and similar tools

**Evidence**:
- GitHub issue: https://github.com/dotnet/runtime/issues/68038
- AOT requires all code paths known at compile time
- ILRepack obscures code paths from AOT analyzer

### 2. Strong-Name Signing and AOT

**Good News**: Strong-name signing is compatible with AOT.

**Note**: Modern .NET (Core/.NET 5+) largely ignores strong-name verification at runtime, so strong-name issues are primarily .NET Framework concerns.

## Recommendations by Scenario

### Scenario 1: Staying on .NET Framework
**Status**: ✅ ILRepack is safe to use

- .NET Framework doesn't support AOT
- ILRepack works fine for strong-name scenarios
- Follow recommendations in `STRONG_NAME_RUNTIME_ERROR.md`

### Scenario 2: Targeting .NET 6+ WITHOUT AOT
**Status**: ✅ ILRepack is safe to use

- If you don't enable `<PublishAot>true</PublishAot>`
- ILRepack still works as a post-build tool
- No compatibility issues

### Scenario 3: Targeting .NET 6+ WITH AOT
**Status**: ❌ Do NOT use ILRepack

**Alternatives**:

#### Option A: Source Embedding (Recommended)
Vendor the ACadSharp source code directly into CADability:
```xml
<ItemGroup>
  <Compile Include="..\ThirdParty\ACadSharp\**\*.cs" LinkBase="ACadSharp" />
</ItemGroup>
```
**Pros**: AOT compatible, full control, single assembly
**Cons**: Maintenance burden, need to track upstream changes

#### Option B: Separate Assemblies
Keep ACadSharp as a separate assembly:
- Modern .NET doesn't enforce strong-name checks
- Let the AOT compiler handle both assemblies
- Use trimming to reduce size

**Pros**: Simple, AOT compatible
**Cons**: Multiple assembly files to deploy

#### Option C: Request Signed Package
Request ACadSharp maintainers to publish strong-name signed packages:
- Open issue: https://github.com/DomCR/ACadSharp/issues
- Wait for official support

**Pros**: Official, no maintenance
**Cons**: May take time or be rejected

## Testing AOT Compatibility

If you plan to enable AOT in the future, test early:

### 1. Enable AOT in Test Project
```xml
<PropertyGroup>
  <PublishAot>true</PublishAot>
  <InvariantGlobalization>false</InvariantGlobalization>
</PropertyGroup>
```

### 2. Run Publish
```bash
dotnet publish -c Release
```

### 3. Check for Warnings
Look for:
- `IL2026`: Using reflection in AOT
- `IL3050`: AOT analysis warnings
- Build failures with ILRepack

### 4. Test Runtime
```bash
./bin/Release/net8.0/win-x64/publish/YourApp.exe
```

## Migration Path for AOT

If you currently use ILRepack and want to enable AOT:

1. **Remove ILRepack**
   - Remove ILRepack NuGet packages
   - Remove post-build targets from .csproj

2. **Choose Alternative**
   - Source embedding (best for single assembly)
   - Separate assemblies (simplest)
   - Wait for signed ACadSharp package

3. **Test AOT Build**
   - Enable PublishAot
   - Fix any AOT warnings
   - Test thoroughly

4. **Validate Runtime**
   - Test all code paths
   - Verify reflection usage
   - Check for missing dependencies

## Best Practices

### DO ✅
- Consider future AOT compatibility when making architectural decisions
- Use source embedding or separate assemblies for AOT scenarios
- Test AOT compatibility early if targeting .NET 6+
- Keep strong-name signing (it's AOT compatible)

### DON'T ❌
- Use ILRepack if you plan to enable AOT
- Use ILMerge or similar IL manipulation tools with AOT
- Enable AOT without thorough testing
- Assume ILRepack warnings can be ignored in AOT scenarios

## Future Outlook

### .NET Trends
- AOT is becoming more important for cloud-native apps
- Trimming and AOT reduce costs in serverless scenarios
- Desktop apps benefit less but still gain startup performance

### CADability Considerations
- Current netstandard2.0 target is pre-AOT
- net48 App target is .NET Framework (no AOT)
- If modernizing to .NET 8+, consider AOT-friendly architecture
- Avoid ILRepack in new projects targeting modern .NET

## Summary

| Tool/Approach | .NET Framework | .NET 6+ (no AOT) | .NET 6+ (with AOT) |
|---------------|----------------|------------------|-------------------|
| **ILRepack** | ✅ Safe | ✅ Safe | ❌ **NOT Compatible** |
| **Source Embedding** | ✅ Safe | ✅ Safe | ✅ Safe |
| **Separate Assemblies** | ⚠️ Strong-name issues | ✅ Safe | ✅ Safe |
| **Strong-Name Signing** | ✅ Safe | ✅ Safe | ✅ Safe |

## References

- [.NET AOT Deployment](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/)
- [AOT Compatibility Requirements](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/compatibility)
- [ILRepack GitHub Issue on AOT](https://github.com/gluck/il-repack/issues/370)
- [Microsoft's Native AOT Docs](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/)

---

**Last Updated**: 2026-02-15  
**Status**: ILRepack NOT currently implemented in CADability  
**Recommendation**: Document AOT incompatibility before implementing ILRepack
