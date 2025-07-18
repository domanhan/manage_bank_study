"""
Created on Sat Jul 20 11:00:00 2025
Final version with a fully functional cascading filter and bug fixes.
@author: manha (with AI assistance)
"""

import tkinter as tk
from tkinter import ttk, filedialog, simpledialog, scrolledtext, messagebox
import os
import re
import subprocess
import tempfile
import shutil
import threading
import hashlib

# Cần cài đặt: pip install Pillow PyMuPDF
try:
    from PIL import Image, ImageTk
    import fitz  # PyMuPDF
except ImportError:
    messagebox.showerror("Thiếu thư viện", "Vui lòng cài đặt: pip install Pillow PyMuPDF")
    exit()

# --- CÀI ĐẶT CHUNG ---
DEFAULT_QUESTION_BANK_DIR = "NganHangCauHoi"
DEFAULT_IMAGE_BANK_DIR = "ImageBank"
STAGING_DIR_NAME = "KhoTam"
UNASSIGNED_DIR_NAME = "ChuaID"
STAGING_IMAGE_DIR_NAME = "AnhTam"
FONTS = {'normal': ("Arial", 11), 'code': ("Consolas", 10), 'bold': ("Arial", 11, "bold")}

# === LATEX_PREAMBLE (Không thay đổi) ===
LATEX_PREAMBLE_FOR_RENDER = r"""
\documentclass[12pt,tikz]{standalone}
\usepackage[utf8]{vietnam}
\usepackage{amsmath,amssymb,yhmath,mathrsfs}
\usepackage{fancyhdr,multicol}
\usepackage[hidelinks]{hyperref}
\usepackage{enumerate}
\usepackage[many]{tcolorbox}
\usepackage{tikz,tkz-euclide,tikz-3dplot,tkz-tab}
\usetikzlibrary{shapes.geometric,arrows,calc,intersections,angles,patterns,snakes}
\usepackage[image]{ex_forms}
\TuyChonExForms[de]{image}
\TuyChonLoigiai[ex]{An}
\def\congtac{An}
\AnHienSTTex{HienHien}
\TuyChonDongKeBang[step=0.25,gray,opacity=0.3]{Hien}
\LoaiBang{green}
\def\ChieuNgangEXForms{13cm}
\def\ChieuDocEXForms{0cm}
\usepackage{ifthen}
\usepackage{xparse}
\usepackage{etoolbox}
\newenvironment{alphaenum}{%
	\begin{enumerate}
		\renewcommand{\labelenumi}{\alph{enumi})}%
		\renewcommand{\labelenumii}{\roman{enumii}.}%
	}{\end{enumerate}}
\newcommand{\choiceTF}[5][]{%
	\begin{alphaenum}
		\item #2
		\item #3
		\item #4
		\item #5
	\end{alphaenum}%
}
\let\True\relax
\newenvironment{itemchoice}{\begin{alphaenum}}{\end{alphaenum}}
\let\itemch\item
\NewDocumentCommand{\shortans}{ O{} m }{
	\par\noindent\textbf{Kết quả:~}%
	\ifthenelse{\equal{\congtac}{Hien}}
	{
		\begin{tikzpicture}[baseline=-0.3ex]
			\node[draw, fill=yellow!20, inner sep=2pt] {#2};
		\end{tikzpicture}%
	}
	{
		\begin{tikzpicture}[baseline=-0.3ex]
			\node[draw, minimum width=3.2cm, minimum height=0.6cm] {};
		\end{tikzpicture}%
	}
}
\begin{document}
"""
LATEX_POSTAMBLE = r"\end{document}"

# --- LỚP DỮ LIỆU & HÀM LOGIC ---

class ParsedQuestion:
    # MODIFIED: Nâng cấp lớp ParsedQuestion để chứa nhiều thông tin chi tiết hơn
    def __init__(self, file_path, seq_num, env_type, base_id, full_content):
       self.file_path = file_path
       self.seq_num = seq_num
       self.env_type = env_type
       self.base_id = base_id
       self.full_content = full_content
       
       file_hash = hashlib.md5(os.path.normpath(file_path).encode()).hexdigest()[:8]
       self.unique_id = f"{base_id}#{seq_num}_{file_hash}"
       self.image_path = os.path.join(DEFAULT_IMAGE_BANK_DIR, f"{self.unique_id}.png")
       
       # --- CÁC DÒNG ĐƯỢC THÊM LẠI ĐỂ SỬA LỖI ---
       self.status = self.get_status()
       self.question_type = self.get_question_type()
       # -----------------------------------------

       img_match = re.search(r'\\includegraphics.*?\{([^}]*)\}', full_content)
       self.included_image_path = img_match.group(1) if img_match else None
       
       self.grade, self.subject, self.level = None, None, None
       self.chapter_key, self.lesson_key, self.form_key = None, None, None
       self.ma_id = "N/A"
       
       id_match = re.match(r'^(?P<grade>\d+)(?P<subject>[DHC])(?P<chapter>\d+)(?P<level>[NHVC])(?P<lesson>\d+)-(?P<form>\d+)$', base_id)
       if id_match:
           data = id_match.groupdict()
           self.grade = data['grade']
           self.subject = data['subject']
           self.level = data['level']
           self.chapter_key = f"{data['grade']}{data['subject']}{data['chapter']}"
           self.lesson_key = f"{self.chapter_key}.{data['lesson']}"
           self.form_key = f"{self.lesson_key}-{data['form']}"
           self.ma_id = f"{self.lesson_key}-{data['form']}"

    def get_question_type(self):
        if r'\choiceTF' in self.full_content: return "DS"
        if r'\choice' in self.full_content: return "TN"
        if r'\shortans' in self.full_content: return "DK"
        return "TL"

    def get_status(self):
        if not os.path.exists(self.image_path): return "Chưa render"
        try:
            if os.path.getmtime(self.file_path) > os.path.getmtime(self.image_path): return "Cần cập nhật"
        except FileNotFoundError: return "Chưa render"
        return "OK"

