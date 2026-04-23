---
name: epub-translate
description: Use this skill when you need to translate an EPUB file to another language. Combines the EPUB processing workflow with a reliable batch translation approach using sub-agents. Triggers include requests like "translate epub", "translate this book to Chinese", "convert epub to Chinese", etc.
---

# EPUB Translation Guide

This skill combines the EPUB extraction workflow with a batch translation approach.

## Core Workflow (Overview)

```
1. Extract EPUB → 2. Identify files → 3. Batch translate → 4. Verify → 5. Fix metadata → 6. Repack
```

---

## Step 1: Extract the EPUB

EPUB is a ZIP archive. Extract it first:

```bash
# Copy and rename to .zip (Windows)
Copy-Item "source.epub" "source.zip"
Expand-Archive -Path "source.zip" -DestinationPath "book_extracted"

# Or on Linux/Mac
cp source.epub source.zip
unzip -o source.zip -d book_extracted/
```

### Windows-specific commands

```powershell
# Must use .zip extension since PowerShell doesn't recognize .epub
$epub = "C:\path\to\book.epub"
$zip = "C:\path\to\temp.zip"
Copy-Item $epub $zip
Expand-Archive -Path $zip -DestinationPath "C:\path\to\book_extracted"
Remove-Item $zip
```

---

## Step 2: Identify Translation Scope

### Find content files

```bash
# List all XHTML files
Get-ChildItem -Path "book_extracted" -Recurse -Name *.xhtml

# Or on Linux/Mac
find book_extracted/ -name "*.xhtml" | sort
```

### Common EPUB structure

```
book_extracted/
├── mimetype
├── META-INF/
│   └── container.xml
├── *.opf                    # Package file (metadata)
└── [content_folder]/
    ├── xhtml/
    │   ├── ch01.xhtml      # Chapter files
    │   ├── ch02.xhtml
    │   ├── ...
    │   ├── part01.xhtml    # Part section files
    │   ├── toc.ncx         # Table of contents (EPUB2)
    │   ├── nav.xhtml       # Navigation (EPUB3)
    │   └── ...
    ├── images/
    ├── styles/
    └── fonts/
```

### Determine what to translate

| File Type | Translate? | Notes |
|----------|------------|-------|
| Chapters (ch01-ch41) | ✅ Yes | Main content |
| Part files (part01-part06) | ✅ Yes | Section headers |
| toc.ncx | ✅ Yes | Table of contents |
| nav.xhtml | ✅ Yes | Navigation document |
| prologue, bm01 (conclusion) | ❓ Ask user | Optional |
| endnotes, index | ❓ Ask user | Usually skip |
| signup pages | ❓ Ask user | Skip |

---

## Step 3: Batch Translation with Sub-Agents

### Translation Rules (define once, reuse)

```
Name Translations (example for English→Chinese):
- Steve Jobs → 史蒂夫·乔布斯
- Steve Wozniak → 史蒂夫·沃兹尼亚克
- Tim Cook → 蒂姆·库克
- Foxconn → 富士康
- TSMC → 台积电
- Terry Gou → 郭台铭
- Macintosh → 麦金塔

HTML Preservation Rules:
- Keep ALL HTML tags intact
- Keep class names, ids, attributes
- Keep page break markers (<span aria-label="...">)
- Keep footnote references (<span id="ennote..."/>)
- Change lang="en-us" → lang="zh-cn"
- Change xml:lang="en-us" → xml:lang="zh-cn"
```

### Sub-agent Prompt Template

```markdown
Translate `{file_path}` to Chinese.

Rules:
- Translate all English text to Chinese
- Keep HTML tags intact
- Names: Steve Jobs→史蒂夫·乔布斯, Tim Cook→蒂姆·库克, Foxconn→富士康
- Keep lang="zh-cn" xml:lang="zh-cn"

Write back to same file. Return "done" when complete.
```

### Batch Process (3 sub-agents at a time)

```
1. Record original file sizes (before translation)
2. Launch 3 sub-agents (each translates 1 file)
3. Wait for all 3 to complete
4. Verify each file (size > 50% of original)
5. If any fail → retry once
6. All pass → next batch
```

---

## Step 4: Verification (CRITICAL)

### Verification 4-Step Check

| # | Check | Command | Pass Criteria |
|---|-------|---------|---------------|
| 1 | File exists | `ls file.xhtml` | File listed |
| 2 | File size vs original | Compare sizes | Translated > 50% of original |
| 3 | Has Chinese | `head -5 file.xhtml` | Contains Chinese chars |
| 4 | lang attribute | `Select-String "lang" file.xhtml` | Contains "zh-cn" |

### Record Original Sizes (Before Translation)

Before starting batch translation, record original file sizes:

```powershell
# PowerShell: Save original sizes to a hashtable
$originalSizes = @{}
Get-ChildItem "book_extracted\text\*.html" | ForEach-Object { 
    $originalSizes[$_.Name] = $_.Length 
}
```

Or on Linux/Mac:

```bash
# Save original sizes to a file
ls -la book_extracted/text/*.html | awk '{print $9, $5}' > original_sizes.txt
```

### Failure Handling

| Failure | Action |
|---------|--------|
| File not exists | Retry translation immediately |
| File too small (<50% of original) | Retry translation immediately |
| No Chinese content | Retry translation immediately |
| lang="en-us" | Fix with: `(Get-Content $f) -replace 'en-us','zh-cn'` |
| Retry fails | Log error, continue to next file |

