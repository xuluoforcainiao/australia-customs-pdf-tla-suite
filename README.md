# Australia Customs PDF + TLA Suite

End-to-end Australian customs document workflow: convert Excel commercial invoices to PDF, package the converter into a standalone Windows GUI executable, and bundle it with TLA IMS upload tools for offline team deployment.

## Use Cases

- Need a complete loop: Excel to PDF + customs upload
- Colleagues don't have Python installed, need double-click-to-run Windows programs
- Want a tkinter GUI for folder selection and batch conversion
- Need to bundle multiple tools into a single zip distribution package

## Workflow Overview

```
Excel files -> tkinter GUI -> Batch PDF conversion -> Auto launch TLA upload tool
                |
                v
        PyInstaller --onefile --windowed -> Standalone exe
                |
                v
        Zip bundle (PDF tool + TLA tool)
```

## Core Features

- **Batch conversion**: Process all .xlsx files in a folder at once
- **GUI interface**: Source/output folder picker, convert button, real-time log, color-coded status
- **TLA handoff**: After conversion, auto-detect and ask whether to launch TLA upload tool
- **Standalone distribution**: PyInstaller single-file exe, zero environment dependency

## Packaging Command

```bash
python -m PyInstaller --onefile --windowed --name "ExcelToPDF" \
  --distpath "./gui_dist" --workpath "./gui_build" \
  "excel_to_pdf_gui.py"
```

## Bundle Structure

```
CustomsToolSuite/
├── ExcelToPDF.exe
├── README.txt
└── TLAUploadTool/
    ├── start_upload.bat
    ├── TLAUpload.exe
    └── ...
```

## Related Projects

- [australia-customs-excel-to-pdf](https://github.com/xuluoforcainiao/australia-customs-excel-to-pdf) - Core PDF conversion logic
- [toplogistics-customs-upload](https://github.com/xuluoforcainiao/toplogistics-customs-upload) - TLA IMS file upload tool