# --- Các hàm và lớp khác ---
def render_batch_of_questions(questions_to_render, progress_callback):
    # (Không thay đổi)
    if not questions_to_render: return
    all_content = "\n".join([q.full_content for q in questions_to_render])
    full_tex_code = f"{LATEX_PREAMBLE_FOR_RENDER}\n{all_content}\n{LATEX_POSTAMBLE}"
    try:
        os.makedirs(DEFAULT_IMAGE_BANK_DIR, exist_ok=True)
        with tempfile.TemporaryDirectory() as tempdir:
            for sty_file in ["ex_test.sty", "ex_forms.sty"]:
                if os.path.exists(sty_file): shutil.copy(sty_file, tempdir)
                else: messagebox.showerror("Lỗi thiếu file", f"Không tìm thấy file '{sty_file}'."); return
            tex_filename = 'batch_render'; tex_path = os.path.join(tempdir, f'{tex_filename}.tex')
            log_path = os.path.join(tempdir, f'{tex_filename}.log'); pdf_path = os.path.join(tempdir, f'{tex_filename}.pdf')
            with open(tex_path, 'w', encoding='utf-8') as f: f.write(full_tex_code)
            process = subprocess.run(['pdflatex', '-synctex=1', '--shell-escape', '-interaction=nonstopmode', f'-jobname={tex_filename}', '-output-directory', tempdir, tex_path], capture_output=True, timeout=180)
            if process.returncode != 0 or not os.path.exists(pdf_path):
                with open(log_path, 'r', encoding='utf-8', errors='ignore') as log_file: log_content = log_file.read()
                messagebox.showerror("Lỗi biên dịch LaTeX", f"Biên dịch ra PDF thất bại.\n{log_content[-1000:]}"); return
            doc = fitz.open(pdf_path)
            if len(doc) != len(questions_to_render):
                messagebox.showwarning("Cảnh báo", f"Số câu hỏi ({len(questions_to_render)}) không khớp với số trang PDF ({len(doc)}).")
            for i, q_obj in enumerate(questions_to_render):
                if i < len(doc):
                    page = doc.load_page(i); pix = page.get_pixmap(dpi=200)
                    pix.save(q_obj.image_path)
                    if progress_callback: progress_callback(q_obj, "OK", f"Đã hóa ảnh: {q_obj.unique_id}")
                elif progress_callback: progress_callback(q_obj, "Lỗi", f"Không tìm thấy trang PDF cho: {q_obj.unique_id}")
            doc.close()
    except Exception as e: messagebox.showerror("Lỗi ngoại lệ", f"Đã có lỗi xảy ra: {e}")

