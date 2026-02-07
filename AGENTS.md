# AGENTS.md - Development Guide for GFWList2AGH

This document provides guidelines for coding agents working on the GFWList2AGH project, which converts GFWList and China domain lists into DNS server configuration formats (AdGuard Home, Bind9, DNSMasq, SmartDNS, Unbound, etc.).

## Project Overview

**Type**: Shell script-based DNS configuration generator  
**Language**: Bash (POSIX-compliant shell scripts)  
**License**: Apache License 2.0 with Commons Clause v1.0  
**Purpose**: Fetch domain lists from multiple sources, process them, and generate DNS configuration files for various DNS servers

## Build & Test Commands

### Primary Build Command
```bash
# Run the complete build process (fetches data, processes, generates all outputs)
bash release.sh
```

### Testing Individual Components
```bash
# Test data fetching only (creates Temp directory with intermediate files)
bash release.sh  # Then Ctrl+C after GetData completes, inspect ./Temp

# Validate generated output format for AdGuard Home
head -20 gfwlist2adguardhome/blacklist_full.txt

# Validate generated output format for DNSMasq
head -20 gfwlist2dnsmasq/blacklist_full.conf

# Check output file counts (should be 8 files per directory)
ls -1 gfwlist2adguardhome/ | wc -l
```

### Validation Commands
```bash
# Verify no duplicate domains in output files
sort gfwlist2domain/blacklist_full.txt | uniq -d

# Check domain format validity (basic regex test)
grep -v -E '^[a-z0-9]([a-z0-9-]*[a-z0-9])?(\.[a-z0-9]([a-z0-9-]*[a-z0-9])?)*$' gfwlist2domain/blacklist_full.txt

# Verify output directories were created
ls -ld gfwlist2* | grep ^d
```

### CI/CD
- **GitHub Actions**: `.github/workflows/main.yml`
- **Schedule**: Daily at 00:00 UTC via cron
- **Manual trigger**: Available via workflow_dispatch

## Code Style Guidelines

### Shell Script Conventions

#### File Structure
- **Single file architecture**: All logic in `release.sh`
- **Function-based**: Organized into `GetData()`, `AnalyseData()`, `GenerateRules()`, `OutputData()`
- **No external dependencies**: Pure bash + standard Unix tools (curl, sed, grep, awk, sort, base64)

#### Naming Conventions
- **Functions**: PascalCase (e.g., `GetData`, `GenerateRules`, `FileName`)
- **Variables**: snake_case (e.g., `cnacc_domain`, `gfwlist_data`, `software_name`)
- **Arrays**: snake_case with plural names (e.g., `domestic_dns`, `foreign_dns`)
- **Loop counters**: `<array_name>_task` (e.g., `cnacc_domain_task`, `foreign_dns_task`)

#### Variable Scope
```bash
# Global arrays for data sources (defined at function start)
cnacc_domain=(...)
gfwlist_base64=(...)

# Local function variables (no explicit 'local' keyword used in this codebase)
# Variables are scoped by function context

# Parameters passed via environment variables
software_name="adguardhome"
generate_file="black"
generate_mode="full"
dns_mode="default"
```

#### String Quoting
- **URLs and paths**: Always use double quotes `"${variable}"`
- **Commands**: Use quotes for arguments: `sed "s/^\.//g"`
- **Array expansion**: Use `"${!array[@]}"` for indices, `"${array[@]}"` for values

#### Array Iteration
```bash
# Correct pattern used throughout codebase
for array_name_task in "${!array_name[@]}"; do
    process "${array_name[$array_name_task]}"
done
```

#### Pipes and Redirects
- **Append to temp files**: `>> ./file.tmp`
- **Overwrite with echo**: `echo "content" > "${file_path}"`
- **Inline processing**: `curl | sed | grep | sort | uniq`
- **Silent curl**: Always use `-s --connect-timeout 15`

### Data Processing Patterns

#### Domain Filtering
```bash
# Standard domain regex (used throughout)
domain_regex="^(([a-z]{1})|([a-z]{1}[a-z]{1})|([a-z]{1}[0-9]{1})|([0-9]{1}[a-z]{1})|([a-z0-9][-\.a-z0-9]{1,61}[a-z0-9]))\.([a-z]{2,13}|[a-z0-9-]{2,30}\.[a-z]{2,3})$"

# Lite domain regex (for top-level domains only)
lite_domain_regex="^([a-z]{2,13}|[a-z0-9-]{2,30}\.[a-z]{2,3})$"
```

#### Custom Rule Syntax (data/data_modify.txt)
- **Add to both lists**: `(@@@)example.org`
- **Remove from both**: `(!!!)example.org`
- **Exclude pattern**: `(***)example.org`
- **Add to China list only**: `(@%@)example.org`
- **Add to GFW list only**: `(@&@)example.org`
- **Cross-list operations**: `(@%!)`, `(!%@)`, `(@&!)`, `(!&@)`

