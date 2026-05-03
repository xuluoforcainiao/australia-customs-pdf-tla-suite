# Reference: Australia Customs PDF + TLA Suite

## Complete GUI Code Structure

```python
class ConverterApp:
    def __init__(self, root):
        self.root = root
        self.root.title('Excel 转一页纸 PDF 工具')
        self.root.geometry('520x420')
        self.root.resizable(False, False)
        self.root.configure(bg='#f0f2f5')
        self.center_window()

        # Header, inputs, convert button, status, log area (see SKILL.md)
        # ... standard tkinter setup ...

        # TLA hint label (hidden by default)
        self.tla_hint_var = tk.StringVar()
        self.tla_hint = ttk.Label(root, textvariable=self.tla_hint_var,
                                   font=('Microsoft YaHei', 9), foreground='#999')
        self.tla_hint.pack()

    def find_tla_tool(self):
        """Detect TLA upload tool in predefined relative paths."""
        if getattr(sys, 'frozen', False):
            exe_dir = os.path.dirname(sys.executable)
        else:
            exe_dir = os.path.dirname(os.path.abspath(__file__))

        candidates = [
            os.path.join(exe_dir, 'TLA海关查验上传工具包', '启动上传工具.bat'),
            os.path.join(exe_dir, '..', 'TLA海关查验上传工具包', '启动上传工具.bat'),
            os.path.join(exe_dir, 'TLA海关查验上传1.0', '启动上传工具.bat'),
            os.path.join(exe_dir, '..', 'TLA海关查验上传1.0', '启动上传工具.bat'),
            os.path.join(exe_dir, 'TLA海关查验上传工具包', 'TLA海关查验上传.exe'),
            os.path.join(exe_dir, '..', 'TLA海关查验上传工具包', 'TLA海关查验上传.exe'),
        ]
        for path in candidates:
            real_path = os.path.normpath(os.path.abspath(path))
            if os.path.exists(real_path):
                return real_path
        return None

    def launch_tla_tool(self, path):
        try:
            os.startfile(path)
        except Exception as e:
            messagebox.showerror('启动失败', '无法启动 TLA 上传工具:\n' + str(e))

    def on_success(self, log_text, success, failed):
        self.log(log_text)
        color = '#27ae60' if failed == 0 else '#f39c12'
        self.status_var.set('转换完成！成功: {} 个'.format(success))
        self.status.configure(foreground=color)
        self.btn.configure(state='normal', bg='#667eea')

        tla_path = self.find_tla_tool()
        if tla_path:
            result = messagebox.askyesno(
                '转换完成',
                '成功转换 {} 个 Excel 文件为 PDF。\n\n是否立即打开 TLA 上传工具？'.format(success)
            )
            if result:
                self.launch_tla_tool(tla_path)
            self.root.destroy()
        else:
            messagebox.showinfo(
                '转换完成',
                '成功转换 {} 个 Excel 文件为 PDF。\n\n未检测到 TLA 上传工具。'.format(success)
            )
            self.tla_hint_var.set('提示: 未找到 TLA 上传工具')

    def on_error(self, msg):
        self.log('错误: ' + msg)
        self.status_var.set('转换失败: ' + msg)
        self.status.configure(foreground='#e74c3c')
        self.btn.configure(state='normal', bg='#667eea')
```

## PyInstaller Build Commands

### Main GUI Exe

```bash
python -m PyInstaller --onefile --windowed \
  --name "Excel转PDF工具" \
  --distpath "./gui_dist" \
  --workpath "./gui_build" \
  "Excel转PDF工具.py"
```

### ReportLab Python 3.7 Compatibility Fix

ReportLab 4.x uses `md5(data, usedforsecurity=False)` which raises `TypeError` on Python 3.7. Edit these files in your Python `site-packages/reportlab/` directory:

- `pdfdoc.py`
- `canvas.py`
- `cidfonts.py`
- `fontfinder.py`
- `utils.py`

Search for `usedforsecurity=False` and remove the parameter entirely (leave just `md5(data)`).

## Distribution Bundle Creation Script

```python
import os
import shutil

dist_folder = './海关工具套件'
pdf_exe = './gui_dist/Excel转PDF工具.exe'
tla_src = './TLA海关查验上传工具包'

os.makedirs(dist_folder, exist_ok=True)
shutil.copy2(pdf_exe, os.path.join(dist_folder, 'Excel转PDF工具.exe'))

if os.path.exists(os.path.join(dist_folder, 'TLA海关查验上传工具包')):
    shutil.rmtree(os.path.join(dist_folder, 'TLA海关查验上传工具包'))
shutil.copytree(tla_src, os.path.join(dist_folder, 'TLA海关查验上传工具包'))

# Create zip via PowerShell or Python zipfile
```

## BAT File Encoding (Critical)

Windows BAT files MUST use UTF-8 BOM + CRLF. Generating via Python:

```python
lines = [
    '@echo off',
    'chcp 65001 >nul',
    'set "PROG_DIR=%~dp0_internal"',
    'set "PLAYWRIGHT_BROWSERS_PATH=%PROG_DIR%"',
    '"%PROG_DIR%\\TLA海关查验上传.exe"',
]
with open('启动上传工具.bat', 'w', encoding='utf-8-sig', newline='\r\n') as f:
    f.write('\n'.join(lines) + '\n')
```

Never use `echo` with `>` to write bat files — it produces LF-only line endings which cause cmd.exe to silently fail.

## Temp Image Cleanup Pattern

```python
def extract_images(xlsx_path):
    images = []
    tmp_dir = os.path.join(tempfile.gettempdir(), 'excel2pdf_' + str(os.getpid()))
    try:
        os.makedirs(tmp_dir, exist_ok=True)
        z = zipfile.ZipFile(xlsx_path)
        idx = 1
        for name in z.namelist():
            if name.startswith('xl/media/') and not name.endswith('/'):
                data = z.read(name)
                ext = name.split('.')[-1]
                safe_name = os.path.splitext(os.path.basename(xlsx_path))[0]
                out_path = os.path.join(tmp_dir, f'img_{safe_name}_{idx}.{ext}')
                with open(out_path, 'wb') as f:
                    f.write(data)
                images.append(out_path)
                idx += 1
    except Exception:
        pass
    return images, tmp_dir

# After c.save() in convert_single():
if tmp_dir and os.path.exists(tmp_dir):
    try:
        shutil.rmtree(tmp_dir)
    except Exception:
        pass
```

## File References

- `australia-customs-excel-to-pdf` — Core PDF conversion logic and layout
- `toplogistics-customs-upload` — TLA IMS upload automation tool
- `pyinstaller-windows-distribution` — General PyInstaller packaging patterns
