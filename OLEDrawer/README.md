# OLEDrawer
## Файлы
### OLEDrawer.exe
Единственный исполняемый файл. Внутри - сетка для рисования на дисплее с настраиваемым размером, несколько различных инструментов для рисования и выбор порта. Также можно сккопировать код битмапа для дисплея и прошивку OLEDrawer.
## Исходный код
Приложение (OLEDrawer.exe)
```python
import tkinter as tk
from tkinter import filedialog, messagebox, ttk
import serial
import serial.tools.list_ports
import time
import math

# Константа с кодом Ардуино для кнопки
ARDUINO_SKETCH = """
#include <U8g2lib.h>
#include <Wire.h>

// Конструктор для SSD1306 128x32 I2C
U8G2_SSD1306_128X32_UNIVISION_F_HW_I2C u8g2(U8G2_R0, U8X8_PIN_NONE);

uint8_t frame_buffer[512]; 
int bytes_received = 0;

void setup(void) {
  Serial.begin(115200);
  u8g2.begin();
  u8g2.clearBuffer();
  u8g2.setFont(u8g2_font_6x10_tf);
  u8g2.drawStr(35, 20, "OLEDrawer"); 
  u8g2.sendBuffer();
}

void loop(void) {
  while (Serial.available() > 0) {
    uint8_t b = Serial.read();
    if (b == 0xAA) { bytes_received = 0; continue; }
    if (bytes_received < 512) {
      frame_buffer[bytes_received++] = b;
    }
    if (bytes_received == 512) {
      uint8_t* ptr = u8g2.getBufferPtr();
      memcpy(ptr, frame_buffer, 512);
      u8g2.sendBuffer();
      bytes_received = 0;
    }
  }
}
"""

class OLEDrawer:
    def __init__(self, root):
        self.root = root
        self.root.title("OLEDrawer")
        self.root.attributes("-fullscreen", True)
        self.root.configure(bg="black")
        
        self.sw = root.winfo_screenwidth()
        self.sh = root.winfo_screenheight()
        self.ser = None
        self.real_data = [[0 for _ in range(128)] for _ in range(32)]
        self.brush_size = 1
        self.last_send = 0
        self.start_x = self.start_y = self.temp_shape = self.cursor_id = None
        self.preview_data = None

        # ПАНЕЛЬ УПРАВЛЕНИЯ
        self.panel_h = 130
        p = tk.Frame(root, bg="#001515", height=self.panel_h, bd=2, relief="ridge")
        p.pack(fill=tk.X, side=tk.BOTTOM)
        
        # Порты
        port_f = tk.Frame(p, bg="#001515")
        port_f.pack(side=tk.LEFT, padx=20)
        tk.Label(port_f, text="PORT:", fg="#00FFFF", bg="#001515").pack()
        self.port_combo = ttk.Combobox(port_f, values=self.get_ports(), width=10)
        self.port_combo.pack()
        tk.Button(port_f, text="CONNECT", command=self.connect_ser, bg="#008888", fg="white").pack(pady=2)

        # Размер
        size_f = tk.Frame(p, bg="#001515")
        size_f.pack(side=tk.LEFT, padx=20)
        self.size_lbl = tk.Label(size_f, text="SIZE: 1px", font='Consolas 12 bold', fg="#00FFFF", bg="#001515")
        self.size_lbl.pack()
        self.pow_slider = tk.Scale(size_f, from_=0, to=5, orient=tk.HORIZONTAL, command=self.rebuild_ui, 
                                   bg="#001515", fg="#00FFFF", highlightthickness=0, length=150)
        self.pow_slider.set(3); self.pow_slider.pack()

        # Центр: Подсказки + Прошивка
        mid_f = tk.Frame(p, bg="#001515")
        mid_f.pack(side=tk.LEFT, expand=True)
        tk.Button(mid_f, text="VIEW ARDUINO CODE", command=self.show_sketch, bg="#004444", fg="#00FFFF", relief="flat").pack(pady=5)
        help_t = "Shift:Line | Alt:Rect | Ctrl:Circle | ЛКМ:Draw | ПКМ:Erase | Колесо:Brush | S:Save | L:Load"
        tk.Label(mid_f, text=help_t, font='Arial 10', fg="#008888", bg="#001515").pack()
        self.vsym = tk.BooleanVar(); tk.Checkbutton(mid_f, text="V-SYM", variable=self.vsym, fg="#00FFFF", bg="#001515", selectcolor="black").pack(side=tk.LEFT, padx=10)
        self.hsym = tk.BooleanVar(); tk.Checkbutton(mid_f, text="H-SYM", variable=self.hsym, fg="#00FFFF", bg="#001515", selectcolor="black").pack(side=tk.LEFT, padx=10)

        # Правая часть
        btn_f = tk.Frame(p, bg="#001515")
        btn_f.pack(side=tk.RIGHT, padx=20)
        tk.Button(btn_f, text="COPY CODE", command=self.copy_code, bg="#008888", fg="white", width=12).pack(side=tk.LEFT, padx=5)
        tk.Button(btn_f, text="CLEAR", command=self.clear, bg="#770000", fg="white", width=10).pack(side=tk.LEFT, padx=5)
        tk.Button(btn_f, text="EXIT", command=root.quit, bg="#440000", fg="white").pack(side=tk.LEFT, padx=5)

        self.cv = tk.Canvas(root, bg="black", highlightthickness=0)
        self.cv.pack(expand=True, fill=tk.BOTH)
        
        self.cv.bind("<Button-1>", lambda e: self.press(e, 1))
        self.cv.bind("<Button-3>", lambda e: self.press(e, 0))
        self.cv.bind("<B1-Motion>", lambda e: self.move(e, 1))
        self.cv.bind("<B3-Motion>", lambda e: self.move(e, 0))
        self.cv.bind("<ButtonRelease-1>", lambda e: self.release())
        self.cv.bind("<ButtonRelease-3>", lambda e: self.release())
        self.cv.bind("<Motion>", self.update_cursor)
        root.bind("<MouseWheel>", self.on_scroll)
        root.bind("<Escape>", lambda e: root.quit())
        root.bind("s", lambda e: self.save_file())
        root.bind("l", lambda e: self.load_file())
        
        self.rebuild_ui()

    def get_ports(self):
        return [p.device for p in serial.tools.list_ports.comports()]

    def connect_ser(self):
        port = self.port_combo.get()
        baud_rate = 115200 # Жестко прописываем здесь для надежности
        try:
            # Исправлено: используем локальную переменную или 115200 напрямую
            self.ser = serial.Serial(port, baud_rate, timeout=0.01)
            messagebox.showinfo("OK", f"Connected to {port}")
        except Exception as e:
            messagebox.showerror("Error", str(e))


    def show_sketch(self):
        win = tk.Toplevel(self.root)
        win.title("Arduino Firmware")
        win.geometry("600x400")
        txt = tk.Text(win, bg="#111", fg="#00FF00", font=("Consolas", 10))
        txt.insert(tk.END, ARDUINO_SKETCH.strip())
        txt.pack(expand=True, fill="both")
        tk.Button(win, text="Close", command=win.destroy).pack(fill="x")

    def rebuild_ui(self, v=None):
        self.cv.delete("all")
        gv = 2**self.pow_slider.get(); self.rows, self.cols = gv, gv * 4
        self.cw = int((self.sw - 60) // self.cols)
        self.ch = self.cw
        if (self.ch * self.rows) > (self.sh - self.panel_h - 60):
            self.ch = int((self.sh - self.panel_h - 60) // self.rows); self.cw = self.ch
        self.ox = (self.sw - (self.cols * self.cw)) // 2
        self.oy = (self.sh - self.panel_h - (self.rows * self.ch)) // 2
        self.rect_map = {}
        for r in range(self.rows):
            for c in range(self.cols):
                x1, y1 = self.ox + c*self.cw, self.oy + r*self.ch
                color = "#00FFFF" if self.real_data[r*(32//self.rows)][c*(128//self.cols)] else "black"
                self.rect_map[(r, c)] = self.cv.create_rectangle(x1, y1, x1+self.cw, y1+self.ch, outline="#001a1a", fill=color)

    def on_scroll(self, e):
        d = 1 if e.delta > 0 else -1
        self.brush_size = max(1, min(32, self.brush_size + d))
        self.size_lbl.config(text=f"SIZE: {self.brush_size}px")
        self.update_cursor(e)

    def update_cursor(self, e):
        if self.cursor_id: self.cv.delete(self.cursor_id)
        cx = (e.x - self.ox) // self.cw
        ry = (e.y - self.oy) // self.ch
        if 0 <= cx < self.cols and 0 <= ry < self.rows:
            gx, gy = self.ox + cx*self.cw, self.oy + ry*self.ch
            self.cursor_id = self.cv.create_rectangle(gx, gy, gx+self.brush_size*self.cw, gy+self.brush_size*self.ch, outline="#00FFFF", width=2)

    def press(self, e, val):
        self.start_x, self.start_y = e.x, e.y
        self.preview_data = [row[:] for row in self.real_data]
        if not (e.state & 0x0001 or e.state & 0x0004 or e.state & 0x20000): self.apply_brush(e.x, e.y, val)
        self.send()

    def move(self, e, val):
        if e.state & 0x0001 or e.state & 0x0004 or e.state & 0x20000:
            if self.preview_data:
                self.real_data = [row[:] for row in self.preview_data]
                if e.state & 0x0001: self.draw_line(self.start_x, self.start_y, e.x, e.y, val)
                elif e.state & 0x0004: self.draw_circle(self.start_x, self.start_y, e.x, e.y, val)
                elif e.state & 0x20000: self.draw_rect(self.start_x, self.start_y, e.x, e.y, val)
        else: self.apply_brush(e.x, e.y, val)
        self.refresh(); self.update_cursor(e); self.send(True)

    def release(self): self.preview_data = None; self.send()

    def apply_brush(self, ex, ey, val):
        cx, ry = int((ex-self.ox)//self.cw), int((ey-self.oy)//self.ch)
        for i in range(ry, ry + self.brush_size):
            for j in range(cx, cx + self.brush_size): self.set_px_sym(i, j, val)

    def set_px_sym(self, r, c, val):
        pts = [(r, c)]
        if self.vsym.get(): pts.append((r, self.cols - 1 - c))
        if self.hsym.get(): 
            pts.append((self.rows - 1 - r, c))
            if self.vsym.get(): pts.append((self.rows - 1 - r, self.cols - 1 - c))
        for rg, cg in pts:
            if 0 <= rg < self.rows and 0 <= cg < self.cols:
                sv, sh = 32//self.rows, 128//self.cols
                for ir in range(rg*sv, (rg+1)*sv):
                    for ic in range(cg*sh, (cg+1)*sh): self.real_data[ir][ic] = val

    def refresh(self):
        sv, sh = 32//self.rows, 128//self.cols
        for r in range(self.rows):
            for c in range(self.cols):
                color = "#00FFFF" if self.real_data[r*sv][c*sh] else "black"
                self.cv.itemconfig(self.rect_map[(r, c)], fill=color)

    def draw_line(self, x0, y0, x1, y1, v):
        dx, dy = abs(x1-x0), -abs(y1-y0); sx, sy = 1 if x0 < x1 else -1, 1 if y0 < y1 else -1
        err = dx + dy
        while True:
            self.apply_brush(x0, y0, v)
            if x0 == x1 and y0 == y1: break
            e2 = 2*err
            if e2 >= dy: err += dy; x0 += sx
            if e2 <= dx: err += dx; y0 += sy

    def draw_rect(self, x0, y0, x1, y1, v):
        self.draw_line(x0, y0, x1, y0, v); self.draw_line(x1, y0, x1, y1, v)
        self.draw_line(x1, y1, x0, y1, v); self.draw_line(x0, y1, x0, y0, v)

    def draw_circle(self, x0, y0, x1, y1, v):
        r = int(math.hypot(x0-x1, y0-y1)); x, y, err = r, 0, 1-r
        while x >= y:
            for sx, sy in [(1,1),(1,-1),(-1,1),(-1,-1)]:
                self.apply_brush(x0+x*sx, y0+y*sy, v); self.apply_brush(x0+y*sx, y0+x*sy, v)
            y += 1
            if err < 0: err += 2*y+1
            else: x -= 1; err += 2*(y-x)+1

    def copy_code(self):
        buf = bytearray(512)
        for p in range(4):
            for x in range(128):
                b = 0
                for bit in range(8):
                    if self.real_data[p*8 + bit][x]: b |= (1 << bit)
                buf[p*128 + x] = b
        print(", ".join([f"0x{b:02X}" for b in buf]))

    def clear(self):
        self.real_data = [[0 for _ in range(128)] for _ in range(32)]; self.refresh(); self.send()

    def save_file(self):
        p = filedialog.asksaveasfilename(defaultextension=".txt")
        if p:
            with open(p, "w") as f:
                for r in self.real_data: f.write("".join(map(str, r)) + "\n")

    def load_file(self):
        p = filedialog.askopenfilename()
        if p:
            with open(p, "r") as f:
                for r, line in enumerate(f.readlines()[:32]):
                    for c, char in enumerate(line.strip()[:128]): self.real_data[r][c] = int(char)
            self.refresh(); self.send()

    def send(self, throttle=False):
        if not self.ser: return
        t = time.time()
        if throttle and (t - self.last_send < 0.07): return
        self.last_send = t
        buf = bytearray([0xAA])
        data = bytearray(512)
        for p in range(4):
            for x in range(128):
                b = 0
                for bit in range(8):
                    if self.real_data[p*8 + bit][x]: b |= (1 << bit)
                data[p*128 + x] = b
        try: self.ser.write(buf + data)
        except: pass

if __name__ == "__main__":
    root = tk.Tk()
    app = OLEDrawer(root)
    root.mainloop()

```
Прошивка для arduino
```C++
#include <U8g2lib.h>
#include <Wire.h>

U8G2_SSD1306_128X32_UNIVISION_F_HW_I2C u8g2(U8G2_R0, U8X8_PIN_NONE);

uint8_t frame[512]; // Наш собственный буфер
int count = 0;

void setup(void) {
  Serial.begin(115200);
  u8g2.begin();
  u8g2.clearBuffer();
  u8g2.setFont(u8g2_font_6x10_tf);
  u8g2.drawStr(10, 20, "OLEDrawer is ready"); 
  u8g2.sendBuffer();
}

void loop(void) {
  while (Serial.available() > 0) {
    uint8_t b = Serial.read();
    
    // Если поймали маркер начала (0xAA), сбрасываем счетчик
    if (b == 0xAA) {
      count = 0;
      continue;
    }

    if (count < 512) {
      frame[count++] = b;
    }

    // Как только накопили 512 байт — выводим на экран
    if (count == 512) {
      uint8_t* ptr = u8g2.getBufferPtr();
      memcpy(ptr, frame, 512);
      u8g2.sendBuffer();
      count = 0; // Готовы к следующему кадру
    }
  }
}
```
