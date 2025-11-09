# WebSocketPP Build Failure - Debug & Fixes

## 🐛 Issue Summary

**Build Environment:** ARM64 ManyLinux 2014 (GitHub Actions)  
**Failed Package:** `websocketpp`  
**Successful Package:** `boost_full`

## 🔍 Root Cause Analysis

### Primary Issue: Incorrect Dependency Name
The `websocketpp` package configuration referenced `"boost"` as a dependency, but the actual package name is `"boost_full"`. This caused the dependency resolution to fail.

```python
# ❌ Before (incorrect)
"dependencies": ["boost"],

# ✅ After (correct)
"dependencies": ["boost_full"],
```

### Secondary Issues Identified

1. **Missing CMake Configuration Flags**
   - `BUILD_TESTS` and `BUILD_EXAMPLES` should be explicitly disabled
   - These can cause build failures if dependencies are missing

2. **EOF Newline Missing**
   - The GitHub Actions workflow file was missing a newline at the end

## ✅ Applied Fixes

### Fix 1: Corrected Dependency Names

**File:** `pack.py`

```python
# websocketpp configuration
"https://github.com/AI-Infra-Team/websocketpp":{
    "branch": "master",
    "c++": 17,
    "dependencies": ["boost_full"],  # ✅ Changed from "boost"
    "build_type": "Release",
    "define": [
        ["BUILD_STATIC_LIBS", "ON"],
        ["BUILD_SHARED_LIBS", "OFF"],
        ["BUILD_TESTS", "OFF"],        # ✅ Added
        ["BUILD_EXAMPLES", "OFF"],     # ✅ Added
    ],
},

# cpprestsdk configuration
"https://github.com/AI-Infra-Team/cpprestsdk":{
    "branch": "master",
    "c++": 17,
    "dependencies": ["websocketpp","boost_full"],  # ✅ Changed from "boost"
    # ... rest of config
},
```

### Fix 2: Added EOF Newline

**File:** `.github/workflows/build-and-release.yml`

Added newline at the end of the file (line 326).

## 🧪 How to Test

### Local Testing (Docker)

```bash
# Test ARM64 build on ManyLinux 2014
python3 pack_in_container.py --system-name manylinux_2014 --arch arm64

# Test AMD64 build (faster for local testing)
python3 pack_in_container.py --system-name manylinux_2014 --arch amd64

# Test on Ubuntu
python3 pack_in_container.py --system-name ubuntu22.04 --arch amd64
```

### GitHub Actions Testing

1. Create a test tag:
```bash
git add pack.py .github/workflows/build-and-release.yml
git commit -m "Fix websocketpp build failure"
git push origin master

# Create a test tag
git tag v0.1.0-test
git push origin v0.1.0-test
```

2. Monitor the GitHub Actions workflow
3. Check that all 6 build variants complete successfully

## 🔍 Additional Debugging Tips

### 1. Check Build Logs

If the build fails again, check these logs:

```bash
# In the GitHub Actions artifacts or local .output_logs/
cat .output_logs/build_report.json
cat .output_logs/websocketpp_build_errors/CMakeError.log
```

### 2. Verify Boost Installation

```bash
# After boost_full builds, verify it's installed correctly
ls -la output/include/boost/
ls -la output/lib/libboost_*
```

### 3. Test WebSocketPP CMake Configuration

```bash
# Manually test websocketpp configuration
cd build/websocketpp/build
cmake .. \
  -DCMAKE_PREFIX_PATH=/workspace/output \
  -DCMAKE_INSTALL_PREFIX=/workspace/output \
  -DBUILD_TESTS=OFF \
  -DBUILD_EXAMPLES=OFF
```

## 🚨 Known Issues & Workarounds

### Issue: WebSocketPP is Header-Only

WebSocketPP is primarily a header-only library, so it might not produce any libraries. The build should still install the headers correctly.

**Expected result:**
```
output/
  include/
    websocketpp/
      *.hpp
```

### Issue: ARM64 QEMU Emulation Performance

ARM64 builds on GitHub Actions (which run on AMD64) use QEMU emulation and are **2-5x slower**.

**Workaround:**
- Use self-hosted ARM64 runners for production
- Pre-build packages and cache them
- Run builds in parallel

## 📊 Expected Build Results

After the fixes, you should see:

```
Build Results: 2/2 packages successful

Package Status:
------------------------------
✓ boost_full: Built successfully (custom command)
✓ websocketpp: Built successfully
```

## 🔄 Next Steps

1. **Commit and push the fixes**
   ```bash
   git add pack.py .github/workflows/build-and-release.yml DEBUG_FIXES.md
   git commit -m "Fix websocketpp dependency and build configuration"
   git push origin master
   ```

2. **Test the build**
   - Push a test tag or manually trigger the workflow
   - Verify all 6 build combinations succeed

3. **Monitor for other issues**
   - Check if cpprestsdk builds correctly after websocketpp
   - Verify all packages in the dependency chain

## 📝 Additional Recommendations

### 1. Add Retry Logic for Network Issues

Consider adding retry logic for git clone operations:

```python
def clone_repository(self, url: str, branch: str, target_dir: Path) -> bool:
    """克隆Git仓库 with retry logic"""
    max_retries = 3
    for attempt in range(max_retries):
        try:
            # ... existing clone logic ...
            return True
        except Exception as e:
            if attempt < max_retries - 1:
                logger.warning(f"Clone attempt {attempt + 1} failed, retrying...")
                time.sleep(5)
            else:
                logger.error(f"Failed to clone after {max_retries} attempts")
                return False
```

### 2. Add Build Caching

For faster CI/CD, add Docker layer caching:

```yaml
- name: Cache Docker layers
  uses: actions/cache@v3
  with:
    path: /tmp/.buildx-cache
    key: ${{ runner.os }}-buildx-${{ hashFiles('pack.py') }}
```

### 3. Improve Error Messages

Add more descriptive error messages when dependencies are not found:

```python
if not dep_url:
    available_packages = [self.get_package_name(u) for u in packages]
    logger.error(
        f"Dependency '{dep_name}' not found for '{self.get_package_name(url)}'"
    )
    logger.error(f"Available packages: {', '.join(available_packages)}")
```

## 🎯 Success Criteria

- ✅ `boost_full` builds successfully
- ✅ `websocketpp` builds successfully
- ✅ `cpprestsdk` builds successfully (depends on websocketpp)
- ✅ All 6 build variants (3 systems × 2 architectures) complete
- ✅ Output packages contain expected files

## 📚 References

- [WebSocketPP GitHub](https://github.com/zaphoyd/websocketpp)
- [Boost Documentation](https://www.boost.org/)
- [ManyLinux Docker Images](https://github.com/dockcross/dockcross)
- [GitHub Actions QEMU Support](https://github.com/docker/setup-qemu-action)
