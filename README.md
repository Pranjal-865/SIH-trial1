import sqlite3
import datetime
import ttkbootstrap as tb
from ttkbootstrap.constants import *
from tkinter import scrolledtext, messagebox
from matplotlib.figure import Figure
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
import threading
import time
import re

DB = 'student_support.db'

# ----- Database setup -----
def init_db():
    conn = sqlite3.connect(DB)
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS users (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    username TEXT UNIQUE,
                    name TEXT,
                    created_at TEXT
                )''')
    c.execute('''CREATE TABLE IF NOT EXISTS mood_entries (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    user_id INTEGER,
                    timestamp TEXT,
                    mood INTEGER,
                    note TEXT,
                    FOREIGN KEY(user_id) REFERENCES users(id)
                )''')
    c.execute('''CREATE TABLE IF NOT EXISTS journal_entries (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    user_id INTEGER,
                    timestamp TEXT,
                    title TEXT,
                    content TEXT
                )''')
    conn.commit()
    conn.close()

init_db()

# ----- Simple data layer -----
class DataLayer:
    def __init__(self):
        self.conn = sqlite3.connect(DB, check_same_thread=False)
        self.lock = threading.Lock()

    def create_user(self, username, name):
        with self.lock:
            c = self.conn.cursor()
            try:
                c.execute('INSERT INTO users (username, name, created_at) VALUES (?, ?, ?)',
                          (username, name, datetime.datetime.now().isoformat()))
                self.conn.commit()
                return c.lastrowid
            except sqlite3.IntegrityError:
                return None

    def get_user(self, username):
        c = self.conn.cursor()
        c.execute('SELECT id, username, name FROM users WHERE username=?', (username,))
        return c.fetchone()

    def add_mood(self, user_id, mood, note):
        with self.lock:
            c = self.conn.cursor()
            c.execute('INSERT INTO mood_entries (user_id, timestamp, mood, note) VALUES (?, ?, ?, ?)',
                      (user_id, datetime.datetime.now().isoformat(), mood, note))
            self.conn.commit()

    def get_moods(self, user_id, days=30):
        c = self.conn.cursor()
        since = (datetime.datetime.now() - datetime.timedelta(days=days)).isoformat()
        c.execute('SELECT timestamp, mood FROM mood_entries WHERE user_id=? AND timestamp>=? ORDER BY timestamp', (user_id, since))
        return c.fetchall()

    def add_journal(self, user_id, title, content):
        with self.lock:
            c = self.conn.cursor()
            c.execute('INSERT INTO journal_entries (user_id, timestamp, title, content) VALUES (?, ?, ?, ?)',
                      (user_id, datetime.datetime.now().isoformat(), title, content))
            self.conn.commit()

    def get_journals(self, user_id, limit=20):
        c = self.conn.cursor()
        c.execute('SELECT timestamp, title, content FROM journal_entries WHERE user_id=? ORDER BY timestamp DESC LIMIT ?', (user_id, limit))
        return c.fetchall()

data = DataLayer()

# ----- Simple supportive bot (rule-based) -----
SUPPORT_RESPONSES = [
    (re.compile(r'stress|stressed|anxious|anxiety', re.I), "I'm sorry you're feeling anxious. Try taking 3 long, slow breaths with me. Would you like a short breathing guide?"),
    (re.compile(r'sad|depress|depressed|down', re.I), "That sounds tough. Writing one small thing that went well today can sometimes help — want to try a short journal prompt?"),
    (re.compile(r'help|suicide|hurt myself|kill myself', re.I), "If you're thinking about harming yourself, please contact local emergency services immediately or a crisis line. You are not alone. Would you like resources for hotlines?"),
]

DEFAULT_RESP = "Thanks for sharing. Tell me more, or choose a mood check-in so I can better help."

def support_reply(text):
    for pattern, resp in SUPPORT_RESPONSES:
        if pattern.search(text or ''):
            return resp
    # fallback empathy
    return DEFAULT_RESP

# ----- UI -----
class App(tb.Window):
    def __init__(self):
        super().__init__(themename="flatly")
        self.title("CampusCare — Student Mental Health Companion")
        self.geometry('930x680')
        self.resizable(False, False)
        self.user = None
        self._build_login()

    def _build_login(self):
        frm = tb.Frame(self, padding=20)
        frm.pack(fill=BOTH, expand=YES)

        header = tb.Label(frm, text="Welcome to CampusCare", font=(None, 22, 'bold'))
        header.pack(pady=(0,10))

        sub = tb.Label(frm, text="A friendly mental health and wellbeing tool for students", font=(None, 10))
        sub.pack(pady=(0,20))

        entry_frame = tb.Frame(frm)
        entry_frame.pack(pady=10)

        tb.Label(entry_frame, text='Username:', width=12).grid(row=0, column=0, sticky=E, pady=6)
        self.username_var = tb.StringVar()
        tb.Entry(entry_frame, textvariable=self.username_var, width=28).grid(row=0, column=1, pady=6)

        tb.Label(entry_frame, text='Full name:', width=12).grid(row=1, column=0, sticky=E, pady=6)
        self.name_var = tb.StringVar()
        tb.Entry(entry_frame, textvariable=self.name_var, width=28).grid(row=1, column=1, pady=6)

        actions = tb.Frame(frm)
        actions.pack(pady=12)

        tb.Button(actions, text='Sign up', bootstyle='success-outline', command=self._signup).grid(row=0, column=0, padx=8)
        tb.Button(actions, text='Login', bootstyle='primary', command=self._login).grid(row=0, column=1, padx=8)

        info = tb.Label(frm, text='Your data stays on this machine. Not a replacement for professional care.', font=(None,8))
        info.pack(pady=(20,0))

    def _clear(self):
        for w in self.winfo_children():
            w.destroy()

    def _signup(self):
        u = self.username_var.get().strip()
        n = self.name_var.get().strip()
        if not u or not n:
            messagebox.showwarning('Missing', 'Please provide both username and full name.')
            return
        uid = data.create_user(u, n)
        if uid:
            messagebox.showinfo('Success', 'Account created. You are now logged in.')
            self.user = (uid, u, n)
            self._build_main()
        else:
            messagebox.showerror('Error', 'Username already exists. Try logging in or choose another username.')

    def _login(self):
        u = self.username_var.get().strip()
        rec = data.get_user(u)
        if rec:
            self.user = rec
            messagebox.showinfo('Welcome back', f'Hello {rec[2]}!')
            self._build_main()
        else:
            messagebox.showerror('Not found', 'User not found. Please sign up first.')

    def _build_main(self):
        self._clear()
        # Top navigation
        top = tb.Frame(self, padding=10)
        top.pack(fill=X)
        tb.Label(top, text=f'CampusCare — {self.user[2]}', font=(None, 14, 'bold')).pack(side=LEFT)
        tb.Button(top, text='Sign out', bootstyle='danger-outline', command=self._signout).pack(side=RIGHT)

        body = tb.Frame(self, padding=12)
        body.pack(fill=BOTH, expand=YES)

        left = tb.Frame(body)
        left.pack(side=LEFT, fill=Y, padx=(0,12))

        # Mood check-in card
        mood_card = tb.LabelFrame(left, text='Mood Check-in', padding=10)
        mood_card.pack(fill=X, pady=6)
        tb.Label(mood_card, text='How are you feeling right now?').pack(anchor=W)

        moods = [('\U0001F622',1), ('\U0001F641',2), ('\U0001F610',3), ('\U0001F642',4), ('\U0001F60A',5)]
        btn_fr = tb.Frame(mood_card)
        btn_fr.pack(pady=8)
        for emoji, val in moods:
            b = tb.Button(btn_fr, text=emoji, width=3, command=lambda v=val: self._prompt_mood(v))
            b.pack(side=LEFT, padx=6)

        tb.Label(mood_card, text='Optional note:').pack(anchor=W, pady=(8,0))
        self.mood_note = tb.Entry(mood_card, width=28)
        self.mood_note.pack()
        tb.Button(mood_card, text='Submit Mood', bootstyle='primary', command=lambda: self._submit_mood(None)).pack(pady=8)

        # Journal card
        journal_card = tb.LabelFrame(left, text='Quick Journal', padding=10)
        journal_card.pack(fill=X, pady=6)
        tb.Button(journal_card, text='New Entry', bootstyle='info-outline', command=self._open_journal_window).pack(fill=X)


        # Bot card
        bot_card = tb.LabelFrame(left, text='Talk to a Supportive Bot', padding=10)
        bot_card.pack(fill=X, pady=6)
        self.bot_entry = tb.Entry(bot_card)
        self.bot_entry.pack(fill=X)
        tb.Button(bot_card, text='Send', bootstyle='secondary', command=self._bot_respond).pack(pady=6)
        self.bot_output = tb.Label(bot_card, text='', wraplength=220)
        self.bot_output.pack()

        # Right area: charts, resources
        right = tb.Frame(body)
        right.pack(side=LEFT, fill=BOTH, expand=YES)

        chart_card = tb.LabelFrame(right, text='Mood Over Time', padding=10)
        chart_card.pack(fill=BOTH, expand=YES, pady=6)
        self.fig = Figure(figsize=(6,3), dpi=100)
        self.ax = self.fig.add_subplot(111)
        self.ax.set_ylim(0.5,5.5)
        self.ax.set_yticks([1,2,3,4,5])
        self.ax.set_yticklabels(['Very sad','Sad','Neutral','Okay','Happy'])
        self.canvas = FigureCanvasTkAgg(self.fig, master=chart_card)
        self.canvas.get_tk_widget().pack(fill=BOTH, expand=YES)

        res_card = tb.LabelFrame(right, text='Resources & Exercises', padding=10)
        res_card.pack(fill=X, pady=6)
        tb.Button(res_card, text='Guided Breathing', command=self._start_breathing).pack(fill=X, pady=4)
        tb.Button(res_card, text='View Emergency Contacts', command=self._show_contacts).pack(fill=X, pady=4)
        tb.Button(res_card, text='View Journal', command=self._view_journal).pack(fill=X, pady=4)

        # initial draw
        self._draw_mood_chart()

    def _signout(self):
        self.user = None
        self._clear()
        self._build_login()

    def _prompt_mood(self, val):
        self.mood_note.delete(0, 'end')
        self.mood_note.insert(0, '')
        self._submit_mood(val)

    def _submit_mood(self, val):
        note = self.mood_note.get().strip()
        if val is None:
            messagebox.showinfo('Pick mood', 'Please choose an emoji mood or use the quick emoji buttons.')
            return
        data.add_mood(self.user[0], val, note)
        messagebox.showinfo('Thanks', 'Mood recorded.')
        self._draw_mood_chart()

    def _draw_mood_chart(self):
        rows = data.get_moods(self.user[0], days=60)
        times = [datetime.datetime.fromisoformat(r[0]) for r in rows]
        moods = [r[1] for r in rows]
        self.ax.clear()
        self.ax.set_ylim(0.5,5.5)
        self.ax.set_yticks([1,2,3,4,5])
        self.ax.set_yticklabels(['Very sad','Sad','Neutral','Okay','Happy'])
        if times:
            self.ax.plot(times, moods, marker='o')
            self.ax.fill_between(times, moods, alpha=0.1)
        else:
            self.ax.text(0.5, 0.5, 'No mood entries yet', transform=self.ax.transAxes, ha='center')
        self.fig.autofmt_xdate()
        self.canvas.draw()

    def _open_journal_window(self):
        win = tb.Toplevel(self)
        win.title('New Journal Entry')
        win.geometry('520x420')
        tb.Label(win, text='Title').pack(anchor=W, padx=8, pady=(8,0))
        title_e = tb.Entry(win)
        title_e.pack(fill=X, padx=8)
        tb.Label(win, text='Write freely — this is private to your device.').pack(anchor=W, padx=8, pady=(6,0))
        content = scrolledtext.ScrolledText(win, height=15)
        content.pack(fill=BOTH, expand=YES, padx=8, pady=6)
        def save():
            t = title_e.get().strip() or 'Untitled'
            c = content.get('1.0', 'end').strip()
            if not c:
                messagebox.showwarning('Empty', 'Journal cannot be empty.')
                return
            data.add_journal(self.user[0], t, c)
            messagebox.showinfo('Saved', 'Journal saved locally.')
            win.destroy()
        tb.Button(win, text='Save', bootstyle='success', command=save).pack(pady=8)

    def _view_journal(self):
        win = tb.Toplevel(self)
        win.title('Journals')
        win.geometry('660x480')
        rows = data.get_journals(self.user[0], limit=50)
        for ts, title, content in rows:
            card = tb.LabelFrame(win, text=f'{title} — {ts[:19]}', padding=8)
            card.pack(fill=X, padx=8, pady=6)
            txt = tb.Label(card, text=content, wraplength=600, justify=LEFT)
            txt.pack()

    def _bot_respond(self):
        txt = self.bot_entry.get().strip()
        if not txt:
            return
        resp = support_reply(txt)
        self.bot_output.config(text=resp)

    def _start_breathing(self):
        win = tb.Toplevel(self)
        win.title('Guided Breathing')
        win.geometry('360x220')
        lbl = tb.Label(win, text='Follow the circle: Breathe in — Hold — Breathe out', font=(None,12))
        lbl.pack(pady=8)
        canvas = tb.Canvas(win, width=200, height=200)
        canvas.pack()
        circle = canvas.create_oval(60,60,140,140, fill='')
        text = canvas.create_text(100,170, text='', font=(None, 10))

        def animate():
            steps = [ (0.8, 'Breathe in'), (1.2, 'Hold'), (0.8, 'Breathe out') ]
            for factor, label in steps*4:
                # animate scale
                for i in range(10):
                    cur = canvas.coords(circle)
                    # simple size pulse
                    base = 100
                    r = base * factor * (0.6 + i*0.04)
                    canvas.coords(circle, 100-r/2, 100-r/2, 100+r/2, 100+r/2)
                    canvas.itemconfig(text, text=label)
                    win.update()
                    time.sleep(0.08)
            canvas.itemconfig(text, text='Done. Hope you feel calmer.')

        threading.Thread(target=animate, daemon=True).start()

    def _show_contacts(self):
        contacts = [
            ('Campus Counselling', '+91-XXXXXXXXXX'),
            ('Local Emergency', '112'),
            ('National Helpline', '+91-XXXXXXXXXX')
        ]
        s = '\n'.join(f'{n}: {p}' for n,p in contacts)
        messagebox.showinfo('Emergency Contacts', s)


if __name__ == '__main__':
    app = App()
    app.mainloop()
