# Security Review Checklist for cclsp Upstream Pulls

This checklist helps identify potential security regressions when pulling changes from the upstream cclsp repository. Review each section carefully before merging upstream changes.

## 🔒 Quick Security Assessment

Before diving into detailed review, quickly assess the scope of changes:

- [ ] **High Risk**: Changes to process spawning, configuration loading, or file operations
- [ ] **Medium Risk**: New MCP tools, dependency updates, or input handling changes
- [ ] **Low Risk**: Documentation, tests, or UI-only changes

## 1. 🚀 Process Execution Security Review

**Critical Files to Review:**
- `src/lsp-client.ts` (lines 127-135, spawn calls)
- `src/setup.ts` (lines 153, 209 - setup spawn calls)
- Any new files that import `child_process`

### Checklist Items:

- [ ] **Command Execution Changes**
  - No new `spawn()`, `exec()`, or `execSync()` calls added
  - Existing spawn calls still use command arrays from config only
  - No string concatenation in command construction
  
- [ ] **Command Array Validation**
  - Commands still extracted from `serverConfig.command` array
  - No direct user input concatenated into commands
  - Command validation logic hasn't been weakened
  
- [ ] **Server Configuration Changes**
  - New server configurations use safe, known LSP servers only
  - No shell injection possibilities in command arrays
  - `rootDir` handling remains secure (no path traversal)

**Red Flags to Watch For:**
```javascript
// DANGEROUS - Avoid these patterns:
exec(`some command ${userInput}`)        // Shell injection
spawn('sh', ['-c', userCommand])         // Shell execution
command.push(userInput)                  // Unvalidated command args
```

## 2. ⚙️ Configuration System Security

**Critical Files to Review:**
- `src/lsp-client.ts` (lines 60-105, config loading)
- `src/language-servers.ts` (server definitions)
- `src/setup.ts` (config generation)

### Checklist Items:

- [ ] **Config Loading Security**
  - Config still loaded only from expected locations (`cclsp.json`, `CCLSP_CONFIG_PATH`)
  - No new environment variables that affect config loading
  - JSON parsing errors handled safely without code execution
  
- [ ] **Config Validation**
  - New config options properly validated
  - No `eval()` or similar dynamic code execution on config values
  - Config schema validation maintained or improved
  
- [ ] **Environment Variable Handling**
  - No new environment variables that control executable paths
  - `CCLSP_CONFIG_PATH` handling unchanged or more secure
  - No environment variable injection vulnerabilities

**Red Flags to Watch For:**
```javascript
// DANGEROUS - Avoid these patterns:
eval(config.someOption)                  // Code execution
require(config.module)                   // Dynamic requires
process.env[config.envVar] = value       // Env manipulation
```

## 3. 📁 File System Operations Security

**Critical Files to Review:**
- `src/file-editor.ts` (file modification logic)
- `index.ts` (file path handling)
- Any new file I/O operations

### Checklist Items:

- [ ] **Path Security**
  - All file paths still resolved using `path.resolve()`
  - No new direct file operations without path validation
  - Path traversal protection maintained (`../` handling)
  
- [ ] **File Modification Safety**
  - Backup creation logic unchanged or improved
  - Symlink handling remains secure
  - File existence validation maintained
  
- [ ] **Permission Handling**
  - No new operations that require elevated privileges
  - File operations limited to project scope
  - No creation of executable files

**Red Flags to Watch For:**
```javascript
// DANGEROUS - Avoid these patterns:
writeFileSync(userPath, content)         // Unvalidated paths
unlinkSync(`../${userInput}`)            // Path traversal
chmodSync(file, 0o777)                   // Executable permissions
```

## 4. ✅ Input Validation & Sanitization

**Critical Files to Review:**
- `index.ts` (MCP tool handlers)
- `src/lsp-client.ts` (LSP request handling)
- New parameter validation logic

### Checklist Items:

- [ ] **MCP Tool Parameters**
  - All tool parameters properly validated
  - File paths resolved before use
  - Symbol names sanitized appropriately
  
- [ ] **LSP Communication**
  - LSP protocol messages properly parsed
  - No direct evaluation of LSP responses as code
  - Error messages don't expose sensitive information
  
- [ ] **User Input Handling**
  - No new direct user input that bypasses validation
  - Input length limits maintained
  - Special characters properly escaped

**Red Flags to Watch For:**
```javascript
// DANGEROUS - Avoid these patterns:
JSON.parse(userInput)                    // No validation
new Function(userCode)                   // Code injection
`template ${userInput} string`           // Template injection
```

## 5. 📦 Dependency & Supply Chain Security

**Critical Files to Review:**
- `package.json` (dependency changes)
- `bun.lock` (lock file updates)
- `src/setup.ts` (auto-installation logic)

### Checklist Items:

- [ ] **Dependency Updates**
  - New dependencies are from trusted sources
  - Version updates are to stable, non-vulnerable versions
  - No dependencies with known security issues
  
- [ ] **Auto-Installation Security**
  - Setup wizard doesn't install untrusted packages
  - Package manager commands properly validated
  - Installation prompts remain user-controlled
  
- [ ] **Build & Runtime Dependencies**
  - No new runtime dependencies that execute code
  - Build tools updates are safe and necessary
  - Peer dependencies properly constrained

**Tools to Help:**
```bash
# Check for known vulnerabilities
npm audit
bun audit

# Verify dependency integrity
npm ci --audit
```

## 6. 🔍 Code Pattern Security Analysis

### Dangerous Patterns to Watch For:

- [ ] **Dynamic Code Execution**
  - `eval()`, `Function()`, `vm.runInContext()`
  - Template literal injection
  - Dynamic `require()` or `import()`
  
- [ ] **Process Execution**
  - New `child_process` usage
  - Shell command construction
  - Process spawning with user input
  
- [ ] **File System Access**
  - Path traversal vulnerabilities
  - Unvalidated file operations
  - Permission escalation attempts
  
- [ ] **Network/Communication**
  - Arbitrary network requests
  - Protocol injection vulnerabilities
  - Unsafe deserialization

## 🚨 Security Regression Indicators

Stop and investigate further if you see:

1. **New `child_process` imports** - Potential command execution
2. **Config structure changes** - May affect validation
3. **New environment variable usage** - Potential privilege escalation
4. **File operation changes** - Risk of path traversal
5. **Error handling modifications** - May expose sensitive info
6. **Validation logic removal** - Direct security regression

## ✅ Final Security Verification

Before approving the upstream pull:

- [ ] Run all tests to ensure security controls still work
- [ ] Review the diff for any bypasses of existing security measures
- [ ] Check if new features introduce new attack vectors
- [ ] Verify that security documentation is updated appropriately
- [ ] Test with potentially malicious inputs if applicable

## 🔧 Tools for Security Review

```bash
# Static security analysis
npm audit
semgrep --config=security .

# Dependency vulnerability check
snyk test

# Code quality review
bun run lint
bun run typecheck

# Full test suite
bun run test:all
```

## 📋 Review Checklist Summary

For each upstream pull, verify:

- [ ] No new arbitrary code execution paths
- [ ] Configuration loading remains secure
- [ ] File operations maintain path validation
- [ ] Input validation not weakened
- [ ] Dependencies are trustworthy
- [ ] No security regressions introduced
- [ ] All tests pass
- [ ] Documentation updated appropriately

---

**Remember**: When in doubt, create a test environment to verify the security of changes before merging into your production fork.