### Verification Commands (Windows PowerShell)

```powershell
# Compare translated file size with original (using saved hashtable)
$translatedSize = (Get-Item "part0005.html").Length
$originalSize = $originalSizes["part0005.html"]
$percentage = [math]::Round(($translatedSize / $originalSize) * 100, 1)
Write-Host "Translated: $translatedSize bytes ($percentage% of original)"

if ($translatedSize -lt ($originalSize * 0.5)) { 
    Write-Host "FAIL: Translated file is less than 50% of original size"
}

# Check for lang attribute
Select-String -Path "part0005.html" -Pattern "lang="

# Check first few lines
Get-Content "part0005.html" -TotalCount 5
```

---

## Step 5: Metadata Fix

### OPF File (package metadata)

Edit the `.opf` file to update language:

```xml
<!-- Before -->
<package ... xml:lang="en-us">
  <dc:language>EN-GB</dc:language>

<!-- After -->
<package ... xml:lang="zh-CN">
  <dc:language>ZH-CN</dc:language>
```

### Batch Fix All Lang Attributes

```powershell
# Fix all XHTML files
Get-ChildItem "*.xhtml" | ForEach-Object { 
  (Get-Content $_.FullName) -replace 'lang="en-us"', 'lang="zh-cn"' | 
  Set-Content $_.FullName 
}

# Fix xml:lang too
Get-ChildItem "*.xhtml" | ForEach-Object { 
  (Get-Content $_.FullName) -replace 'xml:lang="en-us"', 'xml:lang="zh-cn"' | 
  Set-Content $_.FullName 
}
```

---

## Step 6: Repack EPUB

### Windows

```powershell
Set-Location "book_extracted"
Compress-Archive -Path "mimetype","META-INF","*.opf","e978*" -DestinationPath "..\translated.epub" -Force

# Rename if needed
Move-Item "translated.epub" "translated_CN.epub"
```

### Linux/Mac

```bash
cd book_extracted/
zip -0 -X ../translated.epub mimetype
zip -r ../translated.epub . --exclude mimetype
mv translated.epub translated_CN.epub
```

---

## Complete Translation Plan Template

### Initial Analysis

1. Extract EPUB
2. List all XHTML files
3. Identify chapter count (e.g., ch01-ch41)
4. Identify part files (e.g., part01-part06)
5. Identify TOC files (toc.ncx, nav.xhtml)
6. Ask user what to translate

### Plan to Show User(Example)

```
Translation Scope:
- 41 chapters (ch01-ch41)
- 6 part files
- 2 TOC files (toc.ncx, nav.xhtml)
- Total: 49 files

Process:
- 17 batches × 3 files = 49 files
- Each batch: translate → verify (4 checks)
- If verify fails → retry once
- After all: metadata fix → pack → final verify

Estimated time: ~30-60 minutes
```

### Per-Batch Execution

```
Batch N: ch0X, ch0Y, ch0Z
├─ Launch 3 sub-agents
├─ Wait for completion
├─ Verify:
│  ├─ ls -la ch0X.html     → exists?
│  ├─ compare size         → >50% of original?
│  ├─ head -5              → has Chinese?
│  └─ grep "lang"         → zh-cn?
└─ Result: all pass → next batch
```

---

## Quick Reference

| Task | Command |
|------|---------|
| Extract EPUB | `Expand-Archive -Path *.epub -DestinationPath dir` |
| List content | `Get-ChildItem -Recurse -Name *.xhtml` |
| Record original sizes | `$originalSizes = @{...}` or `ls > sizes.txt` |
| Batch translate | Launch 3 sub-agents, wait, verify |
| Verify file size | Compare translated vs original (must be >50%) |
| Check lang | `Select-String -Path file.xhtml -Pattern "lang="` |
| Batch fix lang | `Get-ChildItem *.xhtml \| %{(Get-Content $_) -replace 'en-us','zh-cn'}` |
| Pack EPUB | `Compress-Archive -Path * -DestinationPath out.epub` |
| Verify pack | `Expand-Archive -Path out.epub -DestinationPath test` |

---

## Combined EPUB Skill (Merged)

For reference, here's the complete EPUB processing merged with translation:

### EPUB is a ZIP Archive

```
.epub = ZIP with specific structure
```

### Extraction

```bash
# Copy, rename, extract (Windows)
cp book.epub book.zip
Expand-Archive -Path book.zip -DestinationPath book_extracted/
```

### Navigation File Priority

1. `nav.xhtml` (EPUB3)
2. `toc.ncx` (EPUB2)
3. Files with "nav" or "toc" in name

### OPF File

Contains metadata and reading order. Located via `META-INF/container.xml`.

### Verify Structure

```
book_extracted/
├── mimetype
├── META-INF/container.xml
├── *.opf
└── [folder]/
    ├── xhtml/
    │   ├── ch01.xhtml
    │   ├── nav.xhtml
    │   └── toc.ncx
    └── images/
```

### Troubleshooting

| Issue | Solution |
|-------|----------|
| No nav file | `find . -name "*.xhtml" \| xargs grep -l "epub:type"` |
| Encoding error | Use `encoding="utf-8", errors="ignore"` |
| Namespace in XML | Pass `ns` dict to `find`/`findall` |

---

## Required Tools

- PowerShell or bash
- Sub-agent capability (for parallel translation)
- Text editor (for metadata fixes)

No special Python packages required for basic operations.
