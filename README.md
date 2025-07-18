using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Data;
using System.Drawing;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Windows.Forms;
using Đồ_án_lập_trình_ứng_dụng_1__lần_2_;
using Newtonsoft.Json;
using System.IO;
using PdfSharp.Pdf;
using PdfSharp.Drawing;
using PdfSharp.Drawing.Layout;


namespace Đồ_án_lập_trình_ứng_dụng_1__cách_2_
{
    public partial class Form1 : Form
    {
        private List<Note> allNotes; // Danh sách lưu trữ tất cả ghi chú
        private List<Note> displayedNotes; // Danh sách ghi chú đang được hiển thị (sau khi lọc, tìm kiếm)
        private string dataFilePath = "personal_notes.json";
        public Form1()
        {
            InitializeComponent();
            allNotes = new List<Note>();
            LoadNotesFromFile();
            displayedNotes = new List<Note>();
            // TODO: Tải ghi chú từ file nếu có ở đây
            UpdateNoteListDisplay();
            UpdateTagFilterComboBox();
        }
        // Cập nhật ListBox hiển thị danh sách ghi chú
        private void UpdateNoteListDisplay(List<Note> notesToDisplay = null)
        {
            if (notesToDisplay == null)
            {
                displayedNotes = allNotes.OrderByDescending(n => n.CreationDate).ToList();
            }
            else
            {
                displayedNotes = notesToDisplay.OrderByDescending(n => n.CreationDate).ToList();
            }

            lstNotes.DataSource = null; // Xóa DataSource cũ để refresh
            lstNotes.DataSource = displayedNotes; // Gán DataSource mới
            // lstNotes.DisplayMember = "Title"; // Hoặc sử dụng ToString() đã override trong class Note

            // Cập nhật trạng thái các nút
            bool noteSelected = lstNotes.SelectedItem != null;
            btnEditNote.Enabled = noteSelected;
            btnDeleteNote.Enabled = noteSelected;
            btnExportToPdf.Enabled = noteSelected; // Chỉ xuất khi có ghi chú được chọn
        }

        // Cập nhật ComboBox lọc nhãn
        private void UpdateTagFilterComboBox()
        {
            var allTags = allNotes.SelectMany(n => n.Tags)
                                  .Distinct()
                                  .OrderBy(tag => tag)
                                  .ToList();

            cmbFilterTags.Items.Clear();
            cmbFilterTags.Items.Add("Tất cả nhãn"); // Lựa chọn mặc định
            if (allTags.Any())
            {
                cmbFilterTags.Items.AddRange(allTags.ToArray());
            }
            cmbFilterTags.SelectedIndex = 0; // Chọn "Tất cả nhãn"
        }
        private void Form1_Load(object sender, EventArgs e)
        {
        }
        // Xử lý sự kiện khi chọn một mục trong ListBox
        private void lstNotes_SelectedIndexChanged(object sender, EventArgs e)
        {
            UpdateNoteListDisplay(displayedNotes);
        }
        // Nút "Tạo Mới"
        private void btnNewNote_Click(object sender, EventArgs e)
        {
            using (NoteEditorForm editorForm = new NoteEditorForm())
            {
                if (editorForm.ShowDialog() == DialogResult.OK)
                {
                    Note newNote = editorForm.CurrentNote;
                    allNotes.Add(newNote);
                    SaveNotesToFile();
                    // TODO: Lưu thay đổi vào file
                    UpdateNoteListDisplay();
                    UpdateTagFilterComboBox();
                }
            }
        }
        // Nút "Sửa"
        private void btnEditNote_Click(object sender, EventArgs e)
        {
            if (lstNotes.SelectedItem is Note selectedNote)
            {
                using (NoteEditorForm editorForm = new NoteEditorForm(selectedNote))
                {
                    if (editorForm.ShowDialog() == DialogResult.OK)
                    {
                        // selectedNote đã được cập nhật bên trong editorForm
                        selectedNote.LastModifiedDate = DateTime.Now;
                        // TODO: Lưu thay đổi vào file
                        SaveNotesToFile();
                        UpdateNoteListDisplay();
                        UpdateTagFilterComboBox(); // Nhãn có thể đã thay đổi
                    }
                }
            }
        }
        // Nút "Xoá"
        private void btnDeleteNote_Click(object sender, EventArgs e)
        {
            if (lstNotes.SelectedItem is Note selectedNote)
            {
                var confirmResult = MessageBox.Show($"Bạn có chắc chắn muốn xoá ghi chú '{selectedNote.Title}' không?",
                                     "Xác nhận Xoá",
                                     MessageBoxButtons.YesNo, MessageBoxIcon.Warning);
                if (confirmResult == DialogResult.Yes)
                {
                    allNotes.Remove(selectedNote);
                    // TODO: Lưu thay đổi vào file
                    SaveNotesToFile();
                    UpdateNoteListDisplay();
                    UpdateTagFilterComboBox();
                }
            }
        }
        // Nút "Tìm Kiếm"
        private void btnSearch_Click(object sender, EventArgs e)
        {
            PerformSearchAndFilter();
        }
        