### Output Format Generation

#### Case Statement Structure
```bash
case ${software_name} in
    adguardhome)
        # Define DNS servers
        domestic_dns=(...)
        foreign_dns=(...)
        
        # Define format functions
        function GenerateRulesHeader() { ... }
        function GenerateRulesBody() { ... }
        function GenerateRulesFooter() { ... }
    ;;
    other_software)
        # Similar structure
    ;;
    *)
        exit 1
    ;;
esac
```

#### Output File Naming
- **Pattern**: `{blacklist|whitelist}_{full|lite}[_combine].{txt|conf}`
- **Full**: Complete domain list with all subdomains
- **Lite**: Top-level domains only
- **Combine**: Single-line format (for AdGuard Home)

### Error Handling

#### Current Practice
- **No explicit error checking** for curl/network failures
- **Silent failures**: Most commands use `|| true` implicitly via pipes
- **Exit on unknown software**: `*) exit 1 ;;` in case statements

#### Best Practices for New Code
```bash
# Add error checking for critical operations
if ! curl -s --connect-timeout 15 "${url}" > file.tmp; then
    echo "Error: Failed to fetch ${url}" >&2
fi

# Validate intermediate files exist before processing
if [ ! -f "./cnacc_domain.tmp" ]; then
    echo "Error: Missing intermediate file" >&2
    exit 1
fi
```

### Comments

#### Function Documentation
```bash
## Function
# Get Data
function GetData() {
    # Array definitions with source URLs
    cnacc_domain=(...)
}
```

#### Inline Comments
- **Minimal**: Code is mostly self-documenting via function names
- **Section markers**: Use `##` for major sections
- **Commented code**: DNS server options shown as examples with `#` prefix

### File Organization

#### Directory Structure
```
.
├── .github/workflows/main.yml    # CI/CD configuration
├── data/data_modify.txt          # Custom domain rules
├── release.sh                    # Main build script
├── LICENSE                       # Apache 2.0 + Commons Clause
├── gfwlist2adguardhome/          # Generated AdGuard Home configs
├── gfwlist2adguardhome_new/      # New AdGuard Home format
├── gfwlist2bind9/                # Generated Bind9 configs
├── gfwlist2dnsmasq/              # Generated DNSMasq configs
├── gfwlist2domain/               # Plain domain lists
├── gfwlist2smartdns/             # Generated SmartDNS configs
└── gfwlist2unbound/              # Generated Unbound configs
```

#### Temporary Files
- **Location**: `./Temp/` (created during build, deleted after)
- **Naming**: `*.tmp` suffix
- **Cleanup**: `rm -rf ./Temp` at start and end

## Making Changes

### Adding New Data Sources
1. Add URL to appropriate array in `GetData()` function
2. Ensure download loop appends to correct `.tmp` file
3. Test with full build to verify data integration

### Adding New DNS Software Support
1. Add new case in `GenerateRules()` function
2. Define DNS server arrays (domestic_dns, foreign_dns)
3. Implement three functions: `GenerateRulesHeader()`, `GenerateRulesBody()`, `GenerateRulesFooter()`
4. Add output calls in `OutputData()` function
5. Test all modes: full, lite, black, white, combine

### Modifying Custom Rules
- Edit `data/data_modify.txt` following syntax patterns
- Use markers: `@` (add), `!` (remove), `*` (exclude)
- Combine markers for cross-list operations: `%` (China only), `&` (GFW only)

### Output Format Changes
1. Locate the relevant case block in `GenerateRules()`
2. Modify `GenerateRulesHeader()`, `GenerateRulesBody()`, or `GenerateRulesFooter()`
3. Test with: `bash release.sh && head -50 gfwlist2{software}/{type}.{ext}`

## Common Pitfalls

1. **Long lines**: The script has very long lines (6000+ chars). Be careful with line continuations.
2. **Array indices**: Always use `"${!array[@]}"` for indices, not `"${array[@]}"`.
3. **Temp directory**: Ensure `./Temp` cleanup doesn't break if script exits early.
4. **Silent failures**: Network errors are not caught; add validation if critical.
5. **Regex escaping**: Domain regexes use `\.` for literal dots; be careful with escaping.

## Testing Checklist

- [ ] Run `bash release.sh` completes without errors
- [ ] All 6 output directories contain 4-8 files each
- [ ] Sample files from each directory have correct format
- [ ] No duplicate domains in output (test with `sort | uniq -d`)
- [ ] File sizes are reasonable (>100KB for full lists, >50KB for lite)
- [ ] Custom rules in `data/data_modify.txt` are applied correctly
- [ ] GitHub Actions workflow validates changes