class StagingManagerWindow(tk.Toplevel):
    # (Đã ổn định, không thay đổi)
    def __init__(self, parent):
        super().__init__(parent)
        self.transient(parent); self.title("Quản lý kho câu hỏi tạm")
        self.parent = parent; self.geometry("1200x750")
        self.staging_dir = os.path.join(self.parent.current_directory, STAGING_DIR_NAME)
        self.staging_image_dir = os.path.join(self.parent.current_directory, STAGING_IMAGE_DIR_NAME)
        os.makedirs(self.staging_image_dir, exist_ok=True)
        self.setup_ui(); self.reload_staging_area(); self.grab_set()
    def _get_preview_image_path(self, staged_tex_filepath):
        filename_hash = hashlib.md5(os.path.normpath(staged_tex_filepath).encode()).hexdigest()
        return os.path.join(self.staging_image_dir, f"{filename_hash}.png")
    def setup_ui(self):
        main_pane = ttk.PanedWindow(self, orient=tk.HORIZONTAL)
        main_pane.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)
        left_pane = ttk.Frame(main_pane); main_pane.add(left_pane, weight=1)
        btn_frame = ttk.Frame(left_pane); btn_frame.pack(fill=tk.X, pady=(0, 5))
        ttk.Button(btn_frame, text="Tải lại kho", command=self.reload_staging_area).pack(side=tk.LEFT)
        self.status_label = ttk.Label(btn_frame, text=" Sẵn sàng"); self.status_label.pack(side=tk.LEFT, padx=10)
        tree_frame = ttk.Frame(left_pane); tree_frame.pack(fill='both', expand=True)
        self.tree = ttk.Treeview(tree_frame, columns=("filename",), show="headings", selectmode='extended')
        self.tree.heading("filename", text="Câu hỏi trong kho tạm (ID - Hash)")
        vsb = ttk.Scrollbar(tree_frame, orient="vertical", command=self.tree.yview)
        hsb = ttk.Scrollbar(tree_frame, orient="horizontal", command=self.tree.xview)
        self.tree.configure(yscrollcommand=vsb.set, xscrollcommand=hsb.set)
        vsb.pack(side='right', fill='y'); hsb.pack(side='bottom', fill='x')
        self.tree.pack(fill='both', expand=True)
        self.tree.bind("<<TreeviewSelect>>", self.on_tree_select)
        right_pane = ttk.PanedWindow(main_pane, orient=tk.VERTICAL); main_pane.add(right_pane, weight=3)
        editor_pane = ttk.Frame(right_pane); right_pane.add(editor_pane, weight=2)
        editor_btn_frame = ttk.Frame(editor_pane); editor_btn_frame.pack(fill=tk.X, pady=(0,5))
        ttk.Button(editor_btn_frame, text="✅ Duyệt vào Ngân hàng", command=self.approve_to_bank).pack(side=tk.LEFT, padx=2)
        ttk.Button(editor_btn_frame, text="💾 Lưu thay đổi", command=self.save_changes).pack(side=tk.LEFT, padx=2)
        ttk.Button(editor_btn_frame, text="🖼️ Hóa ảnh", command=self.render_image).pack(side=tk.LEFT, padx=2)
        ttk.Button(editor_btn_frame, text="🗑️ Xóa vĩnh viễn", command=self.delete_permanently).pack(side=tk.RIGHT, padx=2)
        self.editor = scrolledtext.ScrolledText(editor_pane, font=FONTS['code'], wrap=tk.WORD, undo=True)
        self.editor.pack(fill=tk.BOTH, expand=True)
        image_frame = ttk.LabelFrame(right_pane, text="Xem trước ảnh", padding=5)
        self.image_preview = ttk.Label(image_frame, text="Nhấn 'Hóa ảnh' để xem trước", justify=tk.CENTER, anchor="center")
        self.image_preview.pack(fill=tk.BOTH, expand=True)
        right_pane.add(image_frame, weight=1)
    def reload_staging_area(self):
        for i in self.tree.get_children(): self.tree.delete(i)
        os.makedirs(self.staging_dir, exist_ok=True)
        files = [f for f in os.listdir(self.staging_dir) if f.endswith('.tex')]
        self.status_label.config(text=f"Tìm thấy {len(files)} câu hỏi.")
        for filename in sorted(files):
            filepath = os.path.join(self.staging_dir, filename)
            self.tree.insert("", "end", iid=filepath, values=(filename,))
        self.editor.delete("1.0", tk.END)
        self.image_preview.config(image='', text="Chọn một câu hỏi từ danh sách"); self.image_preview.image = None
    def on_tree_select(self, event=None):
        selected_iids = self.tree.selection()
        self.image_preview.config(image='', text="Nhấn 'Hóa ảnh' để xem trước"); self.image_preview.image = None
        if len(selected_iids) == 1:
            filepath = selected_iids[0]
            try:
                with open(filepath, 'r', encoding='utf-8') as f: content = f.read()
                self.editor.delete("1.0", tk.END); self.editor.insert("1.0", content)
                self.editor.config(state='normal')
                preview_path = self._get_preview_image_path(filepath)
                if os.path.exists(preview_path): self._display_image(preview_path)
                else: self.image_preview.config(image='', text="Chưa có ảnh. Nhấn 'Hóa ảnh'.")
            except Exception as e: messagebox.showerror("Lỗi đọc file", f"Không thể đọc file {os.path.basename(filepath)}:\n{e}", parent=self)
        elif len(selected_iids) > 1:
            self.editor.delete("1.0", tk.END); self.editor.insert("1.0", f"Đã chọn {len(selected_iids)} câu hỏi.")
            self.editor.config(state='disabled')
        else: self.editor.delete("1.0", tk.END); self.editor.config(state='disabled')
    def save_changes(self):
        selected_iids = self.tree.selection()
        if len(selected_iids) != 1: messagebox.showwarning("Hành động không hợp lệ", "Vui lòng chỉ chọn một câu hỏi để lưu.", parent=self); return
        filepath = selected_iids[0]
        new_content = self.editor.get("1.0", tk.END).strip()
        try:
            with open(filepath, 'w', encoding='utf-8') as f: f.write(new_content)
            messagebox.showinfo("Thành công", "Đã lưu thay đổi vào file trong kho tạm.", parent=self)
        except Exception as e: messagebox.showerror("Lỗi", f"Không thể lưu file: {e}", parent=self)
    def render_image(self):
        selected_iids = self.tree.selection()
        if not selected_iids: messagebox.showwarning("Chưa chọn", "Vui lòng chọn ít nhất một câu hỏi để hóa ảnh.", parent=self); return
        tasks = []
        for filepath in selected_iids:
            try:
                with open(filepath, 'r', encoding='utf-8') as f: content = f.read()
                output_path = self._get_preview_image_path(filepath)
                tasks.append((content, output_path))
            except Exception: continue
        if not tasks: messagebox.showwarning("Không có gì để hóa ảnh", "Không đọc được file đã chọn.", parent=self); return
        self.status_label.config(text=f"Chuẩn bị hóa ảnh {len(tasks)} câu...")
        threading.Thread(target=self._render_bulk_thread, args=(tasks,), daemon=True).start()
    def _render_bulk_thread(self, tasks):
        total = len(tasks)
        for i, (content, output_path) in enumerate(tasks):
            self.after(0, self.status_label.config, {'text': f'Đang hóa ảnh {i+1}/{total}...'})
            try:
                with tempfile.TemporaryDirectory() as latex_temp_dir:
                    tex_path = os.path.join(latex_temp_dir, "preview.tex")
                    with open(tex_path, 'w', encoding='utf-8') as f: f.write(f"{LATEX_PREAMBLE_FOR_RENDER}\n{content}\n{LATEX_POSTAMBLE}")
                    process = subprocess.run(['pdflatex', '-interaction=nonstopmode', '-output-directory', latex_temp_dir, tex_path], capture_output=True, timeout=60)
                    if process.returncode != 0: continue
                    pdf_path = os.path.join(latex_temp_dir, "preview.pdf")
                    if not os.path.exists(pdf_path): continue
                    doc = fitz.open(pdf_path); pix = doc.load_page(0).get_pixmap(dpi=150)
                    pix.save(output_path); doc.close()
            except Exception: continue
        self.after(0, self.status_label.config, {"text": "Hoàn tất hóa ảnh!"})
    def _display_image(self, img_path):
        try:
            img_data = Image.open(img_path)
            w, h = self.image_preview.winfo_width(), self.image_preview.winfo_height()
            if w < 20 or h < 20: self.after(100, self._display_image, img_path); return
            img_data.thumbnail((w - 10, h - 10), Image.Resampling.LANCZOS)
            photo = ImageTk.PhotoImage(img_data)
            self.image_preview.config(image=photo, text=""); self.image_preview.image = photo
        except Exception as e: self.image_preview.config(image="", text=f"Lỗi hiển thị ảnh: {e}")
    def approve_to_bank(self):
        selected_iids = self.tree.selection()
        if not selected_iids: messagebox.showwarning("Chưa chọn", "Vui lòng chọn (các) câu hỏi để duyệt.", parent=self); return
        approved_count = 0
        for filepath in selected_iids:
            try:
                with open(filepath, 'r', encoding='utf-8') as f: content = f.read()
                base_id_match = re.search(r"%\[(\d+[DHC]\d+[NHVC]\d+-\d+)\]", content)
                if not base_id_match: continue
                base_id = base_id_match.group(1)
                ma_id_parsed = re.match(r'(\d+[DHC]\d+)[NHVC](\d+-\d+)', base_id)
                if not ma_id_parsed: continue
                ma_id = f"{ma_id_parsed.group(1)}.{ma_id_parsed.group(2)}"
                grade = re.match(r'^\d+', ma_id).group(0)
                mon = re.search(r'\d+([DHC])', ma_id).group(1)
                dest_dir = os.path.join(self.parent.current_directory, grade, mon)
                os.makedirs(dest_dir, exist_ok=True)
                dest_file = os.path.join(dest_dir, f"{ma_id}.tex")
                next_seq = 1
                if os.path.exists(dest_file):
                    with open(dest_file, 'r', encoding='utf-8') as f_dest: file_content = f_dest.read()
                    seq_matches = re.findall(r"%\s*Cau\s*(\d+)", file_content)
                    if seq_matches: next_seq = max(map(int, seq_matches)) + 1
                with open(dest_file, 'a', encoding='utf-8') as f_dest: f_dest.write(f"\n\n%Cau {next_seq}\n{content}")
                preview_path = self._get_preview_image_path(filepath)
                if os.path.exists(preview_path): os.remove(preview_path)
                os.remove(filepath)
                approved_count += 1
            except Exception as e: messagebox.showerror("Lỗi", f"Không thể chuyển file {os.path.basename(filepath)}: {e}", parent=self); break
        if approved_count > 0:
            messagebox.showinfo("Thành công", f"Đã duyệt {approved_count} câu hỏi vào ngân hàng chính.", parent=self)
            self.reload_staging_area(); self.parent.load_directory(refresh=True)
    def delete_permanently(self):
        selected_iids = self.tree.selection()
        if not selected_iids: messagebox.showwarning("Chưa chọn", "Vui lòng chọn câu hỏi để xóa.", parent=self); return
        if not messagebox.askyesno("Xác nhận", f"Xóa vĩnh viễn {len(selected_iids)} file này? Hành động này không thể hoàn tác.", parent=self): return
        for filepath in selected_iids:
            try:
                preview_path = self._get_preview_image_path(filepath)
                if os.path.exists(preview_path): os.remove(preview_path)
                os.remove(filepath)
            except Exception as e: messagebox.showerror("Lỗi", f"Không thể xóa file {os.path.basename(filepath)}: {e}", parent=self)
        self.reload_staging_area()

