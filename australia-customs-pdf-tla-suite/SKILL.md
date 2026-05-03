---
name: australia-customs-pdf-tla-suite
description: Build an end-to-end Australian customs document workflow that converts commercial invoice Excel files to single-page landscape A4 PDFs, packages the converter into a standalone Windows GUI executable, and bundles it with TLA IMS upload tools for offline team deployment. Use when the user needs to create offline-deployable PDF conversion tools, bundle Excel-to-PDF with customs upload workflows, build tkinter GUI executables via PyInstaller, or distribute Python tools to colleagues without Python installed.
---

# Australia Customs PDF + TLA Suite

## Purpose

Convert Australian customs commercial invoice Excel files to single-page landscape A4 PDFs, wrap the converter in a native Windows GUI executable, and bundle it with the TLA IMS upload tool so colleagues can run the entire workflow without Python installed.

## When to Use

- User needs to convert Excel invoices to PDF and deploy the tool offline
- User wants a GUI window (folder path inputs, convert button, log display)
- User needs to package a Python script into a Windows exe for colleagues
- User wants the PDF converter to automatically hand off to the TLA upload tool
- User mentions "线下部署", "打包成exe", "同事没装Python", "双击运行"
- User wants to bundle multiple tools into a single distribution zip

## Requirements

- Python 3.7+ with `openpyxl`, `reportlab`, `PyInstaller`
- Windows environment for exe packaging
- TLA upload tool package (separate skill: `toplogistics-customs-upload`)

## Workflow Overview

```
Excel files -> tkinter GUI -> batch PDF conversion -> popup ask TLA? -> launch TLA tool
                  |
                  v
          PyInstaller --onefile --windowed -> standalone exe
                  |
                  v
          zip bundle (PDF tool exe + TLA tool folder)
```

## Step 1: Core PDF Conversion

Reuse the conversion logic from `australia-customs-excel-to-pdf`. Key additions for this suite:

- **Temp image cleanup**: Extract embedded images to `tempfile.gettempdir()`, delete via `shutil.rmtree()` after `c.save()`
- **Batch mode**: Process all `.xlsx` files in a source folder, write PDFs to output folder

## Step 2: tkinter GUI

Build a compact single-window GUI:

- Two `tk.Entry` fields for source/output folder paths
- "Start Conversion" button (disabled during processing)
- `tk.Text` log area (8 lines, read-only)
- Status label with color coding (blue=processing, green=success, orange=partial, red=error)
- Background thread for conversion so UI doesn't freeze
- **No auto-close timer** — conversion completion triggers the TLA handoff dialog instead

Critical: All UI updates from the background thread must use `root.after(0, callback)` to schedule on the main thread.

## Step 3: TLA Tool Integration

After `batch_convert()` finishes successfully:

1. Call `find_tla_tool()` to detect the TLA upload tool in predefined relative paths
2. If found: show `messagebox.askyesno("转换完成", "...是否立即打开 TLA 上传工具？")`
   - Yes -> `os.startfile(tla_path)` then `root.destroy()`
   - No -> `root.destroy()`
3. If not found: show `messagebox.showinfo("转换完成", "未检测到 TLA 上传工具...")`, keep window open

### Path Detection Strategy

```python
def find_tla_tool():
    exe_dir = os.path.dirname(sys.executable) if getattr(sys, 'frozen', False) else os.path.dirname(__file__)
    candidates = [
        os.path.join(exe_dir, 'TLA海关查验上传工具包', '启动上传工具.bat'),
        os.path.join(exe_dir, '..', 'TLA海关查验上传工具包', '启动上传工具.bat'),
        os.path.join(exe_dir, 'TLA海关查验上传1.0', '启动上传工具.bat'),
        os.path.join(exe_dir, '..', 'TLA海关查验上传1.0', '启动上传工具.bat'),
    ]
    for path in candidates:
        real_path = os.path.normpath(os.path.abspath(path))
        if os.path.exists(real_path):
            return real_path
    return None
```

Priority: BAT over EXE because the BAT sets `PLAYWRIGHT_BROWSERS_PATH` environment variable.

## Step 4: PyInstaller Packaging

```bash
python -m PyInstaller --onefile --windowed --name "Excel转PDF工具" \
  --distpath "./gui_dist" --workpath "./gui_build" \
  "Excel转PDF工具.py"
```

**Pitfall**: In frozen exe, use `os.path.dirname(sys.executable)` not `os.path.dirname(__file__)` for portable paths.

**Pitfall**: ReportLab 4.x uses `md5(usedforsecurity=False)` which is incompatible with Python 3.7. If this error occurs, edit the ReportLab source files to remove the `usedforsecurity=False` parameter.

**Pitfall**: Any `input()` calls in the script will cause `EOFError` in non-interactive terminals (e.g., launched from HTA or subprocess). Wrap with `try/except EOFError: pass`.

## Step 5: Joint Distribution Bundle

Create a zip containing both tools at the same level:

```
海关工具套件/
├── Excel转PDF工具.exe
├── 使用说明.txt
└── TLA海关查验上传工具包/
    ├── 启动上传工具.bat
    ├── TLA海关查验上传.exe
    ├── 查看运行报表.exe
    └── chromium-1067/
```

Use a Python script to copy files and `Compress-Archive` to create the zip. Do not use `attrib +h` on folders before zipping — it causes `Compress-Archive` to skip them.

## Common Pitfalls

| Issue | Solution |
|-------|----------|
| BAT file silently fails | Must use UTF-8 BOM + CRLF (`encoding='utf-8-sig', newline='\r\n'`) |
| Chinese paths in `file:///` URLs | Browser may crash; copy HTML to `tempfile.gettempdir()` as fallback |
| HTA `shell.Exec` fails with exe | Abandon HTA, use tkinter GUI instead |
| `Compress-Archive` skips hidden folders | Remove `attrib +h` before zipping |
| TLA tool not found after bundling | Ensure relative path matches `find_tla_tool()` candidates |

## Reference

For complete code, build commands, and detailed troubleshooting, see [reference.md](reference.md).