        // Xử lý khi thay đổi Text trong ô tìm kiếm (tìm kiếm ngay khi gõ)
        private void txtSearch_TextChanged(object sender, EventArgs e)
        {
            // Để tìm kiếm ngay khi gõ, có thể gọi PerformSearchAndFilter() ở đây
            // Hoặc bạn có thể giữ nút Tìm Kiếm riêng biệt
            PerformSearchAndFilter(); // Ví dụ: tìm luôn khi gõ
        }
        private void cmbFilterTags_SelectedIndexChanged(object sender, EventArgs e)
        {
            PerformSearchAndFilter();
        }
        private void PerformSearchAndFilter()
        {
            string keyword = txtSearch.Text.ToLower().Trim();
            string selectedTag = cmbFilterTags.SelectedItem as string;

            IEnumerable<Note> filteredNotes = allNotes;

            // Lọc theo nhãn
            if (!string.IsNullOrEmpty(selectedTag) && selectedTag != "Tất cả nhãn")
            {
                filteredNotes = filteredNotes.Where(n => n.Tags.Contains(selectedTag));
            }

            // Lọc theo từ khóa
            if (!string.IsNullOrEmpty(keyword))
            {
                filteredNotes = filteredNotes.Where(n => n.Title.ToLower().Contains(keyword) ||
                                                         n.Content.ToLower().Contains(keyword) ||
                                                         n.Tags.Any(tag => tag.ToLower().Contains(keyword)));
            }

            UpdateNoteListDisplay(filteredNotes.ToList());
        }
        private void btnExportToPdf_Click(object sender, EventArgs e)
        {
            if (lstNotes.SelectedItem is Note selectedNote)
            {
                SaveFileDialog saveFileDialog = new SaveFileDialog
                {
                    Filter = "PDF Files (*.pdf)|*.pdf",
                    Title = "Lưu ghi chú thành PDF",
                    FileName = SanitizeFileName(selectedNote.Title) + ".pdf" // Gợi ý tên file
                };

                if (saveFileDialog.ShowDialog() == DialogResult.OK)
                {
                    try
                    {
                        ExportNoteToPdf(selectedNote, saveFileDialog.FileName);
                        MessageBox.Show("Xuất PDF thành công!", "Thông báo", MessageBoxButtons.OK, MessageBoxIcon.Information);
                    }
                    catch (Exception ex)
                    {
                        MessageBox.Show($"Lỗi khi xuất PDF: {ex.Message}", "Lỗi", MessageBoxButtons.OK, MessageBoxIcon.Error);
                    }
                }
            }
            else
            {
                MessageBox.Show("Vui lòng chọn một ghi chú để xuất.", "Chưa chọn ghi chú", MessageBoxButtons.OK, MessageBoxIcon.Warning);
            }
        }
        private void ExportNoteToPdf(Note note, string filePath)
        {
            PdfDocument document = new PdfDocument();
            document.Info.Title = note.Title;
            document.Info.Author = "Ứng dụng Ghi Chú Cá Nhân";

            PdfPage page = document.AddPage();
            XGraphics gfx = XGraphics.FromPdfPage(page);

            XFont fontTitle = new XFont("Times New Roman", 20, XFontStyleEx.Bold);
            XFont fontContent = new XFont("Times New Roman", 12, XFontStyleEx.Regular);
            XFont fontTags = new XFont("Times New Roman", 10, XFontStyleEx.Italic);
            XFont fontDate = new XFont("Times New Roman", 9, XFontStyleEx.Regular);

            double yPosition = 40; // Vị trí bắt đầu vẽ trên trục Y

            // Tiêu đề
            gfx.DrawString(note.Title, fontTitle, XBrushes.Black,
                new XRect(40, yPosition, page.Width - 80, 30), XStringFormats.TopCenter);
            yPosition += 40;

            // Ngày tạo và sửa
            gfx.DrawString($"Ngày tạo: {note.CreationDate.ToString("dd/MM/yyyy HH:mm:ss")}", fontDate, XBrushes.Gray,
                new XPoint(40, yPosition));
            yPosition += 15;
            gfx.DrawString($"Sửa đổi lần cuối: {note.LastModifiedDate.ToString("dd/MM/yyyy HH:mm:ss")}", fontDate, XBrushes.Gray,
                new XPoint(40, yPosition));
            yPosition += 30;

            // Nhãn (Tags)
            if (note.Tags.Any())
            {
                gfx.DrawString("Nhãn: " + string.Join(", ", note.Tags), fontTags, XBrushes.DarkSlateGray,
                    new XRect(40, yPosition, page.Width - 80, 20), XStringFormats.TopLeft);
                yPosition += 25;
            }

            // Nội dung
            // Xử lý ngắt dòng cho nội dung
            XTextFormatter tf = new XTextFormatter(gfx);
            tf.Alignment = XParagraphAlignment.Left;
            tf.DrawString(note.Content, fontContent, XBrushes.Black,
                new XRect(40, yPosition, page.Width - 80, page.Height - yPosition - 40)
                // ,XStringFormats.TopLeft // Không cần thiết khi dùng TextFormatter
                );

            document.Save(filePath);
        }
        // Hàm tiện ích để tạo tên file an toàn
        private string SanitizeFileName(string name)
        {
            string invalidChars = System.Text.RegularExpressions.Regex.Escape(new string(System.IO.Path.GetInvalidFileNameChars()));
            string invalidRegStr = string.Format(@"([{0}]*\.+$)|([{0}]+)", invalidChars);
            return System.Text.RegularExpressions.Regex.Replace(name, invalidRegStr, "_");
        }
        // Thêm sự kiện FormClosing để có thể nhắc lưu dữ liệu (nếu bạn muốn)
        private void MainForm_FormClosing(object sender, FormClosingEventArgs e)
        {
            // TODO: Nếu có thay đổi chưa lưu, hỏi người dùng có muốn lưu không.
            // Ví dụ: SaveNotesToFile();
        }
        private void SaveNotesToFile()
        {
            try
            {
                // Chuyển đổi danh sách allNotes thành một chuỗi JSON có định dạng đẹp mắt
                string json = JsonConvert.SerializeObject(allNotes, Formatting.Indented);
                // Ghi chuỗi JSON vào file. Nếu file đã tồn tại, nó sẽ được ghi đè.
                File.WriteAllText(dataFilePath, json);
            }
            catch (Exception ex)
            {
                // Hiển thị thông báo nếu có lỗi xảy ra trong quá trình lưu
                MessageBox.Show($"Không thể lưu ghi chú: {ex.Message}", "Lỗi Lưu File", MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }
        private void LoadNotesFromFile()
        {
            try
            {
                // Chỉ đọc file nếu nó tồn tại
                if (File.Exists(dataFilePath))
                {
                    string json = File.ReadAllText(dataFilePath);
                    // Chuyển đổi chuỗi JSON thành danh sách các đối tượng Note
                    allNotes = JsonConvert.DeserializeObject<List<Note>>(json);
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Không thể tải ghi chú: {ex.Message}", "Lỗi Tải File", MessageBoxButtons.OK, MessageBoxIcon.Error);
            }

            // Nếu file không tồn tại hoặc có lỗi, đảm bảo allNotes không bị null
            if (allNotes == null)
            {
                allNotes = new List<Note>();
            }
        }
    }
}