# --- GIAO DIỆN CHÍNH ---

class AddNewQuestionDialog(simpledialog.Dialog):
    # (Không thay đổi)
    def body(self, master):
        self.title("Thêm câu hỏi mới"); self.result = None
        ttk.Label(master, text="Mã dạng toán (ví dụ: 10D1.1-1):").grid(row=0, sticky='w')
        self.ma_id_entry = ttk.Entry(master, width=40); self.ma_id_entry.grid(row=1, sticky='we', pady=(0, 5))
        ttk.Label(master, text="Mức độ (N, H, V, C):").grid(row=2, sticky='w')
        self.level_entry = ttk.Entry(master, width=10); self.level_entry.grid(row=3, sticky='w', pady=(0, 10))
        ttk.Label(master, text="Nội dung câu hỏi (bên trong môi trường ex):").grid(row=4, sticky='w')
        self.content_editor = scrolledtext.ScrolledText(master, width=80, height=15, font=FONTS['code'])
        self.content_editor.grid(row=5, columnspan=2, sticky='nsew')
        return self.ma_id_entry
    def apply(self):
        self.result = {"ma_id": self.ma_id_entry.get().strip(), "level": self.level_entry.get().strip().upper(), "content": self.content_editor.get("1.0", tk.END).strip()}

class BankManagerApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("Công cụ Quản lý & Chuẩn hóa Ngân hàng câu hỏi"); self.geometry("1400x850")
        self.all_questions = {}; self.current_directory = None
        self.filter_data = {'grades': set(), 'subjects': set(), 'chapters': set(), 'lessons': set(), 'forms': set()}
        self.setup_ui()
        self.after(100, self.check_dependencies)
    
    def setup_ui(self):
        top_frame = ttk.Frame(self, padding=10); top_frame.pack(fill='x')
        ttk.Button(top_frame, text="📁 Nạp Ngân hàng", command=self.load_directory).pack(side='left', padx=(0,5))
        ttk.Button(top_frame, text="📥 Import file vào kho tạm", command=self.start_import_process).pack(side='left', padx=(0,5))
        ttk.Button(top_frame, text="🏪 Quản lý kho tạm", command=self.open_staging_manager).pack(side='left', padx=(0,5))
        self.status_label = ttk.Label(top_frame, text=" Sẵn sàng.", font=FONTS['normal'])
        self.status_label.pack(side='left', padx=10)
        main_pane = ttk.PanedWindow(self, orient=tk.HORIZONTAL); main_pane.pack(fill='both', expand=True, padx=10, pady=10)
        left_pane = ttk.Frame(main_pane); main_pane.add(left_pane, weight=2)
        self.setup_detailed_filter_ui(left_pane)
        tree_frame = ttk.LabelFrame(left_pane, text="Danh sách câu hỏi", padding=5)
        tree_frame.pack(fill='both', expand=True, pady=(10, 0))
        self.tree = ttk.Treeview(tree_frame, columns=("type", "status", "img"), show="tree headings", selectmode='extended')
        self.tree.heading("#0", text="File / Câu hỏi"); self.tree.column("#0", width=300)
        self.tree.heading("type", text="Loại"); self.tree.column("type", width=50, anchor='center')
        self.tree.heading("status", text="Trạng thái"); self.tree.column("status", width=100, anchor='center')
        self.tree.heading("img", text="Ảnh"); self.tree.column("img", width=50, anchor='center')
        vsb = ttk.Scrollbar(tree_frame, orient="vertical", command=self.tree.yview)
        hsb = ttk.Scrollbar(tree_frame, orient="horizontal", command=self.tree.xview)
        self.tree.configure(yscrollcommand=vsb.set, xscrollcommand=hsb.set)
        vsb.pack(side='right', fill='y'); hsb.pack(side='bottom', fill='x')
        self.tree.pack(side='left', fill='both', expand=True)
        right_pane = ttk.PanedWindow(main_pane, orient=tk.VERTICAL); main_pane.add(right_pane, weight=3)
        editor_frame = ttk.LabelFrame(right_pane, text="Trình soạn thảo TeX", padding=5)
        self.editor = scrolledtext.ScrolledText(editor_frame, font=FONTS['code'], wrap=tk.WORD, undo=True, height=10)
        self.editor.pack(fill='both', expand=True); right_pane.add(editor_frame, weight=2)
        image_frame = ttk.LabelFrame(right_pane, text="Xem trước hình ảnh", padding=5)
        self.image_preview_label = ttk.Label(image_frame, text="Chưa có ảnh xem trước.", justify=tk.CENTER)
        self.image_preview_label.pack(fill='both', expand=True); right_pane.add(image_frame, weight=1)
        self.tree.bind("<<TreeviewSelect>>", self.on_tree_select)
        bottom_frame = ttk.Frame(self, padding=10); bottom_frame.pack(fill='x')
        ttk.Button(bottom_frame, text="➕ Thêm câu mới", command=self.add_new_question).pack(side='left')
        ttk.Button(bottom_frame, text="💾 Lưu và Hóa ảnh", command=self.save_and_render).pack(side='left', padx=10)
        ttk.Button(bottom_frame, text="🖼️ Hóa ảnh câu được chọn", command=self.render_selected_question).pack(side='left', padx=10)
        ttk.Button(bottom_frame, text="🗑️ Xóa câu hỏi", command=self.delete_selected_questions).pack(side='left', padx=10)
        ttk.Button(bottom_frame, text="🔍 Chuẩn hóa Lời giải", command=self.start_loigiai_standardization).pack(side='right', padx=10)
        ttk.Button(bottom_frame, text="🔄 Đồng bộ & Hóa ảnh tất cả", command=self.start_sync_and_render).pack(side='right')

    def setup_detailed_filter_ui(self, parent):
        filter_frame = ttk.LabelFrame(parent, text="Bộ lọc chi tiết", padding=10)
        filter_frame.pack(fill='x', pady=5)
        
        def on_filter_change(event=None, level=0):
            if level <= 0: self._update_chapter_filter()
            if level <= 1: self._update_lesson_filter()
            if level <= 2: self._update_form_filter()

        ttk.Label(filter_frame, text="Khối:").grid(row=0, column=0, sticky="w");
        self.grade_filter = ttk.Combobox(filter_frame, state="readonly", width=8)
        self.grade_filter.grid(row=0, column=1, padx=5, pady=2, sticky="ew")
        self.grade_filter.bind("<<ComboboxSelected>>", lambda e: on_filter_change(e, 0))
        ttk.Label(filter_frame, text="Môn:").grid(row=0, column=2, sticky="w", padx=5)
        self.subject_filter = ttk.Combobox(filter_frame, state="readonly", width=12)
        self.subject_filter.grid(row=0, column=3, padx=5, pady=2, sticky="ew")
        self.subject_filter.bind("<<ComboboxSelected>>", lambda e: on_filter_change(e, 0))
        ttk.Label(filter_frame, text="Chương:").grid(row=1, column=0, sticky="w")
        self.chapter_filter = ttk.Combobox(filter_frame, state="readonly")
        self.chapter_filter.grid(row=1, column=1, columnspan=3, padx=5, pady=2, sticky="ew")
        self.chapter_filter.bind("<<ComboboxSelected>>", lambda e: on_filter_change(e, 1))
        ttk.Label(filter_frame, text="Bài:").grid(row=2, column=0, sticky="w")
        self.lesson_filter = ttk.Combobox(filter_frame, state="readonly")
        self.lesson_filter.grid(row=2, column=1, columnspan=3, padx=5, pady=2, sticky="ew")
        self.lesson_filter.bind("<<ComboboxSelected>>", lambda e: on_filter_change(e, 2))
        ttk.Label(filter_frame, text="Dạng:").grid(row=3, column=0, sticky="w")
        self.form_filter = ttk.Combobox(filter_frame, state="readonly")
        self.form_filter.grid(row=3, column=1, columnspan=3, padx=5, pady=2, sticky="ew")
        ttk.Label(filter_frame, text="Loại:").grid(row=4, column=0, sticky="w")
        self.type_filter = ttk.Combobox(filter_frame, values=["Tất cả", "TL", "TN", "DS", "DK"], state="readonly", width=8)
        self.type_filter.set("Tất cả"); self.type_filter.grid(row=4, column=1, padx=5, pady=2, sticky="ew")
        ttk.Label(filter_frame, text="Cấp độ:").grid(row=4, column=2, sticky="w", padx=5)
        self.level_filter = ttk.Combobox(filter_frame, values=["Tất cả", "N", "H", "V", "C"], state="readonly", width=12)
        self.level_filter.set("Tất cả"); self.level_filter.grid(row=4, column=3, padx=5, pady=2, sticky="ew")
        ttk.Label(filter_frame, text="Từ khóa:").grid(row=5, column=0, sticky="w", pady=5)
        self.keyword_filter = ttk.Entry(filter_frame)
        self.keyword_filter.grid(row=5, column=1, columnspan=3, sticky="ew")
        btn_frame = ttk.Frame(filter_frame)
        btn_frame.grid(row=6, column=0, columnspan=4, pady=(10,0))
        ttk.Button(btn_frame, text="Lọc", command=self.apply_filters).pack(side="left", padx=5)
        ttk.Button(btn_frame, text="Xóa lọc", command=self.clear_filters).pack(side="left", padx=5)
        filter_frame.columnconfigure(1, weight=1); filter_frame.columnconfigure(3, weight=1)
    
    def apply_filters(self):
        self.tree.delete(*self.tree.get_children())
        grade_f = self.grade_filter.get()
        subject_f_raw = self.subject_filter.get(); subject_f = subject_f_raw.split(" ")[-1].replace("(", "").replace(")", "") if subject_f_raw != "Tất cả" else "Tất cả"
        chapter_f = self.chapter_filter.get(); lesson_f = self.lesson_filter.get(); form_f = self.form_filter.get()
        type_f = self.type_filter.get(); level_f = self.level_filter.get()
        keyword_f = self.keyword_filter.get().lower()

        filtered_questions = self.all_questions.values()
        if grade_f != "Tất cả": filtered_questions = [q for q in filtered_questions if q.grade == grade_f]
        if subject_f != "Tất cả": filtered_questions = [q for q in filtered_questions if q.subject == subject_f]
        if chapter_f != "Tất cả": filtered_questions = [q for q in filtered_questions if q.chapter_key == chapter_f]
        if lesson_f != "Tất cả": filtered_questions = [q for q in filtered_questions if q.lesson_key == lesson_f]
        if form_f != "Tất cả": filtered_questions = [q for q in filtered_questions if q.form_key == form_f]
        if type_f != "Tất cả": filtered_questions = [q for q in filtered_questions if q.get_question_type() == type_f]
        if level_f != "Tất cả": filtered_questions = [q for q in filtered_questions if q.level == level_f]
        if keyword_f: filtered_questions = [q for q in filtered_questions if keyword_f in q.full_content.lower() or keyword_f in q.base_id.lower()]
        
        file_to_questions = {}
        for q_obj in filtered_questions:
            if q_obj.file_path not in file_to_questions: file_to_questions[q_obj.file_path] = []
            file_to_questions[q_obj.file_path].append(q_obj)
        for file_path, questions in sorted(file_to_questions.items()):
            file_node = self.tree.insert("", "end", iid=file_path, text=os.path.basename(file_path), open=True)
            for q_obj in sorted(questions, key=lambda q: int(q.seq_num)):
                self.tree.insert(file_node, "end", iid=q_obj.unique_id, text=f"Câu {q_obj.seq_num} ({q_obj.base_id})", values=(q_obj.get_question_type(), q_obj.get_status(), "✔️" if q_obj.included_image_path else ""))
        self.status_label.config(text=f"Lọc hoàn tất. Tìm thấy {len(filtered_questions)} câu hỏi.")

    def clear_filters(self):
        self.grade_filter.set("Tất cả"); self.subject_filter.set("Tất cả")
        self.type_filter.set("Tất cả"); self.level_filter.set("Tất cả")
        self.keyword_filter.delete(0, tk.END)
        self._update_chapter_filter(); self._update_lesson_filter(); self._update_form_filter()
        self.apply_filters()

    def _populate_filter_data(self):
        self.filter_data = {'grades': set(), 'subjects': set(), 'chapters': set(), 'lessons': set(), 'forms': set()}
        for q in self.all_questions.values():
            if q.grade: self.filter_data['grades'].add(q.grade)
            if q.subject: self.filter_data['subjects'].add(q.subject)
            if q.chapter_key: self.filter_data['chapters'].add(q.chapter_key)
            if q.lesson_key: self.filter_data['lessons'].add(q.lesson_key)
            if q.form_key: self.filter_data['forms'].add(q.form_key)
        
        self.grade_filter['values'] = ["Tất cả"] + sorted(list(self.filter_data['grades']))
        subject_map = {'D': 'Đại số (D)', 'H': 'Hình học (H)', 'C': 'Chuyên đề (C)'}
        self.subject_filter['values'] = ["Tất cả"] + [subject_map.get(s, s) for s in sorted(list(self.filter_data['subjects']))]
        self.grade_filter.set("Tất cả"); self.subject_filter.set("Tất cả")
        self._update_chapter_filter()

    def _update_chapter_filter(self):
        grade_f = self.grade_filter.get()
        subject_f_raw = self.subject_filter.get(); subject_f = subject_f_raw.split(" ")[-1].replace("(", "").replace(")", "") if subject_f_raw != "Tất cả" else "Tất cả"
        chapters = {q.chapter_key for q in self.all_questions.values() if q.chapter_key and (grade_f == "Tất cả" or q.grade == grade_f) and (subject_f == "Tất cả" or q.subject == subject_f)}
        self.chapter_filter['values'] = ["Tất cả"] + sorted(list(chapters))
        self.chapter_filter.set("Tất cả")
    
    def _update_lesson_filter(self):
        chapter_f = self.chapter_filter.get()
        lessons = {q.lesson_key for q in self.all_questions.values() if q.lesson_key and (chapter_f == "Tất cả" or q.chapter_key == chapter_f)}
        self.lesson_filter['values'] = ["Tất cả"] + sorted(list(lessons))
        self.lesson_filter.set("Tất cả")

    def _update_form_filter(self):
        lesson_f = self.lesson_filter.get()
        forms = {q.form_key for q in self.all_questions.values() if q.form_key and (lesson_f == "Tất cả" or q.lesson_key == lesson_f)}
        self.form_filter['values'] = ["Tất cả"] + sorted(list(forms))
        self.form_filter.set("Tất cả")
        
    def check_dependencies(self):
        if not shutil.which("pdflatex"): messagebox.showwarning("Thiếu TeX Live", "Không tìm thấy 'pdflatex'.")

    def load_directory(self, refresh=False):
        dir_path = self.current_directory
        if not dir_path or refresh == False:
            dir_path = filedialog.askdirectory(initialdir=self.current_directory or DEFAULT_QUESTION_BANK_DIR, title="Chọn thư mục Ngân hàng câu hỏi")
        if not dir_path: return
        self.current_directory = dir_path
        global DEFAULT_IMAGE_BANK_DIR
        DEFAULT_IMAGE_BANK_DIR = os.path.join(self.current_directory, "ImageBank")
        os.makedirs(DEFAULT_IMAGE_BANK_DIR, exist_ok=True)
        os.makedirs(os.path.join(self.current_directory, STAGING_DIR_NAME), exist_ok=True)
        os.makedirs(os.path.join(self.current_directory, UNASSIGNED_DIR_NAME), exist_ok=True)
        os.makedirs(os.path.join(self.current_directory, STAGING_IMAGE_DIR_NAME), exist_ok=True)
        self.status_label.config(text=f"Đang quét {dir_path}..."); self.tree.delete(*self.tree.get_children()); self.all_questions.clear()
        threading.Thread(target=self._scan_and_populate_folder, args=(dir_path,), daemon=True).start()

    def _scan_and_populate_folder(self, dir_path):
        special_dirs = {STAGING_DIR_NAME, UNASSIGNED_DIR_NAME, "ImageBank", STAGING_IMAGE_DIR_NAME}
        for root, dirs, files in os.walk(dir_path):
            dirs[:] = [d for d in dirs if d not in special_dirs]
            for file in files:
                if file.endswith(".tex"): self._scan_and_populate_file(os.path.join(root, file))
        self.status_label.config(text="Quét xong. Sẵn sàng.")
        self.after(0, self._populate_filter_data)
        self.after(1, self.apply_filters)

    def _scan_and_populate_file(self, file_path):
        with open(file_path, 'r', encoding='utf-8') as f: content = f.read()
        question_blocks = re.split(r'(%\s*Cau\s*\d+)', content)
        seq_num_counter = 0
        for i in range(1, len(question_blocks), 2):
            comment = question_blocks[i]; block = question_blocks[i+1]
            seq_num_match = re.search(r"(\d+)", comment)
            seq_num = seq_num_match.group(1) if seq_num_match else f"auto_{seq_num_counter}"; seq_num_counter += 1
            env_match = re.search(r"\\begin\{(ex|bt|vd)\}(.*?)\\end\{\1\}", block, re.DOTALL | re.IGNORECASE)
            if env_match:
                full_block_content = env_match.group(0).strip()
                id_candidates = re.findall(r"%\[([^\]]*)\]", full_block_content)
                base_id = "CHƯA CÓ ID"
                for candidate in reversed(id_candidates):
                    if re.match(r'^\d+[DHC]\d+[NHVC]\d+-\d+$', candidate): base_id = candidate; break
                q_obj = ParsedQuestion(file_path, seq_num, env_match.group(1), base_id, f"{comment.strip()}\n{full_block_content}")
                self.all_questions[q_obj.unique_id] = q_obj

    def on_tree_select(self, event=None):
        selected_iids = self.tree.selection()
        if len(selected_iids) == 1:
            selected_iid = selected_iids[0]
            self.editor.config(state="normal")
            if selected_iid in self.all_questions:
                q_obj = self.all_questions[selected_iid]
                self.editor.delete("1.0", tk.END); self.editor.insert("1.0", q_obj.full_content)
                if os.path.exists(q_obj.image_path):
                    try:
                        img = Image.open(q_obj.image_path)
                        preview_width = self.image_preview_label.winfo_width()
                        if preview_width > 1:
                            aspect_ratio = img.height / img.width
                            new_height = int(preview_width * aspect_ratio) if aspect_ratio > 0 else 0
                            if new_height > 0: img = img.resize((preview_width, new_height), Image.Resampling.LANCZOS)
                        photo = ImageTk.PhotoImage(img)
                        self.image_preview_label.config(image=photo, text=""); self.image_preview_label.image = photo
                    except Exception as e: self.image_preview_label.config(image='', text=f"Lỗi tải ảnh: {e}")
                else: self.image_preview_label.config(image='', text="Chưa có ảnh xem trước.")
            else: self.editor.delete("1.0", tk.END); self.editor.config(state="disabled")
        elif len(selected_iids) > 1:
            self.editor.config(state="normal")
            self.editor.delete("1.0", tk.END)
            self.editor.insert("1.0", f"Đã chọn {len(selected_iids)} câu hỏi.")
            self.editor.config(state="disabled")
        else: self.editor.config(state="disabled"); self.editor.delete("1.0", tk.END)

    def save_changes(self):
        selected_iids = self.tree.selection()
        if len(selected_iids) != 1: messagebox.showwarning("Hành động không hợp lệ", "Chỉ có thể lưu khi chọn một câu hỏi."); return None
        q_obj = self.all_questions.get(selected_iids[0])
        if not q_obj: return None
        new_content = self.editor.get("1.0", tk.END).strip()
        with open(q_obj.file_path, 'r+', encoding='utf-8') as f:
            file_content = f.read()
            updated_file_content = file_content.replace(q_obj.full_content, new_content)
            f.seek(0); f.write(updated_file_content); f.truncate()
        self.load_directory(refresh=True)
        return q_obj

    def save_and_render(self):
        updated_q_obj = self.save_changes()
        if updated_q_obj and messagebox.askyesno("Lưu thành công", "Đã lưu thay đổi. Bạn có muốn hóa ảnh lại câu hỏi này ngay bây giờ không?"):
            self.render_selected_question()

    def add_new_question(self):
        if not self.current_directory: messagebox.showwarning("Cảnh báo", "Vui lòng 'Nạp Ngân hàng' trước."); return
        dialog = AddNewQuestionDialog(self)
        if not dialog.result: return
        data = dialog.result
        if not all([data['ma_id'], data['level'] in ['N', 'H', 'V', 'C'], data['content']]):
            messagebox.showerror("Lỗi", "Vui lòng điền đủ và đúng định dạng các trường (Mức độ: N, H, V, C)."); return
        try:
            match = re.match(r'^(\d+[DHC]\d+)\.(\d+-\d+)$', data['ma_id'])
            if not match: messagebox.showerror("Lỗi định dạng", f"Mã dạng toán '{data['ma_id']}' không hợp lệ."); return
            prefix, suffix = match.groups()
            base_id = f"{prefix}{data['level']}{suffix}"
            grade = re.match(r'^\d+', data['ma_id']).group(0)
            mon = re.search(r'\d+([DHC])', data['ma_id']).group(1)
            dir_path = os.path.join(self.current_directory, str(grade), mon); os.makedirs(dir_path, exist_ok=True)
            dest_file_path = os.path.join(dir_path, f"{data['ma_id']}.tex")
            next_seq = 1
            if os.path.exists(dest_file_path):
                with open(dest_file_path, 'r', encoding='utf-8') as f: content = f.read()
                seq_matches = re.findall(r"%\s*Cau\s*(\d+)", content, re.IGNORECASE)
                if seq_matches: next_seq = max(map(int, seq_matches)) + 1
            new_question_text = f"\n\n%Cau {next_seq}\n\\begin{{ex}}%[{base_id}]\n{data['content']}\n\\end{{ex}}"
            with open(dest_file_path, 'a', encoding='utf-8') as f: f.write(new_question_text)
            messagebox.showinfo("Thành công", f"Đã thêm câu hỏi mới vào file:\n{os.path.basename(dest_file_path)}")
            self.load_directory(refresh=True)
        except Exception as e: messagebox.showerror("Lỗi", f"Không thể thêm câu hỏi: {e}")

    def render_selected_question(self):
        selected_iids = self.tree.selection()
        questions_to_render = [self.all_questions.get(uid) for uid in selected_iids if uid in self.all_questions]
        if not questions_to_render: messagebox.showwarning("Cảnh báo", "Vui lòng chọn ít nhất một câu hỏi để hóa ảnh."); return
        self.status_label.config(text=f"Đang xử lý {len(questions_to_render)} câu hỏi...")
        threading.Thread(target=render_batch_of_questions, args=(questions_to_render, self.update_ui_from_thread), daemon=True).start()
    
    def delete_selected_questions(self):
        selected_iids = self.tree.selection()
        questions_to_delete = [self.all_questions.get(uid) for uid in selected_iids if uid in self.all_questions]
        if not questions_to_delete: messagebox.showwarning("Chưa chọn", "Vui lòng chọn (các) câu hỏi cần xóa."); return
        if not messagebox.askyesno("Xác nhận xóa", f"Bạn có chắc muốn xóa vĩnh viễn {len(questions_to_delete)} câu hỏi đã chọn?"): return
        deletions_by_file = {}
        for q in questions_to_delete:
            if q.file_path not in deletions_by_file: deletions_by_file[q.file_path] = []
            deletions_by_file[q.file_path].append(q)
        for file_path, q_list in deletions_by_file.items():
            try:
                with open(file_path, 'r', encoding='utf-8') as f: content = f.read()
                for q_obj in q_list:
                    content = content.replace(q_obj.full_content, '')
                    if os.path.exists(q_obj.image_path): os.remove(q_obj.image_path)
                with open(file_path, 'w', encoding='utf-8') as f: f.write(content)
            except Exception as e: messagebox.showerror("Lỗi", f"Lỗi khi xóa câu hỏi từ file {os.path.basename(file_path)}:\n{e}")
        messagebox.showinfo("Hoàn tất", f"Đã xóa {len(questions_to_delete)} câu hỏi.")
        self.load_directory(refresh=True)
    # NEW: Hàm bắt đầu quá trình chuẩn hóa lời giải
    def start_loigiai_standardization(self):
        """Bắt đầu quá trình quét và thêm \loigiai nếu thiếu."""
        if not self.all_questions:
            messagebox.showwarning("Ngân hàng trống", "Vui lòng 'Nạp Ngân hàng' trước khi chuẩn hóa.")
            return

        if messagebox.askyesno("Xác nhận", 
                               "Quá trình này sẽ quét toàn bộ ngân hàng câu hỏi.\n"
                               "Những câu bị thiếu sẽ được tự động thêm `\\loigiai{Lời giải chưa hoàn chỉnh}`.\n\n"
                               "File gốc sẽ bị ghi đè. Bạn có muốn tiếp tục?"):
            
            self.status_label.config(text="Bắt đầu chuẩn hóa lời giải...")
            threading.Thread(target=self._scan_and_fix_loigiai_thread, daemon=True).start()

    def _scan_and_fix_loigiai_thread(self):
        """Luồng xử lý việc quét và sửa file."""
        # Lấy danh sách các file duy nhất từ các câu hỏi đã nạp
        all_files = set(q.file_path for q in self.all_questions.values())
        
        files_scanned = 0
        files_fixed = 0

        for i, file_path in enumerate(all_files):
            self.after(0, self.status_label.config, {'text': f"Đang quét file {i+1}/{len(all_files)}..."})
            try:
                with open(file_path, 'r', encoding='utf-8') as f:
                    original_content = f.read()

                # Hàm thay thế cho re.sub
                def add_loigiai_if_missing(match):
                    # match.group(1) là toàn bộ nội dung bên trong \begin{ex}...\end{ex}
                    block_content = match.group(1)
                    if r'\loigiai' not in block_content:
                        # Thêm \loigiai ngay trước \end{ex}
                        return f"\\begin{{ex}}{block_content}\\loigiai{{Lời giải chưa hoàn chỉnh}}\\end{{ex}}"
                    else:
                        # Trả về nguyên bản nếu đã có
                        return match.group(0)

                # Tìm tất cả các môi trường ex và áp dụng hàm thay thế
                # re.DOTALL cho phép '.' khớp với cả ký tự xuống dòng
                new_content = re.sub(r'(\\begin\{ex\}(.*?)\\end\{ex\})', add_loigiai_if_missing, original_content, flags=re.DOTALL)
                
                if new_content != original_content:
                    with open(file_path, 'w', encoding='utf-8') as f:
                        f.write(new_content)
                    files_fixed += 1
                
                files_scanned += 1

            except Exception as e:
                print(f"Lỗi khi xử lý file {file_path}: {e}")

        # Cập nhật giao diện sau khi hoàn tất
        def show_summary():
            messagebox.showinfo("Hoàn tất", f"Đã quét {files_scanned} file.\nĐã chuẩn hóa {files_fixed} file.")
            self.status_label.config(text="Chuẩn hóa lời giải hoàn tất.")
            if files_fixed > 0:
                self.load_directory(refresh=True)

        self.after(0, show_summary)
    def start_sync_and_render(self):
        tasks_to_run = [q for q in self.all_questions.values() if q.status != "OK"]
        if not tasks_to_run: messagebox.showinfo("Thông báo", "Tất cả câu hỏi đã được 'hóa ảnh' và cập nhật."); return
        if messagebox.askokcancel("Xác nhận", f"Tìm thấy {len(tasks_to_run)} câu hỏi cần xử lý.\nBạn có muốn tiếp tục?"):
            self.status_label.config(text="Bắt đầu đồng bộ hàng loạt...")
            threading.Thread(target=self._run_render_in_chunks, args=(tasks_to_run,), daemon=True).start()
    
    def _run_render_in_chunks(self, tasks):
        chunk_size = 20
        chunks = [tasks[i:i + chunk_size] for i in range(0, len(tasks), chunk_size)]
        for i, chunk in enumerate(chunks):
            self.status_label.config(text=f"Đang xử lý lô {i+1}/{len(chunks)}...")
            render_batch_of_questions(chunk, self.update_ui_from_thread)
        self.status_label.config(text="Đồng bộ hoàn tất!")

    def import_and_stage_file(self, source_file_path):
        staging_dir = os.path.join(self.current_directory, STAGING_DIR_NAME)
        unassigned_dir = os.path.join(self.current_directory, UNASSIGNED_DIR_NAME)
        try:
            with open(source_file_path, 'r', encoding='utf-8') as f: content = f.read()
        except Exception as e: messagebox.showerror("Lỗi đọc file", f"Không thể đọc file nguồn: {e}"); return
        valid_count, invalid_count = 0, 0
        question_pattern = re.compile(r'(\\begin\{(ex|bt|vd)\}.*?\\end\{\2\})', re.DOTALL)
        all_blocks = question_pattern.findall(content)
        for block_content, _ in all_blocks:
            standardized_block = re.sub(r'\\begin\{(bt|vd)\}', r'\\begin{ex}', block_content)
            standardized_block = re.sub(r'\\end\{(bt|vd)\}', r'\\end{ex}', standardized_block)
            if r'\loigiai' not in standardized_block:
                standardized_block = standardized_block.replace(r'\end{ex}', '\n\\loigiai{}\n\\end{ex}')
            id_match = re.search(r"%\[(\d+[DHC]\d+[NHVC]\d+-\d+)\]", standardized_block)
            if id_match:
                base_id = id_match.group(1)
                file_hash = hashlib.md5(standardized_block.encode()).hexdigest()[:10]
                staging_filepath = os.path.join(staging_dir, f"{base_id}_{file_hash}.tex")
                with open(staging_filepath, 'w', encoding='utf-8') as f: f.write(standardized_block)
                valid_count += 1
            else:
                unassigned_filepath = os.path.join(unassigned_dir, f"unassigned_{hashlib.md5(standardized_block.encode()).hexdigest()[:10]}.tex")
                with open(unassigned_filepath, 'w', encoding='utf-8') as f: f.write(standardized_block)
                invalid_count += 1
        messagebox.showinfo("Hoàn tất", f"Đã xử lý file.\n- {valid_count} câu hợp lệ đã vào Kho tạm.\n- {invalid_count} câu không có ID đã vào kho '{UNASSIGNED_DIR_NAME}'.")
        self.status_label.config(text="Import hoàn tất.")

    def start_import_process(self):
        if not self.current_directory: messagebox.showwarning("Cảnh báo", "Vui lòng 'Nạp Ngân hàng' trước."); return
        source_file = filedialog.askopenfilename(title="Chọn file TeX cần Import", filetypes=[("TeX files", "*.tex"), ("All files", "*.*")])
        if not source_file: return
        self.status_label.config(text=f"Đang import: {os.path.basename(source_file)}...")
        threading.Thread(target=self.import_and_stage_file, args=(source_file,), daemon=True).start()
    
    def open_staging_manager(self):
        if not self.current_directory:
            messagebox.showwarning("Cảnh báo", "Vui lòng 'Nạp Ngân hàng' để xác định thư mục làm việc trước.")
            return
        StagingManagerWindow(self)

    def update_ui_from_thread(self, q_obj, status, message):
        if self.tree.exists(q_obj.unique_id):
            self.tree.set(q_obj.unique_id, "status", status)
            if q_obj.unique_id in self.tree.selection(): self.after(100, self.on_tree_select)
        self.status_label.config(text=message)

if __name__ == "__main__":
    if not os.path.isdir(DEFAULT_QUESTION_BANK_DIR):
        os.makedirs(DEFAULT_QUESTION_BANK_DIR)
    app = BankManagerApp()
    app.mainloop()
