
    

class LoginSystem:
    def __init__(self, root):
        self.root = root
        self.db_conn = self.connect_to_db()
        self.current_user = None
        self.create_users_table()
        self.create_permissions_table()
        self.check_admin_exists()
        self.setup_login_window()

    def connect_to_db(self):
        try:
            conn = sqlite3.connect("attendance.db")
            return conn
        except Exception as e:
            messagebox.showerror("خطأ في قاعدة البيانات", f"لا يمكن الاتصال بقاعدة البيانات: {str(e)}")
            exit(1)

    def create_users_table(self):
        try:
            with self.db_conn:
                try:
                    self.db_conn.execute("ALTER TABLE users DROP COLUMN role")
                except:
                    pass
                self.db_conn.execute("""
                    CREATE TABLE IF NOT EXISTS users (
                        id INTEGER PRIMARY KEY AUTOINCREMENT,
                        username TEXT UNIQUE,
                        password TEXT,
                        full_name TEXT,
                        created_date TEXT,
                        last_login TEXT,
                        is_active INTEGER DEFAULT 1
                    )
                """)
        except Exception as e:
            messagebox.showerror("خطأ", f"تعذّر إنشاء/تعديل جدول المستخدمين: {str(e)}")

    def create_permissions_table(self):
        try:
            with self.db_conn:
                self.db_conn.execute("""
                    CREATE TABLE IF NOT EXISTS user_permissions (
                        user_id INTEGER PRIMARY KEY,
                        can_edit_attendance INTEGER DEFAULT 1,
                        can_add_students INTEGER DEFAULT 1,
                        can_edit_students INTEGER DEFAULT 1,
                        can_delete_students INTEGER DEFAULT 0,
                        can_view_edit_history INTEGER DEFAULT 0,
                        can_reset_attendance INTEGER DEFAULT 0,
                        can_export_data INTEGER DEFAULT 1,
                        can_import_data INTEGER DEFAULT 0,
                        is_admin INTEGER DEFAULT 0,
                        FOREIGN KEY (user_id) REFERENCES users(id)
                    )
                """)
        except Exception as e:
            messagebox.showerror("خطأ", f"تعذّر إنشاء جدول الصلاحيات: {str(e)}")

    def check_admin_exists(self):
        cursor = self.db_conn.cursor()
        cursor.execute("SELECT COUNT(*) FROM users WHERE username='admin'")
        count = cursor.fetchone()[0]
        if count == 0:
            hashed_pwd = hashlib.sha256("admin123".encode()).hexdigest()
            try:
                with self.db_conn:
                    self.db_conn.execute("""
                        INSERT INTO users (username, password, full_name, created_date, is_active)
                        VALUES (?, ?, ?, ?, ?)
                    """, (
                        'admin',
                        hashed_pwd,
                        'المسؤول الرئيسي',
                        datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
                        1
                    ))
                    cursor.execute("SELECT id FROM users WHERE username='admin'")
                    admin_id = cursor.fetchone()[0]
                    self.db_conn.execute("""
                        INSERT INTO user_permissions (
                            user_id, can_edit_attendance, can_add_students, 
                            can_edit_students, can_delete_students, can_view_edit_history,
                            can_reset_attendance, can_export_data, can_import_data, is_admin
                        ) VALUES (?, 1, 1, 1, 1, 1, 1, 1, 1, 1)
                    """, (admin_id,))
            except Exception as e:
                messagebox.showerror("خطأ", f"تعذّر إنشاء حساب المدير الرئيسي: {str(e)}")

    def setup_login_window(self):
        self.colors = {
            "primary": "#1E40AF",
            "secondary": "#3B82F6",
            "background": "#F1F5F9",
            "card": "#FFFFFF",
            "text": "#1F2937",
            "border": "#E5E7EB",
            "error": "#EF4444"
        }
        self.fonts = {
            "heading": ("Tajawal", 28, "bold"),
            "title": ("Tajawal", 18, "bold"),
            "normal": ("Tajawal", 14),
            "bold": ("Tajawal", 14, "bold"),
            "small": ("Tajawal", 12)
        }

        self.root.title(" نظام إدارة الدورات التخصصية - تسجيل الدخول")
        self.root.geometry("900x600")
        self.root.resizable(False, False)

        screen_width = self.root.winfo_screenwidth()
        screen_height = self.root.winfo_screenheight()
        x = (screen_width - 900) // 2
        y = (screen_height - 600) // 2
        self.root.geometry(f"900x600+{x}+{y}")

        main_frame = tk.Frame(self.root, bg=self.colors["background"])
        main_frame.pack(fill=tk.BOTH, expand=True)

        left_frame = tk.Frame(main_frame, bg=self.colors["primary"], width=350)
        left_frame.pack(side=tk.LEFT, fill=tk.Y)

        right_frame = tk.Frame(main_frame, bg=self.colors["background"])
        right_frame.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)

        left_title = tk.Label(
            left_frame,
            text="قــســم\nشــؤون\nالمـدربـين",
            font=self.fonts["heading"],
            bg=self.colors["primary"],
            fg="white",
            justify=tk.LEFT
        )
        left_title.place(x=30, y=150)

        left_footer = tk.Label(
            left_frame,
            text="© 2025\nجميع الحقوق محفوظة \n للمهندس / عبدالرحمن جفال الشمري ",
            font=self.fonts["small"],
            bg=self.colors["primary"],
            fg="white"
        )
        left_footer.place(x=30, y=520)

        card = tk.Frame(right_frame, bg=self.colors["card"], bd=1, relief=tk.RIDGE, padx=40, pady=30)
        card.place(relx=0.5, rely=0.5, anchor=tk.CENTER, width=420, height=380)

        login_label = tk.Label(card, text="تسجيل الدخول", font=self.fonts["title"], fg=self.colors["primary"],
                               bg=self.colors["card"])
        login_label.pack(pady=(0, 20))

        username_label = tk.Label(card, text="اسم المستخدم:", font=self.fonts["bold"], bg=self.colors["card"],
                                  fg=self.colors["text"])
        username_label.pack(anchor="w", pady=(5, 0))

        self.username_entry = tk.Entry(card, font=self.fonts["normal"], bg=self.colors["card"], fg=self.colors["text"],
                                       highlightthickness=1, highlightbackground=self.colors["border"], relief=tk.FLAT)
        self.username_entry.pack(fill=tk.X, pady=(0, 10), ipady=6)
        self.username_entry.focus_set()

        password_label = tk.Label(card, text="كلمة المرور:", font=self.fonts["bold"], bg=self.colors["card"],
                                  fg=self.colors["text"])
        password_label.pack(anchor="w", pady=(5, 0))

        self.password_entry = tk.Entry(card, font=self.fonts["normal"], bg=self.colors["card"], fg=self.colors["text"],
                                       highlightthickness=1, highlightbackground=self.colors["border"], show="•",
                                       relief=tk.FLAT)
        self.password_entry.pack(fill=tk.X, pady=(0, 20), ipady=6)
        self.password_entry.bind("<Return>", lambda event: self.login())

        login_button = tk.Button(card, text="دخول", font=self.fonts["bold"], bg=self.colors["secondary"], fg="white",
                                 bd=0, relief=tk.FLAT, cursor="hand2", command=self.login)
        login_button.pack(fill=tk.X, pady=(0, 10), ipady=8)

        self.status_label = tk.Label(card, text="", font=self.fonts["small"], bg=self.colors["card"],
                                     fg=self.colors["error"])
        self.status_label.pack()

    def login(self):
        username = self.username_entry.get().strip()
        password = self.password_entry.get().strip()

        if not username or not password:
            messagebox.showwarning("تنبيه", "الرجاء إدخال اسم المستخدم وكلمة المرور")
            return

        hashed_pwd = hashlib.sha256(password.encode()).hexdigest()
        cursor = self.db_conn.cursor()
        cursor.execute("""
            SELECT u.id, u.username, u.full_name
            FROM users u
            WHERE u.username=? AND u.password=? AND u.is_active=1
        """, (username, hashed_pwd))
        user = cursor.fetchone()

        if user:
            cursor.execute("""
                SELECT * FROM user_permissions WHERE user_id=?
            """, (user[0],))
            permissions = cursor.fetchone()

            if not permissions:
                is_admin = 1 if username == 'admin' else 0
                with self.db_conn:
                    cursor.execute("""
                        INSERT INTO user_permissions (
                            user_id, can_edit_attendance, can_add_students, 
                            can_edit_students, can_delete_students, can_view_edit_history,
                            can_reset_attendance, can_export_data, can_import_data, is_admin
                        ) VALUES (?, 1, 1, 1, ?, ?, ?, 1, ?, ?)
                    """, (user[0], is_admin, is_admin, is_admin, is_admin, is_admin))

                cursor.execute("SELECT * FROM user_permissions WHERE user_id=?", (user[0],))
                permissions = cursor.fetchone()

            with self.db_conn:
                self.db_conn.execute("UPDATE users SET last_login=? WHERE id=?",
                                     (datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S"), user[0]))

            self.current_user = {
                "id": user[0],
                "username": user[1],
                "full_name": user[2],
                "permissions": {
                    "can_edit_attendance": bool(permissions[1]),
                    "can_add_students": bool(permissions[2]),
                    "can_edit_students": bool(permissions[3]),
                    "can_delete_students": bool(permissions[4]),
                    "can_view_edit_history": bool(permissions[5]),
                    "can_reset_attendance": bool(permissions[6]),
                    "can_export_data": bool(permissions[7]),
                    "can_import_data": bool(permissions[8]),
                    "is_admin": bool(permissions[9])
                }
            }

            self.root.destroy()

            new_root = tk.Tk()
            ModernAttendanceSystem(new_root, self.current_user, self.db_conn)
            new_root.mainloop()
        else:
            messagebox.showwarning("خطأ", "اسم المستخدم أو كلمة المرور غير صحيحة")


class UserManagement:
    def __init__(self, root, conn, current_user, colors, fonts):
        self.root = root
        self.conn = conn
        self.current_user = current_user
        self.colors = colors
        self.fonts = fonts
        self.create_user_management_window()

    def create_user_management_window(self):
        self.user_window = tk.Toplevel(self.root)
        self.user_window.title("إدارة المستخدمين")
        self.user_window.geometry("900x700")
        self.user_window.configure(bg=self.colors["light"])
        # self.user_window.transient(self.root)  # قم بتعليق هذا السطر أو حذفه
        self.user_window.grab_set()

        # تفعيل خاصية تغيير حجم النافذة
        self.user_window.resizable(True, True)

        x = (self.user_window.winfo_screenwidth() - 900) // 2
        y = (self.user_window.winfo_screenheight() - 700) // 2
        self.user_window.geometry(f"900x700+{x}+{y}")

        tk.Label(
            self.user_window,
            text="إدارة المستخدمين",
            font=self.fonts["large_title"],
            bg=self.colors["primary"],
            fg="white",
            padx=10, pady=10, width=900
        ).pack(fill=tk.X)

        button_frame = tk.Frame(self.user_window, bg=self.colors["light"], pady=10)
        button_frame.pack(fill=tk.X, padx=10)

        add_user_btn = tk.Button(
            button_frame, text="إضافة مستخدم جديد", font=self.fonts["text_bold"], bg=self.colors["success"], fg="white",
            padx=10, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=self.add_user
        )
        add_user_btn.pack(side=tk.RIGHT, padx=5)

        edit_user_btn = tk.Button(
            button_frame, text="تعديل المستخدم المحدد", font=self.fonts["text_bold"], bg=self.colors["warning"],
            fg="white",
            padx=10, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=self.edit_user
        )
        edit_user_btn.pack(side=tk.RIGHT, padx=5)

        toggle_active_btn = tk.Button(
            button_frame, text="تفعيل/تعطيل المستخدم", font=self.fonts["text_bold"], bg=self.colors["secondary"],
            fg="white",
            padx=10, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=self.toggle_user_active
        )
        toggle_active_btn.pack(side=tk.RIGHT, padx=5)

        delete_user_btn = tk.Button(
            button_frame, text="حذف المستخدم المحدد", font=self.fonts["text_bold"], bg=self.colors["danger"],
            fg="white",
            padx=10, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=self.delete_user
        )
        delete_user_btn.pack(side=tk.RIGHT, padx=5)

        manage_permissions_btn = tk.Button(
            button_frame, text="إدارة صلاحيات المستخدم", font=self.fonts["text_bold"], bg="#9333EA", fg="white",
            padx=10, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=self.manage_user_permissions
        )
        manage_permissions_btn.pack(side=tk.RIGHT, padx=5)

        table_frame = tk.Frame(self.user_window, bg=self.colors["light"], padx=10, pady=10)
        table_frame.pack(fill=tk.BOTH, expand=True)

        tree_scroll = tk.Scrollbar(table_frame)
        tree_scroll.pack(side=tk.RIGHT, fill=tk.Y)

        self.users_tree = ttk.Treeview(
            table_frame,
            columns=("id", "username", "full_name", "created_date", "last_login", "status", "is_admin"),
            show="headings",
            yscrollcommand=tree_scroll.set
        )
        self.users_tree.column("id", width=50, anchor=tk.CENTER)
        self.users_tree.column("username", width=120, anchor=tk.CENTER)
        self.users_tree.column("full_name", width=150, anchor=tk.CENTER)
        self.users_tree.column("created_date", width=120, anchor=tk.CENTER)
        self.users_tree.column("last_login", width=120, anchor=tk.CENTER)
        self.users_tree.column("status", width=80, anchor=tk.CENTER)
        self.users_tree.column("is_admin", width=80, anchor=tk.CENTER)

        self.users_tree.heading("id", text="الرقم")
        self.users_tree.heading("username", text="اسم المستخدم")
        self.users_tree.heading("full_name", text="الاسم الكامل")
        self.users_tree.heading("created_date", text="تاريخ الإنشاء")
        self.users_tree.heading("last_login", text="آخر تسجيل دخول")
        self.users_tree.heading("status", text="الحالة")
        self.users_tree.heading("is_admin", text="مشرف")

        self.users_tree.pack(fill=tk.BOTH, expand=True)
        tree_scroll.config(command=self.users_tree.yview)

        self.users_tree.tag_configure("active", background="#e8f5e9")
        self.users_tree.tag_configure("inactive", background="#ffebee")
        self.users_tree.tag_configure("admin", background="#e1f5fe")

        self.load_users()

        close_btn = tk.Button(
            self.user_window, text="إغلاق", font=self.fonts["text_bold"], bg=self.colors["dark"], fg="white",
            padx=15, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=self.user_window.destroy
        )
        close_btn.pack(pady=10)

    def load_users(self):
        for item in self.users_tree.get_children():
            self.users_tree.delete(item)
        cursor = self.conn.cursor()
        cursor.execute("""
            SELECT u.id, u.username, u.full_name, u.created_date, u.last_login, u.is_active,
                   COALESCE(p.is_admin, 0) as is_admin
            FROM users u
            LEFT JOIN user_permissions p ON u.id = p.user_id
        """)
        users = cursor.fetchall()
        for user in users:
            user_id, username, full_name, created_date, last_login, is_active, is_admin = user
            status = "نشط" if is_active else "معطل"
            admin_status = "نعم" if is_admin else "لا"
            if not last_login:
                last_login = "لم يسجل الدخول بعد"
            item_id = self.users_tree.insert("", tk.END, values=(
                user_id, username, full_name, created_date, last_login, status, admin_status))

            if not is_active:
                self.users_tree.item(item_id, tags=("inactive",))
            elif is_admin:
                self.users_tree.item(item_id, tags=("admin",))
            else:
                self.users_tree.item(item_id, tags=("active",))

    def add_user(self):
        add_window = tk.Toplevel(self.user_window)
        add_window.title("إضافة مستخدم جديد")
        add_window.geometry("400x430")
        add_window.configure(bg=self.colors["light"])
        add_window.transient(self.user_window)
        add_window.grab_set()

        x = (add_window.winfo_screenwidth() - 400) // 2
        y = (add_window.winfo_screenheight() - 430) // 2
        add_window.geometry(f"400x430+{x}+{y}")

        tk.Label(
            add_window,
            text="إضافة مستخدم جديد",
            font=self.fonts["title"],
            bg=self.colors["primary"],
            fg="white",
            padx=10, pady=10, width=400
        ).pack(fill=tk.X)

        form_frame = tk.Frame(add_window, bg=self.colors["light"], padx=20, pady=20)
        form_frame.pack(fill=tk.BOTH)

        tk.Label(form_frame, text="اسم المستخدم:", font=self.fonts["text_bold"], bg=self.colors["light"],
                 anchor=tk.E).grid(row=0, column=1, padx=5, pady=8, sticky=tk.E)
        username_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25)
        username_entry.grid(row=0, column=0, padx=5, pady=8, sticky=tk.W)

        tk.Label(form_frame, text="الاسم الكامل:", font=self.fonts["text_bold"], bg=self.colors["light"],
                 anchor=tk.E).grid(row=1, column=1, padx=5, pady=8, sticky=tk.E)
        fullname_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25)
        fullname_entry.grid(row=1, column=0, padx=5, pady=8, sticky=tk.W)

        tk.Label(form_frame, text="كلمة المرور:", font=self.fonts["text_bold"], bg=self.colors["light"],
                 anchor=tk.E).grid(row=2, column=1, padx=5, pady=8, sticky=tk.E)
        password_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25, show="*")
        password_entry.grid(row=2, column=0, padx=5, pady=8, sticky=tk.W)

        tk.Label(form_frame, text="تأكيد كلمة المرور:", font=self.fonts["text_bold"], bg=self.colors["light"],
                 anchor=tk.E).grid(row=3, column=1, padx=5, pady=8, sticky=tk.E)
        confirm_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25, show="*")
        confirm_entry.grid(row=3, column=0, padx=5, pady=8, sticky=tk.W)

        is_admin_var = tk.IntVar(value=0)
        admin_check = tk.Checkbutton(
            form_frame,
            text="جعل هذا المستخدم مشرفًا",
            variable=is_admin_var,
            font=self.fonts["text"],
            bg=self.colors["light"]
        )
        admin_check.grid(row=4, column=0, columnspan=2, padx=5, pady=8, sticky=tk.W)

        button_frame = tk.Frame(add_window, bg=self.colors["light"], pady=10)
        button_frame.pack(fill=tk.X)

        def save_user():
            username = username_entry.get().strip()
            fullname = fullname_entry.get().strip()
            password = password_entry.get().strip()
            confirm = confirm_entry.get().strip()
            is_admin = is_admin_var.get()

            if not all([username, fullname, password, confirm]):
                messagebox.showwarning("تنبيه", "يجب ملء جميع الحقول")
                return
            if password != confirm:
                messagebox.showwarning("تنبيه", "كلمات المرور غير متطابقة")
                return
            if not re.match(r'^[a-zA-Z0-9_-]+$', username):
                messagebox.showwarning("تنبيه", "اسم المستخدم يجب أن يتكون من حروف إنجليزية وأرقام وشرطات فقط")
                return
            cursor = self.conn.cursor()
            cursor.execute("SELECT COUNT(*) FROM users WHERE username=?", (username,))
            count = cursor.fetchone()[0]
            if count > 0:
                messagebox.showwarning("تنبيه", "اسم المستخدم موجود بالفعل")
                return

            hashed_pwd = hashlib.sha256(password.encode()).hexdigest()
            try:
                with self.conn:
                    self.conn.execute("""
                        INSERT INTO users (username, password, full_name, created_date, is_active)
                        VALUES (?, ?, ?, ?, ?)
                    """, (
                        username,
                        hashed_pwd,
                        fullname,
                        datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
                        1
                    ))

                    cursor.execute("SELECT id FROM users WHERE username=?", (username,))
                    user_id = cursor.fetchone()[0]

                    if is_admin:
                        self.conn.execute("""
                            INSERT INTO user_permissions (
                                user_id, can_edit_attendance, can_add_students, 
                                can_edit_students, can_delete_students, can_view_edit_history,
                                can_reset_attendance, can_export_data, can_import_data, is_admin
                            ) VALUES (?, 1, 1, 1, 1, 1, 1, 1, 1, 1)
                        """, (user_id,))
                    else:
                        self.conn.execute("""
                            INSERT INTO user_permissions (
                                user_id, can_edit_attendance, can_add_students, 
                                can_edit_students, can_delete_students, can_view_edit_history,
                                can_reset_attendance, can_export_data, can_import_data, is_admin
                            ) VALUES (?, 1, 1, 1, 0, 0, 0, 1, 0, 0)
                        """, (user_id,))

                messagebox.showinfo("نجاح", "تم إضافة المستخدم بنجاح")
                add_window.destroy()
                self.load_users()
            except Exception as e:
                messagebox.showerror("خطأ", f"حدث خطأ أثناء إضافة المستخدم: {str(e)}")

        save_btn = tk.Button(button_frame, text="حفظ", font=self.fonts["text_bold"], bg=self.colors["success"],
                             fg="white",
                             padx=15, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=save_user)
        save_btn.pack(side=tk.LEFT, padx=10)
        cancel_btn = tk.Button(button_frame, text="إلغاء", font=self.fonts["text_bold"], bg=self.colors["danger"],
                               fg="white",
                               padx=15, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=add_window.destroy)
        cancel_btn.pack(side=tk.RIGHT, padx=10)

    def edit_user(self):
        selected_item = self.users_tree.selection()
        if not selected_item:
            messagebox.showinfo("تنبيه", "الرجاء تحديد مستخدم من القائمة")
            return
        values = self.users_tree.item(selected_item, "values")
        user_id = values[0]
        cursor = self.conn.cursor()
        cursor.execute("SELECT * FROM users WHERE id=?", (user_id,))
        user = cursor.fetchone()
        if not user:
            messagebox.showerror("خطأ", "لم يتم العثور على المستخدم")
            return

        cursor.execute("SELECT * FROM user_permissions WHERE user_id=?", (user_id,))
        permissions = cursor.fetchone()
        is_admin = 0
        if permissions:
            is_admin = permissions[9]

        edit_window = tk.Toplevel(self.user_window)
        edit_window.title("تعديل المستخدم")
        edit_window.geometry("400x430")
        edit_window.configure(bg=self.colors["light"])
        edit_window.transient(self.user_window)
        edit_window.grab_set()

        x = (edit_window.winfo_screenwidth() - 400) // 2
        y = (edit_window.winfo_screenheight() - 430) // 2
        edit_window.geometry(f"400x430+{x}+{y}")

        tk.Label(
            edit_window,
            text=f"تعديل المستخدم: {user[1]}",
            font=self.fonts["title"],
            bg=self.colors["primary"],
            fg="white",
            padx=10, pady=10, width=400
        ).pack(fill=tk.X)

        form_frame = tk.Frame(edit_window, bg=self.colors["light"], padx=20, pady=20)
        form_frame.pack(fill=tk.BOTH)

        tk.Label(form_frame, text="اسم المستخدم:", font=self.fonts["text_bold"], bg=self.colors["light"],
                 anchor=tk.E).grid(row=0, column=1, padx=5, pady=8, sticky=tk.E)
        username_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25)
        username_entry.insert(0, user[1])
        username_entry.grid(row=0, column=0, padx=5, pady=8, sticky=tk.W)

        tk.Label(form_frame, text="الاسم الكامل:", font=self.fonts["text_bold"], bg=self.colors["light"],
                 anchor=tk.E).grid(row=1, column=1, padx=5, pady=8, sticky=tk.E)
        fullname_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25)
        fullname_entry.insert(0, user[3])
        fullname_entry.grid(row=1, column=0, padx=5, pady=8, sticky=tk.W)

        tk.Label(form_frame, text="كلمة المرور الجديدة:", font=self.fonts["text_bold"], bg=self.colors["light"],
                 anchor=tk.E).grid(row=2, column=1, padx=5, pady=8, sticky=tk.E)
        password_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25, show="*")
        password_entry.grid(row=2, column=0, padx=5, pady=8, sticky=tk.W)

        tk.Label(form_frame, text="تأكيد كلمة المرور:", font=self.fonts["text_bold"], bg=self.colors["light"],
                 anchor=tk.E).grid(row=3, column=1, padx=5, pady=8, sticky=tk.E)
        confirm_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25, show="*")
        confirm_entry.grid(row=3, column=0, padx=5, pady=8, sticky=tk.W)

        is_admin_var = tk.IntVar(value=is_admin)
        admin_check = tk.Checkbutton(
            form_frame,
            text="هذا المستخدم مشرف",
            variable=is_admin_var,
            font=self.fonts["text"],
            bg=self.colors["light"]
        )
        admin_check.grid(row=4, column=0, columnspan=2, padx=5, pady=8, sticky=tk.W)

        button_frame = tk.Frame(edit_window, bg=self.colors["light"], pady=10)
        button_frame.pack(fill=tk.X)

        def save_changes():
            username = username_entry.get().strip()
            fullname = fullname_entry.get().strip()
            password = password_entry.get().strip()
            confirm = confirm_entry.get().strip()
            is_admin = is_admin_var.get()

            if not all([username, fullname]):
                messagebox.showwarning("تنبيه", "يجب ملء الحقول الأساسية")
                return

            if password:
                if password != confirm:
                    messagebox.showwarning("تنبيه", "كلمات المرور غير متطابقة")
                    return
            try:
                with self.conn:
                    if password:
                        hashed_pwd = hashlib.sha256(password.encode()).hexdigest()
                        self.conn.execute("UPDATE users SET username=?, full_name=?, password=? WHERE id=?",
                                          (username, fullname, hashed_pwd, user[0]))
                    else:
                        self.conn.execute("UPDATE users SET username=?, full_name=? WHERE id=?",
                                          (username, fullname, user[0]))

                    cursor = self.conn.cursor()
                    cursor.execute("SELECT COUNT(*) FROM user_permissions WHERE user_id=?", (user[0],))
                    has_permissions = cursor.fetchone()[0] > 0

                    if has_permissions:
                        self.conn.execute("UPDATE user_permissions SET is_admin=? WHERE user_id=?",
                                          (is_admin, user[0]))
                    else:
                        if is_admin:
                            self.conn.execute("""
                                INSERT INTO user_permissions (
                                    user_id, can_edit_attendance, can_add_students, 
                                    can_edit_students, can_delete_students, can_view_edit_history,
                                    can_reset_attendance, can_export_data, can_import_data, is_admin
                                ) VALUES (?, 1, 1, 1, 1, 1, 1, 1, 1, 1)
                            """, (user[0],))
                        else:
                            self.conn.execute("""
                                INSERT INTO user_permissions (
                                    user_id, can_edit_attendance, can_add_students, 
                                    can_edit_students, can_delete_students, can_view_edit_history,
                                    can_reset_attendance, can_export_data, can_import_data, is_admin
                                ) VALUES (?, 1, 1, 1, 0, 0, 0, 1, 0, 0)
                            """, (user[0],))

                messagebox.showinfo("نجاح", "تم تحديث بيانات المستخدم بنجاح")
                edit_window.destroy()
                self.load_users()
            except Exception as e:
                messagebox.showerror("خطأ", f"حدث خطأ أثناء تحديث المستخدم: {str(e)}")

        save_btn = tk.Button(button_frame, text="حفظ التغييرات", font=self.fonts["text_bold"],
                             bg=self.colors["warning"], fg="white",
                             padx=15, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=save_changes)
        save_btn.pack(side=tk.LEFT, padx=10)
        cancel_btn = tk.Button(button_frame, text="إلغاء", font=self.fonts["text_bold"], bg=self.colors["danger"],
                               fg="white",
                               padx=15, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=edit_window.destroy)
        cancel_btn.pack(side=tk.RIGHT, padx=10)

    def toggle_user_active(self):
        selected_item = self.users_tree.selection()
        if not selected_item:
            messagebox.showinfo("تنبيه", "الرجاء تحديد مستخدم من القائمة")
            return
        values = self.users_tree.item(selected_item, "values")
        user_id = values[0]
        username = values[1]
        status_text = values[5]
        if username == self.current_user["username"]:
            messagebox.showwarning("تنبيه", "لا يمكن تعطيل المستخدم الحالي")
            return
        new_status = 0 if status_text == "نشط" else 1
        status_msg = "تفعيل" if new_status == 1 else "تعطيل"
        if not messagebox.askyesnocancel("تأكيد", f"هل تريد {status_msg} المستخدم {username}؟"):
            return
        try:
            with self.conn:
                self.conn.execute("UPDATE users SET is_active=? WHERE id=?", (new_status, user_id))
            messagebox.showinfo("نجاح", f"تم {status_msg} المستخدم بنجاح")
            self.load_users()
        except Exception as e:
            messagebox.showerror("خطأ", f"حدث خطأ: {str(e)}")

    def delete_user(self):
        selected_item = self.users_tree.selection()
        if not selected_item:
            messagebox.showinfo("تنبيه", "الرجاء تحديد مستخدم من القائمة")
            return
        values = self.users_tree.item(selected_item, "values")
        user_id = values[0]
        username = values[1]
        if username == self.current_user["username"]:
            messagebox.showwarning("تنبيه", "لا يمكن حذف المستخدم الحالي")
            return
        if not messagebox.askyesnocancel("تأكيد", f"هل تريد حذف المستخدم {username}؟\nلا يمكن التراجع عن العملية!"):
            return
        try:
            with self.conn:
                self.conn.execute("DELETE FROM user_permissions WHERE user_id=?", (user_id,))
                self.conn.execute("DELETE FROM users WHERE id=?", (user_id,))
            messagebox.showinfo("نجاح", "تم حذف المستخدم بنجاح")
            self.load_users()
        except Exception as e:
            messagebox.showerror("خطأ", f"حدث خطأ أثناء حذف المستخدم: {str(e)}")

    def manage_user_permissions(self):
        selected_item = self.users_tree.selection()
        if not selected_item:
            messagebox.showinfo("تنبيه", "الرجاء تحديد مستخدم من القائمة")
            return
        values = self.users_tree.item(selected_item, "values")
        user_id = values[0]
        username = values[1]

        cursor = self.conn.cursor()
        cursor.execute("SELECT * FROM user_permissions WHERE user_id=?", (user_id,))
        permissions = cursor.fetchone()

        if not permissions:
            is_admin = 1 if values[6] == "نعم" else 0
            with self.conn:
                cursor.execute("""
                    INSERT INTO user_permissions (
                        user_id, can_edit_attendance, can_add_students, 
                        can_edit_students, can_delete_students, can_view_edit_history,
                        can_reset_attendance, can_export_data, can_import_data, is_admin
                    ) VALUES (?, 1, 1, 1, ?, ?, ?, 1, ?, ?)
                """, (user_id, is_admin, is_admin, is_admin, is_admin, is_admin))

            cursor.execute("SELECT * FROM user_permissions WHERE user_id=?", (user_id,))
            permissions = cursor.fetchone()

        perm_window = tk.Toplevel(self.user_window)
        perm_window.title(f"إدارة صلاحيات المستخدم: {username}")
        perm_window.geometry("500x550")
        perm_window.configure(bg=self.colors["light"])
        perm_window.transient(self.user_window)
        perm_window.grab_set()

        x = (perm_window.winfo_screenwidth() - 500) // 2
        y = (perm_window.winfo_screenheight() - 550) // 2
        perm_window.geometry(f"500x550+{x}+{y}")

        tk.Label(
            perm_window,
            text=f"صلاحيات المستخدم: {username}",
            font=self.fonts["title"],
            bg=self.colors["primary"],
            fg="white",
            padx=10, pady=10
        ).pack(fill=tk.X)

        perm_frame = tk.Frame(perm_window, bg=self.colors["light"], padx=20, pady=20)
        perm_frame.pack(fill=tk.BOTH, expand=True)

        is_admin_var = tk.IntVar(value=permissions[9])
        can_edit_attendance_var = tk.IntVar(value=permissions[1])
        can_add_students_var = tk.IntVar(value=permissions[2])
        can_edit_students_var = tk.IntVar(value=permissions[3])
        can_delete_students_var = tk.IntVar(value=permissions[4])
        can_view_edit_history_var = tk.IntVar(value=permissions[5])
        can_reset_attendance_var = tk.IntVar(value=permissions[6])
        can_export_data_var = tk.IntVar(value=permissions[7])
        can_import_data_var = tk.IntVar(value=permissions[8])

        def update_permissions():
            is_admin = is_admin_var.get()
            if is_admin:
                for var in [can_edit_attendance_var, can_add_students_var, can_edit_students_var,
                            can_delete_students_var, can_view_edit_history_var, can_reset_attendance_var,
                            can_export_data_var, can_import_data_var]:
                    var.set(1)

                for checkbox in permission_checkboxes:
                    checkbox.config(state=tk.DISABLED)
            else:
                for checkbox in permission_checkboxes:
                    checkbox.config(state=tk.NORMAL)

        admin_title = tk.Label(perm_frame, text="صلاحيات عامة:", font=self.fonts["text_bold"], bg=self.colors["light"])
        admin_title.grid(row=0, column=0, sticky=tk.W, pady=(0, 10))

        admin_check = tk.Checkbutton(
            perm_frame,
            text="هذا المستخدم مشرف (يملك كل الصلاحيات)",
            variable=is_admin_var,
            font=self.fonts["text_bold"],
            bg=self.colors["light"],
            command=update_permissions
        )
        admin_check.grid(row=1, column=0, sticky=tk.W, pady=5)

        specific_title = tk.Label(perm_frame, text="صلاحيات محددة:", font=self.fonts["text_bold"],
                                  bg=self.colors["light"])
        specific_title.grid(row=2, column=0, sticky=tk.W, pady=(20, 10))

        permission_options = [
            (can_edit_attendance_var, "تعديل سجلات الحضور والغياب"),
            (can_add_students_var, "إضافة طلاب جدد"),
            (can_edit_students_var, "تعديل بيانات الطلاب"),
            (can_delete_students_var, "حذف الطلاب"),
            (can_view_edit_history_var, "عرض سجل التعديلات (من عدّل ومتى)"),
            (can_reset_attendance_var, "إعادة تعيين سجلات الحضور"),
            (can_export_data_var, "تصدير البيانات"),
            (can_import_data_var, "استيراد البيانات من Excel")
        ]

        permission_checkboxes = []
        for i, (var, text) in enumerate(permission_options):
            checkbox = tk.Checkbutton(
                perm_frame,
                text=text,
                variable=var,
                font=self.fonts["text"],
                bg=self.colors["light"]
            )
            checkbox.grid(row=i + 3, column=0, sticky=tk.W, pady=5)
            permission_checkboxes.append(checkbox)

        update_permissions()

        button_frame = tk.Frame(perm_window, bg=self.colors["light"], pady=10)
        button_frame.pack(fill=tk.X, padx=20)

        def save_permissions():
            try:
                with self.conn:
                    self.conn.execute("""
                        UPDATE user_permissions SET
                            is_admin=?,
                            can_edit_attendance=?,
                            can_add_students=?,
                            can_edit_students=?,
                            can_delete_students=?,
                            can_view_edit_history=?,
                            can_reset_attendance=?,
                            can_export_data=?,
                            can_import_data=?
                        WHERE user_id=?
                    """, (
                        is_admin_var.get(),
                        can_edit_attendance_var.get(),
                        can_add_students_var.get(),
                        can_edit_students_var.get(),
                        can_delete_students_var.get(),
                        can_view_edit_history_var.get(),
                        can_reset_attendance_var.get(),
                        can_export_data_var.get(),
                        can_import_data_var.get(),
                        user_id
                    ))
                messagebox.showinfo("نجاح", "تم تحديث صلاحيات المستخدم بنجاح")
                perm_window.destroy()
                self.load_users()
            except Exception as e:
                messagebox.showerror("خطأ", f"حدث خطأ أثناء تحديث الصلاحيات: {str(e)}")

        save_btn = tk.Button(button_frame, text="حفظ الصلاحيات", font=self.fonts["text_bold"],
                             bg=self.colors["success"],
                             fg="white", padx=15, pady=5, bd=0, relief=tk.FLAT, cursor="hand2",
                             command=save_permissions)
        save_btn.pack(side=tk.LEFT, padx=10)

        cancel_btn = tk.Button(button_frame, text="إلغاء", font=self.fonts["text_bold"], bg=self.colors["danger"],
                               fg="white", padx=15, pady=5, bd=0, relief=tk.FLAT, cursor="hand2",
                               command=perm_window.destroy)
        cancel_btn.pack(side=tk.RIGHT, padx=10)

    
    # تخزين الحجم الأصلي للخطوط قبل التعديل
    self.original_fonts = {
        "large_title": ("Tajawal", 24, "bold"),
        "title": ("Tajawal", 18, "bold"),
        "subtitle": ("Tajawal", 16, "bold"),
        "text": ("Tajawal", 12),
        "text_bold": ("Tajawal", 12, "bold"),
        "small": ("Tajawal", 10)
    }

    # تعريف الألوان
    self.colors = {
        "primary": "#1a73e8",
        "secondary": "#4285f4",
        "success": "#34a853",
        "danger": "#ea4335",
        "warning": "#fbbc05",
        "light": "#f0f4f8",
        "dark": "#202124",
        "present": "#34a853",
        "absent": "#ea4335",
        "late": "#fbbc05",
        "excused": "#4285f4",
        "not_started": "#FFA500",
        "excluded": "#9C27B0",
        "field_application": "#909090",
        "student_day": "#A9A9A9",
        "evening_remote": "#A0A0A0",
        "death_case": "#7E57C2",
        "hospital": "#26A69A",
    }

    # تحديد التخطيط الأمثل بناءً على حجم الشاشة
    self.determine_best_layout()

    # تعريف الخطوط بعد تحديد الحجم المناسب
    self.fonts = self.original_fonts.copy()

    self.style = ttk.Style(self.root)
    self.style.theme_use("clam")
    self.setup_styles()

    # ربط حدث تغيير حجم النافذة بدالة التكيف التلقائي
    self.root.bind('<Configure>', self.on_window_resize)

    self.tab_control = ttk.Notebook(self.root, style="Bold.TNotebook")
    self.tab_control.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)

    if conn:
        self.conn = conn
    else:
        self.conn = sqlite3.connect("attendance.db")

    self.create_tables()
    self.create_indexes()

    self.today = datetime.datetime.now().strftime("%Y-%m-%d")

    # تعريف متغيرات الإحصائيات
    self.total_students_var = tk.StringVar(value="0")
    self.present_students_var = tk.StringVar(value="0")
    self.absent_students_var = tk.StringVar(value="0")
    self.late_students_var = tk.StringVar(value="0")
    self.excused_students_var = tk.StringVar(value="0")
    self.not_started_students_var = tk.StringVar(value="0")
    self.field_application_var = tk.StringVar(value="0")
    self.student_day_var = tk.StringVar(value="0")
    self.evening_remote_var = tk.StringVar(value="0")
    self.attendance_rate_var = tk.StringVar(value="0%")
    self.death_case_var = tk.StringVar(value="0")
    self.hospital_var = tk.StringVar(value="0")

    # تخزين إشارات لبطاقات الإحصائيات للتحكم فيها لاحقًا
    self.stats_cards = []

    self.create_header()

    self.attendance_tab = tk.Frame(self.tab_control, bg=self.colors["light"])
    self.tab_control.add(self.attendance_tab, text="سجل الحضور")

    self.attendance_log_tab = tk.Frame(self.tab_control, bg=self.colors["light"])
    self.tab_control.add(self.attendance_log_tab, text="استعراض الحضور")

    self.students_tab = tk.Frame(self.tab_control, bg=self.colors["light"])
    self.tab_control.add(self.students_tab, text="إدارة المتدربين")

    # إعداد قاعدة البيانات
    if conn:
        self.conn = conn
    else:
        # فتح الاتصال باستخدام خيارات تحسين الأداء
        self.conn = sqlite3.connect("attendance.db", isolation_level=None)

        # تحسين أداء قاعدة البيانات
        self.conn.execute("PRAGMA journal_mode = WAL")  # استخدام وضع WAL للتخزين
        self.conn.execute("PRAGMA synchronous = NORMAL")  # تقليل وقت الانتظار للكتابة
        self.conn.execute("PRAGMA cache_size = -20000")  # استخدام ذاكرة تخزين مؤقت أكبر (حوالي 20 ميجابايت)
        self.conn.execute("PRAGMA temp_store = MEMORY")  # استخدام الذاكرة للتخزين المؤقت

    # إنشاء وتحسين الجداول والفهارس
    self.create_tables()
    self.create_indexes()  # دالة جديدة تمت إضافتها

    if self.current_user["permissions"]["is_admin"]:
        self.users_tab = tk.Frame(self.tab_control, bg=self.colors["light"])
        self.tab_control.add(self.users_tab, text="إدارة المستخدمين")
        self.setup_users_tab()

    self.setup_attendance_tab()
    self.setup_attendance_log_tab()
    self.setup_students_tab()

    self.status_bar = tk.Label(
        self.root,
        text=f"مرحبًا {self.current_user['full_name']} (مستخدم: {self.current_user['username']})",
        font=self.fonts["small"], bg=self.colors["primary"], fg="white", pady=5
    )
    self.status_bar.pack(side=tk.BOTTOM, fill=tk.X)

    self.archive_manager = ArchiveManager(self.root, self, self.colors, self.fonts)

    # إضافة تبويب الأرشيف
    self.archive_tab = tk.Frame(self.tab_control, bg=self.colors["light"])
    self.tab_control.add(self.archive_tab, text="أرشيف الدورات")
    self.setup_archive_tab()

    # إضافة متغيرات تتبع النشاط لتسجيل الخروج التلقائي - الجزء الجديد
    self.last_activity_time = time.time()
    self.inactivity_timeout = 2400  # 30 ثانية للتجربة (يمكن تغييرها إلى 1200 للإعداد النهائي - 20 دقيقة)
    self.activity_check_id = None

    # ربط حركات المستخدم بتحديث وقت النشاط - الجزء الجديد
    self.root.bind("<Motion>", self.reset_activity_timer)
    self.root.bind("<Button-1>", self.reset_activity_timer)
    self.root.bind("<ButtonRelease-1>", self.reset_activity_timer)
    self.root.bind("<Key>", self.reset_activity_timer)

    # ربط دالة إغلاق النافذة - الجزء الجديد
    self.root.protocol("WM_DELETE_WINDOW", self.on_closing)

    # تحديث واجهة البرنامج
    self.update_students_tree()
    self.update_statistics()
    self.update_attendance_display()

    # تطبيق التخطيط المناسب بعد إنشاء كل العناصر
    if self.screen_info["is_small_screen"]:
        self.apply_compact_layout()
    else:
        self.apply_expanded_layout()

    # بدء فحص النشاط - الجزء الجديد
    self.check_inactivity()

def determine_best_layout(self):
    """تحديد التخطيط الأمثل بناءً على إعدادات الشاشة"""
    screen_width = self.root.winfo_screenwidth()
    screen_height = self.root.winfo_screenheight()

    # حساب حجم النافذة المناسب (90% من حجم الشاشة مع حد أقصى)
    window_width = min(int(screen_width * 0.9), 1400)
    window_height = min(int(screen_height * 0.9), 800)

    # توسيط النافذة
    x = (screen_width - window_width) // 2
    y = (screen_height - window_height) // 2

    # تعيين حجم وموقع النافذة
    self.root.geometry(f"{window_width}x{window_height}+{x}+{y}")

    # تعيين الحد الأدنى لحجم النافذة
    self.root.minsize(800, 600)

    # حفظ معلومات الشاشة لاستخدامها لاحقاً
    self.screen_info = {
        "screen_width": screen_width,
        "screen_height": screen_height,
        "window_width": window_width,
        "window_height": window_height,
        "is_small_screen": screen_width < 1200,
        "is_high_dpi": screen_width > 2000,
        "scale_factor": min(window_width / 1366, window_height / 768)  # عامل القياس النسبي
    }

    # تعديل أحجام الخطوط بناءً على عامل القياس إذا كانت شاشة عالية الدقة
    if self.screen_info["is_high_dpi"]:
        self.adjust_font_sizes(self.screen_info["scale_factor"])

def setup_styles(self):
    """إعداد أنماط العناصر الرسومية"""
    self.style = ttk.Style()  # ✅ ضروري تعريف الكائن قبل الاستخدام

    self.style.configure("Bold.TNotebook.Tab", font=self.fonts["subtitle"])
    self.style.configure(
        "Bold.Treeview",
        background=self.colors["light"],
        foreground=self.colors["dark"],
        rowheight=30,
        fieldbackground=self.colors["light"],
        font=self.fonts["text_bold"]
    )
    self.style.configure(
        "Bold.Treeview.Heading",
        font=self.fonts["text_bold"],
        background=self.colors["primary"],
        foreground="white"
    )
    self.style.map('Bold.Treeview', background=[('selected', self.colors["primary"])])

    self.style.configure(
        "Profile.Treeview",
        background=self.colors["light"],
        foreground=self.colors["dark"],
        rowheight=32,
        fieldbackground=self.colors["light"],
        font=self.fonts["text_bold"]
    )
    self.style.configure(
        "Profile.Treeview.Heading",
        font=self.fonts["subtitle"],
        background=self.colors["primary"],
        foreground="white"
    )

def on_window_resize(self, event=None):
    """تستجيب لتغيير حجم النافذة وتعدل العناصر تلقائياً"""
    # تجاهل الأحداث الصغيرة جدًا لتحسين الأداء
    if hasattr(self, 'last_width') and hasattr(self, 'last_height'):
        width_diff = abs(self.root.winfo_width() - self.last_width)
        height_diff = abs(self.root.winfo_height() - self.last_height)
        if width_diff < 10 and height_diff < 10:
            return

    # تخزين الحجم الحالي
    self.last_width = self.root.winfo_width()
    self.last_height = self.root.winfo_height()

    # تحديث معلومات الشاشة
    self.screen_info["window_width"] = self.last_width
    self.screen_info["window_height"] = self.last_height
    self.screen_info["is_small_screen"] = self.last_width < 1200

    # تعديل عرض الأعمدة في الجداول
    self.adjust_column_widths()

    # تعديل حجم النصوص في علامات التبويب
    self.adjust_tab_text()

    # تطبيق التخطيط المناسب
    if self.screen_info["is_small_screen"]:
        self.apply_compact_layout()
    else:
        self.apply_expanded_layout()

def adjust_font_sizes(self, scale_factor):
    """تعديل أحجام الخطوط بناءً على عامل القياس"""
    # تحديث قيم الخطوط بناءً على عامل القياس
    self.fonts = {
        "large_title": ("Tajawal", int(self.original_fonts["large_title"][1] * scale_factor), "bold"),
        "title": ("Tajawal", int(self.original_fonts["title"][1] * scale_factor), "bold"),
        "subtitle": ("Tajawal", int(self.original_fonts["subtitle"][1] * scale_factor), "bold"),
        "text": ("Tajawal", int(self.original_fonts["text"][1] * scale_factor)),
        "text_bold": ("Tajawal", int(self.original_fonts["text_bold"][1] * scale_factor), "bold"),
        "small": ("Tajawal", int(self.original_fonts["small"][1] * scale_factor))
    }

    # تحديث أنماط العناصر الرسومية
    self.setup_styles()

def adjust_column_widths(self):
    """تعديل عرض الأعمدة في جداول العرض بناءً على حجم النافذة"""
    try:
        # تعديل جدول سجل الحضور
        if hasattr(self, 'attendance_tree'):
            available_width = self.attendance_tree.winfo_width()
            if available_width > 50:  # تأكد من تهيئة العنصر
                # تحديد النسب المئوية للأعمدة - زيادة نسبة عمود الاسم
                col_ratios = [0.12, 0.28, 0.10, 0.12, 0.10, 0.10, 0.10, 0.08]  # زيادة عرض الاسم من 0.20 إلى 0.28

                # حساب العرض الفعلي لكل عمود
                for i, ratio in enumerate(col_ratios):
                    width = int(available_width * ratio)
                    if width > 10:  # تجنب القيم السالبة أو الصغيرة جدًا
                        self.attendance_tree.column(self.attendance_tree["columns"][i], width=width)

        # تعديل جدول المتدربين
        if hasattr(self, 'students_tree'):
            available_width = self.students_tree.winfo_width()
            if available_width > 50:
                col_ratios = [0.15, 0.35, 0.15, 0.15, 0.15, 0.05]  # زيادة عرض الاسم من 0.30 إلى 0.35
                for i, ratio in enumerate(col_ratios):
                    width = int(available_width * ratio)
                    if width > 10:
                        self.students_tree.column(self.students_tree["columns"][i], width=width)
    except Exception as e:
        print(f"خطأ عند تعديل عرض الأعمدة: {str(e)}")

def adjust_tab_text(self):
    """تعديل نصوص علامات التبويب حسب المساحة المتاحة"""
    window_width = self.root.winfo_width()

    # على الشاشات الصغيرة، استخدم أسماء مختصرة
    if window_width < 800:
        self.tab_control.tab(0, text="الحضور")
        self.tab_control.tab(1, text="السجل")
        self.tab_control.tab(2, text="المتدربين")
        if self.current_user["permissions"]["is_admin"]:
            self.tab_control.tab(3, text="المستخدمين")
            self.tab_control.tab(4, text="الأرشيف")
        else:
            self.tab_control.tab(3, text="الأرشيف")
    else:
        # على الشاشات الكبيرة، استخدم الأسماء الكاملة
        self.tab_control.tab(0, text="سجل الحضور")
        self.tab_control.tab(1, text="استعراض الحضور")
        self.tab_control.tab(2, text="إدارة المتدربين")
        if self.current_user["permissions"]["is_admin"]:
            self.tab_control.tab(3, text="إدارة المستخدمين")
            self.tab_control.tab(4, text="أرشيف الدورات")
        else:
            self.tab_control.tab(3, text="أرشيف الدورات")

def apply_compact_layout(self):
    """تطبيق التخطيط المضغوط للشاشات الصغيرة"""
    # تخزين وضع التخطيط الحالي
    self.current_layout = "compact"

    # تنظيم الإحصائيات في عمود واحد
    self.organize_stats_in_one_column()

    # تعديل عدد الأزرار المعروضة
    self.organize_buttons_for_small_screen()

def apply_expanded_layout(self):
    """تطبيق التخطيط الموسع للشاشات الكبيرة"""
    # تخزين وضع التخطيط الحالي
    self.current_layout = "expanded"

    # تنظيم الإحصائيات في صفين
    self.organize_stats_in_two_rows()

    # عرض كامل للأزرار
    self.show_all_buttons()

def organize_stats_in_one_column(self):
    """تنظيم بطاقات الإحصائيات في عمود واحد للشاشات الصغيرة"""
    # التنفيذ فقط إذا كان التخطيط الحالي ليس مضغوطًا
    if hasattr(self, 'current_layout') and self.current_layout == "compact":
        return

    if hasattr(self, 'stats_cards') and self.stats_cards:
        stats_frame = self.find_parent_frame(self.stats_cards[0])

        if stats_frame:
            # إزالة الصفوف القديمة
            for child in stats_frame.winfo_children():
                if child != self.stats_cards[0].master:  # حفظ الإطار الرئيسي
                    child.destroy()

            # إنشاء إطار واحد للعمود
            column_frame = tk.Frame(stats_frame, bg=self.colors["light"])
            column_frame.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)

            # إعادة تنظيم بطاقات الإحصائيات
            for i, card in enumerate(self.stats_cards):
                card.pack_forget()  # إزالة من التخطيط الحالي
                card.pack(in_=column_frame, fill=tk.X, padx=5, pady=2)  # إعادة تنظيم في العمود

def organize_stats_in_two_rows(self):
    """تنظيم بطاقات الإحصائيات في صفين للشاشات الكبيرة"""
    # التنفيذ فقط إذا كان التخطيط الحالي ليس موسعًا
    if hasattr(self, 'current_layout') and self.current_layout == "expanded":
        return

    if hasattr(self, 'stats_cards') and self.stats_cards:
        stats_frame = self.find_parent_frame(self.stats_cards[0])

        if stats_frame:
            # إزالة العمود القديم
            for child in stats_frame.winfo_children():
                child.destroy()

            # إنشاء إطارين للصفين
            top_counter_frame = tk.Frame(stats_frame, bg=self.colors["light"])
            top_counter_frame.pack(fill=tk.X, padx=5, pady=5)

            bottom_counter_frame = tk.Frame(stats_frame, bg=self.colors["light"])
            bottom_counter_frame.pack(fill=tk.X, padx=5, pady=5)

            # توزيع بطاقات الإحصائيات على الصفين
            half_count = len(self.stats_cards) // 2

            for i, card in enumerate(self.stats_cards):
                card.pack_forget()  # إزالة من التخطيط الحالي

                if i < half_count:
                    # الصف الأول
                    card.pack(in_=top_counter_frame, side=tk.RIGHT, padx=5, fill=tk.X, expand=True)
                else:
                    # الصف الثاني
                    card.pack(in_=bottom_counter_frame, side=tk.RIGHT, padx=5, fill=tk.X, expand=True)

def find_parent_frame(self, widget):
    """العثور على إطار الأب لعنصر واجهة"""
    if widget is None:
        return None

    parent = widget.master
    while parent is not None:
        if isinstance(parent, tk.LabelFrame) and parent.cget("text") == "إحصائيات اليوم":
            return parent
        parent = parent.master

    return None

def organize_buttons_for_small_screen(self):
    """تنظيم الأزرار للشاشات الصغيرة"""
    # تنفيذ فقط عند الضرورة
    if hasattr(self, 'current_layout') and self.current_layout == "compact":
        return

    # هنا يمكن تنفيذ تغييرات على تنظيم الأزرار
    # مثل إنشاء قائمة منسدلة لبعض الأزرار الأقل استخداماً
    # أو تصغير حجم الأزرار أو تقليل النص المعروض

    pass  # يمكن تنفيذ المزيد حسب الاحتياج

def show_all_buttons(self):
    """عرض جميع الأزرار للشاشات الكبيرة"""
    # تنفيذ فقط عند الضرورة
    if hasattr(self, 'current_layout') and self.current_layout == "expanded":
        return

    # إعادة الأزرار إلى حالتها الطبيعية
    # مثل إظهار جميع الأزرار وإعادة النصوص الكاملة

    pass  # يمكن تنفيذ المزيد حسب الاحتياج

def setup_users_tab(self):
    user_management_frame = tk.Frame(self.users_tab, bg=self.colors["light"])
    user_management_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)

    tk.Label(
        user_management_frame,
        text="إدارة مستخدمي النظام (محمي بكلمة مرور) - خاص بالمشرف",
        font=self.fonts["title"],
        bg=self.colors["primary"],
        fg="white",
        padx=10, pady=10
    ).pack(fill=tk.X)

    open_button = tk.Button(
        user_management_frame,
        text="فتح نافذة إدارة المستخدمين",
        font=self.fonts["text_bold"],
        bg=self.colors["secondary"],
        fg="white",
        padx=20, pady=10, bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=self.protected_open_user_management
    )
    open_button.pack(pady=50)

    # إضافة إطار للنسخ الاحتياطي
    backup_frame = tk.Frame(user_management_frame, bg=self.colors["light"], pady=20)
    backup_frame.pack(pady=20)

    tk.Label(
        backup_frame,
        text="إدارة النسخ الاحتياطية لقاعدة البيانات",
        font=self.fonts["text_bold"],
        bg=self.colors["light"],
        fg=self.colors["dark"]
    ).pack(pady=(0, 10))

    # إضافة أزرار النسخ الاحتياطي والاسترداد
    backup_btn = tk.Button(
        backup_frame,
        text="إنشاء نسخة احتياطية",
        font=self.fonts["text_bold"],
        bg=self.colors["primary"],
        fg="white",
        padx=15, pady=5,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=self.backup_database
    )
    backup_btn.pack(side=tk.LEFT, padx=5)

    restore_btn = tk.Button(
        backup_frame,
        text="استرداد نسخة احتياطية",
        font=self.fonts["text_bold"],
        bg=self.colors["warning"],
        fg="white",
        padx=15, pady=5,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=self.restore_database
    )
    restore_btn.pack(side=tk.LEFT, padx=5)

    optimize_db_btn = tk.Button(
        backup_frame,
        text="تحسين أداء قاعدة البيانات",
        font=self.fonts["text_bold"],
        bg=self.colors["secondary"],
        fg="white",
        padx=15, pady=5,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=self.optimize_database
    )
    optimize_db_btn.pack(side=tk.LEFT, padx=5)

    tk.Label(
        user_management_frame,
        text="لن يتم فتح نافذة إدارة المستخدمين إلا بعد إدخال كلمة مرور المشرف.",
        font=self.fonts["text"],
        bg=self.colors["light"],
        fg=self.colors["dark"],
        padx=10, pady=10, wraplength=700
    ).pack(fill=tk.X)

def protected_open_user_management(self):
    if not self.current_user["permissions"]["is_admin"]:
        messagebox.showerror("خطأ", "لا تملك صلاحية!")
        return
    admin_pass = simpledialog.askstring("إدخال كلمة المرور", "أدخل كلمة المرور الخاصة بالمشرف:", show='*')
    if not admin_pass:
        return
    cur = self.conn.cursor()
    cur.execute("SELECT password FROM users WHERE username='admin'")
    row = cur.fetchone()
    if not row:
        messagebox.showerror("خطأ", "لا يوجد حساب مشرف رئيسي!")
        return
    admin_real_hash = row[0]
    hashed_input = hashlib.sha256(admin_pass.encode()).hexdigest()
    if hashed_input == admin_real_hash:
        UserManagement(self.root, self.conn, self.current_user, self.colors, self.fonts)
    else:
        messagebox.showerror("خطأ", "كلمة المرور غير صحيحة!")

def create_tables(self):
    try:
        with self.conn:
            # تعديل جدول المتدربين لإضافة حقول الاستبعاد
            self.conn.execute("""
                CREATE TABLE IF NOT EXISTS trainees (
                    national_id TEXT PRIMARY KEY,
                    name TEXT,
                    rank TEXT,
                    course TEXT,
                    phone TEXT,
                    is_excluded INTEGER DEFAULT 0,
                    exclusion_reason TEXT DEFAULT '',
                    excluded_date TEXT DEFAULT ''
                )
            """)

            self.conn.execute("""
                CREATE TABLE IF NOT EXISTS attendance (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    national_id TEXT,
                    name TEXT,
                    rank TEXT,
                    course TEXT,
                    time TEXT,
                    date TEXT,
                    status TEXT,
                    original_status TEXT,
                    registered_by TEXT,
                    excuse_reason TEXT DEFAULT '',
                    updated_by TEXT,
                    updated_at TEXT,
                    modification_reason TEXT DEFAULT ''
                )
            """)

            # إضافة جدول الفصول
            self.conn.execute("""
                CREATE TABLE IF NOT EXISTS course_sections (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    course_name TEXT NOT NULL,
                    section_name TEXT NOT NULL,
                    created_date TEXT,
                    UNIQUE(course_name, section_name)
                )
            """)

            # إضافة جدول تسجيل المتدربين في الفصول
            self.conn.execute("""
                CREATE TABLE IF NOT EXISTS student_sections (
                    national_id TEXT NOT NULL,
                    course_name TEXT NOT NULL,
                    section_name TEXT NOT NULL,
                    assigned_date TEXT,
                    PRIMARY KEY (national_id, course_name),
                    FOREIGN KEY (national_id) REFERENCES trainees(national_id)
                )
            """)

            # تحديث جدول معلومات الدورات لإضافة تاريخ النهاية وفئة الدورة
            self.conn.execute("""
                            CREATE TABLE IF NOT EXISTS course_info (
                                course_name TEXT PRIMARY KEY,
                                start_day TEXT,
                                start_month TEXT,
                                start_year TEXT,
                                end_day TEXT,
                                end_month TEXT,
                                end_year TEXT,
                                end_date_system TEXT,  -- تاريخ نهاية الدورة في النظام
                                course_category TEXT,  -- فئة الدورة
                                created_date TEXT
                            )
                        """)

            # إضافة جدول المخالفات
            self.conn.execute("""
                                           CREATE TABLE IF NOT EXISTS student_violations (
                                               id INTEGER PRIMARY KEY AUTOINCREMENT,
                                               national_id TEXT,
                                               violation_date TEXT,
                                               violation_type TEXT,
                                               description TEXT,
                                               action_taken TEXT,
                                               action_date TEXT,
                                               recorded_by TEXT,
                                               notes TEXT,
                                               attachment_path TEXT,
                                               FOREIGN KEY (national_id) REFERENCES trainees(national_id)
                                           )
                                       """)

            # إضافة الأعمدة الجديدة إذا لم تكن موجودة
            cursor = self.conn.cursor()
            cursor.execute("PRAGMA table_info(course_info)")
            columns = [column[1] for column in cursor.fetchall()]

            if "end_date_system" not in columns:
                self.conn.execute("ALTER TABLE course_info ADD COLUMN end_date_system TEXT")
            if "course_category" not in columns:
                self.conn.execute("ALTER TABLE course_info ADD COLUMN course_category TEXT")

            # إضافة أعمدة الاستبعاد للمتدربين الحاليين إذا لم تكن موجودة
            cursor = self.conn.cursor()
            cursor.execute("PRAGMA table_info(trainees)")
            columns = [column[1] for column in cursor.fetchall()]

            if "is_excluded" not in columns:
                self.conn.execute("ALTER TABLE trainees ADD COLUMN is_excluded INTEGER DEFAULT 0")
            if "exclusion_reason" not in columns:
                self.conn.execute("ALTER TABLE trainees ADD COLUMN exclusion_reason TEXT DEFAULT ''")
            if "excluded_date" not in columns:
                self.conn.execute("ALTER TABLE trainees ADD COLUMN excluded_date TEXT DEFAULT ''")

            # فحص وإضافة الأعمدة المفقودة في جدول attendance
            # فحص وإضافة الأعمدة المفقودة في جدول attendance
            cursor.execute("PRAGMA table_info(attendance)")
            columns = [column[1] for column in cursor.fetchall()]

            # إضافة الأعمدة المفقودة إذا لم تكن موجودة
            if "original_status" not in columns:
                self.conn.execute("ALTER TABLE attendance ADD COLUMN original_status TEXT")
            if "updated_by" not in columns:
                self.conn.execute("ALTER TABLE attendance ADD COLUMN updated_by TEXT")
            if "updated_at" not in columns:
                self.conn.execute("ALTER TABLE attendance ADD COLUMN updated_at TEXT")

    except Exception as e:
        messagebox.showerror("خطأ", f"حدث خطأ أثناء إنشاء/تعديل الجداول: {str(e)}")

def create_indexes(self):
    """إنشاء فهارس لتحسين أداء قاعدة البيانات"""
    try:
        cursor = self.conn.cursor()

        # فهارس للمتدربين
        cursor.execute("CREATE INDEX IF NOT EXISTS idx_trainees_course ON trainees (course)")
        cursor.execute("CREATE INDEX IF NOT EXISTS idx_trainees_name ON trainees (name)")
        cursor.execute("CREATE INDEX IF NOT EXISTS idx_trainees_excluded ON trainees (is_excluded)")

        # فهارس سجلات الحضور
        cursor.execute("CREATE INDEX IF NOT EXISTS idx_attendance_national_id ON attendance (national_id)")
        cursor.execute("CREATE INDEX IF NOT EXISTS idx_attendance_date ON attendance (date)")
        cursor.execute("CREATE INDEX IF NOT EXISTS idx_attendance_status ON attendance (status)")
        cursor.execute("CREATE INDEX IF NOT EXISTS idx_attendance_course ON attendance (course)")
        cursor.execute(
            "CREATE INDEX IF NOT EXISTS idx_attendance_date_national_id ON attendance (date, national_id)")

        # فهارس الفصول
        cursor.execute("CREATE INDEX IF NOT EXISTS idx_sections_course ON course_sections (course_name)")
        cursor.execute(
            "CREATE INDEX IF NOT EXISTS idx_student_sections_national_id ON student_sections (national_id)")
        cursor.execute("CREATE INDEX IF NOT EXISTS idx_student_sections_course ON student_sections (course_name)")

        self.conn.commit()
        print("تم إنشاء الفهارس بنجاح")
    except Exception as e:
        print(f"خطأ في إنشاء الفهارس: {str(e)}")

def clean_deleted_courses(self):
    """تنظيف بيانات الدورات المحذوفة من جميع الجداول"""
    try:
        cursor = self.conn.cursor()

        # الحصول على قائمة الدورات الموجودة فعلياً في جدول المتدربين
        cursor.execute("""
            SELECT DISTINCT course 
            FROM trainees 
            WHERE course IS NOT NULL AND course != ''
        """)
        active_courses = [row[0] for row in cursor.fetchall()]

        if active_courses:
            # حذف بيانات الدورات غير الموجودة من جدول course_info
            cursor.execute("""
                DELETE FROM course_info 
                WHERE course_name NOT IN ({})
                AND course_name IS NOT NULL
            """.format(','.join('?' * len(active_courses))), active_courses)
        else:
            # إذا لم تكن هناك دورات نشطة، احذف كل شيء من course_info
            cursor.execute("DELETE FROM course_info")

        # حذف بيانات الحضور للمتدربين غير الموجودين
        cursor.execute("""
            DELETE FROM attendance 
            WHERE national_id NOT IN (SELECT national_id FROM trainees)
        """)

        # حذف بيانات الفصول للدورات غير الموجودة
        if active_courses:
            cursor.execute("""
                DELETE FROM course_sections 
                WHERE course_name NOT IN ({})
                AND course_name IS NOT NULL
            """.format(','.join('?' * len(active_courses))), active_courses)
        else:
            cursor.execute("DELETE FROM course_sections")

        self.conn.commit()

        messagebox.showinfo("نجاح", "تم تنظيف قاعدة البيانات من بقايا الدورات المحذوفة")

    except Exception as e:
        messagebox.showerror("خطأ", f"حدث خطأ أثناء تنظيف قاعدة البيانات: {str(e)}")

def create_header(self):
    header_frame = tk.Frame(self.root, bg=self.colors["primary"], height=70)
    header_frame.pack(fill=tk.X)

    # منع الإطار من الانكماش
    header_frame.pack_propagate(False)

    # استخدام تخطيط أكثر مرونة للعناوين
    logo_container = tk.Frame(header_frame, bg=self.colors["primary"])
    logo_container.pack(fill=tk.BOTH, expand=True, padx=20, pady=15)

    logo_label = tk.Label(
        logo_container,
        text="قسم شؤون المدربين - نظام إدارة الدورات التخصصية",
        font=self.fonts["large_title"],
        bg=self.colors["primary"],
        fg="white"
    )
    logo_label.pack(side=tk.RIGHT)

    # إطار منفصل لمعلومات المستخدم
    user_frame = tk.Frame(logo_container, bg=self.colors["primary"])
    user_frame.pack(side=tk.LEFT)

    user_type = "مشرف" if self.current_user["permissions"]["is_admin"] else "مستخدم"
    user_label = tk.Label(
        user_frame,
        text=f"المستخدم: {self.current_user['full_name']} ({user_type})",
        font=self.fonts["text_bold"],
        bg=self.colors["primary"],
        fg="white"
    )
    user_label.pack(side=tk.LEFT)

    logout_btn = tk.Button(
        user_frame,
        text="تسجيل الخروج",
        font=self.fonts["text"],
        bg="#ff5252",
        fg="white",
        padx=10,
        pady=2,
        bd=0,
        relief=tk.FLAT,
        cursor="hand2",
        command=self.logout
    )
    logout_btn.pack(side=tk.LEFT, padx=15)

    # تعديل حجم العنوان للشاشات الصغيرة
    def adjust_header_for_screen_size(event=None):
        window_width = self.root.winfo_width()
        if window_width < 800:
            logo_label.config(text=" نظام إدارة الدورات التخصصية")
        else:
            logo_label.config(text="قسم شؤون المدربين -  نظام إدارة الدورات التخصصية")

    # ربط وظيفة تغيير الحجم بحدث تغيير حجم النافذة
    self.root.bind('<Configure>', adjust_header_for_screen_size)

def logout(self):
    if messagebox.askyesnocancel("تسجيل الخروج", "هل تريد تسجيل الخروج؟"):
        # إلغاء جدولة فحص النشاط
        if self.activity_check_id:
            self.root.after_cancel(self.activity_check_id)

        self.root.destroy()
        root = tk.Tk()
        LoginSystem(root)
        root.mainloop()

def on_closing(self):
    """التعامل مع إغلاق النافذة الرئيسية"""
    # إلغاء جدولة فحص النشاط
    if self.activity_check_id:
        self.root.after_cancel(self.activity_check_id)

    self.root.destroy()

def reset_activity_timer(self, event=None):
    """تحديث وقت آخر نشاط للمستخدم"""
    self.last_activity_time = time.time()

def check_inactivity(self):
    """فحص مدة عدم النشاط وتسجيل الخروج إذا تجاوزت الحد المسموح"""
    current_time = time.time()
    elapsed_time = current_time - self.last_activity_time

    # إذا تجاوز الوقت المنقضي الحد المسموح
    if elapsed_time >= self.inactivity_timeout:
        # عرض رسالة وتسجيل الخروج
        messagebox.showinfo("تسجيل الخروج التلقائي", "تم تسجيل خروجك تلقائياً بسبب عدم النشاط")
        self.force_logout()
        return

    # جدولة الفحص التالي كل ثانية
    self.activity_check_id = self.root.after(1000, self.check_inactivity)

def force_logout(self):
    """تسجيل الخروج المباشر بدون تأكيد"""
    # إلغاء جدولة فحص النشاط
    if self.activity_check_id:
        self.root.after_cancel(self.activity_check_id)

    # تدمير النافذة الحالية وإعادة تشغيل شاشة تسجيل الدخول
    self.root.destroy()
    root = tk.Tk()
    LoginSystem(root)
    root.mainloop()

def filter_attendance(self, event=None):
    selected_status = self.status_filter_var.get()
    if selected_status == "الكل":
        self.export_button.config(text="تصدير الكل")
        # إخفاء أدوات التحديد عند العرض الكلي
        if hasattr(self, 'selection_frame'):
            self.selection_frame.pack_forget()
    else:
        self.export_button.config(text=f"تصدير {selected_status}")

        # إظهار أدوات التحديد فقط عند اختيار "لم يباشر"
        if selected_status == "لم يباشر" and hasattr(self, 'selection_frame'):
            self.selection_frame.pack(fill=tk.X, padx=10, pady=5, before=self.attendance_tree)
        else:
            if hasattr(self, 'selection_frame'):
                self.selection_frame.pack_forget()

    # مسح قائمة المتدربين المحددين
    self.selected_students = {}

    self.update_attendance_display()

def on_tree_click(self, event):
    """التعامل مع النقر على الشجرة وخاصة عمود الـ checkbox"""
    # الحصول على العنصر الذي تم النقر عليه والعمود
    region = self.attendance_tree.identify_region(event.x, event.y)
    if region == "cell":
        column = self.attendance_tree.identify_column(event.x)
        item = self.attendance_tree.identify_row(event.y)

        # فقط إذا كان النقر على عمود الـ checkbox (العمود الأول)
        if column == "#1" and item:  # عمود الـ checkbox
            # تبديل حالة التحديد
            if item in self.selected_students:
                self.selected_students.pop(item)
                self.attendance_tree.item(item, values=self.update_checkbox_value(item, False))
                self.attendance_tree.item(item, tags=self.get_item_tags(item, False))
            else:
                self.selected_students[item] = True
                self.attendance_tree.item(item, values=self.update_checkbox_value(item, True))
                self.attendance_tree.item(item, tags=self.get_item_tags(item, True))

            # منع معالجة الحدث الافتراضية فقط للنقر على الـ checkbox
            return "break"

        # بالنسبة للنقر على الأعمدة الأخرى، نسمح بالسلوك الافتراضي للشجرة (مثل النقر المزدوج)

def select_all_students(self):
    """تحديد جميع المتدربين الظاهرين في القائمة"""
    for item in self.attendance_tree.get_children():
        self.selected_students[item] = True
        self.attendance_tree.item(item, values=self.update_checkbox_value(item, True))
        self.attendance_tree.item(item, tags=self.get_item_tags(item, True))

def clear_all_selection(self):
    """إلغاء تحديد جميع المتدربين"""
    for item in self.attendance_tree.get_children():
        if item in self.selected_students:
            self.selected_students.pop(item)
        self.attendance_tree.item(item, values=self.update_checkbox_value(item, False))
        self.attendance_tree.item(item, tags=self.get_item_tags(item, False))

def update_checkbox_value(self, item, checked):
    """تحديث قيمة الـ checkbox في صف معين"""
    values = list(self.attendance_tree.item(item, "values"))
    values[0] = "✓" if checked else ""
    return values

def get_item_tags(self, item, is_checked):
    """الحصول على الوسوم للصف بناءً على حالة التحديد والنوع"""
    current_tags = list(self.attendance_tree.item(item, "tags"))
    if "checked" in current_tags and not is_checked:
        current_tags.remove("checked")
    elif "checked" not in current_tags and is_checked:
        current_tags.append("checked")
    return current_tags

def get_all_absences_count(self, national_id):
    cursor = self.conn.cursor()
    cursor.execute("""
        SELECT COUNT(*) 
        FROM attendance
        WHERE national_id=? AND status='غائب'
    """, (national_id,))
    return cursor.fetchone()[0]

def get_all_late_count(self, national_id):
    cursor = self.conn.cursor()
    cursor.execute("""
        SELECT COUNT(*) 
        FROM attendance
        WHERE national_id=? AND status='متأخر'
    """, (national_id,))
    return cursor.fetchone()[0]

def get_all_excused_count(self, national_id):
    cursor = self.conn.cursor()
    cursor.execute("""
        SELECT COUNT(*) 
        FROM attendance
        WHERE national_id=? AND status='غائب بعذر'
    """, (national_id,))
    return cursor.fetchone()[0]

def setup_attendance_tab(self):
    self.setup_stats_panel()
    self.setup_individual_attendance()
    self.setup_bulk_attendance()

def setup_stats_panel(self):
    stats_frame = tk.LabelFrame(
        self.attendance_tab,
        text="إحصائيات اليوم",
        font=self.fonts["subtitle"],
        bg=self.colors["light"],
        fg=self.colors["dark"],
        padx=10, pady=10
    )
    stats_frame.pack(fill=tk.X, padx=10, pady=5)

    # إنشاء إطارين للصفين
    top_counter_frame = tk.Frame(stats_frame, bg=self.colors["light"])
    top_counter_frame.pack(fill=tk.X, padx=5, pady=5)

    bottom_counter_frame = tk.Frame(stats_frame, bg=self.colors["light"])
    bottom_counter_frame.pack(fill=tk.X, padx=5, pady=5)

    # مسح قائمة البطاقات
    self.stats_cards = []

    # الصف الأول من الإحصائيات
    card1 = self.create_stat_card(top_counter_frame, "إجمالي المتدربين", self.total_students_var,
                                  self.colors["primary"])
    card1.pack(side=tk.RIGHT, padx=5, fill=tk.X, expand=True)
    self.stats_cards.append(card1)

    card2 = self.create_stat_card(top_counter_frame, "الحاضرون", self.present_students_var, self.colors["success"])
    card2.pack(side=tk.RIGHT, padx=5, fill=tk.X, expand=True)
    self.stats_cards.append(card2)

    card3 = self.create_stat_card(top_counter_frame, "المتأخرون", self.late_students_var, self.colors["late"])
    card3.pack(side=tk.RIGHT, padx=5, fill=tk.X, expand=True)
    self.stats_cards.append(card3)

    card4 = self.create_stat_card(top_counter_frame, "الغائبون", self.absent_students_var, self.colors["danger"])
    card4.pack(side=tk.RIGHT, padx=5, fill=tk.X, expand=True)
    self.stats_cards.append(card4)

    card5 = self.create_stat_card(top_counter_frame, "غائب بعذر", self.excused_students_var, self.colors["excused"])
    card5.pack(side=tk.RIGHT, padx=5, fill=tk.X, expand=True)
    self.stats_cards.append(card5)

    card6 = self.create_stat_card(top_counter_frame, "لم يباشر", self.not_started_students_var,
                                  self.colors["not_started"])
    card6.pack(side=tk.RIGHT, padx=5, fill=tk.X, expand=True)
    self.stats_cards.append(card6)

    # الصف الثاني من الإحصائيات
    card7 = self.create_stat_card(bottom_counter_frame, "تطبيق ميداني", self.field_application_var,
                                  self.colors["field_application"])
    card7.pack(side=tk.RIGHT, padx=5, fill=tk.X, expand=True)
    self.stats_cards.append(card7)

    card8 = self.create_stat_card(bottom_counter_frame, "يوم طالب", self.student_day_var,
                                  self.colors["student_day"])
    card8.pack(side=tk.RIGHT, padx=5, fill=tk.X, expand=True)
    self.stats_cards.append(card8)

    card9 = self.create_stat_card(bottom_counter_frame, "مسائية / عن بعد", self.evening_remote_var,
                                  self.colors["evening_remote"])
    card9.pack(side=tk.RIGHT, padx=5, fill=tk.X, expand=True)
    self.stats_cards.append(card9)

    card10 = self.create_stat_card(bottom_counter_frame, "حالة وفاة", self.death_case_var,
                                   self.colors["death_case"])
    card10.pack(side=tk.RIGHT, padx=5, fill=tk.X, expand=True)
    self.stats_cards.append(card10)

    card11 = self.create_stat_card(bottom_counter_frame, "منوم", self.hospital_var, self.colors["hospital"])
    card11.pack(side=tk.RIGHT, padx=5, fill=tk.X, expand=True)
    self.stats_cards.append(card11)

    card12 = self.create_stat_card(bottom_counter_frame, "نسبة الحضور", self.attendance_rate_var,
                                   self.colors["warning"])
    card12.pack(side=tk.RIGHT, padx=5, fill=tk.X, expand=True)
    self.stats_cards.append(card12)

def create_stat_card(self, parent, title, variable, color):
    card = tk.Frame(parent, bg=self.colors["light"], bd=1, relief=tk.RIDGE)

    # استخدام نسب مرنة لحجم العنصر
    title_label = tk.Label(card, text=title, font=self.fonts["text_bold"], bg=color, fg="white", padx=5, pady=5)
    title_label.pack(fill=tk.X)

    value_label = tk.Label(card, textvariable=variable, font=self.fonts["title"], bg=self.colors["light"], pady=10)
    value_label.pack(fill=tk.X)

    return card

def setup_individual_attendance(self):
    """إعادة تصميم إطار تسجيل الحضور بتنظيم أفضل وأكثر راحة للعين"""
    attendance_frame = tk.LabelFrame(
        self.attendance_tab,
        text="تسجيل الحضور",
        font=self.fonts["subtitle"],
        bg=self.colors["light"],
        fg=self.colors["dark"],
        padx=15,
        pady=15
    )
    attendance_frame.pack(fill=tk.BOTH, padx=10, pady=5)

    # إنشاء إطار للبحث مع تصميم أفضل
    search_section = tk.Frame(attendance_frame, bg=self.colors["light"])
    search_section.pack(fill=tk.X, pady=(0, 10))

    # تقسيم منطقة البحث إلى قسمين - يمين ويسار
    search_right = tk.Frame(search_section, bg=self.colors["light"])
    search_right.pack(side=tk.RIGHT, fill=tk.Y)

    search_left = tk.Frame(search_section, bg=self.colors["light"])
    search_left.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)

    # منطقة التاريخ على اليمين
    date_frame = tk.Frame(search_right, bg=self.colors["light"])
    date_frame.pack(side=tk.RIGHT, padx=10, fill=tk.Y)

    date_label = tk.Label(
        date_frame,
        text="التاريخ:",
        font=self.fonts["text_bold"],
        bg=self.colors["light"]
    )
    date_label.pack(side=tk.RIGHT, padx=5)

    self.date_entry = DateEntry(
        date_frame,
        width=12,
        background=self.colors["primary"],
        foreground='white',
        borderwidth=2,
        date_pattern='yyyy-mm-dd',
        font=self.fonts["text"],
        firstweekday="sunday",
        disableddays=(5, 6)
    )
    self.date_entry.pack(side=tk.RIGHT, padx=5)
    self.date_entry.set_date(self.today)
    self.date_entry.bind("<<DateEntrySelected>>", lambda e: self.update_statistics())

    # منطقة البحث على اليسار مع تصميم محسن
    search_box_frame = tk.Frame(search_left, bg=self.colors["light"])
    search_box_frame.pack(fill=tk.X, padx=10)

    search_icon_label = tk.Label(
        search_box_frame,
        text="🔍",
        font=(self.fonts["text"][0], 14),
        bg=self.colors["light"],
        fg=self.colors["primary"]
    )
    search_icon_label.pack(side=tk.RIGHT, padx=(0, 5))

    search_label = tk.Label(
        search_box_frame,
        text="بحث بالاسم أو الهوية:",
        font=self.fonts["text_bold"],
        bg=self.colors["light"]
    )
    search_label.pack(side=tk.RIGHT, padx=5)

    # مربع بحث بحجم أصغر ومظهر أفضل
    self.name_search_entry = tk.Entry(
        search_box_frame,
        font=self.fonts["text"],
        width=20,  # تقليل العرض
        bd=2,
        relief=tk.GROOVE
    )
    self.name_search_entry.pack(side=tk.RIGHT, padx=5, fill=tk.X, expand=False)
    self.name_search_entry.bind("<KeyRelease>", self.dynamic_name_search)

    # تحسين مظهر قائمة النتائج
    results_frame = tk.Frame(attendance_frame, bg=self.colors["light"], pady=5)
    results_frame.pack(fill=tk.X, padx=10)

    results_label = tk.Label(
        results_frame,
        text="نتائج البحث:",
        font=self.fonts["text_bold"],
        bg=self.colors["light"],
        fg=self.colors["primary"]
    )
    results_label.pack(side=tk.RIGHT, anchor=tk.N, padx=(0, 5))

    # إطار لقائمة النتائج مع شريط تمرير
    listbox_frame = tk.Frame(results_frame, bg=self.colors["light"])
    listbox_frame.pack(fill=tk.X, expand=True, side=tk.LEFT)

    # إضافة شريط تمرير
    listbox_scrollbar = tk.Scrollbar(listbox_frame)
    listbox_scrollbar.pack(side=tk.RIGHT, fill=tk.Y)

    self.name_listbox = tk.Listbox(
        listbox_frame,
        font=self.fonts["text"],
        height=4,
        selectbackground=self.colors["primary"],
        bd=2,
        relief=tk.GROOVE,
        yscrollcommand=listbox_scrollbar.set
    )
    self.name_listbox.pack(fill=tk.X, expand=True)
    self.name_listbox.bind("<<ListboxSelect>>", self.on_name_select)

    # ربط شريط التمرير بالقائمة
    listbox_scrollbar.config(command=self.name_listbox.yview)

    # حقل خفي لتخزين الهوية
    self.id_entry = tk.Entry(self.root)

    # تحسين تصميم أزرار تسجيل الحضور - تعديل الأزرار المعروضة
    buttons_frame = tk.Frame(attendance_frame, bg=self.colors["light"])
    buttons_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=5)

    # توزيع الأعمدة بشكل متساوٍ
    for i in range(5):
        buttons_frame.columnconfigure(i, weight=1)

    # إنشاء إطارات الصفوف
    row1_frame = tk.Frame(buttons_frame, bg=self.colors["light"])
    row1_frame.pack(fill=tk.X, pady=(0, 5))

    row2_frame = tk.Frame(buttons_frame, bg=self.colors["light"])
    row2_frame.pack(fill=tk.X, pady=(5, 0))

    # الصف الأول من الأزرار - أزرار الحضور الأساسية
    buttons_row1 = [
        ("حاضر", self.colors["success"], lambda: self.insert_attendance_record("حاضر")),
        ("متأخر", self.colors["late"], lambda: self.insert_attendance_record("متأخر")),
        ("غائب", self.colors["danger"], lambda: self.insert_attendance_record("غائب")),
        ("غياب بعذر", self.colors["excused"], self.register_excused_absence),
        ("لم يباشر", self.colors["not_started"], lambda: self.insert_attendance_record("لم يباشر"))
    ]

    # الصف الثاني من الأزرار - الحالات الخاصة فقط
    buttons_row2 = [
        ("حالة وفاة", self.colors["death_case"], lambda: self.register_special_case("حالة وفاة")),
        ("منوم", self.colors["hospital"], lambda: self.register_special_case("منوم"))
    ]

    # إنشاء أزرار الصف الأول
    for i, (text, color, command) in enumerate(buttons_row1):
        btn = tk.Button(
            row1_frame,
            text=text,
            font=self.fonts["text_bold"],
            bg=color,
            fg="white",
            padx=5,
            pady=8,
            bd=0,
            relief=tk.FLAT,
            cursor="hand2",
            command=command
        )
        btn.pack(side=tk.RIGHT, padx=3, fill=tk.X, expand=True)

    # إنشاء أزرار الصف الثاني
    for i, (text, color, command) in enumerate(buttons_row2):
        btn = tk.Button(
            row2_frame,
            text=text,
            font=self.fonts["text_bold"],
            bg=color,
            fg="white",
            padx=5,
            pady=8,
            bd=0,
            relief=tk.FLAT,
            cursor="hand2",
            command=command
        )
        btn.pack(side=tk.RIGHT, padx=3, fill=tk.X, expand=True)

    # إضافة مؤشر آخر تسجيل بتصميم محسن
    status_frame = tk.Frame(attendance_frame, bg=self.colors["light"], pady=5)
    status_frame.pack(fill=tk.X, padx=10, pady=(10, 0))

    self.last_registered_label = tk.Label(
        status_frame,
        text="",
        font=self.fonts["text_bold"],
        fg=self.colors["primary"],
        bg=self.colors["light"],
        anchor=tk.W  # محاذاة النص إلى اليمين
    )
    self.last_registered_label.pack(fill=tk.X)

def register_attendance(self, event=None):
    self.insert_attendance_record("حاضر")

def register_excused_absence(self):
    if not self.current_user["permissions"]["can_edit_attendance"]:
        messagebox.showwarning("تنبيه", "ليس لديك صلاحية تسجيل الغياب بعذر")
        return

    reason = simpledialog.askstring("غياب بعذر", "اكتب سبب الغياب إن وُجد:")
    if reason is None:
        return
    self.insert_attendance_record("غائب بعذر", excuse_reason=reason)

def register_all_unmarked_as_present(self):
    if not self.current_user["permissions"]["can_edit_attendance"]:
        messagebox.showwarning("تنبيه", "ليس لديك صلاحية تسجيل الحضور والغياب")
        return

    current_date = self.date_entry.get_date().strftime("%Y-%m-%d")
    current_time = datetime.datetime.now().strftime("%H:%M:%S")

    # إنشاء نافذة تقدم العملية
    progress_window = tk.Toplevel(self.root)
    progress_window.title("تسجيل حضور جماعي")
    progress_window.geometry("450x150")
    progress_window.configure(bg=self.colors["light"])
    progress_window.transient(self.root)
    progress_window.grab_set()

    x = (progress_window.winfo_screenwidth() - 450) // 2
    y = (progress_window.winfo_screenheight() - 150) // 2
    progress_window.geometry(f"450x150+{x}+{y}")

    progress_var = tk.DoubleVar()
    tk.Label(
        progress_window,
        text="جاري تسجيل الحضور للمتدربين غير المسجلين...",
        font=self.fonts["text_bold"],
        bg=self.colors["light"],
        pady=10
    ).pack()

    progress_bar = ttk.Progressbar(
        progress_window,
        variable=progress_var,
        maximum=100,
        length=400
    )
    progress_bar.pack(pady=10)

    status_label = tk.Label(
        progress_window,
        text="جاري الإعداد...",
        font=self.fonts["text"],
        bg=self.colors["light"]
    )
    status_label.pack(pady=5)

    progress_window.update()

    try:
        cursor = self.conn.cursor()

        # الحصول على المتدربين غير المسجلين بطريقة أكثر كفاءة مع الاستعلامات المجمعة
        status_label.config(text="جاري تحديد المتدربين غير المسجلين...")
        progress_window.update()

        # استخدام جملة INSERT طويلة بدلاً من العديد من العمليات المنفصلة
        cursor.execute("""
            INSERT INTO attendance (
                national_id, name, rank, course, time, date, status, 
                original_status, registered_by, excuse_reason, updated_by, updated_at
            )
            SELECT 
                t.national_id, t.name, t.rank, t.course, ?, ?, 'حاضر', 'حاضر', ?, '', '', ''
            FROM trainees t
            WHERE t.is_excluded=0
            AND NOT EXISTS (
                SELECT 1 FROM attendance a 
                WHERE a.national_id = t.national_id AND a.date = ?
            )
        """, (current_time, current_date, self.current_user["full_name"], current_date))

        # حفظ التغييرات والحصول على عدد الصفوف المتأثرة
        self.conn.commit()
        rows_affected = cursor.rowcount

        progress_var.set(100)
        status_label.config(text="تم الانتهاء بنجاح!")
        progress_window.update()

        # إغلاق نافذة التقدم بعد ثانيتين
        progress_window.after(2000, progress_window.destroy)

        if rows_affected > 0:
            messagebox.showinfo("نجاح", f"تم تسجيل حضور {rows_affected} متدرب غير مسجل بنجاح.")
        else:
            messagebox.showinfo("ملاحظة", "لا يوجد متدربين غير مسجلين اليوم.")

        self.update_statistics()
        self.update_attendance_display()

    except Exception as e:
        try:
            progress_window.destroy()
        except:
            pass
        messagebox.showerror("خطأ", f"حدث خطأ: {str(e)}")

def register_bulk_lateness(self):
    if not self.current_user["permissions"]["can_edit_attendance"]:
        messagebox.showwarning("تنبيه", "ليس لديك صلاحية تسجيل الحضور والغياب")
        return

    if not messagebox.askyesnocancel("تأكيد",
                                     "هل تريد تسجيل تأخير جماعي لجميع المتدربين الذين لم يتم تسجيلهم اليوم؟"):
        return

    current_date = self.date_entry.get_date().strftime("%Y-%m-%d")
    current_time = datetime.datetime.now().strftime("%H:%M:%S")

    cursor = self.conn.cursor()
    # استثناء المتدربين المستبعدين
    cursor.execute("""
        SELECT national_id, name, rank, course 
        FROM trainees 
        WHERE is_excluded=0
    """)
    all_students = cursor.fetchall()
    cursor.execute("SELECT DISTINCT national_id FROM attendance WHERE date=?", (current_date,))
    already_recorded = set(row[0] for row in cursor.fetchall())

    new_late_rows = []
    for student in all_students:
        if student[0] not in already_recorded:
            new_late_rows.append((
                student[0], student[1], student[2], student[3],
                current_time, current_date,
                "متأخر", "متأخر",
                self.current_user["full_name"], "",
                "", ""
            ))

    if not new_late_rows:
        messagebox.showinfo("ملاحظة", "لا يوجد متدربين غير مسجلين.")
        return

    try:
        with self.conn:
            self.conn.executemany("""
                INSERT INTO attendance (
                    national_id, name, rank, course,
                    time, date, status, original_status,
                    registered_by, excuse_reason,
                    updated_by, updated_at
                )
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
            """, new_late_rows)
        messagebox.showinfo("نجاح", f"تم تسجيل تأخير {len(new_late_rows)} متدرب بنجاح")
        self.update_statistics()
        self.update_attendance_display()
    except Exception as e:
        messagebox.showerror("خطأ", str(e))

def register_special_case(self, status):
    """دالة تسجيل الحالات الخاصة (وفاة، منوم) مع طلب تفاصيل إضافية"""
    if not self.current_user["permissions"]["can_edit_attendance"]:
        messagebox.showwarning("تنبيه", "ليس لديك صلاحية تسجيل الحضور والغياب")
        return

    details = simpledialog.askstring(f"تفاصيل {status}", f"أدخل تفاصيل {status}:")
    if details is None:  # إذا ضغط المستخدم على زر الإلغاء
        return

    self.insert_attendance_record(status, excuse_reason=details)

def bulk_register(self):
    if not self.current_user["permissions"]["can_edit_attendance"]:
        messagebox.showwarning("تنبيه", "ليس لديك صلاحية تسجيل الحضور والغياب")
        return

    text_data = self.bulk_text.get("1.0", tk.END).strip()
    if not text_data:
        return

    current_date = self.date_entry.get_date().strftime("%Y-%m-%d")
    lines = text_data.split("\n")

    cursor = self.conn.cursor()
    new_rows = []

    for line in lines:
        nid = line.strip()
        if not nid:
            continue

        # التحقق من استبعاد المتدرب
        cursor.execute("""
            SELECT national_id, name, rank, course, is_excluded 
            FROM trainees 
            WHERE national_id=?
        """, (nid,))

        trainee = cursor.fetchone()
        if not trainee:
            continue

        # تخطي المتدربين المستبعدين
        if trainee[4] == 1:
            continue

        cursor.execute("SELECT status FROM attendance WHERE national_id=? AND date=?", (trainee[0], current_date))
        existing_record = cursor.fetchone()
        if existing_record:
            continue

        current_time = datetime.datetime.now().strftime("%H:%M:%S")
        new_rows.append((
            trainee[0], trainee[1], trainee[2], trainee[3],
            current_time, current_date,
            "حاضر", "حاضر",
            self.current_user["full_name"], "",
            "", ""
        ))

    if not new_rows:
        messagebox.showinfo("ملاحظة", "لا يوجد متدربين جدد للتسجيل.")
        return

    try:
        with self.conn:
            self.conn.executemany("""
                INSERT INTO attendance (
                    national_id, name, rank, course,
                    time, date, status, original_status,
                    registered_by, excuse_reason,
                    updated_by, updated_at
                )
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
            """, new_rows)
        messagebox.showinfo("نجاح", f"تم تسجيل حضور {len(new_rows)} متدرب بنجاح")
        self.bulk_text.delete("1.0", tk.END)
        self.update_statistics()
        self.update_attendance_display()
    except Exception as e:
        messagebox.showerror("خطأ", str(e))

def register_whole_course(self):
    """تسجيل حضور لدورات متعددة بحالة محددة"""
    if not self.current_user["permissions"]["can_edit_attendance"]:
        messagebox.showwarning("تنبيه", "ليس لديك صلاحية تسجيل الحضور والغياب")
        return

    # إنشاء نافذة اختيار الدورات والحالة
    select_window = tk.Toplevel(self.root)
    select_window.title("تسجيل حضور دورات")
    select_window.geometry("800x600")
    select_window.configure(bg=self.colors["light"])
    select_window.transient(self.root)
    select_window.grab_set()

    x = (select_window.winfo_screenwidth() - 800) // 2
    y = (select_window.winfo_screenheight() - 600) // 2
    select_window.geometry(f"800x600+{x}+{y}")

    # عنوان النافذة
    title_label = tk.Label(
        select_window,
        text="تسجيل حضور دورات",
        font=self.fonts["title"],
        bg=self.colors["primary"],
        fg="white",
        padx=10, pady=10
    )
    title_label.pack(fill=tk.X)

    # إطار البحث
    search_frame = tk.Frame(select_window, bg=self.colors["light"], padx=10, pady=10)
    search_frame.pack(fill=tk.X)

    tk.Label(
        search_frame,
        text="بحث عن دورة:",
        font=self.fonts["text_bold"],
        bg=self.colors["light"]
    ).pack(side=tk.RIGHT, padx=5)

    search_var = tk.StringVar()
    search_entry = tk.Entry(
        search_frame,
        textvariable=search_var,
        font=self.fonts["text"],
        width=30
    )
    search_entry.pack(side=tk.RIGHT, padx=5)

    # الحصول على قائمة الدورات الحالية
    cursor = self.conn.cursor()
    cursor.execute("SELECT DISTINCT course FROM trainees WHERE is_excluded=0 ORDER BY course")
    courses = [row[0] for row in cursor.fetchall() if row[0]]

    # قائمة الدورات مع إمكانية التمرير
    courses_frame = tk.Frame(select_window, bg=self.colors["light"], padx=10, pady=10)
    courses_frame.pack(fill=tk.BOTH, expand=True)

    courses_label = tk.Label(
        courses_frame,
        text="حدد الدورات المطلوبة:",
        font=self.fonts["text_bold"],
        bg=self.colors["light"],
        anchor=tk.W
    )
    courses_label.pack(fill=tk.X, pady=(0, 5))

    # إطار لعرض قائمة الدورات مع شريط تمرير
    list_frame = tk.Frame(courses_frame, bg=self.colors["light"])
    list_frame.pack(fill=tk.BOTH, expand=True)

    scrollbar = tk.Scrollbar(list_frame)
    scrollbar.pack(side=tk.LEFT, fill=tk.Y)

    courses_listbox = tk.Listbox(
        list_frame,
        font=self.fonts["text"],
        selectmode=tk.MULTIPLE,  # السماح بتحديد متعدد
        yscrollcommand=scrollbar.set
    )
    courses_listbox.pack(fill=tk.BOTH, expand=True, side=tk.LEFT)
    scrollbar.config(command=courses_listbox.yview)

    # إضافة الدورات إلى القائمة
    for course in courses:
        courses_listbox.insert(tk.END, course)

    # متغير لتخزين الدورات المحددة
    selected_courses = []

    # دالة تحديث الدورات المعروضة حسب البحث
    def update_courses(*args):
        search_text = search_var.get().strip()
        courses_listbox.delete(0, tk.END)
        for course in courses:
            if search_text.lower() in course.lower():
                courses_listbox.insert(tk.END, course)

    # ربط دالة البحث بتغيير النص
    search_var.trace_add("write", update_courses)

    # إطار أزرار الحالات
    status_frame = tk.LabelFrame(
        select_window,
        text="اختر الحالة المطلوب تسجيلها",
        font=self.fonts["text_bold"],
        bg=self.colors["light"],
        fg=self.colors["dark"],
        padx=10, pady=10
    )
    status_frame.pack(fill=tk.X, padx=10, pady=10)

    # إطار الصف الأول من أزرار الحالات
    status_row1 = tk.Frame(status_frame, bg=self.colors["light"])
    status_row1.pack(fill=tk.X, pady=5)

    # إطار الصف الثاني من أزرار الحالات
    status_row2 = tk.Frame(status_frame, bg=self.colors["light"])
    status_row2.pack(fill=tk.X, pady=5)

    # دالة تسجيل الحضور للدورات المحددة
    def register_status(status):
        # الحصول على الدورات المحددة
        selected_indices = courses_listbox.curselection()
        if not selected_indices:
            messagebox.showwarning("تنبيه", "الرجاء تحديد دورة واحدة على الأقل")
            return

        selected_courses = [courses_listbox.get(i) for i in selected_indices]

        # التأكيد قبل التسجيل
        confirm_msg = f"هل تريد تسجيل حالة '{status}' لجميع متدربين الدورات التالية:\n"
        for course in selected_courses:
            confirm_msg += f"- {course}\n"

        if not messagebox.askyesno("تأكيد", confirm_msg):
            return

        # إنشاء نافذة تقدم العملية
        progress_window = tk.Toplevel(select_window)
        progress_window.title("تسجيل الحضور")
        progress_window.geometry("450x150")
        progress_window.configure(bg=self.colors["light"])
        progress_window.transient(select_window)
        progress_window.grab_set()

        # توسيط النافذة
        x = (progress_window.winfo_screenwidth() - 450) // 2
        y = (progress_window.winfo_screenheight() - 150) // 2
        progress_window.geometry(f"450x150+{x}+{y}")

        progress_var = tk.DoubleVar()
        progress_bar = ttk.Progressbar(
            progress_window,
            variable=progress_var,
            maximum=100,
            length=400
        )
        progress_bar.pack(pady=20)

        status_label = tk.Label(
            progress_window,
            text="جاري تحضير البيانات...",
            font=self.fonts["text"],
            bg=self.colors["light"]
        )
        status_label.pack(pady=10)

        progress_window.update()

        try:
            # تهيئة متغيرات الإحصائيات
            total_students = 0
            new_registered = 0
            already_registered = 0

            current_date = self.date_entry.get_date().strftime("%Y-%m-%d")
            current_time = datetime.datetime.now().strftime("%H:%M:%S")

            # معالجة كل دورة
            for course_idx, course_name in enumerate(selected_courses):
                # تحديث شريط التقدم
                progress_var.set((course_idx / len(selected_courses)) * 50)
                status_label.config(text=f"معالجة دورة: {course_name}")
                progress_window.update()

                # الحصول على جميع المتدربين في الدورة المحددة
                cursor.execute("""
                    SELECT national_id, name, rank, course 
                    FROM trainees 
                    WHERE course=? AND is_excluded=0
                """, (course_name,))
                students = cursor.fetchall()

                if not students:
                    continue

                total_students += len(students)

                # التحقق من المتدربين المسجلين مسبقًا
                cursor.execute("""
                    SELECT a.national_id, a.status
                    FROM attendance a
                    JOIN trainees t ON a.national_id = t.national_id
                    WHERE a.date=? AND t.course=? AND t.is_excluded=0
                """, (current_date, course_name))

                already_registered_ids = {row[0]: row[1] for row in cursor.fetchall()}

                # إعداد البيانات للإدخال
                new_records = []
                course_progress_increment = 50 / len(selected_courses)

                for i, student in enumerate(students):
                    student_id, student_name, student_rank, student_course = student

                    # تحديث شريط التقدم للمتدربين
                    progress = 50 + (course_idx / len(selected_courses) * 50) + (
                            i / len(students) * course_progress_increment)
                    if i % 10 == 0:
                        progress_var.set(progress)
                        status_label.config(text=f"تسجيل المتدربين في دورة {course_name} ({i + 1}/{len(students)})")
                        progress_window.update()

                    # تخطي المتدربين المسجلين بالفعل
                    if student_id in already_registered_ids:
                        already_registered += 1
                        continue

                    new_records.append((
                        student_id, student_name, student_rank, student_course,
                        current_time, current_date,
                        status, status,
                        self.current_user["full_name"], "",
                        "", ""
                    ))
                    new_registered += 1

                # تنفيذ الإدخال الجماعي
                if new_records:
                    with self.conn:
                        self.conn.executemany("""
                            INSERT INTO attendance (
                                national_id, name, rank, course,
                                time, date, status, original_status,
                                registered_by, excuse_reason,
                                updated_by, updated_at
                            )
                            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
                        """, new_records)

            # إكمال شريط التقدم
            progress_var.set(100)
            status_label.config(text="تم التسجيل بنجاح!")
            progress_window.update()

            # إغلاق نافذة التقدم بعد ثانيتين
            progress_window.after(2000, progress_window.destroy)

            # عرض ملخص النتائج
            messagebox.showinfo(
                "نجاح",
                f"تم تسجيل حالة '{status}' بنجاح:\n"
                f"- عدد الدورات: {len(selected_courses)}\n"
                f"- إجمالي المتدربين: {total_students}\n"
                f"- المتدربين المسجلين حديثًا: {new_registered}\n"
                f"- المتدربين المسجلين مسبقًا: {already_registered}"
            )

            select_window.destroy()
            self.update_statistics()
            self.update_attendance_display()

        except Exception as e:
            # إغلاق نافذة التقدم في حالة الخطأ
            try:
                progress_window.destroy()
            except:
                pass
            messagebox.showerror("خطأ", f"حدث خطأ أثناء تسجيل الحضور: {str(e)}")

    # أزرار الحالات (الصف الأول)
    status_buttons_row1 = [
        ("حاضر", self.colors["success"]),
        ("متأخر", self.colors["late"]),
        ("غائب", self.colors["danger"]),
        ("غائب بعذر", self.colors["excused"])
    ]

    # أزرار الحالات (الصف الثاني)
    status_buttons_row2 = [
        ("لم يباشر", self.colors["not_started"]),
        ("تطبيق ميداني", self.colors["field_application"]),
        ("يوم طالب", self.colors["student_day"]),
        ("مسائية / عن بعد", self.colors["evening_remote"])
    ]

    # إنشاء أزرار الصف الأول
    for text, color in status_buttons_row1:
        btn = tk.Button(
            status_row1,
            text=text,
            font=self.fonts["text_bold"],
            bg=color,
            fg="white",
            padx=10,
            pady=8,
            bd=0,
            relief=tk.FLAT,
            cursor="hand2",
            command=lambda s=text: register_status(s)
        )
        btn.pack(side=tk.RIGHT, padx=5, fill=tk.X, expand=True)

    # إنشاء أزرار الصف الثاني
    for text, color in status_buttons_row2:
        btn = tk.Button(
            status_row2,
            text=text,
            font=self.fonts["text_bold"],
            bg=color,
            fg="white",
            padx=10,
            pady=8,
            bd=0,
            relief=tk.FLAT,
            cursor="hand2",
            command=lambda s=text: register_status(s)
        )
        btn.pack(side=tk.RIGHT, padx=5, fill=tk.X, expand=True)

    # زر الإغلاق
    close_btn = tk.Button(
        select_window,
        text="إغلاق",
        font=self.fonts["text_bold"],
        bg=self.colors["dark"],
        fg="white",
        padx=15, pady=5,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=select_window.destroy
    )
    close_btn.pack(pady=10)

def dynamic_name_search(self, event):
    try:
        text = self.name_search_entry.get().strip()
        self.name_listbox.delete(0, tk.END)
        if not text:
            return
        cursor = self.conn.cursor()
        # البحث بالاسم أو برقم الهوية معًا
        cursor.execute("""
            SELECT name, national_id 
            FROM trainees 
            WHERE (name LIKE ? OR national_id LIKE ?) AND is_excluded=0
        """, ('%' + text + '%', '%' + text + '%',))

        results = cursor.fetchall()
        for row in results:
            self.name_listbox.insert(tk.END, f"{row[0]} ({row[1]})")
    except (tk.TclError, AttributeError):
        # تجاهل الخطأ إذا لم يعد العنصر موجوداً
        pass

def on_name_select(self, event):
    selection = self.name_listbox.curselection()
    if not selection:
        return
    selected_text = self.name_listbox.get(selection[0])
    # استخراج رقم الهوية من النص المحدد (الاسم (الهوية))
    try:
        national_id = selected_text.split("(")[1].split(")")[0]
        # تخزين رقم الهوية في الحقل الخفي
        self.id_entry.delete(0, tk.END)
        self.id_entry.insert(0, national_id)
    except:
        pass  # في حال حدوث خطأ في تنسيق النص

def setup_attendance_log_tab(self):
    """تعديل دالة إعداد تبويب سجل الحضور لجعل الأزرار مرنة"""
    table_frame = tk.LabelFrame(self.attendance_log_tab, text="سجل الحضور", font=self.fonts["subtitle"],
                                bg=self.colors["light"], fg=self.colors["dark"], padx=10, pady=10)
    table_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)

    # إنشاء إطارين منفصلين للتحكم (العلوي والسفلي)
    # الإطار العلوي للبحث والتاريخ والتصفية
    top_controls = tk.Frame(table_frame, bg=self.colors["light"])
    top_controls.pack(fill=tk.X, pady=(5, 2))

    # الإطار السفلي للأزرار
    button_controls = tk.Frame(table_frame, bg=self.colors["light"])
    button_controls.pack(fill=tk.X, pady=(2, 5))

    # إطار إضافي لأزرار تحديد الكل واستبعاد المحددين - سيظهر فقط عند تحديد "لم يباشر"
    self.selection_controls = tk.Frame(table_frame, bg=self.colors["light"])

    # ---------- الإطار العلوي ----------
    # إطار اليمين - التاريخ والتصفية
    right_frame = tk.Frame(top_controls, bg=self.colors["light"])
    right_frame.pack(side=tk.RIGHT, fill=tk.Y)

    tk.Label(right_frame, text="التاريخ:", font=self.fonts["text_bold"], bg=self.colors["light"]).pack(
        side=tk.RIGHT, padx=5)
    self.log_date_entry = DateEntry(
        right_frame,
        width=15,
        background=self.colors["primary"],
        foreground='white',
        borderwidth=2,
        date_pattern='yyyy-mm-dd',
        font=self.fonts["text"],
        firstweekday="sunday",
        disableddays=(5, 6)
    )
    self.log_date_entry.pack(side=tk.RIGHT, padx=5)
    self.log_date_entry.set_date(self.today)
    self.log_date_entry.bind("<<DateEntrySelected>>", lambda e: self.update_attendance_display())

    tk.Label(right_frame, text="تصفية حسب الحالة:", font=self.fonts["text"], bg=self.colors["light"]).pack(
        side=tk.RIGHT, padx=5)

    self.status_filter_var = tk.StringVar()
    self.status_filter = ttk.Combobox(
        right_frame,
        textvariable=self.status_filter_var,
        values=["الكل", "حاضر", "متأخر", "غائب", "غائب بعذر", "لم يباشر",
                "تطبيق ميداني", "يوم طالب", "مسائية / عن بعد", "حالة وفاة", "منوم"],
        state="readonly",
        width=15,
        font=self.fonts["text"]
    )
    self.status_filter.current(0)
    self.status_filter.pack(side=tk.RIGHT, padx=5)
    self.status_filter.bind("<<ComboboxSelected>>", self.filter_attendance)

    # إطار اليسار - البحث
    left_frame = tk.Frame(top_controls, bg=self.colors["light"])
    left_frame.pack(side=tk.LEFT, fill=tk.Y)

    tk.Label(left_frame, text="بحث (الاسم/الهوية):", font=self.fonts["text"], bg=self.colors["light"]).pack(
        side=tk.LEFT, padx=5)
    self.log_search_var = tk.StringVar()
    self.log_search_entry = tk.Entry(left_frame, textvariable=self.log_search_var, font=self.fonts["text"],
                                     width=20)
    self.log_search_entry.pack(side=tk.LEFT, padx=5)
    self.log_search_entry.bind("<KeyRelease>", lambda e: self.update_attendance_display())

    # ---------- الإطار السفلي (للأزرار) ----------
    # استخدام Grid لتوزيع الأزرار بشكل مرن
    button_controls.columnconfigure(0, weight=1)  # للمساحة على اليمين
    button_controls.columnconfigure(1, weight=0)  # للزر الأول
    button_controls.columnconfigure(2, weight=0)  # للزر الثاني
    button_controls.columnconfigure(3, weight=0)  # للزر الثالث
    button_controls.columnconfigure(4, weight=0)  # للزر الرابع
    button_controls.columnconfigure(5, weight=0)  # للزر الخامس - زر استبعاد المتدربين
    button_controls.columnconfigure(6, weight=1)  # للمساحة على اليسار

    # إضافة زر إحصائيات الغياب والتأخير
    col_index = 1  # نبدأ من العمود 1

    # إضافة زر أعلى معدلات الغياب والتأخير
    top_absence_button = tk.Button(
        button_controls,
        text="أعلى معدلات الغياب والتأخير",
        font=self.fonts["text_bold"],
        bg="#673AB7",  # لون بنفسجي مميز
        fg="white",
        padx=10,
        pady=3,
        bd=0,
        relief=tk.FLAT,
        cursor="hand2",
        command=self.show_top_absence_statistics
    )
    top_absence_button.grid(row=0, column=col_index, padx=5, pady=5, sticky="ew")
    col_index += 1

    # إضافة زر التصدير إذا كان المستخدم لديه صلاحية
    if self.current_user["permissions"]["can_export_data"]:
        self.export_button = tk.Button(
            button_controls,
            text="تصدير الكل",
            font=self.fonts["text_bold"],
            bg=self.colors["primary"],
            fg="white",
            padx=10,
            pady=3,
            bd=0,
            relief=tk.FLAT,
            cursor="hand2",
            command=self.export_based_on_filter
        )
        self.export_button.grid(row=0, column=col_index, padx=5, pady=5, sticky="ew")
        col_index += 1

    # إضافة زر تصدير تكميل الدورات (متاح لجميع المستخدمين)
    completion_export_button = tk.Button(
        button_controls,
        text="تصدير تكميل الدورات",
        font=self.fonts["text_bold"],
        bg=self.colors["secondary"],
        fg="white",
        padx=10,
        pady=3,
        bd=0,
        relief=tk.FLAT,
        cursor="hand2",
        command=self.export_course_completion
    )
    completion_export_button.grid(row=0, column=col_index, padx=5, pady=5, sticky="ew")
    col_index += 1

    # إضافة زر التكميل الرسمي
    official_completion_button = tk.Button(
        button_controls,
        text="التكميل الرسمي",
        font=self.fonts["text_bold"],
        bg="#E91E63",  # لون وردي مميز
        fg="white",
        padx=10,
        pady=3,
        bd=0,
        relief=tk.FLAT,
        cursor="hand2",
        command=self.export_official_completion
    )
    official_completion_button.grid(row=0, column=col_index, padx=5, pady=5, sticky="ew")
    col_index += 1

    # إضافة زر إعادة تعيين اليوم إذا كان المستخدم لديه صلاحية
    if self.current_user["permissions"]["can_reset_attendance"]:
        reset_button = tk.Button(
            button_controls,
            text="إعادة تعيين اليوم",
            font=self.fonts["text_bold"],
            bg=self.colors["dark"],
            fg="white",
            padx=5,
            pady=3,
            bd=0,
            relief=tk.FLAT,
            command=self.reset_attendance_day,
            cursor="hand2"
        )
        reset_button.grid(row=0, column=col_index, padx=5, pady=5, sticky="ew")
        col_index += 1

    # إضافة زر استبعاد المتدربين المحددين (للمشرفين فقط)
    if self.current_user["permissions"]["is_admin"]:
        self.exclude_button = tk.Button(
            button_controls,
            text="استبعاد المحددين",
            font=self.fonts["text_bold"],
            bg=self.colors["excluded"],  # لون الاستبعاد
            fg="white",
            padx=5,
            pady=3,
            bd=0,
            relief=tk.FLAT,
            command=self.exclude_selected_students,
            cursor="hand2"
        )
        self.exclude_button.grid(row=0, column=col_index, padx=5, pady=5, sticky="ew")

    if self.current_user["permissions"]["is_admin"]:
        # زر تنظيف قاعدة البيانات
        clean_db_button = tk.Button(
            button_controls,
            text="تنظيف قاعدة البيانات",
            font=self.fonts["text_bold"],
            bg="#FF5722",  # لون برتقالي مميز
            fg="white",
            padx=10,
            pady=3,
            bd=0,
            relief=tk.FLAT,
            cursor="hand2",
            command=self.clean_deleted_courses
        )
        clean_db_button.grid(row=0, column=col_index + 1, padx=5, pady=5, sticky="ew")

    # إضافة إطار أزرار التحديد (سيظهر فقط عند اختيار "لم يباشر")
    self.selection_frame = tk.Frame(table_frame, bg=self.colors["light"], pady=5)

    # زر تحديد الكل
    self.select_all_button = tk.Button(
        self.selection_frame,
        text="تحديد الكل",
        font=self.fonts["text_bold"],
        bg="#4CAF50",  # لون أخضر
        fg="white",
        padx=10,
        pady=3,
        bd=0,
        relief=tk.FLAT,
        cursor="hand2",
        command=self.select_all_students
    )
    self.select_all_button.pack(side=tk.LEFT, padx=5)

    # زر إلغاء تحديد الكل
    self.clear_selection_button = tk.Button(
        self.selection_frame,
        text="إلغاء تحديد الكل",
        font=self.fonts["text_bold"],
        bg="#FFA000",  # لون برتقالي
        fg="white",
        padx=10,
        pady=3,
        bd=0,
        relief=tk.FLAT,
        cursor="hand2",
        command=self.clear_all_selection
    )
    self.clear_selection_button.pack(side=tk.LEFT, padx=5)

    # متغير لتخزين حالة التحديد للصفوف
    self.selected_students = {}

    self.tree_scroll = tk.Scrollbar(table_frame)
    self.tree_scroll.pack(side=tk.RIGHT, fill=tk.Y)

    # إضافة عمود checkbox
    if self.current_user["permissions"]["can_view_edit_history"]:
        columns = (
            "checkbox",  # عمود جديد للـ checkbox
            "id",
            "name",
            "rank",
            "course",
            "status",
            "absences",
            "late_count",
            "excused",
            "updated_by",
            "updated_at"
        )
    else:
        columns = (
            "checkbox",  # عمود جديد للـ checkbox
            "id", "name", "rank", "course", "status",
            "absences", "late_count", "excused"
        )

    # إنشاء الجدول مع الاحتفاظ بجميع الأعمدة
    self.attendance_tree = ttk.Treeview(table_frame, columns=columns, show="headings",
                                        yscrollcommand=self.tree_scroll.set, style="Bold.Treeview")

    # تعيين عرض الأعمدة المرئية
    self.attendance_tree.column("checkbox", width=50, anchor=tk.CENTER)
    self.attendance_tree.column("id", width=150, anchor=tk.CENTER)
    self.attendance_tree.column("name", width=250, anchor=tk.CENTER)
    self.attendance_tree.column("rank", width=100, anchor=tk.CENTER)
    self.attendance_tree.column("course", width=150, anchor=tk.CENTER)
    self.attendance_tree.column("status", width=120, anchor=tk.CENTER)

    # إخفاء الأعمدة المطلوبة
    self.attendance_tree.column("absences", width=0, minwidth=0, stretch=False)
    self.attendance_tree.column("late_count", width=0, minwidth=0, stretch=False)
    self.attendance_tree.column("excused", width=0, minwidth=0, stretch=False)

    if self.current_user["permissions"]["can_view_edit_history"]:
        self.attendance_tree.column("updated_by", width=120, anchor=tk.CENTER)
        self.attendance_tree.column("updated_at", width=130, anchor=tk.CENTER)

    # تعيين عناوين الأعمدة
    self.attendance_tree.heading("checkbox", text="✓")
    self.attendance_tree.heading("id", text="رقم الهوية")
    self.attendance_tree.heading("name", text="الاسم")
    self.attendance_tree.heading("rank", text="الرتبة")
    self.attendance_tree.heading("course", text="الدورة")
    self.attendance_tree.heading("status", text="الحالة")
    self.attendance_tree.heading("absences", text="غياب بدون عذر")
    self.attendance_tree.heading("late_count", text="عدد مرات التأخير")
    self.attendance_tree.heading("excused", text="غياب بعذر")

    if self.current_user["permissions"]["can_view_edit_history"]:
        self.attendance_tree.heading("updated_by", text="من عدّل")
        self.attendance_tree.heading("updated_at", text="وقت آخر تعديل")

    self.attendance_tree.pack(fill=tk.BOTH, expand=True)
    self.tree_scroll.config(command=self.attendance_tree.yview)

    # إضافة معالج النقر على عمود الـ checkbox
    self.attendance_tree.bind("<ButtonRelease-1>", self.on_tree_click)

    self.attendance_tree.tag_configure("present", background="#e8f5e9")
    self.attendance_tree.tag_configure("absent", background="#ffebee")
    self.attendance_tree.tag_configure("late", background="#fff8e1")
    self.attendance_tree.tag_configure("excused", background="#e1f5fe")
    self.attendance_tree.tag_configure("not_started", background="#FFE5CC")
    self.attendance_tree.tag_configure("field_application", background="#E0E0E0")
    self.attendance_tree.tag_configure("student_day", background="#ECECEC")
    self.attendance_tree.tag_configure("evening_remote", background="#DDDDDD")
    self.attendance_tree.tag_configure("death_case", background="#E0D6F5")
    self.attendance_tree.tag_configure("hospital", background="#D4F0ED")
    self.attendance_tree.tag_configure("checked", background="#f5f5f5")

    # تمكين تحديد متعدد للصفوف
    self.attendance_tree.configure(selectmode="extended")

    if self.current_user["permissions"]["can_edit_attendance"]:
        self.attendance_tree.bind("<Double-1>", self.on_attendance_double_click)

def exclude_selected_students(self):
    """استبعاد المتدربين المحددين الذين حالتهم 'لم يباشر'"""
    # التحقق من وجود صلاحية المشرف
    if not self.current_user["permissions"]["is_admin"]:
        messagebox.showwarning("تنبيه", "هذه الوظيفة متاحة للمشرفين فقط")
        return

    # التحقق من أن التصفية المحددة هي "لم يباشر"
    current_filter = self.status_filter_var.get()
    if current_filter != "لم يباشر":
        messagebox.showwarning("تنبيه", "يجب تصفية القائمة بحالة 'لم يباشر' أولاً")
        return

    # الحصول على العناصر المحددة من قائمة الـ checkbox
    selected_items = []
    for item in self.attendance_tree.get_children():
        if item in self.selected_students:
            selected_items.append(item)

    # التحقق من أن هناك عناصر محددة
    if not selected_items:
        messagebox.showwarning("تنبيه", "الرجاء تحديد المتدربين المراد استبعادهم بالضغط على مربعات الاختيار")
        return

    # استخراج معلومات المتدربين المحددين
    selected_students = []
    for item in selected_items:
        values = self.attendance_tree.item(item, "values")
        national_id = values[1]  # الآن في العمود الثاني بعد إضافة checkbox
        name = values[2]  # الآن في العمود الثالث

        # التحقق من أن حالة المتدرب هي "لم يباشر"
        status = values[5]  # الآن في العمود السادس
        if status != "لم يباشر":
            messagebox.showwarning("تنبيه", f"المتدرب {name} ليست حالته 'لم يباشر'، لذا لا يمكن استبعاده")
            return

        selected_students.append((national_id, name))

    # طلب سبب الاستبعاد
    exclusion_reason = simpledialog.askstring(
        "سبب الاستبعاد",
        f"أدخل سبب استبعاد {len(selected_students)} متدرب:",
        initialvalue="عدم مباشرة الدورة"
    )

    if not exclusion_reason:
        return  # تم إلغاء العملية

    # تأكيد الاستبعاد
    if not messagebox.askyesno(
            "تأكيد الاستبعاد",
            f"هل أنت متأكد من استبعاد {len(selected_students)} متدرب بسبب:\n{exclusion_reason}"
    ):
        return

    # استبعاد المتدربين المحددين
    current_date = datetime.datetime.now().strftime("%Y-%m-%d")
    excluded_count = 0

    try:
        cursor = self.conn.cursor()
        for national_id, name in selected_students:
            with self.conn:
                # تحديث حالة المتدرب إلى مستبعد
                self.conn.execute("""
                    UPDATE trainees 
                    SET is_excluded=1, 
                        exclusion_reason=?, 
                        excluded_date=?
                    WHERE national_id=?
                """, (exclusion_reason, current_date, national_id))

                excluded_count += 1

        messagebox.showinfo("نجاح", f"تم استبعاد {excluded_count} متدرب بنجاح")

        # تحديث عرض الحضور والإحصائيات بعد الاستبعاد
        self.update_attendance_display()
        self.update_statistics()
        self.update_students_tree()

    except Exception as e:
        messagebox.showerror("خطأ", f"حدث خطأ أثناء استبعاد المتدربين: {str(e)}")

def export_official_completion(self):
    """تصدير التكميل الرسمي للدورات إلى Excel"""
    if not self.current_user["permissions"]["can_export_data"]:
        messagebox.showwarning("تنبيه", "ليس لديك صلاحية تصدير البيانات")
        return

    try:
        # التحقق من تواريخ انتهاء جميع الدورات
        current_date = datetime.datetime.now().date()
        cursor = self.conn.cursor()

        # الحصول على الدورات التي تجاوزت تاريخ النهاية
        cursor.execute("""
            SELECT course_name, end_date_system
            FROM course_info
            WHERE end_date_system IS NOT NULL AND end_date_system != ''
        """)

        expired_courses = []
        for course_name, end_date_str in cursor.fetchall():
            try:
                end_date = datetime.datetime.strptime(end_date_str, "%Y-%m-%d").date()
                if current_date > end_date:
                    expired_courses.append(course_name)
            except:
                continue

        if expired_courses:
            courses_list = "\n".join(f"- {course}" for course in expired_courses)
            messagebox.showwarning(
                "دورات منتهية",
                f"لا يمكن تصدير التكميل الرسمي لوجود دورات انتهت ولم يتم تخريجها:\n\n{courses_list}\n\n"
                "يجب تخريج هذه الدورات من النظام أولاً"
            )
            return

        # الحصول على تاريخ التصدير المطلوب من صفحة استعراض الحضور
        selected_date = self.log_date_entry.get_date().strftime("%Y-%m-%d")

        # التحقق من وجود سجلات حضور في التاريخ المحدد
        cursor.execute("""
            SELECT COUNT(*) 
            FROM attendance 
            WHERE date = ?
        """, (selected_date,))

        attendance_count = cursor.fetchone()[0]

        if attendance_count == 0:
            messagebox.showwarning(
                "لا توجد سجلات",
                f"لا توجد سجلات حضور بتاريخ {selected_date}\n"
                "يجب تسجيل الحضور أولاً قبل تصدير التكميل الرسمي"
            )
            return

        # التحقق من المتدربين غير المسجلين في التاريخ المحدد
        cursor.execute("""
            SELECT COUNT(*)
            FROM trainees t
            WHERE (t.is_excluded = 0 OR (t.is_excluded = 1 AND t.excluded_date > ?))
            AND t.course != '' AND t.course IS NOT NULL
            AND t.national_id NOT IN (
                SELECT a.national_id 
                FROM attendance a 
                WHERE a.date = ?
            )
        """, (selected_date, selected_date))

        unregistered_count = cursor.fetchone()[0]

        if unregistered_count > 0:
            # الحصول على أسماء بعض المتدربين غير المسجلين للعرض
            cursor.execute("""
                SELECT t.name, t.course
                FROM trainees t
                WHERE (t.is_excluded = 0 OR (t.is_excluded = 1 AND t.excluded_date > ?))
                AND t.course != '' AND t.course IS NOT NULL
                AND t.national_id NOT IN (
                    SELECT a.national_id 
                    FROM attendance a 
                    WHERE a.date = ?
                )
                LIMIT 5
            """, (selected_date, selected_date))

            unregistered_students = cursor.fetchall()

            # بناء رسالة التنبيه
            message = f"لا يمكن تصدير التكميل الرسمي\n\n"
            message += f"يوجد عدد ({unregistered_count}) متدرب غير مسجل حضورهم بتاريخ {selected_date}\n\n"

            if unregistered_students:
                message += "المتدربين الغير المسجلين هم :\n"
                for i, (name, course) in enumerate(unregistered_students[:5]):
                    message += f"{i + 1}. {name} - دورة: {course}\n"

                if unregistered_count > 5:
                    message += f"\n... وغيرهم ({unregistered_count - 5} متدرب آخر)\n"

            message += "\nيجب تسجيل حضور جميع المتدربين أولاً"

            messagebox.showwarning("متدربين غير مسجلين", message)
            return

        # اختيار مسار الحفظ
        export_file = filedialog.asksaveasfilename(
            defaultextension=".xlsx",
            filetypes=[("Excel files", "*.xlsx")],
            initialfile=f"التكميل_الرسمي_{selected_date}.xlsx"
        )

        if not export_file:
            return

        # الحصول على بيانات جميع الدورات النشطة أو المستبعدة بعد التاريخ المحدد
        cursor.execute("""
            SELECT DISTINCT t.course, COALESCE(ci.course_category, 'مشتركة') as category
            FROM trainees t
            LEFT JOIN course_info ci ON t.course = ci.course_name
            WHERE (t.is_excluded = 0 OR (t.is_excluded = 1 AND t.excluded_date > ?))
            AND t.course != '' AND t.course IS NOT NULL
            ORDER BY t.course
        """, (selected_date,))

        courses_data = cursor.fetchall()

        if not courses_data:
            messagebox.showinfo("تنبيه", "لا توجد دورات نشطة لتصدير التكميل الرسمي")
            return

        # حساب الإحصائيات لكل فئة (حذف فئة "طلبة")
        categories = ["ضباط", "أفراد", "مشتركة", "مدنيين"]
        stats = {cat: {
            'courses': set(),
            'students': 0,
            'absent': 0,
            'late': 0,
            'sick_leave': 0,
            'not_started': 0
        } for cat in categories}

        # جمع البيانات لليوم المحدد فقط
        for course_name, course_category in courses_data:
            if course_category not in categories:
                course_category = "مشتركة"  # القيمة الافتراضية

            stats[course_category]['courses'].add(course_name)

            # عدد المتدربين في الدورة (شامل الذين تم استبعادهم بعد التاريخ المحدد)
            cursor.execute("""
                SELECT COUNT(*) FROM trainees
                WHERE course = ? 
                AND (is_excluded = 0 OR (is_excluded = 1 AND excluded_date > ?))
            """, (course_name, selected_date))
            stats[course_category]['students'] += cursor.fetchone()[0]

            # إحصائيات الحضور لليوم المحدد فقط
            cursor.execute("""
                SELECT 
                    SUM(CASE WHEN a.status = 'غائب' THEN 1 ELSE 0 END) as absent,
                    SUM(CASE WHEN a.status = 'متأخر' THEN 1 ELSE 0 END) as late,
                    SUM(CASE WHEN a.status = 'منوم' THEN 1 ELSE 0 END) as sick,
                    SUM(CASE WHEN a.status = 'لم يباشر' THEN 1 ELSE 0 END) as not_started,
                    SUM(CASE WHEN a.status = 'حالة وفاة' THEN 1 ELSE 0 END) as death_case
                FROM attendance a
                JOIN trainees t ON a.national_id = t.national_id
                WHERE t.course = ? 
                AND (t.is_excluded = 0 OR (t.is_excluded = 1 AND t.excluded_date > ?))
                AND a.date = ?
            """, (course_name, selected_date, selected_date))

            result = cursor.fetchone()
            if result:
                # حالة الوفاة تُحسب مع الغياب
                stats[course_category]['absent'] += (result[0] or 0) + (result[4] or 0)
                stats[course_category]['late'] += result[1] or 0
                stats[course_category]['sick_leave'] += result[2] or 0  # منوم فقط
                stats[course_category]['not_started'] += result[3] or 0

        # إنشاء DataFrame للإحصائيات
        stats_data = []
        row_labels = ["عدد الدورات", "عدد الملتحقين", "الغياب", "تأخير", "إجازة مرضية", "لم يباشر", "المجموع"]

        for i, label in enumerate(row_labels):
            row = {"م": label}
            for category in categories:
                cat_stats = stats[category]
                if label == "عدد الدورات":
                    row[category] = len(cat_stats['courses'])
                elif label == "عدد الملتحقين":
                    row[category] = cat_stats['students']
                elif label == "الغياب":
                    row[category] = cat_stats['absent']
                elif label == "تأخير":
                    row[category] = cat_stats['late']
                elif label == "إجازة مرضية":
                    row[category] = cat_stats['sick_leave']
                elif label == "لم يباشر":
                    row[category] = cat_stats['not_started']
                elif label == "المجموع":
                    # حساب المجموع = عدد الملتحقين - (الغياب + التأخير + الإجازة المرضية + لم يباشر)
                    total = cat_stats['students'] - (cat_stats['absent'] + cat_stats['late'] +
                                                     cat_stats['sick_leave'] + cat_stats['not_started'])
                    row[category] = total
            stats_data.append(row)

        df_stats = pd.DataFrame(stats_data)

        # إنشاء DataFrame للتفاصيل لليوم المحدد فقط
        details_data = []
        row_num = 1

        # الغياب والتأخير والإجازات المرضية لليوم المحدد
        cursor.execute("""
            SELECT t.name, t.national_id, a.status, COALESCE(ci.course_category, 'مشتركة') as category
            FROM attendance a
            JOIN trainees t ON a.national_id = t.national_id
            LEFT JOIN course_info ci ON t.course = ci.course_name
            WHERE a.status IN ('غائب', 'متأخر', 'منوم', 'حالة وفاة')
            AND (t.is_excluded = 0 OR (t.is_excluded = 1 AND t.excluded_date > ?))
            AND a.date = ?
            ORDER BY a.status, t.name
        """, (selected_date, selected_date))

        for name, national_id, status, category in cursor.fetchall():
            # تحديد الملاحظة
            if status == "غائب":
                note = f"غياب {category}"
            elif status == "متأخر":
                note = f"تأخير {category}"
            elif status == "منوم":
                note = f"إجازة مرضية {category}"
            elif status == "حالة وفاة":
                note = f"غياب {category} - حالة وفاة"

            details_data.append({
                "العدد": row_num,
                "الاسم": name,
                "رقم الهوية": national_id,
                "ملاحظة": note
            })
            row_num += 1

        # إضافة سطر فارغ
        details_data.append({"العدد": "", "الاسم": "", "رقم الهوية": "", "ملاحظة": ""})
        details_data.append({"العدد": "", "الاسم": "", "رقم الهوية": "", "ملاحظة": ""})

        # "لم يباشر" في الأسفل لليوم المحدد
        cursor.execute("""
            SELECT t.name, t.national_id, COALESCE(ci.course_category, 'مشتركة') as category
            FROM attendance a
            JOIN trainees t ON a.national_id = t.national_id
            LEFT JOIN course_info ci ON t.course = ci.course_name
            WHERE a.status = 'لم يباشر'
            AND (t.is_excluded = 0 OR (t.is_excluded = 1 AND t.excluded_date > ?))
            AND a.date = ?
            ORDER BY t.name
        """, (selected_date, selected_date))

        for name, national_id, category in cursor.fetchall():
            details_data.append({
                "العدد": row_num,
                "الاسم": name,
                "رقم الهوية": national_id,
                "ملاحظة": f"لم يباشر {category}"
            })
            row_num += 1

        df_details = pd.DataFrame(details_data)

        # كتابة البيانات إلى Excel
        with pd.ExcelWriter(export_file, engine='openpyxl') as writer:
            # كتابة صفحة الإحصائيات
            df_stats.to_excel(writer, sheet_name='الإحصائيات', index=False, startrow=2)

            # كتابة صفحة التفاصيل
            df_details.to_excel(writer, sheet_name='التفاصيل', index=False, startrow=2)

            # تنسيق الصفحات
            workbook = writer.book

            # الحصول على اليوم والتاريخ
            selected_date_obj = self.log_date_entry.get_date()
            day_name = selected_date_obj.strftime('%A')
            # تحويل اسم اليوم إلى العربية
            days_arabic = {
                'Sunday': 'الأحد',
                'Monday': 'الاثنين',
                'Tuesday': 'الثلاثاء',
                'Wednesday': 'الأربعاء',
                'Thursday': 'الخميس',
                'Friday': 'الجمعة',
                'Saturday': 'السبت'
            }
            day_arabic = days_arabic.get(day_name, day_name)
            date_formatted = selected_date_obj.strftime('%Y/%m/%d')

            # تنسيق صفحة الإحصائيات
            stats_sheet = workbook['الإحصائيات']
            stats_sheet.sheet_view.rightToLeft = True

            # إضافة العنوان الرئيسي - تأكد من دمج الخلايا الصحيحة فقط
            stats_sheet.merge_cells('A1:E1')  # من A إلى E فقط (5 أعمدة: م + 4 فئات)
            stats_sheet[
                'A1'] = f"التكميل اليومي لدورات التخصصية المنعقدة بمدنية تدريب الأمن العام بالمنطقة الشرقية ليوم {day_arabic} الموافق {date_formatted}"
            stats_sheet['A1'].font = openpyxl.styles.Font(bold=True, size=14)
            stats_sheet['A1'].alignment = openpyxl.styles.Alignment(horizontal="center", vertical="center",
                                                                    wrap_text=True)
            stats_sheet['A1'].fill = openpyxl.styles.PatternFill(start_color="E0E0E0", end_color="E0E0E0",
                                                                 fill_type="solid")

            # تنسيق الرؤوس - فقط 5 أعمدة
            for row in stats_sheet['A3:E3']:
                for cell in row:
                    cell.font = openpyxl.styles.Font(bold=True, size=12)
                    cell.fill = openpyxl.styles.PatternFill(start_color="4CAF50", end_color="4CAF50",
                                                            fill_type="solid")
                    cell.font = openpyxl.styles.Font(bold=True, color="FFFFFF", size=12)
                    cell.alignment = openpyxl.styles.Alignment(horizontal="center", vertical="center")
                    cell.border = openpyxl.styles.Border(
                        left=openpyxl.styles.Side(style='thin'),
                        right=openpyxl.styles.Side(style='thin'),
                        top=openpyxl.styles.Side(style='thin'),
                        bottom=openpyxl.styles.Side(style='thin')
                    )

            # تنسيق البيانات - فقط 5 أعمدة
            for row in stats_sheet.iter_rows(min_row=4, max_row=stats_sheet.max_row, min_col=1, max_col=5):
                for cell in row:
                    cell.alignment = openpyxl.styles.Alignment(horizontal="center", vertical="center")
                    cell.border = openpyxl.styles.Border(
                        left=openpyxl.styles.Side(style='thin'),
                        right=openpyxl.styles.Side(style='thin'),
                        top=openpyxl.styles.Side(style='thin'),
                        bottom=openpyxl.styles.Side(style='thin')
                    )
                    # تنسيق العمود الأول (التسميات)
                    if cell.column == 1:
                        cell.font = openpyxl.styles.Font(bold=True)
                        cell.fill = openpyxl.styles.PatternFill(start_color="E0E0E0", end_color="E0E0E0",
                                                                fill_type="solid")

            # ضبط عرض الأعمدة بطريقة صحيحة - 5 أعمدة فقط
            column_widths = {'A': 20, 'B': 15, 'C': 15, 'D': 15, 'E': 15}
            for col_letter, width in column_widths.items():
                stats_sheet.column_dimensions[col_letter].width = width

            # تنسيق صفحة التفاصيل
            details_sheet = workbook['التفاصيل']
            details_sheet.sheet_view.rightToLeft = True

            # إضافة العنوان الرئيسي
            details_sheet.merge_cells('A1:D1')
            details_sheet['A1'] = f"إيضاح و ملاحظات التكميل ليوم {day_arabic} و تاريخ {date_formatted}"
            details_sheet['A1'].font = openpyxl.styles.Font(bold=True, size=14)
            details_sheet['A1'].alignment = openpyxl.styles.Alignment(horizontal="center", vertical="center",
                                                                      wrap_text=True)
            details_sheet['A1'].fill = openpyxl.styles.PatternFill(start_color="E0E0E0", end_color="E0E0E0",
                                                                   fill_type="solid")

            # تنسيق الرؤوس
            for row in details_sheet['A3:D3']:
                for cell in row:
                    cell.font = openpyxl.styles.Font(bold=True, size=12)
                    cell.fill = openpyxl.styles.PatternFill(start_color="4CAF50", end_color="4CAF50",
                                                            fill_type="solid")
                    cell.font = openpyxl.styles.Font(bold=True, color="FFFFFF", size=12)
                    cell.alignment = openpyxl.styles.Alignment(horizontal="center", vertical="center")
                    cell.border = openpyxl.styles.Border(
                        left=openpyxl.styles.Side(style='thin'),
                        right=openpyxl.styles.Side(style='thin'),
                        top=openpyxl.styles.Side(style='thin'),
                        bottom=openpyxl.styles.Side(style='thin')
                    )

            # تنسيق البيانات
            for row in details_sheet.iter_rows(min_row=4, max_row=details_sheet.max_row, min_col=1, max_col=4):
                for cell in row:
                    cell.alignment = openpyxl.styles.Alignment(horizontal="center", vertical="center")
                    cell.border = openpyxl.styles.Border(
                        left=openpyxl.styles.Side(style='thin'),
                        right=openpyxl.styles.Side(style='thin'),
                        top=openpyxl.styles.Side(style='thin'),
                        bottom=openpyxl.styles.Side(style='thin')
                    )

            # ضبط عرض الأعمدة
            details_sheet.column_dimensions['A'].width = 10
            details_sheet.column_dimensions['B'].width = 30
            details_sheet.column_dimensions['C'].width = 20
            details_sheet.column_dimensions['D'].width = 25

        messagebox.showinfo("نجاح", f"تم تصدير التكميل الرسمي ليوم {day_arabic} بنجاح إلى:\n{export_file}")

        # فتح الملف
        try:
            os.startfile(export_file)
        except:
            pass

    except Exception as e:
        messagebox.showerror("خطأ", f"حدث خطأ أثناء تصدير التكميل الرسمي: {str(e)}")

def show_unregistered_students(self):
    """عرض المتدربين الذين لم يتم تسجيلهم في تاريخ معين مع إمكانية التصفية حسب الدورة"""
    # الحصول على التاريخ المحدد
    selected_date = self.date_entry.get_date().strftime("%Y-%m-%d")

    # إنشاء نافذة جديدة
    unregistered_window = tk.Toplevel(self.root)
    unregistered_window.title(f"المتدربين غير المسجلين بتاريخ {selected_date}")
    unregistered_window.geometry("800x600")
    unregistered_window.configure(bg=self.colors["light"])

    # توسيط النافذة
    x = (unregistered_window.winfo_screenwidth() - 800) // 2
    y = (unregistered_window.winfo_screenheight() - 600) // 2
    unregistered_window.geometry(f"800x600+{x}+{y}")

    # عنوان النافذة
    tk.Label(
        unregistered_window,
        text=f"المتدربين غير المسجلين بتاريخ {selected_date}",
        font=self.fonts["title"],
        bg=self.colors["primary"],
        fg="white",
        padx=10, pady=10
    ).pack(fill=tk.X)

    # إطار اختيار الدورة
    filter_frame = tk.Frame(unregistered_window, bg=self.colors["light"], padx=10, pady=10)
    filter_frame.pack(fill=tk.X)

    tk.Label(
        filter_frame,
        text="اختر الدورة:",
        font=self.fonts["text_bold"],
        bg=self.colors["light"]
    ).pack(side=tk.RIGHT, padx=10)

    # الحصول على قائمة الدورات المتاحة
    cursor = self.conn.cursor()
    cursor.execute("SELECT DISTINCT course FROM trainees WHERE is_excluded=0 ORDER BY course")
    courses = ["جميع الدورات"] + [row[0] for row in cursor.fetchall() if row[0]]

    course_var = tk.StringVar(value="جميع الدورات")
    course_combo = ttk.Combobox(
        filter_frame,
        textvariable=course_var,
        values=courses,
        state="readonly",
        width=30,
        font=self.fonts["text"]
    )
    course_combo.pack(side=tk.RIGHT, padx=5)

    # متغيرات التصفح المحدود
    page_size = 100
    current_page = 1

    # زر تطبيق التصفية
    filter_btn = tk.Button(
        filter_frame,
        text="تطبيق",
        font=self.fonts["text_bold"],
        bg=self.colors["primary"],
        fg="white",
        padx=10, pady=5,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=lambda: load_students(1)  # عند تغيير الدورة، ابدأ من الصفحة الأولى
    )
    filter_btn.pack(side=tk.LEFT, padx=5)

    # إطار النتائج والتصفح
    result_frame = tk.Frame(unregistered_window, bg=self.colors["light"], padx=10, pady=10)
    result_frame.pack(fill=tk.BOTH, expand=True)

    # إطار القائمة
    list_frame = tk.Frame(result_frame, bg=self.colors["light"])
    list_frame.pack(fill=tk.BOTH, expand=True)

    # شريط التمرير للقائمة
    list_scroll = tk.Scrollbar(list_frame)
    list_scroll.pack(side=tk.RIGHT, fill=tk.Y)

    # إنشاء قائمة المتدربين
    students_tree = ttk.Treeview(
        list_frame,
        columns=("id", "name", "rank", "course"),
        show="headings",
        yscrollcommand=list_scroll.set,
        style="Bold.Treeview"
    )

    # تعريف أعمدة الجدول
    students_tree.column("id", width=120, anchor=tk.CENTER)
    students_tree.column("name", width=200, anchor=tk.CENTER)
    students_tree.column("rank", width=100, anchor=tk.CENTER)
    students_tree.column("course", width=150, anchor=tk.CENTER)

    # تعريف عناوين الأعمدة
    students_tree.heading("id", text="رقم الهوية")
    students_tree.heading("name", text="الاسم")
    students_tree.heading("rank", text="الرتبة")
    students_tree.heading("course", text="اسم الدورة")

    students_tree.pack(fill=tk.BOTH, expand=True)
    list_scroll.config(command=students_tree.yview)

    # متغير لحفظ ملصق عدد المتدربين
    students_count_var = tk.StringVar(value="إجمالي عدد المتدربين غير المسجلين: 0")
    students_count_label = tk.Label(
        result_frame,
        textvariable=students_count_var,
        font=self.fonts["text_bold"],
        bg=self.colors["light"],
        fg=self.colors["primary"],
        pady=5
    )
    students_count_label.pack(fill=tk.X)

    # إطار أزرار التصفح
    pagination_frame = tk.Frame(result_frame, bg=self.colors["light"])
    pagination_frame.pack(fill=tk.X, pady=5)

    # عناصر التصفح
    prev_btn = tk.Button(
        pagination_frame,
        text="السابق",
        font=self.fonts["text_bold"],
        bg=self.colors["secondary"],
        fg="white",
        padx=10, pady=2,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=lambda: change_page(-1)
    )
    prev_btn.pack(side=tk.LEFT, padx=5)

    page_var = tk.StringVar(value="الصفحة 1")
    page_label = tk.Label(
        pagination_frame,
        textvariable=page_var,
        font=self.fonts["text_bold"],
        bg=self.colors["light"]
    )
    page_label.pack(side=tk.LEFT, padx=10)

    next_btn = tk.Button(
        pagination_frame,
        text="التالي",
        font=self.fonts["text_bold"],
        bg=self.colors["secondary"],
        fg="white",
        padx=10, pady=2,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=lambda: change_page(1)
    )
    next_btn.pack(side=tk.LEFT, padx=5)

    # زر الإغلاق
    tk.Button(
        unregistered_window,
        text="إغلاق",
        font=self.fonts["text_bold"],
        bg=self.colors["dark"],
        fg="white",
        padx=15, pady=5,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=unregistered_window.destroy
    ).pack(side=tk.BOTTOM, pady=10)

    # متغيرات لتتبع عدد الصفحات والصفحة الحالية
    total_pages = 1

    def change_page(delta):
        """تغيير الصفحة الحالية"""
        nonlocal current_page
        new_page = current_page + delta

        if 1 <= new_page <= total_pages:
            current_page = new_page
            load_students(current_page)

    # دالة لتحميل المتدربين حسب الدورة المحددة والصفحة
    def load_students(page=1):
        nonlocal current_page, total_pages
        current_page = page

        # مسح البيانات الحالية
        for item in students_tree.get_children():
            students_tree.delete(item)

        # إعداد الاستعلام حسب الدورة المحددة
        selected_course = course_var.get()

        # استعلام العدد الإجمالي أولاً
        count_query = ""
        count_params = []

        if selected_course == "جميع الدورات":
            # استعلام لجميع الدورات
            count_query = """
                SELECT COUNT(*)
                FROM trainees t
                WHERE t.is_excluded = 0
                AND t.national_id NOT IN (
                    SELECT national_id FROM attendance WHERE date = ?
                )
            """
            count_params = [selected_date]
        else:
            # استعلام للدورة المحددة فقط
            count_query = """
                SELECT COUNT(*)
                FROM trainees t
                WHERE t.is_excluded = 0
                AND t.course = ?
                AND t.national_id NOT IN (
                    SELECT national_id FROM attendance WHERE date = ?
                )
            """
            count_params = [selected_course, selected_date]

        # تنفيذ استعلام العدد
        cursor.execute(count_query, count_params)
        total_count = cursor.fetchone()[0]

        # حساب عدد الصفحات
        total_pages = max(1, (total_count + page_size - 1) // page_size)

        # تحديث متغيرات التصفح
        page_var.set(f"الصفحة {current_page} من {total_pages}")
        students_count_var.set(f"إجمالي عدد المتدربين غير المسجلين: {total_count}")

        # بناء استعلام المتدربين مع حدود الصفحة
        offset = (current_page - 1) * page_size

        query = ""
        params = []

        if selected_course == "جميع الدورات":
            query = """
                SELECT t.national_id, t.name, t.rank, t.course
                FROM trainees t
                WHERE t.is_excluded = 0
                AND t.national_id NOT IN (
                    SELECT national_id FROM attendance WHERE date = ?
                )
                ORDER BY t.course, t.name
                LIMIT ? OFFSET ?
            """
            params = [selected_date, page_size, offset]
        else:
            query = """
                SELECT t.national_id, t.name, t.rank, t.course
                FROM trainees t
                WHERE t.is_excluded = 0
                AND t.course = ?
                AND t.national_id NOT IN (
                    SELECT national_id FROM attendance WHERE date = ?
                )
                ORDER BY t.name
                LIMIT ? OFFSET ?
            """
            params = [selected_course, selected_date, page_size, offset]

        # تنفيذ الاستعلام
        cursor.execute(query, params)
        unregistered_students = cursor.fetchall()

        # إضافة المتدربين إلى القائمة
        for student in unregistered_students:
            students_tree.insert("", tk.END, values=student)

    # تحميل البيانات عند فتح النافذة
    load_students()

def get_total_days(self, national_id):
    """حساب إجمالي عدد أيام الدورة للمتدرب"""
    cursor = self.conn.cursor()
    cursor.execute("""
        SELECT COUNT(DISTINCT date) 
        FROM attendance 
        WHERE national_id=?
    """, (national_id,))
    total = cursor.fetchone()[0]
    return total if total > 0 else 1  # لتجنب القسمة على صفر

def export_based_on_filter(self):
    if not self.current_user["permissions"]["can_export_data"]:
        messagebox.showwarning("تنبيه", "ليس لديك صلاحية تصدير البيانات")
        return

    status_value = self.status_filter_var.get()
    if status_value == "الكل":
        self.export_filtered_records(None)
    else:
        self.export_filtered_records(status_value)

def change_page(self, delta, total_pages):
    """تغيير الصفحة الحالية والتحديث"""
    new_page = self.current_page + delta

    # التأكد من أن الصفحة الجديدة ضمن النطاق المسموح
    if 1 <= new_page <= total_pages:
        self.current_page = new_page
        self.update_attendance_display()

def apply_status_color(self, item_id, status):
    """تطبيق لون مناسب حسب حالة الحضور"""
    if status == "حاضر":
        self.attendance_tree.item(item_id, tags=("present",))
    elif status == "متأخر":
        self.attendance_tree.item(item_id, tags=("late",))
    elif status == "غائب بعذر":
        self.attendance_tree.item(item_id, tags=("excused",))
    elif status == "لم يباشر":
        self.attendance_tree.item(item_id, tags=("not_started",))
    elif status == "تطبيق ميداني":
        self.attendance_tree.item(item_id, tags=("field_application",))
    elif status == "يوم طالب":
        self.attendance_tree.item(item_id, tags=("student_day",))
    elif status == "مسائية / عن بعد":
        self.attendance_tree.item(item_id, tags=("evening_remote",))
    elif status == "حالة وفاة":
        self.attendance_tree.item(item_id, tags=("death_case",))
    elif status == "منوم":
        self.attendance_tree.item(item_id, tags=("hospital",))

def reset_attendance_day(self):
    """إعادة تعيين سجلات حضور اليوم المحدد فقط"""
    if not self.current_user["permissions"]["can_reset_attendance"]:
        messagebox.showwarning("تنبيه", "ليس لديك صلاحية إعادة تعيين سجلات الحضور")
        return

    date_str = self.log_date_entry.get_date().strftime("%Y-%m-%d")

    # تأكيد العملية
    if messagebox.askyesno("تأكيد", f"سيتم حذف كل سجلات الحضور بتاريخ {date_str}، هل أنت متأكد؟"):
        try:
            with self.conn:
                self.conn.execute("DELETE FROM attendance WHERE date=?", (date_str,))
            messagebox.showinfo("نجاح", f"تم حذف السجلات بتاريخ {date_str}")
            self.update_attendance_display()
            self.update_statistics()
        except Exception as e:
            messagebox.showerror("خطأ", str(e))

def export_filtered_records(self, status_value):
    if not self.current_user["permissions"]["can_export_data"]:
        messagebox.showwarning("تنبيه", "ليس لديك صلاحية تصدير البيانات")
        return

    date_str = self.log_date_entry.get_date().strftime("%Y-%m-%d")
    query = """
        SELECT
            a.national_id as 'رقم الهوية',
            a.name as 'الاسم',
            a.rank as 'الرتبة',
            a.course as 'اسم الدورة',
            a.time as 'وقت التسجيل',
            a.date as 'التاريخ',
            a.status as 'الحالة',
            a.registered_by as 'سجّل بواسطة',
            a.excuse_reason as 'السبب'
    """

    if self.current_user["permissions"]["can_view_edit_history"]:
        query += """,
            a.updated_by as 'آخر من عدّل',
            a.updated_at as 'وقت آخر تعديل'
        """

    query += """
        FROM attendance a
        JOIN trainees t ON a.national_id = t.national_id
        WHERE a.date=? AND t.is_excluded=0
    """

    params = [date_str]
    if status_value is not None:
        query += " AND a.status=?"
        params.append(status_value)

    df = pd.read_sql(query, self.conn, params=params)
    if df.empty:
        stxt = "الكل" if (status_value is None) else status_value
        messagebox.showinfo("ملاحظة", f"لا توجد بيانات {stxt} في هذا اليوم.")
        return

    stxt = "الكل" if (status_value is None) else status_value
    export_file = filedialog.asksaveasfilename(
        defaultextension=".xlsx",
        filetypes=[("Excel files", "*.xlsx")],
        initialfile=f"{stxt}_{date_str}.xlsx"
    )
    if not export_file:
        return

    try:
        df.to_excel(export_file, index=False)
        messagebox.showinfo("نجاح", f"تم تصدير {stxt} إلى الملف: {export_file}")
    except Exception as e:
        messagebox.showerror("خطأ", str(e))

def on_attendance_double_click(self, event=None):
    if not self.current_user["permissions"]["can_edit_attendance"]:
        messagebox.showwarning("تنبيه", "ليس لديك صلاحية تعديل سجلات الحضور")
        return

    item = self.attendance_tree.selection()
    if not item:
        return
    values = self.attendance_tree.item(item, "values")
    if not values:
        return

    # تعديل هنا: الهوية الآن في العمود الثاني (index 1) بدلاً من الأول
    national_id = values[1]
    date_str = self.log_date_entry.get_date().strftime("%Y-%m-%d")
    attendance_id = self.get_attendance_record_id(national_id, date_str)
    if not attendance_id:
        messagebox.showinfo("خطأ", "لا يمكن العثور على سجل الحضور المحدد.")
        return
    self.open_edit_attendance_window(attendance_id, values)

def get_attendance_record_id(self, national_id, date_str):
    cursor = self.conn.cursor()
    cursor.execute("SELECT id FROM attendance WHERE national_id=? AND date=?", (national_id, date_str))
    result = cursor.fetchone()
    return result[0] if result else None

def open_edit_attendance_window(self, attendance_id, row_values):
    if not self.current_user["permissions"]["can_edit_attendance"]:
        messagebox.showwarning("تنبيه", "ليس لديك صلاحية تعديل سجلات الحضور")
        return

    # الحصول على الحالة الأصلية للمتدرب
    cursor = self.conn.cursor()
    cursor.execute("SELECT original_status FROM attendance WHERE id=?", (attendance_id,))
    original_status = cursor.fetchone()[0]

    edit_window = tk.Toplevel(self.root)
    edit_window.title("تعديل حالة الحضور")
    edit_window.geometry("500x430")  # زيادة الارتفاع لاستيعاب الحقل الجديد
    edit_window.configure(bg=self.colors["light"])
    edit_window.transient(self.root)
    edit_window.grab_set()

    x = (edit_window.winfo_screenwidth() - 500) // 2
    y = (edit_window.winfo_screenheight() - 430) // 2
    edit_window.geometry(f"500x430+{x}+{y}")

    tk.Label(edit_window, text="تعديل حالة الحضور", font=self.fonts["title"], bg=self.colors["primary"],
             fg="white", padx=10, pady=10, width=500).pack(fill=tk.X)

    info_frame = tk.Frame(edit_window, bg=self.colors["light"], padx=20, pady=10)
    info_frame.pack(fill=tk.X)

    # تعديل هنا: الاسم الآن في العمود الثالث (index 2) بدلاً من الثاني
    tk.Label(info_frame, text=f"المتدرب: {row_values[2]}", font=self.fonts["text_bold"],
             bg=self.colors["light"]).pack(anchor=tk.W)

    # تعديل هنا: الحالة الآن في العمود السادس (index 5) بدلاً من الخامس
    tk.Label(info_frame, text=f"الحالة الحالية: {row_values[5]}", font=self.fonts["text"],
             bg=self.colors["light"]).pack(anchor=tk.W)

    tk.Label(info_frame, text=f"الحالة الأصلية: {original_status}", font=self.fonts["text"],
             bg=self.colors["light"]).pack(anchor=tk.W)

    new_status_frame = tk.LabelFrame(edit_window, text="اختر الحالة الجديدة", font=self.fonts["text_bold"],
                                     bg=self.colors["light"], fg=self.colors["dark"], padx=10, pady=10)
    new_status_frame.pack(fill=tk.X, padx=20, pady=10)

    # تعديل: إضافة جميع الحالات المتاحة للاختيار
    status_options = ["حاضر", "متأخر", "غائب", "غائب بعذر", "لم يباشر", "حالة وفاة", "منوم"]
    status_var = tk.StringVar(value=row_values[5])

    status_combobox = ttk.Combobox(new_status_frame, textvariable=status_var, values=status_options,
                                   state="readonly",
                                   font=self.fonts["text"])
    status_combobox.pack(fill=tk.X, padx=5, pady=5)

    # إطار أسباب الغياب
    reason_frame = tk.Frame(edit_window, bg=self.colors["light"], padx=20, pady=5)
    reason_frame.pack(fill=tk.X)

    reason_label = tk.Label(reason_frame, text="سبب الغياب:", font=self.fonts["text"], bg=self.colors["light"])
    reason_entry = tk.Entry(reason_frame, font=self.fonts["text"], width=40)

    # إطار سبب التعديل
    mod_reason_frame = tk.Frame(edit_window, bg=self.colors["light"], padx=20, pady=5)
    mod_reason_frame.pack(fill=tk.X)

    mod_reason_label = tk.Label(mod_reason_frame, text="سبب التعديل:", font=self.fonts["text"],
                                bg=self.colors["light"])
    mod_reason_entry = tk.Entry(mod_reason_frame, font=self.fonts["text"], width=40)

    def on_status_change(*args):
        # إظهار حقل سبب الغياب إذا كانت الحالة "غائب بعذر"
        if status_var.get() == "غائب بعذر":
            reason_label.pack(anchor=tk.W)
            reason_entry.pack(fill=tk.X, pady=5)
        else:
            reason_label.pack_forget()
            reason_entry.pack_forget()

        # إظهار حقل سبب التعديل فقط إذا كانت الحالة الجديدة مختلفة عن الحالة الأصلية
        if status_var.get() != original_status:
            mod_reason_label.pack(anchor=tk.W)
            mod_reason_entry.pack(fill=tk.X, pady=5)
        else:
            mod_reason_label.pack_forget()
            mod_reason_entry.pack_forget()

    status_var.trace("w", on_status_change)
    on_status_change()

    button_frame = tk.Frame(edit_window, bg=self.colors["light"], pady=10)
    button_frame.pack(fill=tk.X, padx=20)

    def save_changes():
        new_status = status_var.get()
        new_reason = reason_entry.get().strip() if new_status == "غائب بعذر" else ""
        modification_reason = mod_reason_entry.get().strip() if new_status != original_status else ""
        now_str = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        try:
            with self.conn:
                self.conn.execute("""
                    UPDATE attendance
                    SET status=?, excuse_reason=?,
                        updated_by=?, updated_at=?, modification_reason=?
                    WHERE id=?
                """, (
                    new_status,
                    new_reason,
                    self.current_user["full_name"],
                    now_str,
                    modification_reason,
                    attendance_id
                ))
            messagebox.showinfo("نجاح", "تم تحديث حالة الحضور بنجاح")
            edit_window.destroy()
            self.update_attendance_display()
            self.update_statistics()
        except Exception as e:
            messagebox.showerror("خطأ", str(e))

    save_btn = tk.Button(button_frame, text="حفظ التغييرات", font=self.fonts["text_bold"],
                         bg=self.colors["success"],
                         fg="white", padx=15, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=save_changes)
    save_btn.pack(side=tk.LEFT, padx=5)

    cancel_btn = tk.Button(button_frame, text="إلغاء", font=self.fonts["text_bold"], bg=self.colors["danger"],
                           fg="white", padx=15, pady=5, bd=0, relief=tk.FLAT, cursor="hand2",
                           command=edit_window.destroy)
    cancel_btn.pack(side=tk.RIGHT, padx=5)

    def delete_record():
        # إضافة: دالة لحذف التسجيل
        if messagebox.askyesno("تأكيد الحذف",
                               f"هل أنت متأكد من حذف تسجيل المتدرب {row_values[1]} ليوم {self.log_date_entry.get_date().strftime('%Y-%m-%d')}؟\n\nلن يظهر هذا اليوم في سجل المتدرب."):
            try:
                with self.conn:
                    self.conn.execute("DELETE FROM attendance WHERE id=?", (attendance_id,))
                messagebox.showinfo("نجاح", "تم حذف تسجيل المتدرب بنجاح")
                edit_window.destroy()
                self.update_attendance_display()
                self.update_statistics()
            except Exception as e:
                messagebox.showerror("خطأ", str(e))

    # إضافة: زر حذف التسجيل
    delete_btn = tk.Button(button_frame, text="حذف التسجيل", font=self.fonts["text_bold"],
                           bg="#FF5722",  # لون برتقالي مميز
                           fg="white", padx=15, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=delete_record)
    delete_btn.pack(side=tk.RIGHT, padx=5)

def add_new_student(self):
    if not self.current_user["permissions"]["can_add_students"]:
        messagebox.showwarning("تنبيه", "ليس لديك صلاحية إضافة متدربين جدد")
        return

    add_window = tk.Toplevel(self.root)
    add_window.title("إضافة متدرب جديد")
    add_window.geometry("400x350")
    add_window.configure(bg=self.colors["light"])
    add_window.transient(self.root)
    add_window.grab_set()

    x = (add_window.winfo_screenwidth() - 400) // 2
    y = (add_window.winfo_screenheight() - 350) // 2
    add_window.geometry(f"400x350+{x}+{y}")

    tk.Label(
        add_window,
        text="إضافة متدرب جديد",
        font=self.fonts["title"],
        bg=self.colors["primary"],
        fg="white",
        padx=10, pady=10, width=400
    ).pack(fill=tk.X)

    form_frame = tk.Frame(add_window, bg=self.colors["light"], padx=20, pady=20)
    form_frame.pack(fill=tk.BOTH)

    tk.Label(form_frame, text="رقم الهوية:", font=self.fonts["text_bold"], bg=self.colors["light"]).grid(row=0,
                                                                                                         column=1,
                                                                                                         padx=5,
                                                                                                         pady=5,
                                                                                                         sticky=tk.E)
    id_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25)
    id_entry.grid(row=0, column=0, padx=5, pady=5, sticky=tk.W)

    tk.Label(form_frame, text="الاسم:", font=self.fonts["text_bold"], bg=self.colors["light"]).grid(row=1, column=1,
                                                                                                    padx=5, pady=5,
                                                                                                    sticky=tk.E)
    name_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25)
    name_entry.grid(row=1, column=0, padx=5, pady=5, sticky=tk.W)

    tk.Label(form_frame, text="الرتبة:", font=self.fonts["text_bold"], bg=self.colors["light"]).grid(row=2,
                                                                                                     column=1,
                                                                                                     padx=5, pady=5,
                                                                                                     sticky=tk.E)
    rank_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25)
    rank_entry.grid(row=2, column=0, padx=5, pady=5, sticky=tk.W)

    tk.Label(form_frame, text="اسم الدورة:", font=self.fonts["text_bold"], bg=self.colors["light"]).grid(row=3,
                                                                                                         column=1,
                                                                                                         padx=5,
                                                                                                         pady=5,
                                                                                                         sticky=tk.E)

    # عرض قائمة الدورات الحالية
    cursor = self.conn.cursor()
    cursor.execute("SELECT DISTINCT course FROM trainees")
    courses = [row[0] for row in cursor.fetchall() if row[0]]

    if courses:
        course_entry = ttk.Combobox(form_frame, font=self.fonts["text"], width=23, values=courses)
    else:
        course_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25)

    course_entry.grid(row=3, column=0, padx=5, pady=5, sticky=tk.W)

    tk.Label(form_frame, text="رقم الجوال:", font=self.fonts["text_bold"], bg=self.colors["light"]).grid(row=4,
                                                                                                         column=1,
                                                                                                         padx=5,
                                                                                                         pady=5,
                                                                                                         sticky=tk.E)
    phone_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25)
    phone_entry.grid(row=4, column=0, padx=5, pady=5, sticky=tk.W)

    button_frame = tk.Frame(add_window, bg=self.colors["light"], pady=10)
    button_frame.pack(fill=tk.X)

    def save_student():
        nid = id_entry.get().strip()
        name = name_entry.get().strip()
        rank_ = rank_entry.get().strip()
        course = course_entry.get().strip()
        phone = phone_entry.get().strip()
        if not nid or not name:
            messagebox.showwarning("تنبيه", "يجب إدخال رقم الهوية والاسم على الأقل")
            return
        cursor = self.conn.cursor()
        cursor.execute("SELECT COUNT(*) FROM trainees WHERE national_id=?", (nid,))
        exists = cursor.fetchone()[0]

        # في حالة وجود المتدرب بالفعل، نسأل المستخدم إذا كان يريد الاستمرار
        if exists > 0:
            # تحقق ما إذا كان المتدرب موجود بنفس اسم الدورة
            cursor.execute("SELECT course FROM trainees WHERE national_id=?", (nid,))
            current_course = cursor.fetchone()[0]

            if current_course == course:
                messagebox.showwarning("تنبيه", f"رقم الهوية موجود بالفعل في نفس الدورة: {course}")
                return

            if not messagebox.askyesno("تأكيد الإضافة",
                                       f"المتدرب برقم الهوية {nid} موجود في دورة أخرى: {current_course}\n\nهل تريد حذفه من الدورة السابقة وإضافته للدورة الجديدة: {course}؟"):
                return

            try:
                # حذف المتدرب من الدورة القديمة
                with self.conn:
                    # حذف سجلات الحضور للمتدرب
                    self.conn.execute("DELETE FROM attendance WHERE national_id=?", (nid,))
                    # حذف المتدرب نفسه
                    self.conn.execute("DELETE FROM trainees WHERE national_id=?", (nid,))
            except Exception as e:
                messagebox.showerror("خطأ", f"حدث خطأ أثناء حذف السجل القديم: {str(e)}")
                return

        try:
            with self.conn:
                self.conn.execute("""
                    INSERT INTO trainees (national_id, name, rank, course, phone)
                    VALUES (?, ?, ?, ?, ?)
                """, (nid, name, rank_, course, phone))
            messagebox.showinfo("نجاح", "تمت الإضافة بنجاح")
            add_window.destroy()
            self.update_students_tree()
            self.update_statistics()
        except Exception as e:
            messagebox.showerror("خطأ", str(e))

    save_btn = tk.Button(button_frame, text="حفظ", font=self.fonts["text_bold"], bg=self.colors["success"],
                         fg="white",
                         padx=15, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=save_student)
    save_btn.pack(side=tk.LEFT, padx=10)

    cancel_btn = tk.Button(button_frame, text="إلغاء", font=self.fonts["text_bold"], bg=self.colors["danger"],
                           fg="white",
                           padx=15, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=add_window.destroy)
    cancel_btn.pack(side=tk.RIGHT, padx=10)

def edit_student(self, from_selection=False):
    if not self.current_user["permissions"]["can_edit_students"]:
        messagebox.showwarning("تنبيه", "ليس لديك صلاحية تعديل بيانات المتدربين")
        return

    if from_selection:
        selected_item = self.students_tree.selection()
        if not selected_item:
            messagebox.showinfo("تنبيه", "الرجاء تحديد متدرب من القائمة")
            return
        values = self.students_tree.item(selected_item, "values")
        nid = values[0]
    else:
        nid = simpledialog.askstring("تعديل متدرب", "أدخل رقم هوية المتدرب:")
        if not nid:
            return

    cursor = self.conn.cursor()

    # أولاً: استرجاع معلومات المتدرب
    cursor.execute("SELECT * FROM trainees WHERE national_id=?", (nid,))
    student = cursor.fetchone()

    if not student:
        messagebox.showinfo("تنبيه", "لا توجد معلومات عن هذا المتدرب")
        return

    # ثانياً: استرجاع سجلات الحضور بشكل منفصل ومباشر
    cursor.execute("""
            SELECT id, national_id, name, rank, course, time, date, status, original_status, 
                   registered_by, excuse_reason, updated_by, updated_at, modification_reason
            FROM attendance 
            WHERE national_id=?
            ORDER BY date DESC
        """, (nid,))
    attendance_records = cursor.fetchall()

    edit_window = tk.Toplevel(self.root)
    edit_window.title("تعديل بيانات المتدرب")
    edit_window.geometry("400x350")
    edit_window.configure(bg=self.colors["light"])
    edit_window.transient(self.root)
    edit_window.grab_set()

    x = (edit_window.winfo_screenwidth() - 400) // 2
    y = (edit_window.winfo_screenheight() - 350) // 2
    edit_window.geometry(f"400x350+{x}+{y}")

    tk.Label(
        edit_window,
        text="تعديل بيانات المتدرب",
        font=self.fonts["title"],
        bg=self.colors["primary"],
        fg="white",
        padx=10, pady=10, width=400
    ).pack(fill=tk.X)

    form_frame = tk.Frame(edit_window, bg=self.colors["light"], padx=20, pady=20)
    form_frame.pack(fill=tk.BOTH)

    tk.Label(form_frame, text="رقم الهوية:", font=self.fonts["text_bold"], bg=self.colors["light"]).grid(row=0,
                                                                                                         column=1,
                                                                                                         padx=5,
                                                                                                         pady=5,
                                                                                                         sticky=tk.E)
    id_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25)
    id_entry.insert(0, student[0])
    id_entry.config(state="disabled")
    id_entry.grid(row=0, column=0, padx=5, pady=5, sticky=tk.W)

    tk.Label(form_frame, text="الاسم:", font=self.fonts["text_bold"], bg=self.colors["light"]).grid(row=1, column=1,
                                                                                                    padx=5, pady=5,
                                                                                                    sticky=tk.E)
    name_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25)
    name_entry.insert(0, student[1])
    name_entry.grid(row=1, column=0, padx=5, pady=5, sticky=tk.W)

    tk.Label(form_frame, text="الرتبة:", font=self.fonts["text_bold"], bg=self.colors["light"]).grid(row=2,
                                                                                                     column=1,
                                                                                                     padx=5, pady=5,
                                                                                                     sticky=tk.E)
    rank_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25)
    rank_entry.insert(0, student[2])
    rank_entry.grid(row=2, column=0, padx=5, pady=5, sticky=tk.W)

    tk.Label(form_frame, text="اسم الدورة:", font=self.fonts["text_bold"], bg=self.colors["light"]).grid(row=3,
                                                                                                         column=1,
                                                                                                         padx=5,
                                                                                                         pady=5,
                                                                                                         sticky=tk.E)

    # عرض قائمة الدورات الحالية
    cursor = self.conn.cursor()
    cursor.execute("SELECT DISTINCT course FROM trainees")
    courses = [row[0] for row in cursor.fetchall() if row[0]]

    if courses:
        course_entry = ttk.Combobox(form_frame, font=self.fonts["text"], width=23, values=courses)
        course_entry.set(student[3])
    else:
        course_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25)
        course_entry.insert(0, student[3])

    course_entry.grid(row=3, column=0, padx=5, pady=5, sticky=tk.W)

    tk.Label(form_frame, text="رقم الجوال:", font=self.fonts["text_bold"], bg=self.colors["light"]).grid(row=4,
                                                                                                         column=1,
                                                                                                         padx=5,
                                                                                                         pady=5,
                                                                                                         sticky=tk.E)
    phone_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25)
    phone_entry.insert(0, student[4])
    phone_entry.grid(row=4, column=0, padx=5, pady=5, sticky=tk.W)

    button_frame = tk.Frame(edit_window, bg=self.colors["light"], pady=10)
    button_frame.pack(fill=tk.X)

    def save_edit():
        new_name = name_entry.get().strip()
        new_rank = rank_entry.get().strip()
        new_course = course_entry.get().strip()
        new_phone = phone_entry.get().strip()
        if not new_name:
            messagebox.showwarning("تنبيه", "لا يمكن ترك حقل الاسم فارغًا")
            return
        try:
            with self.conn:
                # 1. تحديث بيانات المتدرب في جدول trainees
                self.conn.execute("""
                    UPDATE trainees
                    SET name=?, rank=?, course=?, phone=?
                    WHERE national_id=?
                """, (new_name, new_rank, new_course, new_phone, student[0]))

                # 2. تحديث اسم المتدرب والرتبة في جدول attendance
                if new_name != student[1] or new_rank != student[2]:
                    self.conn.execute("""
                        UPDATE attendance
                        SET name=?, rank=?
                        WHERE national_id=?
                    """, (new_name, new_rank, student[0]))

                # 3. إذا تم تغيير اسم الدورة، يجب تحديث سجلات الحضور أيضًا
                if new_course != student[3]:
                    self.conn.execute("""
                        UPDATE attendance
                        SET course=?
                        WHERE national_id=?
                    """, (new_course, student[0]))

                    # تحديث بيانات الفصول إذا كان هناك تغيير في اسم الدورة
                    self.conn.execute("""
                        UPDATE student_sections
                        SET course_name=?
                        WHERE national_id=? AND course_name=?
                    """, (new_course, student[0], student[3]))

            messagebox.showinfo("نجاح", "تم التعديل بنجاح")
            edit_window.destroy()
            self.update_students_tree()
            self.update_attendance_display()  # تحديث عرض الحضور بعد التعديل
        except Exception as e:
            messagebox.showerror("خطأ", str(e))

    save_btn = tk.Button(button_frame, text="حفظ", font=self.fonts["text_bold"], bg=self.colors["success"],
                         fg="white",
                         padx=15, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=save_edit)
    save_btn.pack(side=tk.LEFT, padx=10)

    cancel_btn = tk.Button(button_frame, text="إلغاء", font=self.fonts["text_bold"], bg=self.colors["danger"],
                           fg="white",
                           padx=15, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=edit_window.destroy)
    cancel_btn.pack(side=tk.RIGHT, padx=10)

def delete_selected_student(self):
    if not self.current_user["permissions"]["can_delete_students"]:
        messagebox.showwarning("تنبيه", "ليس لديك صلاحية حذف المتدربين")
        return

    selected_item = self.students_tree.selection()
    if not selected_item:
        messagebox.showinfo("تنبيه", "الرجاء تحديد متدرب من القائمة")
        return
    values = self.students_tree.item(selected_item, "values")
    nid = values[0]
    if not messagebox.askyesnocancel("تأكيد", f"هل تريد حذف المتدرب صاحب الهوية {nid}؟"):
        return
    try:
        with self.conn:
            self.conn.execute("DELETE FROM trainees WHERE national_id=?", (nid,))
        messagebox.showinfo("نجاح", "تم الحذف بنجاح")
        self.update_students_tree()
        self.update_statistics()
    except Exception as e:
        messagebox.showerror("خطأ", str(e))

def export_course_data(self, course_name):
    """وظيفة تصدير بيانات الدورة مع معلومات تفصيلية عن الغياب والتأخير"""
    if not self.current_user["permissions"]["can_export_data"]:
        messagebox.showwarning("تنبيه", "ليس لديك صلاحية تصدير البيانات")
        return

    try:
        # الحصول على بيانات المتدربين في الدورة
        cursor = self.conn.cursor()

        # استعلام البيانات الأساسية للمتدربين
        cursor.execute("""
            SELECT national_id, name, rank, phone, is_excluded, exclusion_reason, excluded_date
            FROM trainees
            WHERE course=?
        """, (course_name,))
        students_data = cursor.fetchall()

        if not students_data:
            messagebox.showinfo("ملاحظة", f"لا يوجد متدربين مسجلين في الدورة '{course_name}'")
            return

        # إنشاء نافذة حالة لإظهار تقدم التصدير
        progress_window = tk.Toplevel(self.root)
        progress_window.title("جاري تصدير البيانات")
        progress_window.geometry("400x150")
        progress_window.configure(bg=self.colors["light"])
        progress_window.transient(self.root)
        progress_window.resizable(False, False)
        progress_window.grab_set()

        x = (progress_window.winfo_screenwidth() - 400) // 2
        y = (progress_window.winfo_screenheight() - 150) // 2
        progress_window.geometry(f"400x150+{x}+{y}")

        tk.Label(
            progress_window,
            text=f"جاري تصدير بيانات الدورة: {course_name}",
            font=self.fonts["text_bold"],
            bg=self.colors["light"],
            pady=10
        ).pack()

        progress_var = tk.DoubleVar()
        progress_bar = ttk.Progressbar(
            progress_window,
            variable=progress_var,
            maximum=100,
            length=350
        )
        progress_bar.pack(pady=10)

        status_label = tk.Label(
            progress_window,
            text="جاري تحضير البيانات...",
            font=self.fonts["text"],
            bg=self.colors["light"]
        )
        status_label.pack(pady=5)

        progress_window.update()

        # إنشاء قاموس لتخزين بيانات كل متدرب
        students_dict = {}

        total_students = len(students_data)

        for index, student in enumerate(students_data):
            national_id, name, rank, phone, is_excluded, exclusion_reason, excluded_date = student

            # تحديث شريط التقدم
            progress_var.set((index / total_students) * 50)  # نصف التقدم للاستعلامات
            status_label.config(text=f"تحليل بيانات المتدرب {index + 1} من {total_students}: {name}")
            progress_window.update()

            # إحصاء حالات الغياب والتأخير لكل متدرب
            cursor.execute("""
                SELECT status, COUNT(*) 
                FROM attendance 
                WHERE national_id=? AND status IN ('غائب', 'متأخر', 'غائب بعذر')
                GROUP BY status
            """, (national_id,))
            status_counts = dict(cursor.fetchall())

            # الحصول على تواريخ الغياب
            cursor.execute("""
                SELECT date FROM attendance 
                WHERE national_id=? AND status='غائب'
                ORDER BY date
            """, (national_id,))
            absent_dates = [row[0] for row in cursor.fetchall()]
            absent_dates_str = " || ".join(absent_dates) if absent_dates else ""

            # الحصول على تواريخ التأخير
            cursor.execute("""
                SELECT date FROM attendance 
                WHERE national_id=? AND status='متأخر'
                ORDER BY date
            """, (national_id,))
            late_dates = [row[0] for row in cursor.fetchall()]
            late_dates_str = " || ".join(late_dates) if late_dates else ""

            # الحصول على أسباب وتواريخ الغياب بعذر
            cursor.execute("""
                SELECT date, excuse_reason 
                FROM attendance 
                WHERE national_id=? AND status='غائب بعذر'
                ORDER BY date
            """, (national_id,))
            excused_data = cursor.fetchall()

            excused_dates = [row[0] for row in excused_data]
            excused_dates_str = " || ".join(excused_dates) if excused_dates else ""

            # تجميع أسباب الغياب بعذر
            excuses_text = ""
            for date, reason in excused_data:
                if reason:
                    excuses_text += f"{date}: {reason} || "

            if excuses_text.endswith(" || "):
                excuses_text = excuses_text[:-4]

            # تنسيق حالة الاستبعاد
            excluded_status = "مستبعد" if is_excluded else "موجود"
            exclusion_info = exclusion_reason if is_excluded else "لا يوجد"
            exclusion_date = excluded_date if is_excluded else ""

            # تخزين البيانات في القاموس
            students_dict[national_id] = {
                "رقم الهوية": national_id,
                "الاسم": name,
                "الرتبة": rank,
                "الدورة": course_name,
                "رقم الجوال": phone,
                "عدد أيام الغياب": status_counts.get("غائب", 0),
                "تواريخ الغياب": absent_dates_str,
                "عدد أيام التأخير": status_counts.get("متأخر", 0),
                "تواريخ التأخير": late_dates_str,
                "عدد أيام الغياب بعذر": status_counts.get("غائب بعذر", 0),
                "أسباب الغياب بعذر": excuses_text,
                "حالة المتدرب": excluded_status,
                "سبب الاستبعاد": exclusion_info,
                "تاريخ الاستبعاد": exclusion_date
            }

        # تحويل القاموس إلى DataFrame
        progress_var.set(60)
        status_label.config(text="إنشاء ملف التصدير...")
        progress_window.update()

        df = pd.DataFrame(list(students_dict.values()))

        # حفظ الإكسل
        progress_var.set(70)
        status_label.config(text="فتح حوار حفظ الملف...")
        progress_window.update()

        export_file = filedialog.asksaveasfilename(
            defaultextension=".xlsx",
            filetypes=[("Excel files", "*.xlsx")],
            initialfile=f"دورة_{course_name}.xlsx"
        )

        if not export_file:
            progress_window.destroy()
            return

        # تصدير إلى ملف إكسل مع معالجة خاصة للخلايا
        progress_var.set(80)
        status_label.config(text="إنشاء ملف Excel...")
        progress_window.update()

        writer = pd.ExcelWriter(export_file, engine='xlsxwriter')
        df.to_excel(writer, index=False, sheet_name='بيانات الدورة')

        # الحصول على workbook وورقة العمل
        workbook = writer.book
        worksheet = writer.sheets['بيانات الدورة']

        # تنسيق الخلايا
        header_format = workbook.add_format({
            'bold': True,
            'text_wrap': True,
            'valign': 'top',
            'fg_color': '#4F81BD',
            'font_color': 'white',
            'border': 1,
            'align': 'center'
        })

        # تنسيق للمحتوى
        content_format = workbook.add_format({
            'text_wrap': True,
            'valign': 'top',
            'align': 'right',
            'border': 1
        })

        # تطبيق التنسيق على الرؤوس
        progress_var.set(90)
        status_label.config(text="تنسيق الملف...")
        progress_window.update()

        for col_num, value in enumerate(df.columns.values):
            worksheet.write(0, col_num, value, header_format)

        # تعديل عرض الأعمدة وتطبيق التنسيق
        for i, col in enumerate(df.columns):
            # تعيين عرض مناسب لكل عمود
            column_width = max(
                df[col].astype(str).map(len).max(),  # أطول نص في العمود
                len(str(col)) + 2  # عرض عنوان العمود + هامش
            )
            # تحديد حد أقصى
            if column_width > 50:
                column_width = 50
            worksheet.set_column(i, i, column_width, content_format)

        # إضافة تنسيق لجدول البيانات (تصميم الجدول)
        worksheet.add_table(0, 0, len(df), len(df.columns) - 1, {
            'style': 'Table Style Medium 2',
            'columns': [{'header': col} for col in df.columns]
        })

        # حفظ الملف
        progress_var.set(95)
        status_label.config(text="حفظ الملف...")
        progress_window.update()

        writer.close()

        progress_var.set(100)
        status_label.config(text="تم تصدير البيانات بنجاح!")
        progress_window.update()

        # إغلاق نافذة التقدم بعد ثانيتين
        progress_window.after(2000, progress_window.destroy)

        messagebox.showinfo("نجاح", f"تم تصدير بيانات الدورة '{course_name}' بنجاح إلى الملف:\n{export_file}")

    except Exception as e:
        try:
            progress_window.destroy()
        except:
            pass
        messagebox.showerror("خطأ", f"حدث خطأ أثناء تصدير بيانات الدورة: {str(e)}")

def import_new_course(self):
    """
    دالة استيراد دورة جديدة من ملف Excel
    مع إضافة تاريخ بداية ونهاية الدورة وفئة الدورة
    """
    if not self.current_user["permissions"]["can_import_data"]:
        messagebox.showwarning("تنبيه", "ليس لديك صلاحية استيراد البيانات")
        return

    import_window = tk.Toplevel(self.root)
    import_window.title("استيراد دورة جديدة")
    import_window.geometry("500x750")  # زيادة ارتفاع النافذة
    import_window.configure(bg=self.colors["light"])
    import_window.transient(self.root)
    import_window.grab_set()

    x = (import_window.winfo_screenwidth() - 500) // 2
    y = (import_window.winfo_screenheight() - 750) // 2
    import_window.geometry(f"500x750+{x}+{y}")

    tk.Label(
        import_window,
        text="استيراد دورة جديدة",
        font=self.fonts["title"],
        bg=self.colors["primary"],
        fg="white",
        padx=10, pady=10
    ).pack(fill=tk.X)

    input_frame = tk.Frame(import_window, bg=self.colors["light"], padx=20, pady=20)
    input_frame.pack(fill=tk.BOTH, expand=True)

    tk.Label(
        input_frame,
        text="اسم الدورة الجديدة:",
        font=self.fonts["text_bold"],
        bg=self.colors["light"]
    ).pack(anchor=tk.W, pady=(0, 5))

    course_entry = tk.Entry(input_frame, font=self.fonts["text"], width=40)
    course_entry.pack(fill=tk.X, pady=(0, 20))

    # إضافة قائمة منسدلة لفئة الدورة
    category_frame = tk.Frame(input_frame, bg=self.colors["light"])
    category_frame.pack(fill=tk.X, pady=10)

    tk.Label(
        category_frame,
        text="فئة الدورة:",
        font=self.fonts["text_bold"],
        bg=self.colors["light"]
    ).pack(side=tk.RIGHT, padx=5)

    course_categories = ["ضباط", "أفراد", "مشتركة", "مدنيين"]
    category_var = tk.StringVar(value="مشتركة")
    category_combo = ttk.Combobox(
        category_frame,
        textvariable=category_var,
        values=course_categories,
        state="readonly",
        width=20,
        font=self.fonts["text"]
    )
    category_combo.pack(side=tk.RIGHT, padx=5)

    # إضافة إطار لتاريخ نهاية الدورة في النظام
    system_end_date_frame = tk.LabelFrame(
        input_frame,
        text="تاريخ نهاية الدورة في النظام",
        font=self.fonts["text_bold"],
        bg=self.colors["light"],
        padx=10,
        pady=10
    )
    system_end_date_frame.pack(fill=tk.X, pady=10)

    system_end_date_entry = DateEntry(
        system_end_date_frame,
        width=15,
        background=self.colors["primary"],
        foreground='white',
        borderwidth=2,
        date_pattern='yyyy-mm-dd',
        font=self.fonts["text"],
        firstweekday="sunday"
    )
    system_end_date_entry.pack(pady=5)

    # إطار لتاريخ بداية الدورة النصي (للعرض فقط)
    start_date_frame = tk.LabelFrame(input_frame, text="تاريخ بداية الدورة (للعرض)", font=self.fonts["text_bold"],
                                     bg=self.colors["light"], padx=10, pady=10)
    start_date_frame.pack(fill=tk.X, pady=10)

    # حقول تاريخ البداية
    start_date_fields = tk.Frame(start_date_frame, bg=self.colors["light"])
    start_date_fields.pack(fill=tk.X)

    # اليوم
    tk.Label(start_date_fields, text="اليوم:", font=self.fonts["text"], bg=self.colors["light"]).pack(side=tk.RIGHT,
                                                                                                      padx=5)
    start_day_entry = tk.Entry(start_date_fields, font=self.fonts["text"], width=5)
    start_day_entry.pack(side=tk.RIGHT, padx=5)

    # الشهر
    tk.Label(start_date_fields, text="الشهر:", font=self.fonts["text"], bg=self.colors["light"]).pack(side=tk.RIGHT,
                                                                                                      padx=5)
    start_month_entry = tk.Entry(start_date_fields, font=self.fonts["text"], width=5)
    start_month_entry.pack(side=tk.RIGHT, padx=5)

    # السنة
    tk.Label(start_date_fields, text="السنة:", font=self.fonts["text"], bg=self.colors["light"]).pack(side=tk.RIGHT,
                                                                                                      padx=5)
    start_year_entry = tk.Entry(start_date_fields, font=self.fonts["text"], width=8)
    start_year_entry.pack(side=tk.RIGHT, padx=5)

    # إطار لتاريخ نهاية الدورة النصي (للعرض فقط)
    end_date_frame = tk.LabelFrame(input_frame, text="تاريخ نهاية الدورة (للعرض)", font=self.fonts["text_bold"],
                                   bg=self.colors["light"], padx=10, pady=10)
    end_date_frame.pack(fill=tk.X, pady=10)

    # حقول تاريخ النهاية
    end_date_fields = tk.Frame(end_date_frame, bg=self.colors["light"])
    end_date_fields.pack(fill=tk.X)

    # اليوم
    tk.Label(end_date_fields, text="اليوم:", font=self.fonts["text"], bg=self.colors["light"]).pack(side=tk.RIGHT,
                                                                                                    padx=5)
    end_day_entry = tk.Entry(end_date_fields, font=self.fonts["text"], width=5)
    end_day_entry.pack(side=tk.RIGHT, padx=5)

    # الشهر
    tk.Label(end_date_fields, text="الشهر:", font=self.fonts["text"], bg=self.colors["light"]).pack(side=tk.RIGHT,
                                                                                                    padx=5)
    end_month_entry = tk.Entry(end_date_fields, font=self.fonts["text"], width=5)
    end_month_entry.pack(side=tk.RIGHT, padx=5)

    # السنة
    tk.Label(end_date_fields, text="السنة:", font=self.fonts["text"], bg=self.colors["light"]).pack(side=tk.RIGHT,
                                                                                                    padx=5)
    end_year_entry = tk.Entry(end_date_fields, font=self.fonts["text"], width=8)
    end_year_entry.pack(side=tk.RIGHT, padx=5)

    # القيود على حقول التاريخ
    def validate_number(P, max_length):
        if P == "":
            return True
        if not P.isdigit():
            return False
        if len(P) > max_length:
            return False
        return True

    # تسجيل وظائف التحقق
    validate_day = import_window.register(lambda P: validate_number(P, 2))
    validate_month = import_window.register(lambda P: validate_number(P, 2))
    validate_year = import_window.register(lambda P: validate_number(P, 4))

    # تطبيق القيود
    start_day_entry.config(validate="key", validatecommand=(validate_day, "%P"))
    start_month_entry.config(validate="key", validatecommand=(validate_month, "%P"))
    start_year_entry.config(validate="key", validatecommand=(validate_year, "%P"))

    end_day_entry.config(validate="key", validatecommand=(validate_day, "%P"))
    end_month_entry.config(validate="key", validatecommand=(validate_month, "%P"))
    end_year_entry.config(validate="key", validatecommand=(validate_year, "%P"))

    columns_frame = tk.Frame(input_frame, bg=self.colors["light"])
    columns_frame.pack(fill=tk.X, pady=5)

    tk.Label(
        input_frame,
        text="يجب أن يكون ترتيب أسماء الأعمدة كما يلي : (الاسم - الرتبة - رقم الهوية - رقم الجوال)",
        font=self.fonts["text"],
        bg=self.colors["light"],
        fg=self.colors["secondary"]
    ).pack(anchor=tk.W, pady=(0, 10))

    file_frame = tk.Frame(input_frame, bg=self.colors["light"])
    file_frame.pack(fill=tk.X)

    file_path_var = tk.StringVar()
    file_entry = tk.Entry(file_frame, textvariable=file_path_var, font=self.fonts["text"], width=30,
                          state="readonly")
    file_entry.pack(side=tk.LEFT, fill=tk.X, expand=True, padx=(0, 10))

    def browse_file():
        file_path = filedialog.askopenfilename(
            title="اختر ملف Excel",
            filetypes=[("Excel files", "*.xlsx"), ("All files", "*.*")]
        )
        if file_path:
            file_path_var.set(file_path)

    browse_btn = tk.Button(
        file_frame,
        text="استعراض...",
        font=self.fonts["text"],
        bg=self.colors["secondary"],
        fg="white",
        padx=10, pady=3,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=browse_file
    )
    browse_btn.pack(side=tk.RIGHT)

    def import_course():
        course_name = course_entry.get().strip()
        file_path = file_path_var.get().strip()
        category = category_var.get()
        system_end_date = system_end_date_entry.get_date().strftime("%Y-%m-%d")

        start_day = start_day_entry.get().strip()
        start_month = start_month_entry.get().strip()
        start_year = start_year_entry.get().strip()
        end_day = end_day_entry.get().strip()
        end_month = end_month_entry.get().strip()
        end_year = end_year_entry.get().strip()

        # التحقق من البيانات
        if not course_name:
            messagebox.showwarning("تنبيه", "الرجاء إدخال اسم الدورة")
            return

        if not file_path:
            messagebox.showwarning("تنبيه", "الرجاء اختيار ملف Excel")
            return

        # التحقق من تواريخ البداية والنهاية (اختياري)
        date_valid = True
        date_message = ""

        if (start_day or start_month or start_year) and not (start_day and start_month and start_year):
            date_valid = False
            date_message = "يجب إدخال تاريخ بداية الدورة كاملاً (اليوم والشهر والسنة)"

        if (end_day or end_month or end_year) and not (end_day and end_month and end_year):
            date_valid = False
            date_message = "يجب إدخال تاريخ نهاية الدورة كاملاً (اليوم والشهر والسنة)"

        if not date_valid:
            messagebox.showwarning("تنبيه", date_message)
            return

        # فحص المتدربين المتكررين واستكمال عملية الاستيراد
        self.check_duplicate_students_with_dates(file_path, course_name,
                                                 start_day, start_month, start_year,
                                                 end_day, end_month, end_year,
                                                 system_end_date, category)
        import_window.destroy()

    button_frame = tk.Frame(import_window, bg=self.colors["light"], pady=10)
    button_frame.pack(fill=tk.X, padx=20)

    import_btn = tk.Button(
        button_frame,
        text="استيراد",
        font=self.fonts["text_bold"],
        bg=self.colors["success"],
        fg="white",
        padx=15, pady=5,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=import_course
    )
    import_btn.pack(side=tk.LEFT, padx=5)

    cancel_btn = tk.Button(
        button_frame,
        text="إلغاء",
        font=self.fonts["text_bold"],
        bg=self.colors["danger"],
        fg="white",
        padx=15, pady=5,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=import_window.destroy
    )
    cancel_btn.pack(side=tk.RIGHT, padx=5)

def save_course_dates_and_category(self, course_name, start_day, start_month, start_year,
                                   end_day, end_month, end_year, system_end_date, category):
    """
    حفظ تواريخ بداية ونهاية الدورة وفئتها في قاعدة البيانات
    """
    try:
        current_date = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")

        with self.conn:
            # التحقق من وجود الدورة
            cursor = self.conn.cursor()
            cursor.execute("SELECT COUNT(*) FROM course_info WHERE course_name=?", (course_name,))
            exists = cursor.fetchone()[0] > 0

            if exists:
                # تحديث بيانات الدورة الموجودة
                self.conn.execute("""
                    UPDATE course_info 
                    SET start_day=?, start_month=?, start_year=?, 
                        end_day=?, end_month=?, end_year=?,
                        end_date_system=?, course_category=?
                    WHERE course_name=?
                """, (start_day, start_month, start_year, end_day, end_month, end_year,
                      system_end_date, category, course_name))
            else:
                # إضافة دورة جديدة
                self.conn.execute("""
                    INSERT INTO course_info 
                    (course_name, start_day, start_month, start_year, end_day, end_month, end_year, 
                     end_date_system, course_category, created_date)
                    VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
                """, (course_name, start_day, start_month, start_year, end_day, end_month, end_year,
                      system_end_date, category, current_date))

        return True
    except Exception as e:
        print(f"خطأ في حفظ بيانات الدورة: {str(e)}")
        return False

def check_duplicate_students_with_dates(self, file_path, course_name,
                                        start_day, start_month, start_year,
                                        end_day, end_month, end_year,
                                        system_end_date, category):
    """
    فحص المتدربين المتكررين قبل استيراد دورة جديدة مع دعم تواريخ البداية والنهاية وفئة الدورة
    """
    try:
        # إنشاء نافذة حالة لإظهار تقدم العملية
        progress_window = tk.Toplevel(self.root)
        progress_window.title("فحص المتدربين المتكررين")
        progress_window.geometry("400x150")
        progress_window.configure(bg=self.colors["light"])
        progress_window.transient(self.root)
        progress_window.grab_set()

        # توسيط النافذة
        x = (progress_window.winfo_screenwidth() - 400) // 2
        y = (progress_window.winfo_screenheight() - 150) // 2
        progress_window.geometry(f"400x150+{x}+{y}")

        tk.Label(
            progress_window,
            text="جاري فحص المتدربين المتكررين...",
            font=self.fonts["text_bold"],
            bg=self.colors["light"],
            pady=10
        ).pack()

        progress_var = tk.DoubleVar()
        progress_bar = ttk.Progressbar(
            progress_window,
            variable=progress_var,
            maximum=100,
            length=350
        )
        progress_bar.pack(pady=10)

        status_label = tk.Label(
            progress_window,
            text="جاري قراءة ملف Excel...",
            font=self.fonts["text"],
            bg=self.colors["light"]
        )
        status_label.pack(pady=5)

        progress_window.update()

        # قراءة ملف Excel
        df = pd.read_excel(file_path)

        # تحديد الأعمدة المطلوبة (دعم الأسماء العربية والإنجليزية)
        column_mapping = {
            'الاسم': 'name',
            'رقم الهوية': 'national_id',
            'الرتبة': 'rank',
            'رقم الجوال': 'phone',
            'name': 'name',
            'national_id': 'national_id',
            'rank': 'rank',
            'phone': 'phone'
        }

        # تغيير أسماء الأعمدة إلى النموذج الإنجليزي
        df_columns = list(df.columns)
        english_columns = {}

        for col in df_columns:
            if col in column_mapping:
                english_columns[col] = column_mapping[col]

        # التحقق من وجود الأعمدة المطلوبة
        required_cols_ar = ["الاسم", "رقم الهوية", "الرتبة", "رقم الجوال"]
        required_cols_en = ["name", "national_id", "rank", "phone"]

        # التحقق من وجود العمود بأي من اللغتين
        has_name = any(col in ["الاسم", "name"] for col in df_columns)
        has_id = any(col in ["رقم الهوية", "national_id"] for col in df_columns)
        has_rank = any(col in ["الرتبة", "rank"] for col in df_columns)
        has_phone = any(col in ["رقم الجوال", "phone"] for col in df_columns)

        if not (has_name and has_id):
            progress_window.destroy()
            messagebox.showwarning("تحذير",
                                   "يجب أن يحتوي الملف على الأعمدة التالية على الأقل:\n"
                                   "- الاسم (name)\n"
                                   "- رقم الهوية (national_id)")
            return False

        # إعادة تسمية الأعمدة للاستخدام الداخلي
        rename_dict = {}
        for orig_col in df.columns:
            if orig_col in column_mapping:
                rename_dict[orig_col] = column_mapping[orig_col]

        if rename_dict:
            df = df.rename(columns=rename_dict)

        # إضافة الأعمدة المفقودة (اختياري) إذا لم تكن موجودة
        if 'rank' not in df.columns:
            df['rank'] = ''
        if 'phone' not in df.columns:
            df['phone'] = ''

        # قائمة المتدربين المتكررين
        duplicates = []

        # فحص كل متدرب
        progress_var.set(20)
        status_label.config(text="جاري فحص المتدربين المتكررين...")
        progress_window.update()

        total_rows = len(df)
        cursor = self.conn.cursor()

        for i, row in enumerate(df.iterrows()):
            # تحديث شريط التقدم
            progress = 20 + (i / total_rows * 60)
            progress_var.set(progress)

            _, row_data = row
            # تحويل رقم الهوية إلى نص
            nid = str(row_data["national_id"]).strip()
            name = str(row_data["name"]).strip()

            if i % 10 == 0:
                status_label.config(text=f"فحص المتدرب {i + 1} من {total_rows}: {name}")
                progress_window.update()

            # التحقق من وجود المتدرب
            cursor.execute("""
                SELECT t.course, t.name
                FROM trainees t
                WHERE t.national_id=?
            """, (nid,))

            result = cursor.fetchone()
            if result:
                current_course, current_name = result
                duplicates.append({
                    "id": nid,
                    "name": name,
                    "current_course": current_course
                })

        progress_window.destroy()

        # حفظ تواريخ الدورة وفئتها بغض النظر عن وجود متدربين متكررين
        self.save_course_dates_and_category(course_name, start_day, start_month, start_year,
                                            end_day, end_month, end_year, system_end_date, category)

        # عرض النتائج
        if duplicates:
            # عرض رسالة بأسماء المتدربين المتكررين فقط
            duplicate_details = f"تم العثور على {len(duplicates)} متدرب موجودين بالفعل في دورات أخرى:\n\n"

            # عرض أول 10 متدربين فقط لتجنب رسائل طويلة جداً
            display_count = min(10, len(duplicates))
            for i in range(display_count):
                duplicate_details += f"{i + 1}. {duplicates[i]['name']} (هوية: {duplicates[i]['id']}) - دورة: {duplicates[i]['current_course']}\n"

            if len(duplicates) > 10:
                duplicate_details += f"\n... وغيرهم ({len(duplicates) - 10} آخرين)"

            duplicate_details += "\n\nهل تريد نقل هؤلاء المتدربين من دوراتهم السابقة إلى الدورة الجديدة؟"

            choice = messagebox.askquestion("متدربين متكررين", duplicate_details, type=messagebox.YESNOCANCEL)

            if choice == "cancel":
                return False

            # متابعة الاستيراد مع خيار النقل (True) أو التخطي (False)
            update_mode = (choice == "yes")

            # إضافة سؤال عما إذا كانت الدورة متعددة الفصول
            is_multi_section = messagebox.askyesno("نوع الدورة", f"هل الدورة '{course_name}' متعددة الفصول؟")

            sections_count = 1
            if is_multi_section:
                # طلب عدد الفصول
                sections_count_str = simpledialog.askstring("عدد الفصول", "كم عدد الفصول في هذه الدورة؟",
                                                            initialvalue="2")
                if not sections_count_str:
                    return False

                try:
                    sections_count = int(sections_count_str)
                    if sections_count <= 0:
                        messagebox.showwarning("تنبيه", "يجب أن يكون عدد الفصول أكبر من صفر")
                        return False
                except:
                    messagebox.showwarning("تنبيه", "الرجاء إدخال رقم صحيح لعدد الفصول")
                    return False
            else:
                # إذا كانت الدورة غير متعددة الفصول، نجعلها بفصل واحد فقط
                sections_count = 1
                messagebox.showinfo("معلومات",
                                    f"سيتم إنشاء فصل واحد للدورة '{course_name}' ويمكنك إدارة الفصول لاحقًا من 'إدارة الفصول وتصدير الكشوفات'")

            # استدعاء دالة معالجة الاستيراد
            self.process_course_import_arabic(file_path, course_name, is_multi_section, sections_count, update_mode)
            return True
        else:
            messagebox.showinfo("تقرير الفحص",
                                f"لم يتم العثور على متدربين متكررين. يمكنك المتابعة في استيراد الدورة '{course_name}'.")
            # إضافة سؤال عما إذا كانت الدورة متعددة الفصول
            is_multi_section = messagebox.askyesno("نوع الدورة", f"هل الدورة '{course_name}' متعددة الفصول؟")

            sections_count = 1
            if is_multi_section:
                # طلب عدد الفصول
                sections_count_str = simpledialog.askstring("عدد الفصول", "كم عدد الفصول في هذه الدورة؟",
                                                            initialvalue="2")
                if not sections_count_str:
                    return False

                try:
                    sections_count = int(sections_count_str)
                    if sections_count <= 0:
                        messagebox.showwarning("تنبيه", "يجب أن يكون عدد الفصول أكبر من صفر")
                        return False
                except:
                    messagebox.showwarning("تنبيه", "الرجاء إدخال رقم صحيح لعدد الفصول")
                    return False
            else:
                # إذا كانت الدورة غير متعددة الفصول، نجعلها بفصل واحد فقط
                sections_count = 1
                messagebox.showinfo("معلومات",
                                    f"سيتم إنشاء فصل واحد للدورة '{course_name}' ويمكنك إدارة الفصول لاحقًا من 'إدارة الفصول وتصدير الكشوفات'")

            # استدعاء دالة معالجة الاستيراد
            self.process_course_import_arabic(file_path, course_name, is_multi_section, sections_count, False)
            return False

    except Exception as e:
        try:
            progress_window.destroy()
        except:
            pass
        messagebox.showerror("خطأ", f"حدث خطأ أثناء فحص المتدربين المتكررين: {str(e)}")
        return False

def save_course_dates(self, course_name, start_day, start_month, start_year, end_day, end_month, end_year):
    """
    حفظ تواريخ بداية ونهاية الدورة في قاعدة البيانات
    """
    try:
        current_date = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")

        with self.conn:
            # التحقق من وجود الدورة
            cursor = self.conn.cursor()
            cursor.execute("SELECT COUNT(*) FROM course_info WHERE course_name=?", (course_name,))
            exists = cursor.fetchone()[0] > 0

            if exists:
                # تحديث بيانات الدورة الموجودة
                self.conn.execute("""
                    UPDATE course_info 
                    SET start_day=?, start_month=?, start_year=?, 
                        end_day=?, end_month=?, end_year=?
                    WHERE course_name=?
                """, (start_day, start_month, start_year, end_day, end_month, end_year, course_name))
            else:
                # إضافة دورة جديدة
                self.conn.execute("""
                    INSERT INTO course_info 
                    (course_name, start_day, start_month, start_year, end_day, end_month, end_year, created_date)
                    VALUES (?, ?, ?, ?, ?, ?, ?, ?)
                """, (course_name, start_day, start_month, start_year, end_day, end_month, end_year, current_date))

        return True
    except Exception as e:
        print(f"خطأ في حفظ تواريخ الدورة: {str(e)}")
        return False

def edit_course_dates(self, course_name):
    """تعديل اسم الدورة وتواريخ بداية ونهاية الدورة وفئتها"""

    # استرجاع البيانات الحالية من قاعدة البيانات
    cursor = self.conn.cursor()
    cursor.execute("""
        SELECT start_day, start_month, start_year, end_day, end_month, end_year, 
               end_date_system, course_category
        FROM course_info
        WHERE course_name=?
    """, (course_name,))

    current_info = cursor.fetchone()
    current_dates = current_info[:6] if current_info else ["", "", "", "", "", ""]
    current_end_date_system = current_info[6] if current_info and len(current_info) > 6 else None
    current_category = current_info[7] if current_info and len(current_info) > 7 else "مشتركة"

    # إنشاء نافذة تعديل البيانات
    edit_window = tk.Toplevel(self.root)
    edit_window.title(f"تعديل بيانات دورة: {course_name}")
    edit_window.geometry("600x750")  # زيادة الارتفاع لاستيعاب حقل الفئة
    edit_window.configure(bg=self.colors["light"])
    edit_window.transient(self.root)
    edit_window.grab_set()

    # توسيط النافذة
    x = (edit_window.winfo_screenwidth() - 600) // 2
    y = (edit_window.winfo_screenheight() - 750) // 2
    edit_window.geometry(f"600x750+{x}+{y}")

    # عنوان النافذة
    tk.Label(
        edit_window,
        text="تعديل بيانات الدورة",
        font=self.fonts["title"],
        bg=self.colors["primary"],
        fg="white",
        padx=10, pady=10
    ).pack(fill=tk.X)

    # إطار المحتويات
    content_frame = tk.Frame(edit_window, bg=self.colors["light"], padx=20, pady=20)
    content_frame.pack(fill=tk.BOTH, expand=True)

    # حقل اسم الدورة
    name_frame = tk.LabelFrame(
        content_frame,
        text="اسم الدورة",
        font=self.fonts["text_bold"],
        bg=self.colors["light"],
        padx=10, pady=10
    )
    name_frame.pack(fill=tk.X, pady=10)

    new_course_name_var = tk.StringVar(value=course_name)
    course_name_entry = tk.Entry(
        name_frame,
        textvariable=new_course_name_var,
        font=self.fonts["text"],
        width=40
    )
    course_name_entry.pack(pady=5)

    # إطار فئة الدورة
    category_frame = tk.LabelFrame(
        content_frame,
        text="فئة الدورة",
        font=self.fonts["text_bold"],
        bg=self.colors["light"],
        padx=10, pady=10
    )
    category_frame.pack(fill=tk.X, pady=10)

    category_var = tk.StringVar(value=current_category)
    category_combo = ttk.Combobox(
        category_frame,
        textvariable=category_var,
        values=["ضباط", "أفراد", "مشتركة", "مدنيين"],
        state="readonly",
        width=20,
        font=self.fonts["text"]
    )
    category_combo.pack(pady=5)

    # إطار لتاريخ نهاية الدورة في النظام
    system_end_date_frame = tk.LabelFrame(
        content_frame,
        text="تاريخ نهاية الدورة في النظام",
        font=self.fonts["text_bold"],
        bg=self.colors["light"],
        padx=10, pady=10
    )
    system_end_date_frame.pack(fill=tk.X, pady=10)

    system_end_date_entry = DateEntry(
        system_end_date_frame,
        width=15,
        background=self.colors["primary"],
        foreground='white',
        borderwidth=2,
        date_pattern='yyyy-mm-dd',
        font=self.fonts["text"],
        firstweekday="sunday"
    )
    system_end_date_entry.pack(pady=5)

    # تعيين التاريخ الحالي إذا كان موجوداً
    if current_end_date_system:
        try:
            date_obj = datetime.datetime.strptime(current_end_date_system, "%Y-%m-%d")
            system_end_date_entry.set_date(date_obj)
        except:
            pass

    # إضافة إطار لتاريخ بداية الدورة
    start_date_frame = tk.LabelFrame(
        content_frame,
        text="تاريخ بداية الدورة (للعرض)",
        font=self.fonts["text_bold"],
        bg=self.colors["light"],
        padx=10, pady=10
    )
    start_date_frame.pack(fill=tk.X, pady=10)

    # حقول تاريخ البداية
    start_date_fields = tk.Frame(start_date_frame, bg=self.colors["light"])
    start_date_fields.pack(fill=tk.X)

    # اليوم
    tk.Label(start_date_fields, text="اليوم:", font=self.fonts["text"], bg=self.colors["light"]).pack(side=tk.RIGHT,
                                                                                                      padx=5)
    start_day_entry = tk.Entry(start_date_fields, font=self.fonts["text"], width=5)
    start_day_entry.insert(0, current_dates[0] if current_dates[0] else "")
    start_day_entry.pack(side=tk.RIGHT, padx=5)

    # الشهر
    tk.Label(start_date_fields, text="الشهر:", font=self.fonts["text"], bg=self.colors["light"]).pack(side=tk.RIGHT,
                                                                                                      padx=5)
    start_month_entry = tk.Entry(start_date_fields, font=self.fonts["text"], width=5)
    start_month_entry.insert(0, current_dates[1] if current_dates[1] else "")
    start_month_entry.pack(side=tk.RIGHT, padx=5)

    # السنة
    tk.Label(start_date_fields, text="السنة:", font=self.fonts["text"], bg=self.colors["light"]).pack(side=tk.RIGHT,
                                                                                                      padx=5)
    start_year_entry = tk.Entry(start_date_fields, font=self.fonts["text"], width=8)
    start_year_entry.insert(0, current_dates[2] if current_dates[2] else "")
    start_year_entry.pack(side=tk.RIGHT, padx=5)

    # إطار لتاريخ نهاية الدورة
    end_date_frame = tk.LabelFrame(
        content_frame,
        text="تاريخ نهاية الدورة (للعرض)",
        font=self.fonts["text_bold"],
        bg=self.colors["light"],
        padx=10, pady=10
    )
    end_date_frame.pack(fill=tk.X, pady=10)

    # حقول تاريخ النهاية
    end_date_fields = tk.Frame(end_date_frame, bg=self.colors["light"])
    end_date_fields.pack(fill=tk.X)

    # اليوم
    tk.Label(end_date_fields, text="اليوم:", font=self.fonts["text"], bg=self.colors["light"]).pack(side=tk.RIGHT,
                                                                                                    padx=5)
    end_day_entry = tk.Entry(end_date_fields, font=self.fonts["text"], width=5)
    end_day_entry.insert(0, current_dates[3] if current_dates[3] else "")
    end_day_entry.pack(side=tk.RIGHT, padx=5)

    # الشهر
    tk.Label(end_date_fields, text="الشهر:", font=self.fonts["text"], bg=self.colors["light"]).pack(side=tk.RIGHT,
                                                                                                    padx=5)
    end_month_entry = tk.Entry(end_date_fields, font=self.fonts["text"], width=5)
    end_month_entry.insert(0, current_dates[4] if current_dates[4] else "")
    end_month_entry.pack(side=tk.RIGHT, padx=5)

    # السنة
    tk.Label(end_date_fields, text="السنة:", font=self.fonts["text"], bg=self.colors["light"]).pack(side=tk.RIGHT,
                                                                                                    padx=5)
    end_year_entry = tk.Entry(end_date_fields, font=self.fonts["text"], width=8)
    end_year_entry.insert(0, current_dates[5] if current_dates[5] else "")
    end_year_entry.pack(side=tk.RIGHT, padx=5)

    # أزرار الحفظ والإلغاء
    buttons_frame = tk.Frame(edit_window, bg=self.colors["light"], pady=20)
    buttons_frame.pack(fill=tk.X, padx=10)

    def save_changes():
        """حفظ التغييرات"""
        new_name = new_course_name_var.get().strip()
        new_category = category_var.get()
        start_day = start_day_entry.get().strip()
        start_month = start_month_entry.get().strip()
        start_year = start_year_entry.get().strip()
        end_day = end_day_entry.get().strip()
        end_month = end_month_entry.get().strip()
        end_year = end_year_entry.get().strip()
        system_end_date = system_end_date_entry.get_date().strftime("%Y-%m-%d")

        # التحقق من صحة البيانات
        if not new_name:
            messagebox.showwarning("تنبيه", "اسم الدورة لا يمكن أن يكون فارغاً")
            return

        if (start_day or start_month or start_year) and not (start_day and start_month and start_year):
            messagebox.showwarning("تنبيه", "يجب إدخال تاريخ بداية الدورة كاملاً (اليوم والشهر والسنة)")
            return

        if (end_day or end_month or end_year) and not (end_day and end_month and end_year):
            messagebox.showwarning("تنبيه", "يجب إدخال تاريخ نهاية الدورة كاملاً (اليوم والشهر والسنة)")
            return

        try:
            with self.conn:
                # إذا تم تغيير اسم الدورة
                if new_name != course_name:
                    # التحقق من عدم وجود دورة بنفس الاسم الجديد
                    cursor.execute("SELECT COUNT(*) FROM course_info WHERE course_name=?", (new_name,))
                    if cursor.fetchone()[0] > 0:
                        messagebox.showwarning("تنبيه", f"يوجد دورة أخرى بنفس الاسم '{new_name}'")
                        return

                    # تحديث اسم الدورة في جميع الجداول المرتبطة
                    # 1. جدول المتدربين
                    self.conn.execute("UPDATE trainees SET course=? WHERE course=?", (new_name, course_name))

                    # 2. جدول الحضور
                    self.conn.execute("UPDATE attendance SET course=? WHERE course=?", (new_name, course_name))

                    # 3. جدول الفصول
                    self.conn.execute("UPDATE course_sections SET course_name=? WHERE course_name=?",
                                      (new_name, course_name))

                    # 4. جدول توزيع المتدربين على الفصول
                    self.conn.execute("UPDATE student_sections SET course_name=? WHERE course_name=?",
                                      (new_name, course_name))

                    # 5. جدول معلومات الدورة
                    self.conn.execute("""
                        UPDATE course_info 
                        SET course_name=?, start_day=?, start_month=?, start_year=?, 
                            end_day=?, end_month=?, end_year=?, end_date_system=?,
                            course_category=?
                        WHERE course_name=?
                    """, (new_name, start_day, start_month, start_year, end_day, end_month, end_year,
                          system_end_date, new_category, course_name))
                else:
                    # تحديث البيانات فقط
                    self.conn.execute("""
                        UPDATE course_info 
                        SET start_day=?, start_month=?, start_year=?, 
                            end_day=?, end_month=?, end_year=?, end_date_system=?,
                            course_category=?
                        WHERE course_name=?
                    """, (start_day, start_month, start_year, end_day, end_month, end_year,
                          system_end_date, new_category, course_name))

            messagebox.showinfo("نجاح", "تم حفظ التغييرات بنجاح")
            edit_window.destroy()

            # تحديث البيانات المعروضة
            self.update_statistics()
            self.update_students_tree()
            self.update_attendance_display()

        except Exception as e:
            messagebox.showerror("خطأ", f"حدث خطأ أثناء حفظ التغييرات: {str(e)}")

    save_btn = tk.Button(
        buttons_frame,
        text="حفظ التغييرات",
        font=self.fonts["text_bold"],
        bg=self.colors["success"],
        fg="white",
        padx=15, pady=5,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=save_changes
    )
    save_btn.pack(side=tk.LEFT, padx=10)

    cancel_btn = tk.Button(
        buttons_frame,
        text="إلغاء",
        font=self.fonts["text_bold"],
        bg=self.colors["danger"],
        fg="white",
        padx=15, pady=5,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=edit_window.destroy
    )
    cancel_btn.pack(side=tk.RIGHT, padx=10)

def toggle_student_exclusion(self, national_id, exclude, profile_window=None):
    """إضافة أو إزالة استبعاد المتدرب"""
    if not self.current_user["permissions"]["can_edit_students"]:
        messagebox.showwarning("تنبيه", "ليس لديك صلاحية استبعاد المتدربين")
        return

    cursor = self.conn.cursor()
    cursor.execute("SELECT name, is_excluded FROM trainees WHERE national_id=?", (national_id,))
    student = cursor.fetchone()

    if not student:
        messagebox.showwarning("تنبيه", "لم يتم العثور على المتدرب")
        return

    student_name, current_excluded = student

    if exclude and current_excluded == 1:
        messagebox.showinfo("تنبيه", "هذا المتدرب مستبعد بالفعل")
        return

    if not exclude and current_excluded == 0:
        messagebox.showinfo("تنبيه", "هذا المتدرب غير مستبعد بالفعل")
        return

    if exclude:
        reason = simpledialog.askstring("سبب الاستبعاد", "أدخل سبب استبعاد المتدرب:")
        if reason is None:  # الضغط على إلغاء
            return

        current_date = datetime.datetime.now().strftime("%Y-%m-%d")

        try:
            with self.conn:
                self.conn.execute("""
                    UPDATE trainees 
                    SET is_excluded=1, exclusion_reason=?, excluded_date=? 
                    WHERE national_id=?
                """, (reason, current_date, national_id))

            messagebox.showinfo("نجاح", f"تم استبعاد المتدرب {student_name} بنجاح")
        except Exception as e:
            messagebox.showerror("خطأ", f"حدث خطأ أثناء استبعاد المتدرب: {str(e)}")
            return
    else:
        try:
            with self.conn:
                self.conn.execute("""
                    UPDATE trainees 
                    SET is_excluded=0, exclusion_reason='', excluded_date='' 
                    WHERE national_id=?
                """, (national_id,))

            messagebox.showinfo("نجاح", f"تم إلغاء استبعاد المتدرب {student_name} بنجاح")
        except Exception as e:
            messagebox.showerror("خطأ", f"حدث خطأ أثناء إلغاء استبعاد المتدرب: {str(e)}")
            return

    # تحديث الإحصائيات والبيانات
    self.update_students_tree()
    self.update_statistics()
    self.update_attendance_display()

    # إغلاق نافذة ملف المتدرب إذا كانت مفتوحة وإعادة فتحها لتعكس التغييرات
    if profile_window:
        profile_window.destroy()
        self.view_student_profile()

    def export_course_completion(self):
        """وظيفة تصدير مستند تكميل الدورات بتنسيق Word مع ملخص للدورات في الصفحة الأولى مع إضافة تواريخ بداية ونهاية الدورة وفئة الدورة"""
        if not self.current_user["permissions"]["can_export_data"]:
            messagebox.showwarning("تنبيه", "ليس لديك صلاحية تصدير البيانات")
            return

        try:
            # التأكد من وجود مكتبة python-docx
            if 'Document' not in globals():
                messagebox.showerror("خطأ",
                                     "لم يتم العثور على مكتبة python-docx. قم بتثبيتها باستخدام: pip install python-docx")
                return

            # الحصول على تاريخ اليوم المحدد
            selected_date = self.log_date_entry.get_date()
            selected_date_str = selected_date.strftime("%Y-%m-%d")
            # تنسيق التاريخ بشكل أفضل للعرض (يوم/شهر/سنة)
            arabic_date = selected_date.strftime("%d/%m/%Y")

            # تحديد يوم الأسبوع بالعربية
            weekday = selected_date.weekday()
            arabic_weekdays = ["الاثنين", "الثلاثاء", "الأربعاء", "الخميس", "الجمعة", "السبت", "الأحد"]
            arabic_weekday = arabic_weekdays[weekday]

            # استعلام إجمالي عدد المتدربين في النظام (غير المستبعدين)
            cursor = self.conn.cursor()
            cursor.execute("""
                SELECT COUNT(*)
                FROM trainees
                WHERE is_excluded=0
            """)
            total_students_count = cursor.fetchone()[0]

            # الحصول على عدد المتدربين غير المسجلين بعد
            cursor.execute("""
                SELECT COUNT(*)
                FROM trainees t
                WHERE t.is_excluded=0 AND NOT EXISTS (
                    SELECT 1 FROM attendance a
                    WHERE a.national_id = t.national_id AND a.date = ?
                )
            """, (selected_date_str,))

            unrecorded_count = cursor.fetchone()[0]

            # التحقق من عدم وجود متدربين غير مسجلين
            if unrecorded_count > 0:
                messagebox.showwarning("تنبيه",
                                       f"لا يمكن تصدير التكميل، هناك {unrecorded_count} متدرب لم يتم تسجيل حضورهم/غيابهم بعد.")
                return

            # استعلام بيانات المتدربين الغائبين والذين لم يباشروا والغائبين بعذر وجميع الحالات الأخرى
            cursor.execute("""
                SELECT a.national_id, a.name, a.rank, a.course, a.status, a.excuse_reason
                FROM attendance a
                JOIN trainees t ON a.national_id = t.national_id
                WHERE a.date=? AND t.is_excluded=0 AND a.status IN ('غائب', 'لم يباشر', 'غائب بعذر', 'متأخر', 'تطبيق ميداني', 'يوم طالب', 'مسائية / عن بعد', 'حالة وفاة', 'منوم')
                ORDER BY a.course, a.name
            """, (selected_date_str,))
            all_attendance_data = cursor.fetchall()

            if not all_attendance_data and total_students_count == 0:
                messagebox.showinfo("ملاحظة", "لا توجد بيانات غياب أو متدربين في النظام لهذا اليوم.")
                return

            # الحصول على إحصائيات الدورات مع فئة الدورة وتاريخ النهاية
            cursor.execute("""
                SELECT 
                    t.course,
                    COUNT(DISTINCT t.national_id) as total_course_students,
                    COUNT(DISTINCT CASE WHEN a.status = 'حاضر' THEN a.national_id END) as present_count,
                    COUNT(DISTINCT CASE WHEN a.status = 'غائب' THEN a.national_id END) as absent_count,
                    COUNT(DISTINCT CASE WHEN a.status = 'غائب بعذر' THEN a.national_id END) as excused_count,
                    COUNT(DISTINCT CASE WHEN a.status = 'متأخر' THEN a.national_id END) as late_count,
                    COUNT(DISTINCT CASE WHEN a.status = 'لم يباشر' THEN a.national_id END) as not_started_count,
                    COUNT(DISTINCT CASE WHEN a.status = 'تطبيق ميداني' THEN a.national_id END) as field_app_count,
                    COUNT(DISTINCT CASE WHEN a.status = 'يوم طالب' THEN a.national_id END) as student_day_count,
                    COUNT(DISTINCT CASE WHEN a.status = 'مسائية / عن بعد' THEN a.national_id END) as evening_remote_count,
                    COUNT(DISTINCT CASE WHEN a.status = 'حالة وفاة' THEN a.national_id END) as death_case_count,
                    COUNT(DISTINCT CASE WHEN a.status = 'منوم' THEN a.national_id END) as hospital_count,
                    COUNT(DISTINCT CASE WHEN a.status IS NOT NULL THEN a.national_id END) as recorded_count,
                    COALESCE(ci.course_category, 'مشتركة') as category,
                    ci.end_date_system
                FROM 
                    trainees t
                LEFT JOIN 
                    attendance a ON t.national_id = a.national_id AND a.date = ?
                LEFT JOIN 
                    course_info ci ON t.course = ci.course_name
                WHERE 
                    t.is_excluded = 0
                GROUP BY 
                    t.course
                ORDER BY 
                    CASE 
                        WHEN ci.end_date_system IS NULL THEN 1
                        ELSE 0
                    END,
                    ci.end_date_system DESC,
                    t.course
            """, (selected_date_str,))

            courses_stats = cursor.fetchall()

            if not courses_stats:
                messagebox.showinfo("ملاحظة", "لا توجد دورات مسجلة في النظام.")
                return

            # الحصول على المتدربين الذين لديهم غيابات أكثر من يومين حتى التاريخ المحدد
            cursor.execute("""
                SELECT t.national_id, t.name, t.rank, t.course, COUNT(*) as absence_count
                FROM trainees t
                JOIN attendance a ON t.national_id = a.national_id
                WHERE a.status = 'غائب' AND t.is_excluded = 0 AND a.date <= ?
                GROUP BY t.national_id, t.name, t.rank, t.course
                HAVING COUNT(*) > 2
                ORDER BY t.course, absence_count DESC, t.name
            """, (selected_date_str,))
            multiple_absences_data = cursor.fetchall()

            # الحصول على بيانات المستبعدين من جميع الدورات
            today_date = datetime.datetime.now().strftime("%d/%m/%Y")
            cursor.execute("""
                SELECT name, rank, national_id, course, exclusion_reason, excluded_date
                FROM trainees
                WHERE is_excluded = 1
                ORDER BY course, excluded_date DESC, name
            """)
            excluded_students_data = cursor.fetchall()

            # تصنيف البيانات حسب الحالة (مرتبة حسب الدورة)
            absent_data = sorted([student for student in all_attendance_data if student[4] == 'غائب'],
                                 key=lambda x: (x[3], x[1]))  # ترتيب حسب الدورة ثم الاسم
            not_started_data = sorted([student for student in all_attendance_data if student[4] == 'لم يباشر'],
                                      key=lambda x: (x[3], x[1]))
            excused_data = sorted([student for student in all_attendance_data if student[4] == 'غائب بعذر'],
                                  key=lambda x: (x[3], x[1]))
            late_data = sorted([student for student in all_attendance_data if student[4] == 'متأخر'],
                               key=lambda x: (x[3], x[1]))
            field_app_data = sorted([student for student in all_attendance_data if student[4] == 'تطبيق ميداني'],
                                    key=lambda x: (x[3], x[1]))
            student_day_data = sorted([student for student in all_attendance_data if student[4] == 'يوم طالب'],
                                      key=lambda x: (x[3], x[1]))
            evening_remote_data = sorted(
                [student for student in all_attendance_data if student[4] == 'مسائية / عن بعد'],
                key=lambda x: (x[3], x[1]))
            death_case_data = sorted([student for student in all_attendance_data if student[4] == 'حالة وفاة'],
                                     key=lambda x: (x[3], x[1]))
            hospital_data = sorted([student for student in all_attendance_data if student[4] == 'منوم'],
                                   key=lambda x: (x[3], x[1]))

            # استعلام إحصائيات الحضور الإجمالية لهذا اليوم
            cursor.execute("""
                SELECT 
                    COUNT(CASE WHEN a.status = 'حاضر' THEN 1 END) as present_count,
                    COUNT(CASE WHEN a.status = 'غائب' THEN 1 END) as absent_count,
                    COUNT(CASE WHEN a.status = 'غائب بعذر' THEN 1 END) as excused_count,
                    COUNT(CASE WHEN a.status = 'متأخر' THEN 1 END) as late_count,
                    COUNT(CASE WHEN a.status = 'لم يباشر' THEN 1 END) as not_started_count,
                    COUNT(CASE WHEN a.status = 'تطبيق ميداني' THEN 1 END) as field_app_count,
                    COUNT(CASE WHEN a.status = 'يوم طالب' THEN 1 END) as student_day_count,
                    COUNT(CASE WHEN a.status = 'مسائية / عن بعد' THEN 1 END) as evening_remote_count,
                    COUNT(CASE WHEN a.status = 'حالة وفاة' THEN 1 END) as death_case_count,
                    COUNT(CASE WHEN a.status = 'منوم' THEN 1 END) as hospital_count,
                    COUNT(*) as total_recorded_count
                FROM attendance a
                JOIN trainees t ON a.national_id = t.national_id
                WHERE a.date=? AND t.is_excluded=0
            """, (selected_date_str,))
            stats = cursor.fetchone()

            # إذا لم تتوفر إحصائيات
            if not stats:
                stats = (0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0)

            # أعداد الحضور والغياب
            present_count = stats[0] or 0
            absent_count = stats[1] or 0
            excused_count = stats[2] or 0
            late_count = stats[3] or 0
            not_started_count = stats[4] or 0
            field_app_count = stats[5] or 0
            student_day_count = stats[6] or 0
            evening_remote_count = stats[7] or 0
            death_case_count = stats[8] or 0
            hospital_count = stats[9] or 0
            total_recorded_count = stats[10] or 0

            # إنشاء مستند Word جديد
            doc = Document()

            # إعداد صفحة المستند
            section = doc.sections[0]
            section.page_width = Inches(11.69)  # A4 width in landscape
            section.page_height = Inches(8.27)  # A4 height in landscape
            section.orientation = WD_ORIENTATION.LANDSCAPE
            section.left_margin = Inches(0.5)
            section.right_margin = Inches(0.5)
            section.top_margin = Inches(0.7)
            section.bottom_margin = Inches(0.7)

            # ============== الصفحة الأولى: ملخص الدورات ==============
            # إضافة العنوان المعدل
            title = doc.add_heading(
                f'التكميل اليومي لدورات التخصصية المنعقدة بمدينة تدريب الامن العام بالمنطقة الشرقية ليوم {arabic_weekday} بتاريخ {arabic_date}',
                level=0)
            title.alignment = WD_ALIGN_PARAGRAPH.CENTER
            for run in title.runs:
                run.font.rtl = True
                run.font.bold = True
                run.font.size = Pt(16)

            # إضافة خط أفقي فاصل
            border_para = doc.add_paragraph()
            border_para.paragraph_format.border_bottom = True

            # إضافة فقرة اسم الجهة
            dept_para = doc.add_paragraph()
            dept_para.alignment = WD_ALIGN_PARAGRAPH.CENTER
            dept_run = dept_para.add_run("قسم شؤون المدربين")
            dept_run.font.rtl = True
            dept_run.font.bold = True
            dept_run.font.size = Pt(14)

            # إضافة فقرة فاصلة
            doc.add_paragraph()

            # حساب عدد الأعمدة المطلوبة بناءً على الحالات الموجودة
            show_field_app = any(course[7] > 0 for course in courses_stats)
            show_student_day = any(course[8] > 0 for course in courses_stats)
            show_evening_remote = any(course[9] > 0 for course in courses_stats)
            show_death_case = any(course[10] > 0 for course in courses_stats)
            show_hospital = any(course[11] > 0 for course in courses_stats)

            # 10 أعمدة ثابتة مع إضافة عمود فئة الدورة
            cols_count = 10  # العدد، الدورة، فئة الدورة، بداية، نهاية، القوة، لم يباشر، غياب، تأخير، غياب بعذر

            # إضافة أعمدة إضافية للحالات الموجودة فقط
            if show_field_app:
                cols_count += 1
            if show_student_day:
                cols_count += 1
            if show_evening_remote:
                cols_count += 1
            if show_death_case:
                cols_count += 1
            if show_hospital:
                cols_count += 1

            # إضافة عمود العدد الفعلي دائماً
            cols_count += 1

            # إنشاء جدول ملخص الدورات في الصفحة الأولى
            courses_table = doc.add_table(rows=1, cols=cols_count)
            courses_table.style = 'Table Grid'

            # إنشاء قائمة العناوين
            headers = [
                "العدد", "اسم الدورة", "فئة الدورة",
                "تاريخ بداية الدورة", "تاريخ نهاية الدورة",
                "القوة", "لم يباشر", "غياب", "تأخير", "غياب بعذر"
            ]

            # إضافة عناوين الحالات الموجودة فقط
            if show_field_app:
                headers.append("تطبيق ميداني")
            if show_student_day:
                headers.append("يوم طالب")
            if show_evening_remote:
                headers.append("مسائية / عن بعد")
            if show_death_case:
                headers.append("حالة وفاة")
            if show_hospital:
                headers.append("منوم")

            # إضافة عنوان العدد الفعلي دائماً
            headers.append("العدد الفعلي")

            # عناوين جدول الدورات
            header_cells = courses_table.rows[0].cells

            for i, header in enumerate(headers):
                # حساب الموقع المناسب للعناوين (من اليمين إلى اليسار بسبب RTL)
                idx = len(headers) - i - 1
                header_cells[idx].text = header
                header_cells[idx].paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.CENTER

                for run in header_cells[idx].paragraphs[0].runs:
                    run.font.bold = True
                    run.font.rtl = True
                    run.font.size = Pt(11)

                # تطبيق تظليل للرأس
                try:
                    shading_elm = parse_xml(r'<w:shd {} w:fill="DDDDDD"/>'.format(nsdecls('w')))
                    header_cells[idx]._element.get_or_add_tcPr().append(shading_elm)
                except:
                    pass

            # متغيرات لحساب المجاميع
            total_strength = 0
            total_not_started = 0
            total_absent = 0
            total_late = 0
            total_excused = 0
            total_field_app = 0
            total_student_day = 0
            total_evening_remote = 0
            total_death_case = 0
            total_hospital = 0
            total_effective = 0

            # إضافة بيانات الدورات إلى الجدول
            for idx, course_data in enumerate(courses_stats):
                course_name = course_data[0]
                total_course_students = course_data[1] or 0
                present_count = course_data[2] or 0
                absent_count = course_data[3] or 0
                excused_count = course_data[4] or 0
                late_count = course_data[5] or 0
                not_started_count = course_data[6] or 0
                field_app_count = course_data[7] or 0
                student_day_count = course_data[8] or 0
                evening_remote_count = course_data[9] or 0
                death_case_count = course_data[10] or 0
                hospital_count = course_data[11] or 0
                recorded_count = course_data[12] or 0
                course_category = course_data[13] or "مشتركة"
                end_date_system = course_data[14]

                # تحديث المجاميع
                total_strength += total_course_students
                total_not_started += not_started_count
                total_absent += absent_count
                total_late += late_count
                total_excused += excused_count
                total_field_app += field_app_count
                total_student_day += student_day_count
                total_evening_remote += evening_remote_count
                total_death_case += death_case_count
                total_hospital += hospital_count

                # الحصول على تواريخ البداية والنهاية للدورة من قاعدة البيانات
                cursor.execute("""
                    SELECT start_day, start_month, start_year, end_day, end_month, end_year
                    FROM course_info
                    WHERE course_name=?
                """, (course_name,))
                date_info = cursor.fetchone()

                start_date_str = ""
                end_date_str = ""

                if date_info:
                    start_day, start_month, start_year, end_day, end_month, end_year = date_info
                    if start_day and start_month and start_year:
                        start_date_str = f"{start_day}/{start_month}/{start_year}"
                    if end_day and end_month and end_year:
                        end_date_str = f"{end_day}/{end_month}/{end_year}"

                # حساب العدد الفعلي (بعد خصم جميع الحالات المذكورة)
                effective_count = present_count
                total_effective += effective_count

                row_cells = courses_table.add_row().cells

                # تحضير القيم للإدخال في الجدول
                values = [
                    str(idx + 1),  # العدد التسلسلي للدورة
                    course_name,  # اسم الدورة
                    course_category,  # فئة الدورة
                    start_date_str,  # تاريخ بداية الدورة
                    end_date_str,  # تاريخ نهاية الدورة
                    str(total_course_students),  # القوة
                    str(not_started_count),  # عدد حالات "لم يباشر"
                    str(absent_count),  # عدد حالات الغياب
                    str(late_count),  # عدد حالات التأخير
                    str(excused_count)  # عدد حالات الغياب بعذر
                ]

                # إضافة القيم للحالات الموجودة فقط
                if show_field_app:
                    values.append(str(field_app_count))
                if show_student_day:
                    values.append(str(student_day_count))
                if show_evening_remote:
                    values.append(str(evening_remote_count))
                if show_death_case:
                    values.append(str(death_case_count))
                if show_hospital:
                    values.append(str(hospital_count))

                # إضافة العدد الفعلي دائماً
                values.append(str(effective_count))

                # إدخال القيم في الجدول بالترتيب من اليمين إلى اليسار
                for i, value in enumerate(values):
                    idx = len(values) - i - 1
                    row_cells[idx].text = value

                # تنسيق الخلايا
                for cell in row_cells:
                    cell.paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.CENTER
                    for run in cell.paragraphs[0].runs:
                        run.font.rtl = True
                        run.font.size = Pt(10)

            # إضافة صف المجموع
            total_row_cells = courses_table.add_row().cells

            # إعداد قيم صف المجموع
            total_values = [
                "",  # العدد التسلسلي (فارغ)
                "المجموع",  # اسم الدورة
                "",  # فئة الدورة (فارغ)
                "",  # تاريخ بداية (فارغ)
                "",  # تاريخ نهاية (فارغ)
                str(total_strength),  # إجمالي القوة
                str(total_not_started),  # إجمالي لم يباشر
                str(total_absent),  # إجمالي الغياب
                str(total_late),  # إجمالي التأخير
                str(total_excused)  # إجمالي الغياب بعذر
            ]

            # إضافة المجاميع للحالات الموجودة فقط
            if show_field_app:
                total_values.append(str(total_field_app))
            if show_student_day:
                total_values.append(str(total_student_day))
            if show_evening_remote:
                total_values.append(str(total_evening_remote))
            if show_death_case:
                total_values.append(str(total_death_case))
            if show_hospital:
                total_values.append(str(total_hospital))

            # إضافة إجمالي العدد الفعلي
            total_values.append(str(total_effective))

            # إدخال قيم المجموع في الجدول
            for i, value in enumerate(total_values):
                idx = len(total_values) - i - 1
                total_row_cells[idx].text = value

            # تنسيق صف المجموع بشكل مميز
            for cell in total_row_cells:
                cell.paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.CENTER
                for run in cell.paragraphs[0].runs:
                    run.font.rtl = True
                    run.font.size = Pt(11)
                    run.font.bold = True

                # تطبيق تظليل لصف المجموع
                try:
                    shading_elm = parse_xml(r'<w:shd {} w:fill="E0E0E0"/>'.format(nsdecls('w')))
                    cell._element.get_or_add_tcPr().append(shading_elm)
                except:
                    pass

            # ============== صفحة جديدة للتفاصيل ==============
            doc.add_page_break()

            # دالة مساعدة لإضافة جدول المتدربين (معدلة لإضافة اليوم والتاريخ في العنوان)
            def add_students_table(title, students_data, has_reason=False, is_excluded=False,
                                   include_date_in_title=True, has_course_column=False):
                """إضافة جدول المتدربين لحالة معينة"""
                if not students_data:
                    return  # تخطي إذا لم تكن هناك بيانات

                # إضافة عنوان الجدول مع التاريخ
                if include_date_in_title and not is_excluded:
                    # إضافة اليوم والتاريخ في العنوان
                    full_title = f"{title} ليوم {arabic_weekday} بتاريخ {arabic_date}"
                else:
                    full_title = title

                title_para = doc.add_heading(full_title, level=2)
                title_para.alignment = WD_ALIGN_PARAGRAPH.CENTER
                for run in title_para.runs:
                    run.font.rtl = True
                    run.font.bold = True
                    run.font.size = Pt(14)

                # إنشاء جدول المتدربين
                if is_excluded:
                    cols_count = 7  # للمستبعدين: العدد، الاسم، الرتبة، رقم الهوية، الدورة، سبب الاستبعاد، تاريخ الاستبعاد
                else:
                    cols_count = 6 if has_reason else 5  # إضافة عمود للسبب عند الحاجة

                students_table = doc.add_table(rows=1, cols=cols_count)
                students_table.style = 'Table Grid'

                # تحديد عناوين الجدول
                if is_excluded:
                    headers = ["العدد", "الاسم", "الرتبة", "رقم الهوية", "الدورة", "سبب الاستبعاد", "تاريخ الاستبعاد"]
                else:
                    headers = ["العدد", "الاسم", "الرتبة", "رقم الهوية", "الدورة"]
                    if has_reason:
                        headers.append("السبب")  # إضافة عمود السبب إذا كان مطلوباً

                header_cells = students_table.rows[0].cells
                for i, header in enumerate(headers):
                    idx = len(headers) - i - 1  # لترتيب العناوين من اليمين لليسار
                    header_cells[idx].text = header
                    header_cells[idx].paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.CENTER

                    for run in header_cells[idx].paragraphs[0].runs:
                        run.font.bold = True
                        run.font.rtl = True
                        run.font.size = Pt(12)

                    # إضافة تظليل للعناوين
                    try:
                        shading_elm = parse_xml(r'<w:shd {} w:fill="DDDDDD"/>'.format(nsdecls('w')))
                        header_cells[idx]._element.get_or_add_tcPr().append(shading_elm)
                    except:
                        pass

                # إضافة بيانات المتدربين
                for idx, student in enumerate(students_data):
                    row_cells = students_table.add_row().cells

                    if is_excluded:
                        # للمستبعدين: name, rank, national_id, course, exclusion_reason, excluded_date
                        name, rank, national_id, course, exclusion_reason, excluded_date = student

                        row_cells[6].text = str(idx + 1)  # العدد التسلسلي
                        row_cells[5].text = name  # الاسم
                        row_cells[4].text = rank  # الرتبة
                        row_cells[3].text = national_id  # رقم الهوية
                        row_cells[2].text = course  # اسم الدورة
                        row_cells[1].text = exclusion_reason if exclusion_reason else "لم يحدد سبب"  # سبب الاستبعاد
                        row_cells[0].text = excluded_date if excluded_date else ""  # تاريخ الاستبعاد
                    else:
                        # للحالات الأخرى
                        national_id, name, rank, course, status, reason = student

                        if has_reason:
                            row_cells[5].text = str(idx + 1)  # العدد التسلسلي
                            row_cells[4].text = name  # الاسم
                            row_cells[3].text = rank  # الرتبة
                            row_cells[2].text = national_id  # رقم الهوية
                            row_cells[1].text = course  # اسم الدورة
                            row_cells[0].text = reason if reason else "لم يحدد سبب"  # السبب
                        else:
                            row_cells[4].text = str(idx + 1)  # العدد التسلسلي
                            row_cells[3].text = name  # الاسم
                            row_cells[2].text = rank  # الرتبة
                            row_cells[1].text = national_id  # رقم الهوية
                            row_cells[0].text = course  # اسم الدورة

                    # تنسيق خلايا البيانات
                    for cell in row_cells:
                        cell.paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.CENTER
                        for run in cell.paragraphs[0].runs:
                            run.font.rtl = True
                            run.font.size = Pt(11)

                # إضافة إجمالي عدد المتدربين
                summary_para = doc.add_paragraph()
                summary_para.alignment = WD_ALIGN_PARAGRAPH.LEFT
                summary_text = summary_para.add_run(f"إجمالي عدد المتدربين: {len(students_data)}")
                summary_text.font.bold = True
                summary_text.font.rtl = True
                summary_text.font.size = Pt(12)

                # إضافة فراغ بعد الجدول
                doc.add_paragraph()

            # إضافة جداول المتدربين في نفس الصفحة
            # إضافة جدول المتدربين الغائبين
            if absent_data:
                add_students_table("بيان المتدربين الغائبين", absent_data)

            # إضافة جدول المتدربين الغائبين بعذر
            if excused_data:
                add_students_table("بيان المتدربين الغائبين بعذر", excused_data, has_reason=True)

            # إضافة جدول المتدربين المتأخرين
            if late_data:
                add_students_table("بيان المتدربين المتأخرين", late_data)

            # إضافة جدول المتدربين في حالة "لم يباشر"
            if not_started_data:
                add_students_table("بيان المتدربين في حالة لم يباشر", not_started_data)

            # إضافة جدول حالات الوفاة
            if death_case_data:
                add_students_table("بيان المتدربين في حالة وفاة", death_case_data, has_reason=True)

            # إضافة جدول المتدربين المنومين
            if hospital_data:
                add_students_table("بيان المتدربين المنومين", hospital_data, has_reason=True)

            # صفحة جديدة للغيابات المتكررة والمستبعدين
            if multiple_absences_data or excluded_students_data:
                doc.add_page_break()

            # إضافة جدول المتدربين الذين لديهم غيابات أكثر من يومين
            if multiple_absences_data:
                title_para = doc.add_heading(
                    f"بيان المتدربين الذين لديهم غيابات متكررة (أكثر من يومين) حتى تاريخ {arabic_date}", level=2)
                title_para.alignment = WD_ALIGN_PARAGRAPH.CENTER
                for run in title_para.runs:
                    run.font.rtl = True
                    run.font.bold = True
                    run.font.size = Pt(14)

                # إنشاء جدول للغيابات المتكررة
                absence_table = doc.add_table(rows=1, cols=6)
                absence_table.style = 'Table Grid'

                # عناوين الجدول
                headers = ["العدد", "الاسم", "الرتبة", "رقم الهوية", "الدورة", "عدد أيام الغياب"]
                header_cells = absence_table.rows[0].cells

                for i, header in enumerate(headers):
                    idx = len(headers) - i - 1
                    header_cells[idx].text = header
                    header_cells[idx].paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.CENTER

                    for run in header_cells[idx].paragraphs[0].runs:
                        run.font.bold = True
                        run.font.rtl = True
                        run.font.size = Pt(12)

                    # إضافة تظليل للعناوين
                    try:
                        shading_elm = parse_xml(r'<w:shd {} w:fill="FFDDDD"/>'.format(nsdecls('w')))
                        header_cells[idx]._element.get_or_add_tcPr().append(shading_elm)
                    except:
                        pass

                # إضافة بيانات المتدربين
                for idx, (national_id, name, rank, course, absence_count) in enumerate(multiple_absences_data):
                    row_cells = absence_table.add_row().cells

                    row_cells[5].text = str(idx + 1)  # العدد التسلسلي
                    row_cells[4].text = name  # الاسم
                    row_cells[3].text = rank  # الرتبة
                    row_cells[2].text = national_id  # رقم الهوية
                    row_cells[1].text = course  # اسم الدورة
                    row_cells[0].text = str(absence_count)  # عدد أيام الغياب

                    # تنسيق خلايا البيانات
                    for cell in row_cells:
                        cell.paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.CENTER
                        for run in cell.paragraphs[0].runs:
                            run.font.rtl = True
                            run.font.size = Pt(11)

                # إضافة إجمالي عدد المتدربين
                summary_para = doc.add_paragraph()
                summary_para.alignment = WD_ALIGN_PARAGRAPH.LEFT
                summary_text = summary_para.add_run(
                    f"إجمالي عدد المتدربين ذوي الغيابات المتكررة: {len(multiple_absences_data)}")
                summary_text.font.bold = True
                summary_text.font.rtl = True
                summary_text.font.size = Pt(12)

                doc.add_paragraph()

            # إضافة جدول المستبعدين من جميع الدورات في نفس الصفحة
            if excluded_students_data:
                title_para = doc.add_heading(f"المستبعدين من جميع الدورات المنعقدة حتى تاريخ {today_date}", level=2)
                title_para.alignment = WD_ALIGN_PARAGRAPH.CENTER
                for run in title_para.runs:
                    run.font.rtl = True
                    run.font.bold = True
                    run.font.size = Pt(14)

                add_students_table("", excluded_students_data, is_excluded=True, include_date_in_title=False)

            # إضافة التوقيعات في نهاية المستند فقط
            doc.add_paragraph()
            doc.add_paragraph()

            signatures_table = doc.add_table(rows=1, cols=3)
            signatures_table.style = 'Table Grid'

            sig_cells = signatures_table.rows[0].cells
            sig_cells[2].text = "اسم المراقب: ____________"
            sig_cells[1].text = "رئيس قسم الفصول: ______________"
            sig_cells[0].text = "مدير قسم شؤون المدربين: _____________"

            for cell in sig_cells:
                cell.paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.CENTER
                for run in cell.paragraphs[0].runs:
                    run.font.rtl = True
                    run.font.size = Pt(11)

            # حفظ المستند
            export_file = filedialog.asksaveasfilename(
                defaultextension=".docx",
                filetypes=[("Word documents", "*.docx")],
                initialfile=f"تكميل_الدورات_{selected_date_str}.docx"
            )

            if export_file:
                doc.save(export_file)
                messagebox.showinfo("نجاح", f"تم تصدير تكميل الدورات بنجاح إلى:\n{export_file}")
                # محاولة فتح الملف تلقائيًا
                try:
                    os.startfile(export_file)
                except:
                    pass

        except Exception as e:
            messagebox.showerror("خطأ", f"حدث خطأ أثناء تصدير البيانات: {str(e)}")

def update_statistics(self):
    cursor = self.conn.cursor()
    # احتساب عدد المتدربين غير المستبعدين فقط
    cursor.execute("SELECT COUNT(*) FROM trainees WHERE is_excluded=0")
    total_students = cursor.fetchone()[0]
    self.total_students_var.set(str(total_students))

    date_str = self.date_entry.get_date().strftime("%Y-%m-%d")

    # استعلام محسّن: استخدام GROUP BY والدالة المجمعة COUNT مع CASE
    cursor.execute("""
        SELECT 
            COALESCE(SUM(CASE WHEN a.status = 'حاضر' THEN 1 ELSE 0 END), 0) as present_count,
            COALESCE(SUM(CASE WHEN a.status = 'غائب' THEN 1 ELSE 0 END), 0) as absent_count,
            COALESCE(SUM(CASE WHEN a.status = 'متأخر' THEN 1 ELSE 0 END), 0) as late_count,
            COALESCE(SUM(CASE WHEN a.status = 'غائب بعذر' THEN 1 ELSE 0 END), 0) as excused_count,
            COALESCE(SUM(CASE WHEN a.status = 'لم يباشر' THEN 1 ELSE 0 END), 0) as not_started_count,
            COALESCE(SUM(CASE WHEN a.status = 'تطبيق ميداني' THEN 1 ELSE 0 END), 0) as field_app_count,
            COALESCE(SUM(CASE WHEN a.status = 'يوم طالب' THEN 1 ELSE 0 END), 0) as student_day_count,
            COALESCE(SUM(CASE WHEN a.status = 'مسائية / عن بعد' THEN 1 ELSE 0 END), 0) as evening_remote_count,
            COALESCE(SUM(CASE WHEN a.status = 'حالة وفاة' THEN 1 ELSE 0 END), 0) as death_case_count,
            COALESCE(SUM(CASE WHEN a.status = 'منوم' THEN 1 ELSE 0 END), 0) as hospital_count
        FROM attendance a
        JOIN trainees t ON a.national_id = t.national_id
        WHERE a.date=? AND t.is_excluded=0
    """, (date_str,))

    result = cursor.fetchone()

    # تحديث متغيرات العرض
    self.present_students_var.set(str(result[0]))
    self.absent_students_var.set(str(result[1]))
    self.late_students_var.set(str(result[2]))
    self.excused_students_var.set(str(result[3]))
    self.not_started_students_var.set(str(result[4]))
    self.field_application_var.set(str(result[5]))
    self.student_day_var.set(str(result[6]))
    self.evening_remote_var.set(str(result[7]))
    self.death_case_var.set(str(result[8]))
    self.hospital_var.set(str(result[9]))

    # حساب نسبة الحضور
    if total_students > 0:
        # إضافة الحالات الجديدة لحساب نسبة الحضور
        attendance_rate = ((result[0] + result[2] + result[5] + result[6] + result[7]) / total_students) * 100
    else:
        attendance_rate = 0.0

    self.attendance_rate_var.set(f"{attendance_rate:.2f}%")

def check_student_absence(self, national_id, current_date):
    """
    فحص حالة غياب المتدرب ورصد الغياب المتكرر
    يتم استدعاء هذه الدالة عند تسجيل غياب جديد للمتدرب

    المدخلات:
        national_id: رقم هوية المتدرب
        current_date: تاريخ اليوم الحالي بتنسيق YYYY-MM-DD

    المخرجات:
        tuple(bool, str): الأول يشير إلى وجود غياب متكرر، والثاني هو نص رسالة التنبيه
    """
    cursor = self.conn.cursor()

    # الحصول على معلومات المتدرب
    cursor.execute("""
        SELECT name, rank, course 
        FROM trainees 
        WHERE national_id=?
    """, (national_id,))

    student_info = cursor.fetchone()
    if not student_info:
        return False, ""

    student_name, student_rank, student_course = student_info

    # تحويل التاريخ الحالي إلى كائن تاريخ
    current_date_obj = datetime.datetime.strptime(current_date, "%Y-%m-%d").date()

    # الحصول على تواريخ حضور المتدرب في الأيام السابقة (دون اليوم الحالي)
    cursor.execute("""
        SELECT date, status 
        FROM attendance 
        WHERE national_id=? AND date < ? 
        ORDER BY date DESC
    """, (national_id, current_date))

    attendance_records = cursor.fetchall()

    # حساب عدد أيام الغياب المتتالية
    consecutive_absences = 0

    # حساب عدد أيام الغياب الإجمالية
    total_absences = 0

    # التحقق أولاً ما إذا كان اليوم المسجل هو "غائب"
    consecutive_absences = 1  # اليوم الحالي محسوب كغياب (لأننا ندعو هذه الدالة فقط عند تسجيل غياب)
    total_absences = 1

    # معالجة سجلات الحضور السابقة
    last_date = current_date_obj
    for record in attendance_records:
        date_str, status = record
        record_date = datetime.datetime.strptime(date_str, "%Y-%m-%d").date()

        # حساب إجمالي الغياب (غائب وغائب بعذر يُحسبان كغياب للإجمالي)
        if status in ["غائب", "غائب بعذر"]:
            total_absences += 1

            # التحقق من التتابع - يجب أن يكون الفرق يوم واحد للاعتبار متتاليًا
            if (last_date - record_date).days == 1 and status == "غائب":
                consecutive_absences += 1
            else:
                # إذا كان هناك انقطاع في التتابع، نتوقف عن العد
                if status != "غائب":  # إذا كان "غائب بعذر" لا يُحسب في التتابع
                    continue
        else:
            # إذا كان المتدرب حاضرًا أو حالة أخرى، نتوقف عن عد الأيام المتتالية
            break

        last_date = record_date

    # تحديد نوع التنبيه المطلوب
    alert_message = ""
    show_alert = False

    if consecutive_absences >= 3:
        show_alert = True

        if consecutive_absences >= 4:
            # تنبيه أحمر للغياب المتكرر أكثر من 3 أيام متتالية
            alert_message = f"⚠️ تنبيه هام: المتدرب {student_name} ({student_rank}) متغيب {consecutive_absences} أيام متتالية!\n\n" \
                            f"✓ الإجراء المطلوب: رفع محاضر الغياب \nالى سعادة قائد المدينة بخطاب رسمي.\n" \
                            f"✓ الدورة: {student_course}\n" \
                            f"✓ إجمالي أيام الغياب: {total_absences} أيام"
            alert_type = "خطير"
            alert_color = "red"
        else:
            # تنبيه أصفر للغياب 3 أيام متتالية
            alert_message = f"⚠️ تنبيه: المتدرب {student_name} ({student_rank}) متغيب {consecutive_absences} أيام متتالية.\n\n" \
                            f"✓ الدورة: {student_course}\n" \
                            f"✓ إجمالي أيام الغياب: {total_absences} أيام"
            alert_type = "متوسط"
            alert_color = "orange"

    return show_alert, alert_message, alert_type if show_alert else None, alert_color if show_alert else None

def insert_attendance_record(self, status, excuse_reason=""):
    """تعديل دالة تسجيل الحضور الموجودة لإضافة فحص الغياب المتكرر"""
    if not self.current_user["permissions"]["can_edit_attendance"]:
        messagebox.showwarning("تنبيه", "ليس لديك صلاحية تسجيل الحضور والغياب")
        return

    national_id = self.id_entry.get().strip()
    if not national_id:
        messagebox.showwarning("تنبيه", "الرجاء اختيار متدرب من خلال البحث بالاسم أو الهوية")
        return

    cursor = self.conn.cursor()
    cursor.execute("""
        SELECT national_id, name, rank, course, is_excluded 
        FROM trainees 
        WHERE national_id=?
    """, (national_id,))

    trainee = cursor.fetchone()
    if not trainee:
        messagebox.showwarning("تنبيه", "لا يوجد متدرب بهذا الرقم")
        return

    # التحقق من استبعاد المتدرب
    if trainee[4] == 1:
        messagebox.showwarning("تنبيه", "هذا المتدرب مستبعد ولا يمكن تسجيل حضوره")
        return

    current_date = self.date_entry.get_date().strftime("%Y-%m-%d")
    cursor.execute("SELECT status FROM attendance WHERE national_id=? AND date=?", (trainee[0], current_date))
    existing_record = cursor.fetchone()

    if existing_record:
        existing_status = existing_record[0]

        # استخدام نوافذ خطأ بدلاً من معلومات لجذب انتباه المستخدم
        if existing_status == status:
            # إذا كانت نفس الحالة
            messagebox.showerror("خطأ في التكرار",
                                 f"⚠️ تنبيه: المتدرب {trainee[1]} مسجل بالفعل بحالة '{existing_status}' اليوم\n\nلا يمكن تكرار نفس الحالة للمتدرب في نفس اليوم.")
        else:
            # إذا كانت حالة مختلفة
            messagebox.showerror("تعارض في الحالة",
                                 f"⚠️ تنبـــيه: المتدرب {trainee[1]} مسجل بالفعل بحالة '{existing_status}' اليوم\n\nلتغيير الحالة من '{existing_status}' إلى '{status}'، يرجى استخدام خاصية تعديل الحضور من قائمة سجل الحضور.")

        # مسح قيمة الهوية
        self.id_entry.delete(0, tk.END)
        self.name_search_entry.delete(0, tk.END)
        return

    # التحقق من حالة المتدرب في اليوم السابق
    current_date_obj = self.date_entry.get_date()
    yesterday_date_obj = current_date_obj - datetime.timedelta(days=1)
    yesterday_date = yesterday_date_obj.strftime("%Y-%m-%d")

    cursor.execute("SELECT status FROM attendance WHERE national_id=? AND date=?", (trainee[0], yesterday_date))
    yesterday_record = cursor.fetchone()

    # التحقق إذا كان المتدرب مسجل "لم يباشر" في اليوم السابق ويحاول المستخدم تسجيله "غائب"
    if yesterday_record and yesterday_record[0] == "لم يباشر" and status == "غائب":
        response = messagebox.askquestion("تنبيه هام ⚠️",
                                          f"المتدرب {trainee[1]} مسجل كـ 'لم يباشر' في اليوم السابق.\n\n"
                                          "• تأكد من أن المتدرب لم يباشر الدورة فعلاً.\n"
                                          "• إذا حضر المتدرب اليوم، يجب تسجيله كـ 'حاضر' أو 'متأخر'.\n"
                                          "• استمرار تسجيله كـ 'غائب' يعتبر مخالف لتعلميات التدريب المستديمة.\n\n"
                                          "هل تريد تغيير الحالة إلى 'لم يباشر' بدلاً من 'غائب'؟",
                                          icon="warning")
        if response == "yes":
            status = "لم يباشر"
        elif response == "no":
            # إضافة تأكيد إضافي عند الإصرار على الغياب
            confirm = messagebox.askquestion("تأكيد نهائي",
                                             f"هل أنت متأكد من تسجيل المتدرب {trainee[1]} كـ 'غائب' رغم أنه 'لم يباشر' بالأمس؟",
                                             icon="warning")
            if confirm != "yes":
                return

    t_id, t_name, t_rank, t_course, _ = trainee
    current_time = datetime.datetime.now().strftime("%H:%M:%S")

    # معالجة تسجيل حالة الغياب ورصد الغياب المتكرر
    absence_alert = False
    alert_message = ""
    alert_type = None
    alert_color = None

    if status in ["غائب"]:
        absence_alert, alert_message, alert_type, alert_color = self.check_student_absence(t_id, current_date)

    try:
        with self.conn:
            self.conn.execute("""
                INSERT INTO attendance (
                    national_id, name, rank, course,
                    time, date, status, original_status,
                    registered_by, excuse_reason,
                    updated_by, updated_at
                )
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
            """, (
                t_id, t_name, t_rank, t_course,
                current_time, current_date,
                status, status,
                self.current_user["full_name"], excuse_reason,
                "", ""
            ))

        # تحديث رسالة التأكيد في عنصر الواجهة بدلاً من نافذة منبثقة
        if status == "حاضر":
            icon_status = "✅"
        elif status == "غائب":
            icon_status = "❌"
        elif status == "متأخر":
            icon_status = "⏰"
        elif status == "غائب بعذر":
            icon_status = "📝"
        elif status == "لم يباشر":
            icon_status = "⏳"
        else:
            icon_status = "📌"

        # نعرض الرسالة فقط في حقل آخر متدرب سُجّل بدلاً من نافذة منبثقة
        self.last_registered_label.config(text=f"آخر متدرب سُجِّل: {t_name} ({status}) {icon_status}")

        # مسح حقول الإدخال
        self.id_entry.delete(0, tk.END)
        self.name_search_entry.delete(0, tk.END)
        self.name_listbox.delete(0, tk.END)

        self.update_statistics()
        self.update_attendance_display()

        # عرض تنبيه الغياب المتكرر إذا كان مطلوبًا
        if absence_alert:
            self.show_absence_alert(alert_message, alert_type, alert_color)

    except Exception as e:
        messagebox.showerror("خطأ", str(e))

def export_course_diligence_behavior(self, course_name):
    """وظيفة تصدير بيان المواظبة والسلوك للدورة بتنسيق Word مع ترتيب المتدربين حسب الدرجة"""
    if not self.current_user["permissions"]["can_export_data"]:
        messagebox.showwarning("تنبيه", "ليس لديك صلاحية تصدير البيانات")
        return

    try:
        # التأكد من وجود مكتبة python-docx
        if 'Document' not in globals():
            messagebox.showerror("خطأ",
                                 "لم يتم العثور على مكتبة python-docx. قم بتثبيتها باستخدام: pip install python-docx")
            return

        # الحصول على بيانات المتدربين في الدورة (فقط غير المستبعدين)
        cursor = self.conn.cursor()
        cursor.execute("""
            SELECT national_id, name, rank
            FROM trainees
            WHERE course=? AND is_excluded=0
            ORDER BY name
        """, (course_name,))
        students_data = cursor.fetchall()

        if not students_data:
            messagebox.showinfo("ملاحظة", f"لا يوجد متدربين نشطين مسجلين في الدورة '{course_name}'")
            return

        # إنشاء نافذة حالة لإظهار تقدم التصدير
        progress_window = tk.Toplevel(self.root)
        progress_window.title("جاري حساب المواظبة والسلوك")
        progress_window.geometry("400x150")
        progress_window.configure(bg=self.colors["light"])
        progress_window.transient(self.root)
        progress_window.resizable(False, False)
        progress_window.grab_set()

        x = (progress_window.winfo_screenwidth() - 400) // 2
        y = (progress_window.winfo_screenheight() - 150) // 2
        progress_window.geometry(f"400x150+{x}+{y}")

        tk.Label(
            progress_window,
            text=f"جاري حساب نتائج المواظبة والسلوك لدورة: {course_name}",
            font=self.fonts["text_bold"],
            bg=self.colors["light"],
            pady=10
        ).pack()

        progress_var = tk.DoubleVar()
        progress_bar = ttk.Progressbar(
            progress_window,
            variable=progress_var,
            maximum=100,
            length=350
        )
        progress_bar.pack(pady=10)

        status_label = tk.Label(
            progress_window,
            text="جاري تحليل بيانات الحضور والغياب...",
            font=self.fonts["text"],
            bg=self.colors["light"]
        )
        status_label.pack(pady=5)

        progress_window.update()

        # إنشاء مستند جديد
        doc = Document()

        # إعداد المستند للغة العربية (RTL)
        section = doc.sections[0]
        section.page_width = Inches(8.27)  # A4 width
        section.page_height = Inches(11.69)  # A4 height
        section.left_margin = Inches(0.7)
        section.right_margin = Inches(0.7)
        section.top_margin = Inches(0.7)
        section.bottom_margin = Inches(0.7)

        # إضافة عنوان المستند
        title = doc.add_heading(f'بيان المواظبة والسلوك لمتدربين دورة: {course_name}', level=0)
        title.alignment = WD_ALIGN_PARAGRAPH.CENTER
        for run in title.runs:
            run.font.size = Pt(16)
            run.font.bold = True
            run.font.rtl = True

        # إضافة معلومات الطباعة والتاريخ
        date_info = doc.add_paragraph()
        date_info.alignment = WD_ALIGN_PARAGRAPH.LEFT
        today_date = datetime.datetime.now().strftime("%Y-%m-%d")
        date_run = date_info.add_run(f'تاريخ الطباعة: {today_date}')
        date_run.font.size = Pt(10)
        date_run.font.rtl = True

        # إضافة خط أفقي
        border_paragraph = doc.add_paragraph()
        border_paragraph.paragraph_format.border_bottom = True

        # إنشاء جدول للمواظبة والسلوك
        table = doc.add_table(rows=1, cols=6)
        table.style = 'Table Grid'

        # عناوين الجدول (من اليمين إلى اليسار)
        hdr_cells = table.rows[0].cells
        headers = ["عدد", "الاسم", "الرتبة", "رقم الهوية", "المواظبة", "السلوك"]

        for i, header in enumerate(headers):
            # حساب الموقع المناسب للعناوين (من اليمين إلى اليسار)
            idx = len(headers) - i - 1
            hdr_cells[idx].text = header
            hdr_cells[idx].paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.CENTER
            for run in hdr_cells[idx].paragraphs[0].runs:
                run.font.bold = True
                run.font.size = Pt(12)
                run.font.rtl = True

            # تطبيق تظليل للرأس
            try:
                shading_elm = parse_xml(r'<w:shd {} w:fill="D9D9D9"/>'.format(nsdecls('w')))
                hdr_cells[idx]._element.get_or_add_tcPr().append(shading_elm)
            except:
                pass

        # معالجة بيانات كل متدرب وحساب درجة المواظبة
        student_scores = []
        total_students = len(students_data)

        for index, student in enumerate(students_data):
            national_id, name, rank = student

            # تحديث شريط التقدم
            progress_var.set((index / total_students) * 80)  # 80% للمعالجة
            status_label.config(text=f"معالجة المتدرب {index + 1} من {total_students}: {name}")
            progress_window.update()

            # حساب درجة المواظبة:
            # 1. الدرجة الأولية هي 100
            # 2. خصم 4 درجات لكل غياب كامل
            # 3. خصم 1 درجة لكل تأخير
            # 4. خصم 0.5 درجة لكل غياب بعذر

            # الاستعلام عن حالات الحضور للمتدرب
            cursor.execute("""
                SELECT status
                FROM attendance
                WHERE national_id=?
            """, (national_id,))
            attendance_records = cursor.fetchall()

            diligence_score = 100.0  # البداية من 100

            for record in attendance_records:
                status = record[0]
                if status == "غائب" or status == "غائب بعذر":  # تعديل: خصم 4 نقاط لـ "غائب بعذر"
                    diligence_score -= 4.0
                elif status == "متأخر":
                    diligence_score -= 1.0
                elif status == "حالة وفاة" or status == "منوم":  # إضافة: خصم 0.5 نقطة للحالات الجديدة
                    diligence_score -= 0.5

            # التأكد من عدم نزول الدرجة عن صفر
            diligence_score = max(0, diligence_score)

            # حفظ بيانات المتدرب مع الدرجة
            student_scores.append((national_id, name, rank, diligence_score))

        # ترتيب المتدربين تصاعدياً حسب درجة المواظبة (الأقل يأتي أولاً)
        student_scores.sort(key=lambda x: x[3])

        # إضافة بيانات المتدربين إلى الجدول بعد الترتيب
        for index, (national_id, name, rank, diligence_score) in enumerate(student_scores):
            # تحديث شريط التقدم
            progress_var.set(80 + (index / total_students) * 15)  # 15% للترتيب والإضافة
            status_label.config(text=f"إضافة المتدرب {index + 1} من {total_students} إلى التقرير")
            progress_window.update()

            # درجة السلوك دائمًا 100
            behavior_score = 100.0

            # إضافة صف جديد للمتدرب
            row_cells = table.add_row().cells

            # الترتيب من اليمين إلى اليسار
            row_cells[5].text = str(index + 1)  # العدد التسلسلي
            row_cells[4].text = name  # الاسم
            row_cells[3].text = rank  # الرتبة
            row_cells[2].text = national_id  # رقم الهوية
            row_cells[1].text = f"{diligence_score:.1f}"  # المواظبة بدقة رقم عشري واحد
            row_cells[0].text = f"{behavior_score:.0f}"  # السلوك (دائمًا 100)

            # تنسيق الخلايا
            for cell in row_cells:
                cell.paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.CENTER
                for run in cell.paragraphs[0].runs:
                    run.font.rtl = True
                    run.font.size = Pt(11)

            # تلوين الصف حسب درجة المواظبة
            if diligence_score < 90:  # إذا كانت الدرجة أقل من 90، تمييزها بلون فاتح
                try:
                    for cell in row_cells:
                        shading_elm = parse_xml(r'<w:shd {} w:fill="FFDDDD"/>'.format(nsdecls('w')))
                        cell._element.get_or_add_tcPr().append(shading_elm)
                except:
                    pass

        # تنسيق الجدول
        table.autofit = False
        try:
            # تعيين عرض الأعمدة (العرض بالبوصة)
            widths = [0.8, 0.8, 1.2, 1.5, 2.5, 0.5]  # السلوك، المواظبة، الهوية، الرتبة، الاسم، العدد
            for i, width in enumerate(widths):
                table.columns[i].width = Inches(width)
        except:
            pass

        # إضافة فقرة فاصلة بعد الجدول
        doc.add_paragraph()

        # إضافة جدول للتوقيعات
        signature_table = doc.add_table(rows=1, cols=3)
        signature_table.style = 'Table Grid'

        sig_cells = signature_table.rows[0].cells
        sig_cells[2].text = "مسؤول الحضور: _________________"
        sig_cells[1].text = "رئيس القسم: __________________"
        sig_cells[0].text = "مدير التدريب: ________________"

        for cell in sig_cells:
            cell.paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.CENTER
            for run in cell.paragraphs[0].runs:
                run.font.rtl = True
                run.font.size = Pt(11)

        # إضافة نص توضيحي في نهاية المستند
        doc.add_paragraph()
        note_para = doc.add_paragraph()
        note_para.alignment = WD_ALIGN_PARAGRAPH.RIGHT
        note_run = note_para.add_run("ملاحظات حساب المواظبة:")
        note_run.font.bold = True
        note_run.font.rtl = True

        notes = [
            "- تبدأ درجة المواظبة من 100 درجة.",
            "- يتم خصم 4 درجات عن كل يوم غياب.",
            "- يتم خصم 4 درجات عن كل غياب بعذر.",
            "- يتم خصم 1 درجة عن كل حالة تأخير.",
            "- يتم خصم 0.5 درجة عن كل حالة وفاة.",
            "- يتم خصم 0.5 درجة عن كل حالة منوم.",
            "- درجة السلوك 100 درجة للجميع."
        ]

        for note in notes:
            p = doc.add_paragraph()
            p.alignment = WD_ALIGN_PARAGRAPH.RIGHT
            p.add_run(note).font.rtl = True

        # تحديث شريط التقدم
        progress_var.set(95)
        status_label.config(text="فتح حوار حفظ الملف...")
        progress_window.update()

        # حفظ المستند
        export_file = filedialog.asksaveasfilename(
            defaultextension=".docx",
            filetypes=[("Word documents", "*.docx")],
            initialfile=f"بيان_المواظبة_والسلوك_{course_name}.docx"
        )

        if export_file:
            progress_var.set(95)
            status_label.config(text="جاري حفظ الملف...")
            progress_window.update()

            doc.save(export_file)

            progress_var.set(100)
            status_label.config(text="تم تصدير البيان بنجاح!")
            progress_window.update()

            # إغلاق نافذة التقدم بعد ثانيتين
            progress_window.after(2000, progress_window.destroy)

            messagebox.showinfo("نجاح",
                                f"تم تصدير بيان المواظبة والسلوك للدورة '{course_name}' بنجاح إلى:\n{export_file}")

            # محاولة فتح الملف تلقائيًا
            try:
                os.startfile(export_file)
            except:
                pass
        else:
            progress_window.destroy()

    except Exception as e:
        try:
            progress_window.destroy()
        except:
            pass
        messagebox.showerror("خطأ", f"حدث خطأ أثناء تصدير بيان المواظبة والسلوك: {str(e)}")

def manage_multi_section_courses(self):
    """وظيفة إدارة الدورات متعددة الفصول"""
    # إنشاء نافذة جديدة
    multi_window = tk.Toplevel(self.root)
    multi_window.title("إدارة الفصول و تصدير الكشوفات")
    multi_window.geometry("900x600")
    multi_window.configure(bg=self.colors["light"])
    multi_window.grab_set()
    multi_window.resizable(True, True)

    # توسيط النافذة
    x = (multi_window.winfo_screenwidth() - 900) // 2
    y = (multi_window.winfo_screenheight() - 600) // 2
    multi_window.geometry(f"900x600+{x}+{y}")

    # عنوان النافذة
    tk.Label(
        multi_window,
        text="إدارة الفصول و تصدير الكشوفات",
        font=self.fonts["large_title"],
        bg=self.colors["primary"],
        fg="white",
        padx=10, pady=10
    ).pack(fill=tk.X)

    # تعديل: إضافة إطار جديد لعرض معلومات إجمالي المتدربين تحت العنوان مباشرة
    students_info_frame = tk.Frame(multi_window, bg=self.colors["light"], padx=10, pady=5)
    students_info_frame.pack(fill=tk.X)

    # تعديل: متغير لعرض إجمالي عدد المتدربين
    total_students_var = tk.StringVar(value="")
    total_students_label = tk.Label(
        students_info_frame,
        textvariable=total_students_var,
        font=self.fonts["text_bold"],
        bg=self.colors["light"],
        fg=self.colors["primary"]
    )
    total_students_label.pack(pady=5)

    # إطار اختيار الدورة
    course_frame = tk.Frame(multi_window, bg=self.colors["light"], padx=10, pady=10)
    course_frame.pack(fill=tk.X)

    tk.Label(
        course_frame,
        text="اختيار الدورة:",
        font=self.fonts["text_bold"],
        bg=self.colors["light"]
    ).pack(side=tk.RIGHT, padx=5)

    # الحصول على قائمة الدورات
    cursor = self.conn.cursor()
    cursor.execute("SELECT DISTINCT course_name FROM course_sections")
    multi_section_courses = [row[0] for row in cursor.fetchall()]

    course_var = tk.StringVar()
    course_dropdown = ttk.Combobox(
        course_frame,
        textvariable=course_var,
        values=multi_section_courses,
        width=30,
        font=self.fonts["text"]
    )
    course_dropdown.pack(side=tk.RIGHT, padx=5)

    # زر تحديث قائمة الدورات
    refresh_btn = tk.Button(
        course_frame,
        text="تحديث",
        font=self.fonts["text_bold"],
        bg=self.colors["secondary"],
        fg="white",
        padx=10, pady=2,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=lambda: update_sections_list()
    )
    refresh_btn.pack(side=tk.LEFT, padx=5)

    # زر تعديل تاريخ الدورة
    edit_course_info_btn = tk.Button(
        course_frame,
        text="تعديل تواريخ الدورة",
        font=self.fonts["text_bold"],
        bg=self.colors["warning"],
        fg="white",
        padx=10, pady=2,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=lambda: self.edit_course_dates(course_var.get())
    )
    edit_course_info_btn.pack(side=tk.LEFT, padx=5)

    # إضافة زر حذف الدورة كاملة (للمشرفين فقط)
    if self.current_user["permissions"]["is_admin"]:
        delete_course_btn = tk.Button(
            course_frame,
            text="حذف الدورة كاملة",
            font=self.fonts["text_bold"],
            bg=self.colors["danger"],
            fg="white",
            padx=10, pady=2,
            bd=0, relief=tk.FLAT,
            cursor="hand2",
            command=lambda: delete_entire_course()
        )
        delete_course_btn.pack(side=tk.LEFT, padx=5)

    # إطار عرض الفصول
    sections_frame = tk.LabelFrame(
        multi_window,
        text="الفصول المتاحة",
        font=self.fonts["subtitle"],
        bg=self.colors["light"],
        fg=self.colors["dark"],
        padx=10, pady=10
    )
    sections_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=5)

    # قائمة الفصول
    list_frame = tk.Frame(sections_frame, bg=self.colors["light"])
    list_frame.pack(side=tk.LEFT, fill=tk.BOTH, expand=True, padx=5, pady=5)

    list_scroll = tk.Scrollbar(list_frame)
    list_scroll.pack(side=tk.RIGHT, fill=tk.Y)

    sections_listbox = tk.Listbox(
        list_frame,
        font=self.fonts["text"],
        selectbackground=self.colors["primary"],
        selectforeground="white",
        yscrollcommand=list_scroll.set
    )
    sections_listbox.pack(fill=tk.BOTH, expand=True)
    list_scroll.config(command=sections_listbox.yview)

    # إطار التفاصيل
    details_frame = tk.Frame(sections_frame, bg=self.colors["light"], width=350)
    details_frame.pack(side=tk.RIGHT, fill=tk.BOTH, padx=5, pady=5)

    # عنوان التفاصيل
    section_title_var = tk.StringVar(value="اختر فصلاً لعرض تفاصيله")
    section_title = tk.Label(
        details_frame,
        textvariable=section_title_var,
        font=self.fonts["text_bold"],
        bg=self.colors["light"],
        fg=self.colors["primary"]
    )
    section_title.pack(pady=(0, 10))

    # تعديل: نحتفظ بمتغير عدد المتدربين للاستخدام الداخلي دون عرضه في إطار التفاصيل
    students_count_var = tk.StringVar(value="")

    # أزرار الإجراءات
    actions_frame = tk.Frame(details_frame, bg=self.colors["light"], pady=10)
    actions_frame.pack(fill=tk.X)

    export_attendance_btn = tk.Button(
        actions_frame,
        text="تصدير كشف حضور",
        font=self.fonts["text_bold"],
        bg=self.colors["primary"],
        fg="white",
        padx=10, pady=5,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=lambda: export_section_attendance_sheet()
    )
    export_attendance_btn.pack(fill=tk.X, pady=5)

    export_diligence_btn = tk.Button(
        actions_frame,
        text="تصدير كشف المواظبة والسلوك",
        font=self.fonts["text_bold"],
        bg="#8E44AD",
        fg="white",
        padx=10, pady=5,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=lambda: export_section_diligence()
    )
    export_diligence_btn.pack(fill=tk.X, pady=5)

    view_students_btn = tk.Button(
        actions_frame,
        text="عرض المتدربين وإدارة الفصول",
        font=self.fonts["text_bold"],
        bg=self.colors["success"],
        fg="white",
        padx=10, pady=5,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=lambda: manage_section_students()
    )
    view_students_btn.pack(fill=tk.X, pady=5)

    rename_section_btn = tk.Button(
        actions_frame,
        text="تغيير اسم الفصل",
        font=self.fonts["text_bold"],
        bg=self.colors["warning"],
        fg="white",
        padx=10, pady=5,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=lambda: rename_section()
    )
    rename_section_btn.pack(fill=tk.X, pady=5)

    # إضافة زر حذف الفصل مع ترحيل المتدربين (متاح للجميع)
    delete_section_btn = tk.Button(
        actions_frame,
        text="حذف الفصل مع ترحيل المتدربين",
        font=self.fonts["text_bold"],
        bg=self.colors["danger"],
        fg="white",
        padx=10, pady=5,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=lambda: delete_section_with_transfer()
    )
    delete_section_btn.pack(fill=tk.X, pady=5)

    # الإطار السفلي للأزرار
    bottom_frame = tk.Frame(multi_window, bg=self.colors["light"], pady=10)
    bottom_frame.pack(fill=tk.X, padx=10)

    add_section_btn = tk.Button(
        bottom_frame,
        text="إضافة فصل جديد",
        font=self.fonts["text_bold"],
        bg=self.colors["success"],
        fg="white",
        padx=15, pady=5,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=lambda: add_new_section()
    )
    add_section_btn.pack(side=tk.LEFT, padx=5)

    # هنا يتم إضافة الزر الجديد
    import_sections_btn = tk.Button(
        bottom_frame,
        text="استيراد تحديثات الفصول",
        font=self.fonts["text_bold"],
        bg="#FF9800",  # لون برتقالي للتمييز
        fg="white",
        padx=15, pady=5,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=lambda: import_section_updates()
    )
    import_sections_btn.pack(side=tk.LEFT, padx=5)

    # تعريف دالة للإغلاق مع تحديث البيانات
    def on_close_multi_window():
        multi_window.destroy()
        self.update_statistics()
        self.update_students_tree()
        self.update_attendance_display()  # إضافة هذا السطر لتحديث عرض سجل الحضور أيضاً

    close_btn = tk.Button(
        bottom_frame,
        text="إغلاق",
        font=self.fonts["text_bold"],
        bg=self.colors["dark"],
        fg="white",
        padx=15, pady=5,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=on_close_multi_window  # استخدام الدالة الجديدة بدلاً من multi_window.destroy
    )
    close_btn.pack(side=tk.RIGHT, padx=5)

    def import_section_updates():
        """استيراد تحديثات توزيع المتدربين على الفصول من ملف Excel مع دعم الأعمدة باللغة العربية"""
        selected_course = course_var.get().strip()
        if not selected_course:
            messagebox.showwarning("تنبيه", "الرجاء اختيار دورة أولاً")
            return

        # اختيار ملف Excel
        file_path = filedialog.askopenfilename(
            title="اختر ملف تحديثات الفصول",
            filetypes=[("Excel files", "*.xlsx"), ("All files", "*.*")]
        )

        if not file_path:
            return

        # إنشاء نافذة تقدم العملية
        progress_window = tk.Toplevel(multi_window)
        progress_window.title("استيراد تحديثات الفصول")
        progress_window.geometry("450x180")
        progress_window.configure(bg=self.colors["light"])
        progress_window.transient(multi_window)
        progress_window.grab_set()

        # توسيط النافذة
        x = (progress_window.winfo_screenwidth() - 450) // 2
        y = (progress_window.winfo_screenheight() - 180) // 2
        progress_window.geometry(f"450x180+{x}+{y}")

        tk.Label(
            progress_window,
            text=f"جاري معالجة تحديثات الفصول لدورة: {selected_course}",
            font=self.fonts["text_bold"],
            bg=self.colors["light"],
            pady=10
        ).pack()

        progress_var = tk.DoubleVar()
        progress_bar = ttk.Progressbar(
            progress_window,
            variable=progress_var,
            maximum=100,
            length=400
        )
        progress_bar.pack(pady=10)

        status_label = tk.Label(
            progress_window,
            text="جاري قراءة ملف Excel...",
            font=self.fonts["text"],
            bg=self.colors["light"]
        )
        status_label.pack(pady=5)

        progress_window.update()

        try:
            # قراءة ملف Excel
            progress_var.set(10)
            status_label.config(text="جاري قراءة ملف Excel...")
            progress_window.update()

            df = pd.read_excel(file_path)

            # تعريف ترجمة أسماء الأعمدة (دعم العربية والإنجليزية)
            column_mapping = {
                'رقم الهوية': 'national_id',
                'الفصل': 'section_name',
                'اسم الفصل': 'section_name',
                'national_id': 'national_id',
                'section_name': 'section_name'
            }

            # تحويل أسماء الأعمدة من العربية إلى الإنجليزية
            rename_dict = {}
            for orig_col in df.columns:
                if orig_col in column_mapping:
                    rename_dict[orig_col] = column_mapping[orig_col]

            if rename_dict:
                df = df.rename(columns=rename_dict)

            # التحقق من وجود الأعمدة المطلوبة
            has_id = any(col in ["رقم الهوية", "national_id"] for col in df.columns)
            has_section = any(col in ["الفصل", "اسم الفصل", "section_name"] for col in df.columns)

            if not (has_id and has_section):
                progress_window.destroy()
                messagebox.showwarning("تنبيه",
                                       f"يجب أن يحتوي ملف التحديثات على الأعمدة التالية:\n\n" +
                                       "• رقم الهوية (national_id)\n" +
                                       "• الفصل (section_name)")
                return

            # التحقق من صحة الفصول المذكورة في الملف
            progress_var.set(20)
            status_label.config(text="التحقق من صحة بيانات الفصول...")
            progress_window.update()

            # الحصول على قائمة الفصول المتاحة في الدورة
            cursor = self.conn.cursor()
            cursor.execute("""
                SELECT section_name
                FROM course_sections
                WHERE course_name=?
            """, (selected_course,))

            available_sections = {row[0] for row in cursor.fetchall()}

            # التحقق من وجود الفصول المذكورة في الملف
            unique_sections = set(df['section_name'].dropna())
            invalid_sections = unique_sections - available_sections

            if invalid_sections:
                progress_window.destroy()
                messagebox.showwarning("تنبيه",
                                       f"توجد فصول غير موجودة في الدورة: {', '.join(invalid_sections)}\n\n" +
                                       "الفصول المتاحة هي: " + ', '.join(available_sections))
                return

            # التحقق من وجود المتدربين المذكورين في ملف التحديثات
            progress_var.set(30)
            status_label.config(text="التحقق من بيانات المتدربين...")
            progress_window.update()

            # الحصول على قائمة المتدربين في الدورة الحالية
            cursor.execute("""
                SELECT national_id, name
                FROM trainees
                WHERE course=? AND is_excluded=0
            """, (selected_course,))

            students_dict = {row[0]: row[1] for row in cursor.fetchall()}

            # التحقق من وجود جميع المتدربين المذكورين في الملف
            student_ids = df['national_id'].astype(str).tolist()
            invalid_students = [sid for sid in student_ids if sid not in students_dict]

            if invalid_students:
                if len(invalid_students) > 5:
                    invalid_display = ', '.join(invalid_students[:5]) + f' وغيرهم ({len(invalid_students) - 5})'
                else:
                    invalid_display = ', '.join(invalid_students)

                proceed = messagebox.askyesno("تنبيه - متدربين غير موجودين",
                                              f"هناك {len(invalid_students)} متدرب غير موجود في الدورة: {invalid_display}\n\n" +
                                              "هل تريد المتابعة وتجاهل هؤلاء المتدربين؟")

                if not proceed:
                    progress_window.destroy()
                    return

            # تحضير التغييرات
            progress_var.set(50)
            status_label.config(text="تحضير التغييرات...")
            progress_window.update()

            # الحصول على التوزيع الحالي للمتدربين على الفصول
            cursor.execute("""
                SELECT national_id, section_name
                FROM student_sections
                WHERE course_name=?
            """, (selected_course,))

            current_assignments = {row[0]: row[1] for row in cursor.fetchall()}

            # تحضير قائمة التغييرات
            changes = []
            no_changes = []
            new_assignments = []

            for _, row in df.iterrows():
                student_id = str(row['national_id']).strip()
                new_section = str(row['section_name']).strip()

                # تخطي المتدربين غير الموجودين
                if student_id not in students_dict:
                    continue

                # التحقق إذا كان المتدرب في فصل مختلف حاليًا
                if student_id in current_assignments:
                    current_section = current_assignments[student_id]

                    if current_section != new_section:
                        # تغيير الفصل
                        changes.append((student_id, students_dict[student_id], current_section, new_section))
                    else:
                        # لا تغيير
                        no_changes.append((student_id, students_dict[student_id], current_section))
                else:
                    # متدرب جديد ليس في أي فصل سابقًا
                    new_assignments.append((student_id, students_dict[student_id], new_section))

            # عرض ملخص التغييرات المقترحة
            progress_var.set(70)
            status_label.config(text="تجهيز ملخص التغييرات...")
            progress_window.update()

            summary = f"ملخص التغييرات:\n\n"
            summary += f"• عدد المتدربين الذين سيتم نقلهم بين الفصول: {len(changes)}\n"
            summary += f"• عدد المتدربين الجدد المراد تسجيلهم في فصول: {len(new_assignments)}\n"
            summary += f"• عدد المتدربين بدون تغيير: {len(no_changes)}\n"

            if invalid_students:
                summary += f"• عدد المتدربين غير الموجودين في الدورة: {len(invalid_students)} (سيتم تجاهلهم)\n"

            progress_window.destroy()

            # عرض نافذة ملخص التغييرات
            summary_window = tk.Toplevel(multi_window)
            summary_window.title("ملخص التغييرات المقترحة")
            summary_window.geometry("600x500")
            summary_window.configure(bg=self.colors["light"])
            summary_window.transient(multi_window)
            summary_window.grab_set()

            # توسيط النافذة
            x = (summary_window.winfo_screenwidth() - 600) // 2
            y = (summary_window.winfo_screenheight() - 500) // 2
            summary_window.geometry(f"600x500+{x}+{y}")

            tk.Label(
                summary_window,
                text="ملخص التغييرات المقترحة",
                font=self.fonts["title"],
                bg=self.colors["primary"],
                fg="white",
                padx=10, pady=10
            ).pack(fill=tk.X)

            # عرض ملخص التغييرات
            summary_frame = tk.Frame(summary_window, bg=self.colors["light"], padx=10, pady=10)
            summary_frame.pack(fill=tk.BOTH, expand=True)

            tk.Label(
                summary_frame,
                text=summary,
                font=self.fonts["text"],
                bg=self.colors["light"],
                justify=tk.RIGHT,
                anchor=tk.E
            ).pack(fill=tk.X, pady=10)

            # إنشاء نوتبوك لعرض التفاصيل
            details_notebook = ttk.Notebook(summary_frame)
            details_notebook.pack(fill=tk.BOTH, expand=True, pady=10)

            # تبويب المتدربين المنقولين
            if changes:
                changes_frame = tk.Frame(details_notebook, bg=self.colors["light"])
                details_notebook.add(changes_frame, text=f"متدربين سيتم نقلهم ({len(changes)})")

                changes_list = tk.Text(changes_frame, font=self.fonts["text"], width=70, height=15)
                changes_list.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)

                for student_id, name, old_section, new_section in changes:
                    changes_list.insert(tk.END,
                                        f"{name} ({student_id}): من فصل {old_section} إلى فصل {new_section}\n")

                changes_list.configure(state="disabled")

            # تبويب المتدربين الجدد
            if new_assignments:
                new_frame = tk.Frame(details_notebook, bg=self.colors["light"])
                details_notebook.add(new_frame, text=f"تسجيلات جديدة ({len(new_assignments)})")

                new_list = tk.Text(new_frame, font=self.fonts["text"], width=70, height=15)
                new_list.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)

                for student_id, name, section in new_assignments:
                    new_list.insert(tk.END, f"{name} ({student_id}): تسجيل في فصل {section}\n")

                new_list.configure(state="disabled")

            # أزرار التأكيد أو الإلغاء
            button_frame = tk.Frame(summary_window, bg=self.colors["light"], pady=10)
            button_frame.pack(fill=tk.X, padx=10)

            def apply_changes():
                # تنفيذ التغييرات
                try:
                    current_date = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")

                    progress_window = tk.Toplevel(summary_window)
                    progress_window.title("تنفيذ التغييرات")
                    progress_window.geometry("450x180")
                    progress_window.configure(bg=self.colors["light"])
                    progress_window.transient(summary_window)
                    progress_window.grab_set()

                    # توسيط النافذة
                    x = (progress_window.winfo_screenwidth() - 450) // 2
                    y = (progress_window.winfo_screenheight() - 180) // 2
                    progress_window.geometry(f"450x180+{x}+{y}")

                    tk.Label(
                        progress_window,
                        text="جاري تنفيذ التغييرات...",
                        font=self.fonts["text_bold"],
                        bg=self.colors["light"],
                        pady=10
                    ).pack()

                    progress_var = tk.DoubleVar()
                    progress_bar = ttk.Progressbar(
                        progress_window,
                        variable=progress_var,
                        maximum=100,
                        length=400
                    )
                    progress_bar.pack(pady=10)

                    status_label = tk.Label(
                        progress_window,
                        text="جاري الإعداد...",
                        font=self.fonts["text"],
                        bg=self.colors["light"]
                    )
                    status_label.pack(pady=5)

                    progress_window.update()

                    with self.conn:
                        # تنفيذ التغييرات
                        total_operations = len(changes) + len(new_assignments)
                        operations_done = 0

                        # 1. تحديث المتدربين الذين سيتم نقلهم
                        if changes:
                            status_label.config(text="جاري تعديل تسجيلات الفصول الحالية...")
                            progress_window.update()

                            for student_id, _, _, new_section in changes:
                                self.conn.execute("""
                                    UPDATE student_sections
                                    SET section_name=?, assigned_date=?
                                    WHERE national_id=? AND course_name=?
                                """, (new_section, current_date, student_id, selected_course))

                                operations_done += 1
                                progress_percent = (operations_done / total_operations) * 100
                                progress_var.set(progress_percent)
                                progress_window.update()

                        # 2. إضافة المتدربين الجدد
                        if new_assignments:
                            status_label.config(text="جاري إضافة تسجيلات جديدة للفصول...")
                            progress_window.update()

                            for student_id, _, section in new_assignments:
                                self.conn.execute("""
                                    INSERT OR REPLACE INTO student_sections
                                    (national_id, course_name, section_name, assigned_date)
                                    VALUES (?, ?, ?, ?)
                                """, (student_id, selected_course, section, current_date))

                                operations_done += 1
                                progress_percent = (operations_done / total_operations) * 100
                                progress_var.set(progress_percent)
                                progress_window.update()

                    progress_var.set(100)
                    status_label.config(text="تم تنفيذ التغييرات بنجاح!")
                    progress_window.update()

                    # إغلاق نافذة التقدم بعد ثانيتين
                    progress_window.after(2000, progress_window.destroy)

                    # إغلاق نافذة الملخص
                    summary_window.destroy()

                    # عرض رسالة نجاح
                    messagebox.showinfo("نجاح", "تم تنفيذ تحديثات الفصول بنجاح!")

                    # تحديث القوائم
                    update_sections_list()

                except Exception as e:
                    try:
                        progress_window.destroy()
                    except:
                        pass

                    messagebox.showerror("خطأ", f"حدث خطأ أثناء تنفيذ التغييرات: {str(e)}")

            confirm_btn = tk.Button(
                button_frame,
                text="تنفيذ التغييرات",
                font=self.fonts["text_bold"],
                bg=self.colors["success"],
                fg="white",
                padx=15, pady=5,
                bd=0, relief=tk.FLAT,
                cursor="hand2",
                command=apply_changes
            )
            confirm_btn.pack(side=tk.LEFT, padx=5)

            cancel_btn = tk.Button(
                button_frame,
                text="إلغاء",
                font=self.fonts["text_bold"],
                bg=self.colors["danger"],
                fg="white",
                padx=15, pady=5,
                bd=0, relief=tk.FLAT,
                cursor="hand2",
                command=summary_window.destroy
            )
            cancel_btn.pack(side=tk.RIGHT, padx=5)

        except Exception as e:
            try:
                progress_window.destroy()
            except:
                pass

            messagebox.showerror("خطأ", f"حدث خطأ أثناء معالجة ملف التحديثات: {str(e)}")

    # الوظائف المساعدة ضمن النافذة
    def update_sections_list():
        """تحديث قائمة الفصول المتاحة للدورة المحددة"""
        selected_course = course_var.get().strip()
        sections_listbox.delete(0, tk.END)

        # تعديل: إعادة ضبط متغيرات العرض
        section_title_var.set("اختر فصلاً لعرض تفاصيله")
        students_count_var.set("")

        if not selected_course:
            total_students_var.set("")
            return

        # تعديل: تحديث إجمالي عدد المتدربين في الدورة
        cursor = self.conn.cursor()
        cursor.execute("""
            SELECT COUNT(DISTINCT t.national_id)
            FROM trainees t
            WHERE t.course=? AND t.is_excluded=0
        """, (selected_course,))

        total_count = cursor.fetchone()[0]
        total_students_var.set(f"إجمالي المتدربين الملتحقين بدورة \"{selected_course}\": {total_count}")

        cursor = self.conn.cursor()
        cursor.execute("""
            SELECT section_name
            FROM course_sections
            WHERE course_name=?
            ORDER BY section_name
        """, (selected_course,))

        sections = cursor.fetchall()

        for section in sections:
            sections_listbox.insert(tk.END, section[0])

    def on_section_select(event=None):
        """عند اختيار فصل من القائمة"""
        selected_indices = sections_listbox.curselection()
        if not selected_indices:
            return

        selected_course = course_var.get().strip()
        selected_section = sections_listbox.get(selected_indices[0])

        if not selected_course or not selected_section:
            return

        # تحديث عنوان التفاصيل
        section_title_var.set(f"فصل: {selected_section}")

        # حساب عدد المتدربين
        cursor = self.conn.cursor()
        cursor.execute("""
            SELECT COUNT(*)
            FROM student_sections
            WHERE course_name=? AND section_name=?
        """, (selected_course, selected_section))

        count = cursor.fetchone()[0]
        students_count_var.set(f"عدد المتدربين في فصل \"{selected_section}\": {count}")

        # تعديل: تحديث عرض عدد المتدربين في العنوان
        total_students_var.set(students_count_var.get())

    def add_new_section():
        """إضافة فصل جديد للدورة المحددة"""
        selected_course = course_var.get().strip()

        if not selected_course:
            messagebox.showwarning("تنبيه", "الرجاء اختيار دورة أولاً")
            return

        section_name = simpledialog.askstring("إضافة فصل", "أدخل اسم الفصل الجديد:")

        if not section_name:
            return

        # التحقق من وجود الفصل
        cursor = self.conn.cursor()
        cursor.execute("""
            SELECT COUNT(*)
            FROM course_sections
            WHERE course_name=? AND section_name=?
        """, (selected_course, section_name))

        if cursor.fetchone()[0] > 0:
            messagebox.showwarning("تنبيه", f"الفصل '{section_name}' موجود بالفعل في هذه الدورة")
            return

        # إضافة الفصل الجديد
        try:
            current_date = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
            with self.conn:
                self.conn.execute("""
                    INSERT INTO course_sections (course_name, section_name, created_date)
                    VALUES (?, ?, ?)
                """, (selected_course, section_name, current_date))

            messagebox.showinfo("نجاح", f"تم إضافة الفصل '{section_name}' بنجاح")
            update_sections_list()
        except Exception as e:
            messagebox.showerror("خطأ", f"حدث خطأ أثناء إضافة الفصل: {str(e)}")

    def rename_section():
        """تغيير اسم الفصل المحدد"""
        selected_indices = sections_listbox.curselection()
        if not selected_indices:
            messagebox.showwarning("تنبيه", "الرجاء اختيار فصل أولاً")
            return

        selected_course = course_var.get().strip()
        old_section_name = sections_listbox.get(selected_indices[0])

        new_section_name = simpledialog.askstring("تغيير اسم الفصل", "أدخل الاسم الجديد للفصل:",
                                                  initialvalue=old_section_name)

        if not new_section_name or new_section_name == old_section_name:
            return

        # التحقق من وجود الفصل الجديد
        cursor = self.conn.cursor()
        cursor.execute("""
            SELECT COUNT(*)
            FROM course_sections
            WHERE course_name=? AND section_name=?
        """, (selected_course, new_section_name))

        if cursor.fetchone()[0] > 0:
            messagebox.showwarning("تنبيه", f"الفصل '{new_section_name}' موجود بالفعل في هذه الدورة")
            return

        # تحديث اسم الفصل
        try:
            with self.conn:
                # تحديث في جدول الفصول
                self.conn.execute("""
                    UPDATE course_sections
                    SET section_name=?
                    WHERE course_name=? AND section_name=?
                """, (new_section_name, selected_course, old_section_name))

                # تحديث في جدول المتدربين
                self.conn.execute("""
                    UPDATE student_sections
                    SET section_name=?
                    WHERE course_name=? AND section_name=?
                """, (new_section_name, selected_course, old_section_name))

            messagebox.showinfo("نجاح", f"تم تغيير اسم الفصل إلى '{new_section_name}' بنجاح")
            update_sections_list()
        except Exception as e:
            messagebox.showerror("خطأ", f"حدث خطأ أثناء تغيير اسم الفصل: {str(e)}")

    def manage_section_students():
        """إدارة متدربين الفصل المحدد"""
        selected_indices = sections_listbox.curselection()
        if not selected_indices:
            messagebox.showwarning("تنبيه", "الرجاء اختيار فصل أولاً")
            return

        selected_course = course_var.get().strip()
        selected_section = sections_listbox.get(selected_indices[0])

        # فتح نافذة إدارة متدربين الفصل
        self.open_section_students_window(selected_course, selected_section)

    def export_section_attendance_sheet():
        """تصدير كشف حضور للفصل المحدد"""
        selected_indices = sections_listbox.curselection()
        if not selected_indices:
            messagebox.showwarning("تنبيه", "الرجاء اختيار فصل أولاً")
            return

        selected_course = course_var.get().strip()
        selected_section = sections_listbox.get(selected_indices[0])

        # تنفيذ وظيفة تصدير كشف الحضور للفصل
        self.export_section_to_word(selected_course, selected_section)

    def export_section_diligence():
        """تصدير كشف المواظبة والسلوك للفصل المحدد"""
        selected_indices = sections_listbox.curselection()
        if not selected_indices:
            messagebox.showwarning("تنبيه", "الرجاء اختيار فصل أولاً")
            return

        selected_course = course_var.get().strip()
        selected_section = sections_listbox.get(selected_indices[0])

        # تنفيذ وظيفة تصدير كشف المواظبة والسلوك للفصل
        self.export_section_diligence_behavior(selected_course, selected_section)

    # وظيفة حذف الفصل مع ترحيل المتدربين
    def delete_section_with_transfer():
        """حذف الفصل مع ترحيل المتدربين إلى فصل آخر"""
        selected_indices = sections_listbox.curselection()
        if not selected_indices:
            messagebox.showwarning("تنبيه", "الرجاء اختيار فصل للحذف")
            return

        selected_course = course_var.get().strip()
        selected_section = sections_listbox.get(selected_indices[0])

        # الحصول على عدد المتدربين في الفصل المحدد
        cursor = self.conn.cursor()
        cursor.execute("""
            SELECT COUNT(*)
            FROM student_sections
            WHERE course_name=? AND section_name=?
        """, (selected_course, selected_section))

        students_count = cursor.fetchone()[0]

        # التحقق من وجود فصول أخرى
        cursor.execute("""
            SELECT section_name
            FROM course_sections
            WHERE course_name=? AND section_name!=?
        """, (selected_course, selected_section))

        other_sections = [row[0] for row in cursor.fetchall()]

        if not other_sections:
            messagebox.showwarning("تنبيه",
                                   f"لا يوجد فصول أخرى في الدورة '{selected_course}' لترحيل المتدربين إليها.\n\nيجب أن يتوفر فصل واحد على الأقل لنقل المتدربين إليه.")
            return

        # إذا كان هناك متدربين في الفصل، اطلب تحديد فصل للترحيل
        if students_count > 0:
            # عرض نافذة لاختيار الفصل المراد الترحيل إليه
            transfer_window = tk.Toplevel(multi_window)
            transfer_window.title("ترحيل المتدربين")
            transfer_window.geometry("400x300")
            transfer_window.configure(bg=self.colors["light"])
            transfer_window.transient(multi_window)
            transfer_window.grab_set()

            # توسيط النافذة
            x = (transfer_window.winfo_screenwidth() - 400) // 2
            y = (transfer_window.winfo_screenheight() - 300) // 2
            transfer_window.geometry(f"400x300+{x}+{y}")

            tk.Label(
                transfer_window,
                text=f"ترحيل متدربين الفصل: {selected_section}",
                font=self.fonts["title"],
                bg=self.colors["primary"],
                fg="white",
                padx=10, pady=10
            ).pack(fill=tk.X)

            tk.Label(
                transfer_window,
                text=f"يوجد {students_count} متدرب في هذا الفصل.\nاختر الفصل المراد ترحيل المتدربين إليه:",
                font=self.fonts["text"],
                bg=self.colors["light"],
                pady=10
            ).pack()

            # قائمة الفصول المتاحة للترحيل
            target_var = tk.StringVar()
            target_listbox = tk.Listbox(
                transfer_window,
                font=self.fonts["text"],
                selectbackground=self.colors["primary"],
                selectforeground="white",
                height=8
            )
            target_listbox.pack(fill=tk.X, padx=20, pady=10)

            # إضافة أسماء الفصول إلى القائمة
            for section in other_sections:
                target_listbox.insert(tk.END, section)

            # إذا كان هناك فصل واحد فقط، حدده تلقائيًا
            if len(other_sections) == 1:
                target_listbox.select_set(0)

            button_frame = tk.Frame(transfer_window, bg=self.colors["light"], pady=10)
            button_frame.pack(fill=tk.X, padx=20)

            def execute_transfer():
                """تنفيذ عملية الترحيل وحذف الفصل"""
                selected_indices = target_listbox.curselection()
                if not selected_indices:
                    messagebox.showwarning("تنبيه", "الرجاء اختيار فصل للترحيل إليه")
                    return

                target_section = target_listbox.get(selected_indices[0])

                try:
                    with self.conn:
                        # ترحيل المتدربين إلى الفصل المحدد
                        self.conn.execute("""
                            UPDATE student_sections
                            SET section_name=?
                            WHERE course_name=? AND section_name=?
                        """, (target_section, selected_course, selected_section))

                        # حذف الفصل
                        self.conn.execute("""
                            DELETE FROM course_sections
                            WHERE course_name=? AND section_name=?
                        """, (selected_course, selected_section))

                    messagebox.showinfo("نجاح",
                                        f"تم ترحيل {students_count} متدرب من الفصل '{selected_section}' إلى الفصل '{target_section}' وحذف الفصل بنجاح")
                    transfer_window.destroy()
                    update_sections_list()

                    # تحديث الإحصائيات بعد عملية الترحيل
                    self.update_statistics()
                    self.update_students_tree()
                    self.update_attendance_display()

                except Exception as e:
                    messagebox.showerror("خطأ", f"حدث خطأ أثناء الترحيل: {str(e)}")

            transfer_btn = tk.Button(
                button_frame,
                text="ترحيل وحذف",
                font=self.fonts["text_bold"],
                bg=self.colors["warning"],
                fg="white",
                padx=15, pady=5,
                bd=0, relief=tk.FLAT,
                cursor="hand2",
                command=execute_transfer
            )
            transfer_btn.pack(side=tk.LEFT)

            cancel_btn = tk.Button(
                button_frame,
                text="إلغاء",
                font=self.fonts["text_bold"],
                bg=self.colors["danger"],
                fg="white",
                padx=15, pady=5,
                bd=0, relief=tk.FLAT,
                cursor="hand2",
                command=transfer_window.destroy
            )
            cancel_btn.pack(side=tk.RIGHT)

        else:
            # إذا لم يكن هناك متدربين، يمكن حذف الفصل مباشرة
            if messagebox.askyesno("تأكيد", f"هل أنت متأكد من حذف الفصل '{selected_section}'؟"):
                try:
                    with self.conn:
                        self.conn.execute("""
                            DELETE FROM course_sections
                            WHERE course_name=? AND section_name=?
                        """, (selected_course, selected_section))

                    messagebox.showinfo("نجاح", f"تم حذف الفصل '{selected_section}' بنجاح")
                    update_sections_list()

                    # تحديث الإحصائيات بعد حذف الفصل
                    self.update_statistics()
                    self.update_students_tree()
                    self.update_attendance_display()

                except Exception as e:
                    messagebox.showerror("خطأ", f"حدث خطأ أثناء حذف الفصل: {str(e)}")

    # وظيفة حذف الدورة كاملة (للمشرفين فقط)
    def delete_entire_course():
        """حذف الدورة كاملة مع جميع الفصول والمتدربين"""
        if not self.current_user["permissions"]["is_admin"]:
            messagebox.showwarning("تنبيه", "هذه الوظيفة متاحة للمشرفين فقط")
            return

        selected_course = course_var.get().strip()
        if not selected_course:
            messagebox.showwarning("تنبيه", "الرجاء اختيار دورة للحذف")
            return

        # التأكيد قبل الحذف
        confirmation = messagebox.askquestion(
            "تحذير - حذف دورة كاملة",
            f"تحذير! أنت على وشك حذف الدورة '{selected_course}' بالكامل.\n\n"
            "سيؤدي هذا إلى:\n"
            "• حذف جميع الفصول في الدورة\n"
            "• حذف جميع المتدربين المرتبطين بالدورة\n"
            "• حذف جميع سجلات الحضور المرتبطة بالدورة\n\n"
            "هذا الإجراء لا يمكن التراجع عنه.\n\n"
            "هل أنت متأكد من رغبتك في حذف الدورة بالكامل؟",
            icon="warning",
            type="yesnocancel"
        )

        if confirmation != "yes":
            return

        # طلب كلمة مرور المشرف للتأكيد
        admin_password = simpledialog.askstring(
            "تأكيد حذف الدورة",
            "أدخل كلمة مرور المشرف للتأكيد:",
            show="*"
        )

        if not admin_password:
            return

        # التحقق من كلمة المرور
        hashed_password = hashlib.sha256(admin_password.encode()).hexdigest()
        cursor = self.conn.cursor()
        cursor.execute("SELECT password FROM users WHERE username=?", ("admin",))
        result = cursor.fetchone()

        if not result or result[0] != hashed_password:
            messagebox.showwarning("تنبيه", "كلمة المرور غير صحيحة")
            return

        # بدء عملية الحذف
        try:
            # إظهار نافذة تقدم العملية
            progress_window = tk.Toplevel(multi_window)
            progress_window.title("جاري حذف الدورة")
            progress_window.geometry("400x150")
            progress_window.configure(bg=self.colors["light"])
            progress_window.transient(multi_window)
            progress_window.grab_set()

            # توسيط النافذة
            x = (progress_window.winfo_screenwidth() - 400) // 2
            y = (progress_window.winfo_screenheight() - 150) // 2
            progress_window.geometry(f"400x150+{x}+{y}")

            tk.Label(
                progress_window,
                text=f"جاري حذف الدورة '{selected_course}'...",
                font=self.fonts["text_bold"],
                bg=self.colors["light"],
                pady=10
            ).pack()

            progress_var = tk.DoubleVar()
            progress_bar = ttk.Progressbar(
                progress_window,
                variable=progress_var,
                maximum=100,
                length=350
            )
            progress_bar.pack(pady=10)

            status_label = tk.Label(
                progress_window,
                text="جاري تحضير العملية...",
                font=self.fonts["text"],
                bg=self.colors["light"]
            )
            status_label.pack(pady=5)

            progress_window.update()

            # الحصول على جميع أرقام هويات المتدربين في الدورة
            cursor.execute("""
                SELECT national_id 
                FROM trainees 
                WHERE course=?
            """, (selected_course,))
            student_ids = [row[0] for row in cursor.fetchall()]

            total_steps = 3
            current_step = 0

            # 1. حذف سجلات الحضور
            progress_var.set((current_step / total_steps) * 100)
            status_label.config(text="جاري حذف سجلات الحضور...")
            progress_window.update()

            with self.conn:
                for student_id in student_ids:
                    self.conn.execute("""
                        DELETE FROM attendance 
                        WHERE national_id=?
                    """, (student_id,))

            current_step += 1
            progress_var.set((current_step / total_steps) * 100)
            status_label.config(text="جاري حذف سجلات الفصول...")
            progress_window.update()

            # 2. حذف سجلات الفصول
            with self.conn:
                self.conn.execute("""
                    DELETE FROM student_sections 
                    WHERE course_name=?
                """, (selected_course,))

                self.conn.execute("""
                    DELETE FROM course_sections 
                    WHERE course_name=?
                """, (selected_course,))

            current_step += 1
            progress_var.set((current_step / total_steps) * 100)
            status_label.config(text="جاري حذف بيانات المتدربين...")
            progress_window.update()

            # 3. حذف المتدربين
            with self.conn:
                self.conn.execute("""
                    DELETE FROM trainees 
                    WHERE course=?
                """, (selected_course,))

            current_step += 1
            progress_var.set(100)
            status_label.config(text="تم حذف الدورة بنجاح!")
            progress_window.update()

            # تحديث الإحصائيات بعد الحذف
            self.update_statistics()
            self.update_students_tree()
            self.update_attendance_display()

            # إغلاق نافذة التقدم بعد ثلاث ثوان
            progress_window.after(3000, progress_window.destroy)

            messagebox.showinfo("نجاح", f"تم حذف الدورة '{selected_course}' بنجاح مع جميع البيانات المرتبطة بها")

            # تحديث القائمة
            cursor.execute("SELECT DISTINCT course_name FROM course_sections")
            updated_courses = [row[0] for row in cursor.fetchall()]
            course_dropdown['values'] = updated_courses

            # مسح القيمة الحالية إذا تم حذفها
            if selected_course not in updated_courses:
                course_var.set("")

            # تحديث قائمة الفصول
            sections_listbox.delete(0, tk.END)
            section_title_var.set("اختر فصلاً لعرض تفاصيله")
            students_count_var.set("")

        except Exception as e:
            messagebox.showerror("خطأ", f"حدث خطأ أثناء حذف الدورة: {str(e)}")
            try:
                progress_window.destroy()
            except:
                pass

    # ربط وظيفة اختيار الفصل
    sections_listbox.bind("<<ListboxSelect>>", on_section_select)

    # ربط وظيفة تحديث القائمة بتغيير الدورة
    course_dropdown.bind("<<ComboboxSelected>>", lambda e: update_sections_list())

def open_section_students_window(self, course_name, section_name):
    """فتح نافذة إدارة متدربين الفصل"""
    students_window = tk.Toplevel(self.root)
    students_window.title(f"إدارة متدربين فصل {section_name} - {course_name}")
    students_window.geometry("900x600")
    students_window.configure(bg=self.colors["light"])
    students_window.grab_set()
    students_window.resizable(True, True)

    # توسيط النافذة
    x = (students_window.winfo_screenwidth() - 900) // 2
    y = (students_window.winfo_screenheight() - 600) // 2
    students_window.geometry(f"900x600+{x}+{y}")

    # عنوان النافذة
    tk.Label(
        students_window,
        text=f"إدارة متدربين فصل: {section_name} - دورة: {course_name}",
        font=self.fonts["title"],
        bg=self.colors["primary"],
        fg="white",
        padx=10, pady=10
    ).pack(fill=tk.X)

    # إطار البحث
    search_frame = tk.Frame(students_window, bg=self.colors["light"], padx=10, pady=10)
    search_frame.pack(fill=tk.X)

    tk.Label(
        search_frame,
        text="البحث عن متدرب:",
        font=self.fonts["text_bold"],
        bg=self.colors["light"]
    ).pack(side=tk.RIGHT, padx=5)

    search_var = tk.StringVar()
    search_entry = tk.Entry(
        search_frame,
        textvariable=search_var,
        font=self.fonts["text"],
        width=25
    )
    search_entry.pack(side=tk.RIGHT, padx=5)

    search_btn = tk.Button(
        search_frame,
        text="بحث",
        font=self.fonts["text_bold"],
        bg=self.colors["secondary"],
        fg="white",
        padx=10, pady=2,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=lambda: search_students()
    )
    search_btn.pack(side=tk.RIGHT, padx=5)

    # إطار القوائم المزدوجة
    lists_frame = tk.Frame(students_window, bg=self.colors["light"])
    lists_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=5)

    # قائمة المتدربين في الفصل
    section_frame = tk.LabelFrame(
        lists_frame,
        text=f"متدربين فصل {section_name}",
        font=self.fonts["text_bold"],
        bg=self.colors["light"],
        fg=self.colors["dark"],
        padx=5, pady=5
    )
    section_frame.pack(side=tk.LEFT, fill=tk.BOTH, expand=True, padx=(0, 5))

    section_scroll = tk.Scrollbar(section_frame)
    section_scroll.pack(side=tk.RIGHT, fill=tk.Y)

    section_students = tk.Listbox(
        section_frame,
        font=self.fonts["text"],
        selectbackground=self.colors["primary"],
        selectforeground="white",
        yscrollcommand=section_scroll.set
    )
    section_students.pack(fill=tk.BOTH, expand=True, pady=5)
    section_scroll.config(command=section_students.yview)

    # القائمة الوسطى للأزرار
    middle_frame = tk.Frame(lists_frame, bg=self.colors["light"], width=100)
    middle_frame.pack(side=tk.LEFT, fill=tk.Y, padx=5)

    move_to_other_btn = tk.Button(
        middle_frame,
        text=">>",
        font=self.fonts["text_bold"],
        bg=self.colors["primary"],
        fg="white",
        padx=5, pady=2,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=lambda: move_to_other_section()
    )
    move_to_other_btn.pack(pady=5)

    move_to_current_btn = tk.Button(
        middle_frame,
        text="<<",
        font=self.fonts["text_bold"],
        bg=self.colors["success"],
        fg="white",
        padx=5, pady=2,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=lambda: move_to_current_section()
    )
    move_to_current_btn.pack(pady=5)

    # قائمة المتدربين في الدورة بدون فصل أو في فصول أخرى
    other_frame = tk.LabelFrame(
        lists_frame,
        text="متدربين الدورة الآخرين",
        font=self.fonts["text_bold"],
        bg=self.colors["light"],
        fg=self.colors["dark"],
        padx=5, pady=5
    )
    other_frame.pack(side=tk.RIGHT, fill=tk.BOTH, expand=True, padx=(5, 0))

    other_scroll = tk.Scrollbar(other_frame)
    other_scroll.pack(side=tk.RIGHT, fill=tk.Y)

    other_students = tk.Listbox(
        other_frame,
        font=self.fonts["text"],
        selectbackground=self.colors["warning"],
        selectforeground="white",
        yscrollcommand=other_scroll.set
    )
    other_students.pack(fill=tk.BOTH, expand=True, pady=5)
    other_scroll.config(command=other_students.yview)

    # إطار المعلومات
    info_frame = tk.Frame(students_window, bg=self.colors["light"], padx=10, pady=5)
    info_frame.pack(fill=tk.X)

    section_count_var = tk.StringVar(value="عدد متدربين الفصل: 0")
    other_count_var = tk.StringVar(value="عدد المتدربين الآخرين: 0")

    section_count_label = tk.Label(
        info_frame,
        textvariable=section_count_var,
        font=self.fonts["text"],
        bg=self.colors["light"]
    )
    section_count_label.pack(side=tk.RIGHT, padx=10)

    other_count_label = tk.Label(
        info_frame,
        textvariable=other_count_var,
        font=self.fonts["text"],
        bg=self.colors["light"]
    )
    other_count_label.pack(side=tk.LEFT, padx=10)

    # إطار الأزرار السفلي
    button_frame = tk.Frame(students_window, bg=self.colors["light"], pady=10)
    button_frame.pack(fill=tk.X, padx=10)

    save_btn = tk.Button(
        button_frame,
        text="حفظ التغييرات",
        font=self.fonts["text_bold"],
        bg=self.colors["success"],
        fg="white",
        padx=15, pady=5,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=lambda: save_changes()
    )
    save_btn.pack(side=tk.LEFT, padx=5)

    close_btn = tk.Button(
        button_frame,
        text="إغلاق",
        font=self.fonts["text_bold"],
        bg=self.colors["dark"],
        fg="white",
        padx=15, pady=5,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=students_window.destroy
    )
    close_btn.pack(side=tk.RIGHT, padx=5)

    # حفظ التغييرات المؤقتة
    # القواميس تخزن: {الهوية: الاسم}
    current_section_students = {}  # المتدربين في الفصل الحالي
    other_section_students = {}  # المتدربين الآخرين
    modified = False  # هل تم تعديل البيانات

    # الوظائف المساعدة
    def load_students():
        """تحميل بيانات المتدربين"""
        nonlocal current_section_students, other_section_students

        # مسح القوائم
        section_students.delete(0, tk.END)
        other_students.delete(0, tk.END)
        current_section_students.clear()
        other_section_students.clear()

        cursor = self.conn.cursor()

        # 1. الحصول على متدربين الفصل الحالي
        cursor.execute("""
            SELECT t.national_id, t.name
            FROM trainees t
            JOIN student_sections s ON t.national_id = s.national_id
            WHERE t.course=? AND s.section_name=? AND t.is_excluded=0
            ORDER BY t.name
        """, (course_name, section_name))

        for row in cursor.fetchall():
            student_id, student_name = row
            display_text = f"{student_name} ({student_id})"
            section_students.insert(tk.END, display_text)
            current_section_students[student_id] = student_name

        # 2. الحصول على باقي متدربين الدورة (غير مسجلين في فصول أو في فصول أخرى)
        cursor.execute("""
            SELECT t.national_id, t.name, 
                   (SELECT section_name FROM student_sections 
                    WHERE national_id=t.national_id AND course_name=t.course) as section
            FROM trainees t
            WHERE t.course=? AND t.is_excluded=0
            ORDER BY t.name
        """, (course_name,))

        for row in cursor.fetchall():
            student_id, student_name, student_section = row

            # تخطي المتدربين في الفصل الحالي
            if student_section == section_name:
                continue

            # إضافة المتدربين الآخرين
            display_text = f"{student_name} ({student_id})"
            if student_section:
                display_text += f" - فصل: {student_section}"
            else:
                display_text += " - بدون فصل"

            other_students.insert(tk.END, display_text)
            other_section_students[student_id] = student_name

        # تحديث العدادات
        section_count_var.set(f"عدد متدربين الفصل: {len(current_section_students)}")
        other_count_var.set(f"عدد المتدربين الآخرين: {len(other_section_students)}")

    def search_students():
        """البحث عن متدربين"""
        search_text = search_var.get().strip()
        if not search_text:
            load_students()
            return

        # مسح القوائم
        section_students.delete(0, tk.END)
        other_students.delete(0, tk.END)

        # البحث في قائمة متدربين الفصل الحالي
        for student_id, student_name in current_section_students.items():
            if (search_text.lower() in student_name.lower() or
                    search_text in student_id):
                display_text = f"{student_name} ({student_id})"
                section_students.insert(tk.END, display_text)

        # البحث في قائمة المتدربين الآخرين
        for student_id, student_name in other_section_students.items():
            if (search_text.lower() in student_name.lower() or
                    search_text in student_id):
                display_text = f"{student_name} ({student_id})"

                # التحقق من وجود معلومات الفصل
                cursor = self.conn.cursor()
                cursor.execute("""
                    SELECT section_name FROM student_sections
                    WHERE national_id=? AND course_name=?
                """, (student_id, course_name))

                result = cursor.fetchone()

                if result and result[0]:
                    display_text += f" - فصل: {result[0]}"
                else:
                    display_text += " - بدون فصل"

                other_students.insert(tk.END, display_text)

    def move_to_other_section():
        """نقل المتدربين المحددين من الفصل الحالي إلى قائمة المتدربين الآخرين"""
        nonlocal modified

        selected_indices = section_students.curselection()
        if not selected_indices:
            return

        for index in reversed(selected_indices):
            student_text = section_students.get(index)
            student_id = extract_id_from_text(student_text)

            if student_id in current_section_students:
                student_name = current_section_students[student_id]

                # نقل المتدرب إلى القائمة الأخرى
                other_students.insert(tk.END, f"{student_name} ({student_id}) - بدون فصل")
                other_section_students[student_id] = student_name

                # حذف المتدرب من القائمة الحالية
                del current_section_students[student_id]
                section_students.delete(index)

                modified = True

        # تحديث العدادات
        section_count_var.set(f"عدد متدربين الفصل: {len(current_section_students)}")
        other_count_var.set(f"عدد المتدربين الآخرين: {len(other_section_students)}")

    def move_to_current_section():
        """نقل المتدربين المحددين من قائمة المتدربين الآخرين إلى الفصل الحالي"""
        nonlocal modified

        selected_indices = other_students.curselection()
        if not selected_indices:
            return

        for index in reversed(selected_indices):
            student_text = other_students.get(index)
            student_id = extract_id_from_text(student_text)

            if student_id in other_section_students:
                student_name = other_section_students[student_id]

                # نقل المتدرب إلى الفصل الحالي
                section_students.insert(tk.END, f"{student_name} ({student_id})")
                current_section_students[student_id] = student_name

                # حذف المتدرب من القائمة الأخرى
                del other_section_students[student_id]
                other_students.delete(index)

                modified = True

        # تحديث العدادات
        section_count_var.set(f"عدد متدربين الفصل: {len(current_section_students)}")
        other_count_var.set(f"عدد المتدربين الآخرين: {len(other_section_students)}")

    def extract_id_from_text(text):
        """استخراج رقم الهوية من النص المعروض"""
        # النص بشكل: "اسم المتدرب (رقم الهوية) - معلومات إضافية"
        try:
            start = text.find("(") + 1
            end = text.find(")")
            if start > 0 and end > start:
                return text[start:end]
        except:
            pass
        return ""

    def save_changes():
        """حفظ التغييرات في قاعدة البيانات"""
        nonlocal modified

        if not modified:
            messagebox.showinfo("معلومات", "لم يتم إجراء أي تغييرات")
            return

        # تحديث بيانات المتدربين في قاعدة البيانات
        current_date = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")

        try:
            cursor = self.conn.cursor()
            with self.conn:
                # 1. حذف كل المتدربين من الفصل الحالي
                self.conn.execute("""
                    DELETE FROM student_sections
                    WHERE course_name=? AND section_name=?
                """, (course_name, section_name))

                # 2. إضافة المتدربين الحاليين في الفصل
                for student_id in current_section_students:
                    self.conn.execute("""
                        INSERT OR REPLACE INTO student_sections
                        (national_id, course_name, section_name, assigned_date)
                        VALUES (?, ?, ?, ?)
                    """, (student_id, course_name, section_name, current_date))

            messagebox.showinfo("نجاح", "تم حفظ التغييرات بنجاح")
            modified = False
        except Exception as e:
            messagebox.showerror("خطأ", f"حدث خطأ أثناء حفظ التغييرات: {str(e)}")

    # تحميل بيانات المتدربين عند فتح النافذة
    load_students()

def export_section_to_word(self, course_name, section_name):
    """تصدير كشف حضور للفصل المحدد"""
    if not self.current_user["permissions"]["can_export_data"]:
        messagebox.showwarning("تنبيه", "ليس لديك صلاحية تصدير البيانات")
        return

    try:
        # التأكد من وجود مكتبة python-docx
        if 'Document' not in globals():
            messagebox.showerror("خطأ",
                                 "لم يتم العثور على مكتبة python-docx. قم بتثبيتها باستخدام: pip install python-docx")
            return

        # الحصول على بيانات المتدربين في الفصل المحدد (فقط غير المستبعدين)
        cursor = self.conn.cursor()
        cursor.execute("""
            SELECT t.national_id, t.name, t.rank
            FROM trainees t
            JOIN student_sections s ON t.national_id = s.national_id
            WHERE t.course=? AND s.section_name=? AND t.is_excluded=0
            ORDER BY t.name
        """, (course_name, section_name))
        students_data = cursor.fetchall()

        if not students_data:
            messagebox.showinfo("ملاحظة",
                                f"لا يوجد متدربين نشطين مسجلين في فصل '{section_name}' من دورة '{course_name}'")
            return

        # إنشاء مستند جديد
        doc = Document()

        # إعداد المستند للغة العربية (RTL) بتنسيق عمودي
        section = doc.sections[0]
        section.page_width = Inches(8.27)  # A4 width in portrait
        section.page_height = Inches(11.69)  # A4 height in portrait
        section.left_margin = Inches(0.5)
        section.right_margin = Inches(0.5)
        section.top_margin = Inches(0.7)
        section.bottom_margin = Inches(0.7)

        # إعداد الرأس (Header) مع خط فاصل
        header = section.header
        header_para = header.paragraphs[0]
        header_para.alignment = WD_ALIGN_PARAGRAPH.CENTER
        header_run = header_para.add_run(f'كشف حضور وغياب متدربين فصل: {section_name} - دورة: {course_name}')
        header_run.font.size = Pt(14)
        header_run.font.bold = True
        header_run.font.rtl = True

        # إضافة إجمالي عدد المتدربين في الرأس
        header_para = header.add_paragraph()
        header_para.alignment = WD_ALIGN_PARAGRAPH.CENTER
        student_count_run = header_para.add_run(f'إجمالي عدد المتدربين: {len(students_data)}')
        student_count_run.font.size = Pt(12)
        student_count_run.font.bold = True
        student_count_run.font.rtl = True

        # إضافة خط أفقي بعد معلومات الدورة في الرأس
        header_para.paragraph_format.border_bottom = True

        # إضافة تاريخ الطباعة في الرأس
        today_date = datetime.datetime.now().strftime("%Y-%m-%d")
        header_para = header.add_paragraph()
        header_para.alignment = WD_ALIGN_PARAGRAPH.LEFT
        header_date = header_para.add_run(f'تاريخ الطباعة: {today_date}')
        header_date.font.size = Pt(9)
        header_date.font.rtl = True

        # إعداد التذييل بشكل بسيط
        footer = section.footer
        footer_para = footer.paragraphs[0]
        footer_para.alignment = WD_ALIGN_PARAGRAPH.CENTER
        footer_text = footer_para.add_run('نظام إدارة الحضور والغياب - قسم شؤون المدربين')
        footer_text.font.size = Pt(9)
        footer_text.font.rtl = True

        # إضافة فقرة فاصلة قبل الجدول
        doc.add_paragraph()

        # إنشاء جدول للحضور والغياب
        table = doc.add_table(rows=1, cols=8)
        table.style = 'Table Grid'

        # تعريف رأس الجدول
        hdr_cells = table.rows[0].cells
        headers = ["العدد", "الاسم", "رقم الهوية", "الأحد", "الاثنين", "الثلاثاء", "الأربعاء", "الخميس"]

        # إضافة العناوين من اليمين إلى اليسار (عكس الترتيب)
        for i, header in enumerate(reversed(headers)):
            hdr_cells[i].text = header
            # تنسيق العناوين
            hdr_cells[i].paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.CENTER
            for run in hdr_cells[i].paragraphs[0].runs:
                run.font.bold = True
                run.font.size = Pt(11)
                run.font.rtl = True

            # تطبيق تظليل لرأس الجدول بطريقة بسيطة
            try:
                shading_elm = parse_xml(r'<w:shd {} w:fill="D9D9D9"/>'.format(nsdecls('w')))
                hdr_cells[i]._element.get_or_add_tcPr().append(shading_elm)
            except:
                # في حالة حدوث خطأ، نتجاهل التظليل
                pass

        # إضافة بيانات المتدربين
        for i, student in enumerate(students_data):
            national_id, name, rank = student
            row_cells = table.add_row().cells

            # إضافة البيانات من اليمين إلى اليسار (عكس الترتيب)
            # العدد (تسلسلي)
            row_cells[7].text = str(i + 1)
            row_cells[7].paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.CENTER

            # الاسم - تغيير المحاذاة إلى توسيط
            row_cells[6].text = name
            row_cells[6].paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.CENTER

            # رقم الهوية
            row_cells[5].text = national_id
            row_cells[5].paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.CENTER

            # الأيام تبقى فارغة للتعبئة يدوياً
            for day_idx in range(5):
                row_cells[day_idx].paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.CENTER

            # تنسيق النص في الصف
            for cell in row_cells:
                for paragraph in cell.paragraphs:
                    for run in paragraph.runs:
                        run.font.rtl = True
                        run.font.size = Pt(10)

        # ضبط أبعاد الجدول لتناسب التنسيق العمودي - زيادة عرض عمود الاسم
        table.autofit = False
        col_widths = [0.5, 2.6, 1.4, 0.7, 0.7, 0.7, 0.7, 0.7]  # زيادة عرض عمود الاسم (2.6 بدلاً من 2.0)

        # تطبيق العرض المحدد لكل عمود
        try:
            for i, width in enumerate(col_widths):
                table.columns[i].width = Inches(width)
        except:
            # في حالة حدوث خطأ، نتجاهل تعديل العرض
            pass

        # إضافة مساحة بعد الجدول
        doc.add_paragraph()

        # إضافة جدول للتوقيعات
        sig_table = doc.add_table(rows=1, cols=3)
        sig_table.style = 'Table Grid'
        sig_cells = sig_table.rows[0].cells

        sig_cells[2].text = "المسؤول: _________________"
        sig_cells[1].text = "رئيس القسم: ______________"
        sig_cells[0].text = "المدير: __________________"

        for cell in sig_cells:
            cell.paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.CENTER
            for run in cell.paragraphs[0].runs:
                run.font.rtl = True
                run.font.size = Pt(11)

        # إضافة ملاحظات في نهاية المستند
        doc.add_paragraph()
        notes_para = doc.add_paragraph()
        notes_para.alignment = WD_ALIGN_PARAGRAPH.RIGHT
        notes_para.add_run("ملاحظات:").bold = True

        # إضافة خطوط للملاحظات
        for _ in range(3):
            line_para = doc.add_paragraph("_" * 80)
            line_para.alignment = WD_ALIGN_PARAGRAPH.RIGHT

        # حفظ المستند
        export_file = filedialog.asksaveasfilename(
            defaultextension=".docx",
            filetypes=[("Word documents", "*.docx")],
            initialfile=f"كشف_حضور_{section_name}_{course_name}.docx"
        )

        if export_file:
            doc.save(export_file)
            messagebox.showinfo("نجاح", f"تم تصدير كشف الحضور لفصل '{section_name}' بنجاح إلى:\n{export_file}")
            # فتح الملف مباشرة بعد التصدير
            try:
                os.startfile(export_file)
            except:
                # في حالة عدم تمكن النظام من فتح الملف، تجاهل الخطأ
                pass

    except Exception as e:
        messagebox.showerror("خطأ", f"حدث خطأ أثناء تصدير بيانات الفصل: {str(e)}")

def export_section_diligence_behavior(self, course_name, section_name):
    """وظيفة تصدير بيان المواظبة والسلوك للفصل المحدد بتنسيق Word مع ترتيب المتدربين حسب الدرجة"""
    if not self.current_user["permissions"]["can_export_data"]:
        messagebox.showwarning("تنبيه", "ليس لديك صلاحية تصدير البيانات")
        return

    try:
        # التأكد من وجود مكتبة python-docx
        if 'Document' not in globals():
            messagebox.showerror("خطأ",
                                 "لم يتم العثور على مكتبة python-docx. قم بتثبيتها باستخدام: pip install python-docx")
            return

        # الحصول على بيانات المتدربين في الفصل المحدد (فقط غير المستبعدين)
        cursor = self.conn.cursor()
        cursor.execute("""
            SELECT t.national_id, t.name, t.rank
            FROM trainees t
            JOIN student_sections s ON t.national_id = s.national_id
            WHERE t.course=? AND s.section_name=? AND t.is_excluded=0
            ORDER BY t.name
        """, (course_name, section_name))
        students_data = cursor.fetchall()

        if not students_data:
            messagebox.showinfo("ملاحظة",
                                f"لا يوجد متدربين نشطين مسجلين في فصل '{section_name}' من دورة '{course_name}'")
            return

        # إنشاء نافذة حالة لإظهار تقدم التصدير
        progress_window = tk.Toplevel(self.root)
        progress_window.title("جاري حساب المواظبة والسلوك")
        progress_window.geometry("400x150")
        progress_window.configure(bg=self.colors["light"])
        progress_window.transient(self.root)
        progress_window.resizable(False, False)
        progress_window.grab_set()

        x = (progress_window.winfo_screenwidth() - 400) // 2
        y = (progress_window.winfo_screenheight() - 150) // 2
        progress_window.geometry(f"400x150+{x}+{y}")

        tk.Label(
            progress_window,
            text=f"جاري حساب نتائج المواظبة والسلوك لفصل: {section_name}",
            font=self.fonts["text_bold"],
            bg=self.colors["light"],
            pady=10
        ).pack()

        progress_var = tk.DoubleVar()
        progress_bar = ttk.Progressbar(
            progress_window,
            variable=progress_var,
            maximum=100,
            length=350
        )
        progress_bar.pack(pady=10)

        status_label = tk.Label(
            progress_window,
            text="جاري تحليل بيانات الحضور والغياب...",
            font=self.fonts["text"],
            bg=self.colors["light"]
        )
        status_label.pack(pady=5)

        progress_window.update()

        # إنشاء مستند جديد
        doc = Document()

        # إعداد المستند للغة العربية (RTL)
        section = doc.sections[0]
        section.page_width = Inches(8.27)  # A4 width
        section.page_height = Inches(11.69)  # A4 height
        section.left_margin = Inches(0.7)
        section.right_margin = Inches(0.7)
        section.top_margin = Inches(0.7)
        section.bottom_margin = Inches(0.7)

        # إضافة عنوان المستند
        title = doc.add_heading(f'بيان المواظبة والسلوك لمتدربين فصل: {section_name} - دورة: {course_name}',
                                level=0)
        title.alignment = WD_ALIGN_PARAGRAPH.CENTER
        for run in title.runs:
            run.font.size = Pt(16)
            run.font.bold = True
            run.font.rtl = True

        # إضافة معلومات الطباعة والتاريخ
        date_info = doc.add_paragraph()
        date_info.alignment = WD_ALIGN_PARAGRAPH.LEFT
        today_date = datetime.datetime.now().strftime("%Y-%m-%d")
        date_run = date_info.add_run(f'تاريخ الطباعة: {today_date}')
        date_run.font.size = Pt(10)
        date_run.font.rtl = True

        # إضافة خط أفقي
        border_paragraph = doc.add_paragraph()
        border_paragraph.paragraph_format.border_bottom = True

        # إنشاء جدول للمواظبة والسلوك
        table = doc.add_table(rows=1, cols=6)
        table.style = 'Table Grid'

        # عناوين الجدول (من اليمين إلى اليسار)
        hdr_cells = table.rows[0].cells
        headers = ["عدد", "الاسم", "الرتبة", "رقم الهوية", "المواظبة", "السلوك"]

        for i, header in enumerate(headers):
            # حساب الموقع المناسب للعناوين (من اليمين إلى اليسار)
            idx = len(headers) - i - 1
            hdr_cells[idx].text = header
            hdr_cells[idx].paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.CENTER
            for run in hdr_cells[idx].paragraphs[0].runs:
                run.font.bold = True
                run.font.size = Pt(12)
                run.font.rtl = True

            # تطبيق تظليل للرأس
            try:
                shading_elm = parse_xml(r'<w:shd {} w:fill="D9D9D9"/>'.format(nsdecls('w')))
                hdr_cells[idx]._element.get_or_add_tcPr().append(shading_elm)
            except:
                pass

        # معالجة بيانات كل متدرب وحساب درجة المواظبة
        student_scores = []
        total_students = len(students_data)

        for index, student in enumerate(students_data):
            national_id, name, rank = student

            # تحديث شريط التقدم
            progress_var.set((index / total_students) * 80)  # 80% للمعالجة
            status_label.config(text=f"معالجة المتدرب {index + 1} من {total_students}: {name}")
            progress_window.update()

            # حساب درجة المواظبة:
            # 1. الدرجة الأولية هي 100
            # 2. خصم 4 درجات لكل غياب كامل
            # 3. خصم 1 درجة لكل تأخير
            # 4. خصم 0.5 درجة لكل غياب بعذر

            # الاستعلام عن حالات الحضور للمتدرب
            cursor.execute("""
                SELECT status
                FROM attendance
                WHERE national_id=?
            """, (national_id,))
            attendance_records = cursor.fetchall()

            diligence_score = 100.0  # البداية من 100

            for record in attendance_records:
                status = record[0]
                if status == "غائب" or status == "غائب بعذر":  # تعديل: خصم 4 نقاط لـ "غائب بعذر"
                    diligence_score -= 4.0
                elif status == "متأخر":
                    diligence_score -= 1.0
                elif status == "حالة وفاة" or status == "منوم":  # إضافة: خصم 0.5 نقطة للحالات الجديدة
                    diligence_score -= 0.5

            # التأكد من عدم نزول الدرجة عن صفر
            diligence_score = max(0, diligence_score)

            # حفظ بيانات المتدرب مع الدرجة
            student_scores.append((national_id, name, rank, diligence_score))

        # ترتيب المتدربين تصاعدياً حسب درجة المواظبة (الأقل يأتي أولاً)
        student_scores.sort(key=lambda x: x[3])

        # إضافة بيانات المتدربين إلى الجدول بعد الترتيب
        for index, (national_id, name, rank, diligence_score) in enumerate(student_scores):
            # تحديث شريط التقدم
            progress_var.set(80 + (index / total_students) * 15)  # 15% للترتيب والإضافة
            status_label.config(text=f"إضافة المتدرب {index + 1} من {total_students} إلى التقرير")
            progress_window.update()

            # درجة السلوك دائمًا 100
            behavior_score = 100.0

            # إضافة صف جديد للمتدرب
            row_cells = table.add_row().cells

            # الترتيب من اليمين إلى اليسار
            row_cells[5].text = str(index + 1)  # العدد التسلسلي
            row_cells[4].text = name  # الاسم
            row_cells[3].text = rank  # الرتبة
            row_cells[2].text = national_id  # رقم الهوية
            row_cells[1].text = f"{diligence_score:.1f}"  # المواظبة بدقة رقم عشري واحد
            row_cells[0].text = f"{behavior_score:.0f}"  # السلوك (دائمًا 100)

            # تنسيق الخلايا
            for cell in row_cells:
                cell.paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.CENTER
                for run in cell.paragraphs[0].runs:
                    run.font.rtl = True
                    run.font.size = Pt(11)

            # تلوين الصف حسب درجة المواظبة
            if diligence_score < 90:  # إذا كانت الدرجة أقل من 90، تمييزها بلون فاتح
                try:
                    for cell in row_cells:
                        shading_elm = parse_xml(r'<w:shd {} w:fill="FFDDDD"/>'.format(nsdecls('w')))
                        cell._element.get_or_add_tcPr().append(shading_elm)
                except:
                    pass

        # تنسيق الجدول
        table.autofit = False
        try:
            # تعيين عرض الأعمدة (العرض بالبوصة)
            widths = [0.8, 0.8, 1.2, 1.5, 2.5, 0.5]  # السلوك، المواظبة، الهوية، الرتبة، الاسم، العدد
            for i, width in enumerate(widths):
                table.columns[i].width = Inches(width)
        except:
            pass

        # إضافة فقرة فاصلة بعد الجدول
        doc.add_paragraph()

        # إضافة جدول للتوقيعات
        signature_table = doc.add_table(rows=1, cols=3)
        signature_table.style = 'Table Grid'

        sig_cells = signature_table.rows[0].cells
        sig_cells[2].text = "مسؤول الحضور: _________________"
        sig_cells[1].text = "رئيس القسم: __________________"
        sig_cells[0].text = "مدير التدريب: ________________"

        for cell in sig_cells:
            cell.paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.CENTER
            for run in cell.paragraphs[0].runs:
                run.font.rtl = True
                run.font.size = Pt(11)

        # إضافة نص توضيحي في نهاية المستند
        doc.add_paragraph()
        note_para = doc.add_paragraph()
        note_para.alignment = WD_ALIGN_PARAGRAPH.RIGHT
        note_run = note_para.add_run("ملاحظات حساب المواظبة:")
        note_run.font.bold = True
        note_run.font.rtl = True

        notes = [
            "- تبدأ درجة المواظبة من 100 درجة.",
            "- يتم خصم 4 درجات عن كل يوم غياب.",
            "- يتم خصم 4 درجات عن كل غياب بعذر.",
            "- يتم خصم 1 درجة عن كل حالة تأخير.",
            "- يتم خصم 0.5 درجة عن كل حالة وفاة.",
            "- يتم خصم 0.5 درجة عن كل حالة منوم.",
            "- درجة السلوك 100 درجة للجميع."
        ]

        for note in notes:
            p = doc.add_paragraph()
            p.alignment = WD_ALIGN_PARAGRAPH.RIGHT
            p.add_run(note).font.rtl = True

        # تحديث شريط التقدم
        progress_var.set(95)
        status_label.config(text="فتح حوار حفظ الملف...")
        progress_window.update()

        # حفظ المستند
        export_file = filedialog.asksaveasfilename(
            defaultextension=".docx",
            filetypes=[("Word documents", "*.docx")],
            initialfile=f"بيان_المواظبة_والسلوك_{section_name}_{course_name}.docx"
        )

        if export_file:
            progress_var.set(95)
            status_label.config(text="جاري حفظ الملف...")
            progress_window.update()

            doc.save(export_file)

            progress_var.set(100)
            status_label.config(text="تم تصدير البيان بنجاح!")
            progress_window.update()

            # إغلاق نافذة التقدم بعد ثانيتين
            progress_window.after(2000, progress_window.destroy)

            messagebox.showinfo("نجاح",
                                f"تم تصدير بيان المواظبة والسلوك للفصل '{section_name}' بنجاح إلى:\n{export_file}")

            # محاولة فتح الملف تلقائيًا
            try:
                os.startfile(export_file)
            except:
                pass
        else:
            progress_window.destroy()

    except Exception as e:
        try:
            progress_window.destroy()
        except:
            pass
        messagebox.showerror("خطأ", f"حدث خطأ أثناء تصدير بيان المواظبة والسلوك: {str(e)}")

def backup_database(self):
    """إنشاء نسخة احتياطية من قاعدة البيانات"""
    if not self.current_user["permissions"]["is_admin"]:
        messagebox.showwarning("تنبيه", "هذه الوظيفة متاحة للمشرفين فقط")
        return

    import shutil
    import datetime
    import os

    # إنشاء مجلد للنسخ الاحتياطي إذا لم يكن موجوداً
    backup_dir = "backup"
    if not os.path.exists(backup_dir):
        os.makedirs(backup_dir)

    # إنشاء اسم ملف النسخة الاحتياطية مع التاريخ والوقت
    current_time = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
    backup_file = os.path.join(backup_dir, f"attendance_backup_{current_time}.db")

    # إغلاق الاتصال بقاعدة البيانات مؤقتاً
    self.conn.commit()

    try:
        # نسخ ملف قاعدة البيانات
        shutil.copy2("attendance.db", backup_file)
        messagebox.showinfo("نجاح", f"تم إنشاء نسخة احتياطية في: {backup_file}")
    except Exception as e:
        messagebox.showerror("خطأ", f"حدث خطأ أثناء إنشاء النسخة الاحتياطية: {str(e)}")

def restore_database(self):
    """استرداد نسخة احتياطية من قاعدة البيانات"""
    if not self.current_user["permissions"]["is_admin"]:
        messagebox.showwarning("تنبيه", "هذه الوظيفة متاحة للمشرفين فقط")
        return

    import os
    import shutil
    import sys
    import datetime

    # التحقق من وجود مجلد النسخ الاحتياطي
    backup_dir = "backup"
    if not os.path.exists(backup_dir):
        messagebox.showwarning("تنبيه", "لا يوجد مجلد للنسخ الاحتياطية")
        return

    # الحصول على قائمة ملفات النسخ الاحتياطية
    backup_files = [f for f in os.listdir(backup_dir) if f.startswith("attendance_backup_") and f.endswith(".db")]

    if not backup_files:
        messagebox.showwarning("تنبيه", "لا توجد نسخ احتياطية متاحة")
        return

    # إنشاء نافذة لاختيار النسخة الاحتياطية
    restore_window = tk.Toplevel(self.root)
    restore_window.title("استرداد نسخة احتياطية")
    restore_window.geometry("600x400")
    restore_window.configure(bg=self.colors["light"])
    restore_window.transient(self.root)
    restore_window.grab_set()

    # توسيط النافذة
    x = (restore_window.winfo_screenwidth() - 600) // 2
    y = (restore_window.winfo_screenheight() - 400) // 2
    restore_window.geometry(f"600x400+{x}+{y}")

    # عنوان النافذة
    tk.Label(
        restore_window,
        text="استرداد نسخة احتياطية من قاعدة البيانات",
        font=self.fonts["title"],
        bg=self.colors["primary"],
        fg="white",
        padx=10, pady=10
    ).pack(fill=tk.X)

    # إطار القائمة
    list_frame = tk.Frame(restore_window, bg=self.colors["light"])
    list_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)

    tk.Label(
        list_frame,
        text="اختر النسخة الاحتياطية التي تريد استردادها:",
        font=self.fonts["text_bold"],
        bg=self.colors["light"]
    ).pack(anchor=tk.W, pady=(0, 10))

    # إنشاء قائمة النسخ الاحتياطية
    backup_listbox = tk.Listbox(
        list_frame,
        font=self.fonts["text"],
        selectbackground=self.colors["primary"],
        selectforeground="white"
    )
    backup_listbox.pack(fill=tk.BOTH, expand=True, pady=(0, 10))

    # تعبئة القائمة بالملفات مرتبة من الأحدث إلى الأقدم
    backup_files.sort(reverse=True)
    for file in backup_files:
        # استخراج التاريخ والوقت من اسم الملف
        date_str = file.replace("attendance_backup_", "").replace(".db", "")
        try:
            # محاولة تنسيق التاريخ بشكل مقروء
            date_year = date_str[:4]
            date_month = date_str[4:6]
            date_day = date_str[6:8]
            time_hour = date_str[9:11]
            time_min = date_str[11:13]
            time_sec = date_str[13:15]
            display_date = f"{date_year}-{date_month}-{date_day} {time_hour}:{time_min}:{time_sec}"
        except:
            display_date = date_str

        backup_listbox.insert(tk.END, f"{display_date} - {file}")

    # إطار الأزرار
    button_frame = tk.Frame(restore_window, bg=self.colors["light"], pady=10)
    button_frame.pack(fill=tk.X, padx=10)

    # تخزين المتغيرات المطلوبة للدالة الداخلية
    def do_restore():
        """تنفيذ عملية استرداد النسخة الاحتياطية المحددة"""
        selection = backup_listbox.curselection()
        if not selection:
            messagebox.showwarning("تنبيه", "الرجاء اختيار نسخة احتياطية")
            return

        selected_item = backup_listbox.get(selection[0])
        backup_file = selected_item.split(" - ")[1]
        backup_path = os.path.join(backup_dir, backup_file)

        # التأكيد على استرداد النسخة الاحتياطية
        if not messagebox.askokcancel(
                "تأكيد الاسترداد",
                "سيتم استبدال قاعدة البيانات الحالية بالنسخة الاحتياطية المحددة.\n\n"
                "هذه العملية ستؤدي إلى فقدان أي تغييرات تمت منذ إنشاء هذه النسخة.\n\n"
                "هل أنت متأكد من المتابعة؟",
                icon="warning"
        ):
            return

        # إغلاق الاتصال بقاعدة البيانات
        self.conn.close()

        try:
            # إنشاء نسخة احتياطية إضافية من قاعدة البيانات الحالية (للأمان)
            current_time = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
            safety_backup = os.path.join(backup_dir, f"pre_restore_{current_time}.db")

            # نسخ ملف قاعدة البيانات الحالية كإجراء وقائي
            try:
                shutil.copy2("attendance.db", safety_backup)
            except:
                pass  # تجاهل الأخطاء في النسخ الوقائي

            # استبدال قاعدة البيانات الحالية بالنسخة الاحتياطية
            shutil.copy2(backup_path, "attendance.db")

            messagebox.showinfo(
                "نجاح",
                f"تم استرداد النسخة الاحتياطية بنجاح.\n\n"
                f"ستتم إعادة تشغيل النظام الآن لتطبيق التغييرات."
            )

            # إغلاق نافذة الاسترداد
            restore_window.destroy()

            # إعادة تشغيل البرنامج
            self.root.destroy()
            python = sys.executable
            os.execl(python, python, *sys.argv)

        except Exception as e:
            messagebox.showerror(
                "خطأ",
                f"حدث خطأ أثناء استرداد النسخة الاحتياطية:\n{str(e)}\n\n"
                "الرجاء إعادة تشغيل البرنامج يدوياً."
            )
            # محاولة إعادة فتح الاتصال بقاعدة البيانات في حال وجود خطأ
            try:
                self.conn = sqlite3.connect("attendance.db")
            except:
                pass

    restore_btn = tk.Button(
        button_frame,
        text="استرداد النسخة المحددة",
        font=self.fonts["text_bold"],
        bg=self.colors["warning"],
        fg="white",
        padx=15, pady=5,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=do_restore
    )
    restore_btn.pack(side=tk.LEFT, padx=5)

    cancel_btn = tk.Button(
        button_frame,
        text="إلغاء",
        font=self.fonts["text_bold"],
        bg=self.colors["dark"],
        fg="white",
        padx=15, pady=5,
        bd=0, relief=tk.FLAT,
        cursor="hand2",
        command=restore_window.destroy
    )
    cancel_btn.pack(side=tk.RIGHT, padx=5)

def export_student_absence_reports(self, student_info, attendance_records):
    """
    تصدير محاضر غيابات المتدرب (محضر منفصل لكل يوم غياب)

    Args:
        student_info: معلومات المتدرب (national_id, name, rank, course, ...)
        attendance_records: سجلات الحضور والغياب للمتدرب
    """
    try:
        # التأكد من وجود مكتبة python-docx
        if 'Document' not in globals():
            messagebox.showerror("خطأ",
                                 "لم يتم العثور على مكتبة python-docx. قم بتثبيتها باستخدام: pip install python-docx")
            return

        # استخراج المعلومات من البيانات
        nid = student_info[0]
        name = student_info[1]
        rank = student_info[2]
        course = student_info[3]

        # البحث عن سجلات الغياب فقط وترتيبها من الأقدم للأحدث
        absence_records = []
        for record in attendance_records:
            status = record[7]  # Status column
            if status == "غائب":
                absence_records.append(record)

        # ترتيب سجلات الغياب من الأقدم للأحدث
        absence_records = sorted(absence_records, key=lambda x: x[6])

        if not absence_records:
            messagebox.showinfo("معلومات", "لا توجد سجلات غياب لهذا المتدرب")
            return

        # إنشاء مستند جديد
        doc = Document()

        # إعداد المستند للغة العربية (RTL)
        section = doc.sections[0]
        section.page_width = Inches(8.5)
        section.page_height = Inches(11)
        section.left_margin = Inches(1.0)
        section.right_margin = Inches(1.0)
        section.top_margin = Inches(1.0)
        section.bottom_margin = Inches(1.0)

        # لكل سجل غياب، إنشاء صفحة جديدة
        for i, record in enumerate(absence_records):
            if i > 0:
                doc.add_page_break()

            # تاريخ الغياب
            absence_date = record[6]  # Date column

            # إضافة عنوان المستند
            title = doc.add_heading('محضر غياب', level=0)
            title.alignment = WD_ALIGN_PARAGRAPH.CENTER
            for run in title.runs:
                run.font.size = Pt(18)
                run.font.bold = True
                run.font.rtl = True

            # إضافة تاريخ الغياب تحت العنوان
            date_paragraph = doc.add_paragraph()
            date_paragraph.alignment = WD_ALIGN_PARAGRAPH.CENTER
            date_run = date_paragraph.add_run(f"تاريخ الغياب: {absence_date}")
            date_run.font.size = Pt(14)
            date_run.font.bold = True
            date_run.font.rtl = True

            # إضافة خط أفقي
            doc.add_paragraph().paragraph_format.border_bottom = True

            # إضافة جدول معلومات المتدرب
            doc.add_paragraph()  # فراغ قبل الجدول
            student_table = doc.add_table(rows=1, cols=4)
            student_table.style = 'Table Grid'

            # إضافة عناوين الجدول
            header_cells = student_table.rows[0].cells

            # نظراً لأن اللغة العربية RTL، نضيف العناوين بشكل معكوس
            header_cells[3].text = "الاسم"
            header_cells[2].text = "الرتبة"
            header_cells[1].text = "رقم الهوية"
            header_cells[0].text = "الدورة"

            # تنسيق عناوين الجدول
            for cell in header_cells:
                cell.paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.CENTER
                for paragraph in cell.paragraphs:
                    for run in paragraph.runs:
                        run.font.bold = True
                        run.font.rtl = True
                        run.font.size = Pt(12)

                # إضافة تظليل للرأس
                try:
                    shading_elm = parse_xml(r'<w:shd {} w:fill="DDDDDD"/>'.format(nsdecls('w')))
                    cell._element.get_or_add_tcPr().append(shading_elm)
                except:
                    pass

            # إضافة بيانات المتدرب
            data_row = student_table.add_row().cells

            # نضيف البيانات بشكل معكوس بسبب RTL
            data_row[3].text = name
            data_row[2].text = rank
            data_row[1].text = nid
            data_row[0].text = course

            # تنسيق بيانات المتدرب
            for cell in data_row:
                cell.paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.CENTER
                for paragraph in cell.paragraphs:
                    for run in paragraph.runs:
                        run.font.rtl = True
                        run.font.size = Pt(12)

            # ضبط عرض الجدول
            student_table.autofit = False
            try:
                col_widths = [2.5, 1.5, 1.5, 2.0]  # العرض بالبوصة (الدورة، الهوية، الرتبة، الاسم)
                for i, width in enumerate(col_widths):
                    student_table.columns[i].width = Inches(width)
            except:
                pass

            # إضافة نص المحضر
            doc.add_paragraph()  # فراغ بعد الجدول

            absence_paragraph = doc.add_paragraph()
            absence_paragraph.alignment = WD_ALIGN_PARAGRAPH.RIGHT
            absence_paragraph.paragraph_format.space_after = Pt(12)
            absence_text = f"أنه في يوم {self.get_arabic_day_name(absence_date)} الموافق {absence_date} قد تغيب المتدرب المذكور أعلاه عن دورة {course} وبناءً عليه أعد هذا المحضر للاطلاع."
            absence_run = absence_paragraph.add_run(absence_text)
            absence_run.font.rtl = True
            absence_run.font.size = Pt(12)

            # إضافة مكان للتوقيعات
            doc.add_paragraph()
            doc.add_paragraph()

            signature_table = doc.add_table(rows=1, cols=3)
            signature_table.style = 'Table Grid'

            sig_cells = signature_table.rows[0].cells
            sig_cells[2].text = "مسؤول الحضور: _________________"
            sig_cells[1].text = "رئيس القسم: __________________"
            sig_cells[0].text = "مدير التدريب: ________________"

            for cell in sig_cells:
                cell.paragraphs[0].alignment = WD_ALIGN_PARAGRAPH.CENTER
                for run in cell.paragraphs[0].runs:
                    run.font.rtl = True
                    run.font.size = Pt(11)

            # إضافة تاريخ الطباعة
            doc.add_paragraph()

            now = datetime.datetime.now()
            date_print_str = now.strftime("%Y-%m-%d")

            print_date = doc.add_paragraph()
            print_date.alignment = WD_ALIGN_PARAGRAPH.LEFT
            print_date_run = print_date.add_run(f"تاريخ الطباعة: {date_print_str}")
            print_date_run.font.size = Pt(9)
            print_date_run.font.rtl = True

        # حفظ المستند
        export_file = filedialog.asksaveasfilename(
            defaultextension=".docx",
            filetypes=[("Word documents", "*.docx")],
            initialfile=f"محاضر_غياب_{name}_{nid}.docx"
        )

        if export_file:
            doc.save(export_file)
            messagebox.showinfo("نجاح",
                                f"تم تصدير محاضر الغياب ({len(absence_records)} محضر) بنجاح إلى:\n{export_file}")

    except Exception as e:
        messagebox.showerror("خطأ", f"حدث خطأ أثناء تصدير محاضر الغياب: {str(e)}")

def get_arabic_day_name(self, date_str):
    """تحويل التاريخ إلى اسم اليوم بالعربية"""
    try:
        # تحويل النص إلى تاريخ
        date_obj = datetime.datetime.strptime(date_str, '%Y-%m-%d')

        # الحصول على رقم اليوم في الأسبوع (0=الاثنين، 6=الأحد)
        day_num = date_obj.weekday()

        # قائمة أيام الأسبوع بالعربية (مرتبة حسب نظام Python للأيام)
        arabic_days = ["الاثنين", "الثلاثاء", "الأربعاء", "الخميس", "الجمعة", "السبت", "الأحد"]

        return arabic_days[day_num]
    except:
        # في حال حدوث أي خطأ، نعيد نص التاريخ كما هو
        return date_str



    def add_students_to_existing_course(self):
        """إضافة متدربين جدد لدورة موجودة مع خيار لتعيين الفصل مباشرة"""
        if not self.current_user["permissions"]["can_add_students"]:
            messagebox.showwarning("تنبيه", "ليس لديك صلاحية إضافة متدربين جدد")
            return

        # إنشاء نافذة جديدة
        add_window = tk.Toplevel(self.root)
        add_window.title("إضافة متدربين جدد لدورة موجودة")
        add_window.geometry("800x600")
        add_window.configure(bg=self.colors["light"])
        add_window.transient(self.root)
        add_window.grab_set()

        # توسيط النافذة
        x = (add_window.winfo_screenwidth() - 800) // 2
        y = (add_window.winfo_screenheight() - 600) // 2
        add_window.geometry(f"800x600+{x}+{y}")

        # إضافة عنوان للنافذة
        tk.Label(
            add_window,
            text="إضافة متدربين جدد لدورة موجودة",
            font=self.fonts["title"],
            bg=self.colors["primary"],
            fg="white",
            padx=10, pady=10
        ).pack(fill=tk.X)

        # إطار اختيار الدورة والفصل
        course_frame = tk.Frame(add_window, bg=self.colors["light"], padx=20, pady=10)
        course_frame.pack(fill=tk.X)

        tk.Label(course_frame, text="اختر الدورة:", font=self.fonts["text_bold"], bg=self.colors["light"]).grid(row=0,
                                                                                                                column=3,
                                                                                                                padx=5,
                                                                                                                pady=8,
                                                                                                                sticky=tk.E)

        # الحصول على قائمة الدورات المتاحة
        cursor = self.conn.cursor()
        cursor.execute("SELECT DISTINCT course FROM trainees")
        courses = [row[0] for row in cursor.fetchall() if row[0]]

        course_var = tk.StringVar()
        course_combo = ttk.Combobox(
            course_frame,
            textvariable=course_var,
            values=courses,
            font=self.fonts["text"],
            width=30,
            state="readonly"
        )
        course_combo.grid(row=0, column=2, padx=5, pady=8, sticky=tk.W)

        # إضافة عنصر اختيار الفصل
        tk.Label(course_frame, text="اختر الفصل:", font=self.fonts["text_bold"], bg=self.colors["light"]).grid(row=0,
                                                                                                               column=1,
                                                                                                               padx=5,
                                                                                                               pady=8,
                                                                                                               sticky=tk.E)

        section_var = tk.StringVar()
        section_combo = ttk.Combobox(
            course_frame,
            textvariable=section_var,
            font=self.fonts["text"],
            width=20,
            state="readonly"
        )
        section_combo.grid(row=0, column=0, padx=5, pady=8, sticky=tk.W)

        # دالة لتحديث قائمة الفصول عند اختيار دورة
        def update_sections():
            selected_course = course_var.get()
            if not selected_course:
                section_combo['values'] = []
                section_var.set("")
                return

            # الحصول على قائمة الفصول المتاحة للدورة
            cursor = self.conn.cursor()
            cursor.execute("""
                SELECT section_name FROM course_sections
                WHERE course_name=?
                ORDER BY section_name
            """, (selected_course,))
            sections = [row[0] for row in cursor.fetchall()]

            # لا نضيف خيار "بدون فصل" للقائمة المنسدلة
            section_combo['values'] = sections

            if sections:
                section_combo.current(0)  # اختيار أول فصل كافتراضي
            else:
                # إذا لم تكن هناك فصول، اطلب من المستخدم إنشاء فصل أولاً
                messagebox.showwarning("تنبيه",
                                       "لا توجد فصول لهذه الدورة. الرجاء إنشاء فصل أولاً من خلال 'إدارة الفصول وتصدير الكشوفات'.")
                return

        # ربط وظيفة تحديث الفصول بتغيير الدورة
        course_combo.bind("<<ComboboxSelected>>", lambda e: update_sections())

        # إضافة إطار معلومات المتدربين
        info_frame = tk.Frame(add_window, bg=self.colors["light"], padx=20, pady=5)
        info_frame.pack(fill=tk.X)

        tk.Label(
            info_frame,
            text="معلومات المتدربين المضافين:",
            font=self.fonts["subtitle"],
            bg=self.colors["light"],
            fg=self.colors["primary"]
        ).pack(anchor=tk.W, pady=(0, 10))

        # إطار البحث
        search_frame = tk.Frame(add_window, bg=self.colors["light"])
        search_frame.pack(fill=tk.X, pady=5)

        tk.Label(search_frame, text="بحث بالاسم أو الهوية:", font=self.fonts["text_bold"],
                 bg=self.colors["light"]).pack(side=tk.RIGHT, padx=5)

        self.name_search_entry = tk.Entry(search_frame, font=self.fonts["text"], width=30, bd=2, relief=tk.GROOVE)
        self.name_search_entry.pack(side=tk.RIGHT, padx=5, fill=tk.X, expand=True)
        self.name_search_entry.bind("<KeyRelease>", self.dynamic_name_search)

        self.name_listbox = tk.Listbox(add_window, font=self.fonts["text"], height=4,
                                       selectbackground=self.colors["primary"], bd=2, relief=tk.GROOVE)
        self.name_listbox.pack(fill=tk.X, padx=10, pady=(0, 10))
        self.name_listbox.bind("<<ListboxSelect>>", self.on_name_select)

        input_frame = tk.Frame(add_window, bg=self.colors["light"])
        input_frame.pack(fill=tk.X, pady=5)

        self.id_entry = tk.Entry(self.root)

        # تعديل: إضافة إطار للأزرار في الأسفل
        buttons_frame = tk.Frame(add_window, bg=self.colors["light"], pady=20)
        buttons_frame.pack(fill=tk.X, padx=20)

        add_one_btn = tk.Button(
            buttons_frame,
            text="إضافة متدرب واحد",
            font=self.fonts["text_bold"],
            bg=self.colors["success"],
            fg="white",
            padx=10, pady=5,
            bd=0, relief=tk.FLAT,
            cursor="hand2",
            command=lambda: add_single_student(course_var.get(), section_var.get())
        )
        add_one_btn.pack(side=tk.RIGHT, padx=10)

        import_excel_btn = tk.Button(
            buttons_frame,
            text="استيراد من ملف Excel",
            font=self.fonts["text_bold"],
            bg=self.colors["secondary"],
            fg="white",
            padx=10, pady=5,
            bd=0, relief=tk.FLAT,
            cursor="hand2",
            command=lambda: import_from_excel(course_var.get(), section_var.get())
        )
        import_excel_btn.pack(side=tk.RIGHT, padx=10)

        close_btn = tk.Button(
            buttons_frame,
            text="إغلاق",
            font=self.fonts["text_bold"],
            bg=self.colors["dark"],
            fg="white",
            padx=10, pady=5,
            bd=0, relief=tk.FLAT,
            cursor="hand2",
            command=add_window.destroy
        )
        close_btn.pack(side=tk.LEFT, padx=10)

        # دالة إضافة متدرب واحد جديد للدورة
        def add_single_student(course_name, section_name):
            if not course_name:
                messagebox.showwarning("تنبيه", "الرجاء اختيار الدورة أولاً")
                return

            # استخدام نافذة إضافة متدرب موجودة مع إعداد اسم الدورة مسبقاً
            single_window = tk.Toplevel(add_window)
            single_window.title("إضافة متدرب جديد للدورة")
            single_window.geometry("400x350")
            single_window.configure(bg=self.colors["light"])
            single_window.transient(add_window)
            single_window.grab_set()

            x = (single_window.winfo_screenwidth() - 400) // 2
            y = (single_window.winfo_screenheight() - 350) // 2
            single_window.geometry(f"400x350+{x}+{y}")

            tk.Label(
                single_window,
                text=f"إضافة متدرب جديد للدورة: {course_name}",
                font=self.fonts["title"],
                bg=self.colors["primary"],
                fg="white",
                padx=10, pady=10, width=400
            ).pack(fill=tk.X)

            form_frame = tk.Frame(single_window, bg=self.colors["light"], padx=20, pady=20)
            form_frame.pack(fill=tk.BOTH)

            tk.Label(form_frame, text="رقم الهوية:", font=self.fonts["text_bold"], bg=self.colors["light"]).grid(row=0,
                                                                                                                 column=1,
                                                                                                                 padx=5,
                                                                                                                 pady=5,
                                                                                                                 sticky=tk.E)
            id_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25)
            id_entry.grid(row=0, column=0, padx=5, pady=5, sticky=tk.W)

            tk.Label(form_frame, text="الاسم:", font=self.fonts["text_bold"], bg=self.colors["light"]).grid(row=1,
                                                                                                            column=1,
                                                                                                            padx=5,
                                                                                                            pady=5,
                                                                                                            sticky=tk.E)
            name_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25)
            name_entry.grid(row=1, column=0, padx=5, pady=5, sticky=tk.W)

            tk.Label(form_frame, text="الرتبة:", font=self.fonts["text_bold"], bg=self.colors["light"]).grid(row=2,
                                                                                                             column=1,
                                                                                                             padx=5,
                                                                                                             pady=5,
                                                                                                             sticky=tk.E)
            rank_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25)
            rank_entry.grid(row=2, column=0, padx=5, pady=5, sticky=tk.W)

            tk.Label(form_frame, text="رقم الجوال:", font=self.fonts["text_bold"], bg=self.colors["light"]).grid(row=3,
                                                                                                                 column=1,
                                                                                                                 padx=5,
                                                                                                                 pady=5,
                                                                                                                 sticky=tk.E)
            phone_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25)
            phone_entry.grid(row=3, column=0, padx=5, pady=5, sticky=tk.W)

            button_frame = tk.Frame(single_window, bg=self.colors["light"], pady=10)
            button_frame.pack(fill=tk.X)

            def save_student():
                nid = id_entry.get().strip()
                name = name_entry.get().strip()
                rank_ = rank_entry.get().strip()
                phone = phone_entry.get().strip()
                selected_section = section_name  # الفصل المختار

                if not all([nid, name]):
                    messagebox.showwarning("تنبيه", "يجب إدخال رقم الهوية والاسم على الأقل")
                    return

                if not selected_section:
                    messagebox.showwarning("تنبيه", "يجب اختيار فصل للمتدرب")
                    return

                cursor = self.conn.cursor()
                cursor.execute("SELECT COUNT(*) FROM trainees WHERE national_id=?", (nid,))
                exists = cursor.fetchone()[0]

                if exists > 0:
                    # التحقق مما إذا كان المتدرب موجود في نفس الدورة
                    cursor.execute("SELECT course FROM trainees WHERE national_id=?", (nid,))
                    current_course = cursor.fetchone()[0]

                    if current_course == course_name:
                        messagebox.showwarning("تنبيه", f"رقم الهوية موجود بالفعل في نفس الدورة: {course_name}")
                        return

                    if not messagebox.askyesno("تأكيد الإضافة",
                                               f"المتدرب برقم الهوية {nid} موجود في دورة أخرى: {current_course}\n\nهل تريد نقله من الدورة السابقة إلى الدورة الجديدة: {course_name}؟"):
                        return

                    try:
                        # حذف المتدرب من الدورة القديمة
                        with self.conn:
                            # حذف سجلات الحضور للمتدرب
                            self.conn.execute("DELETE FROM attendance WHERE national_id=?", (nid,))
                            # حذف توزيع الفصول السابق
                            self.conn.execute("DELETE FROM student_sections WHERE national_id=?", (nid,))
                            # حذف المتدرب نفسه
                            self.conn.execute("DELETE FROM trainees WHERE national_id=?", (nid,))
                    except Exception as e:
                        messagebox.showerror("خطأ", f"حدث خطأ أثناء حذف السجل القديم: {str(e)}")
                        return

                current_date = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")

                try:
                    with self.conn:
                        # إضافة المتدرب للدورة
                        self.conn.execute("""
                            INSERT INTO trainees (national_id, name, rank, course, phone)
                            VALUES (?, ?, ?, ?, ?)
                        """, (nid, name, rank_, course_name, phone))

                        # إذا تم اختيار فصل (وليس "بدون فصل")، أضف المتدرب إلى الفصل
                        if section_name and section_name != "بدون فصل":
                            self.conn.execute("""
                                INSERT INTO student_sections 
                                (national_id, course_name, section_name, assigned_date)
                                VALUES (?, ?, ?, ?)
                            """, (nid, course_name, section_name, current_date))

                    messagebox.showinfo("نجاح", f"تم إضافة المتدرب {name} بنجاح" +
                                        (
                                            f" إلى فصل {section_name}" if section_name and section_name != "بدون فصل" else ""))
                    single_window.destroy()
                    self.update_students_tree()

                except Exception as e:
                    messagebox.showerror("خطأ", str(e))

            save_btn = tk.Button(button_frame, text="حفظ", font=self.fonts["text_bold"], bg=self.colors["success"],
                                 fg="white",
                                 padx=15, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=save_student)
            save_btn.pack(side=tk.LEFT, padx=10)

            cancel_btn = tk.Button(button_frame, text="إلغاء", font=self.fonts["text_bold"], bg=self.colors["danger"],
                                   fg="white",
                                   padx=15, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=single_window.destroy)
            cancel_btn.pack(side=tk.RIGHT, padx=10)

        def import_from_excel(course_name, section_name):
            """استيراد متدربين من ملف Excel مع إضافتهم إلى الفصل المحدد"""
            if not course_name:
                messagebox.showwarning("تنبيه", "الرجاء اختيار الدورة أولاً")
                return

            # اختيار ملف Excel
            file_path = filedialog.askopenfilename(
                title="اختر ملف Excel يحتوي على بيانات المتدربين",
                filetypes=[("Excel files", "*.xlsx"), ("Excel 97-2003", "*.xls"), ("All files", "*.*")]
            )

            if not file_path:
                return

            # التحقق من وجود مكتبة pandas
            try:
                import pandas as pd
            except ImportError:
                messagebox.showerror("خطأ",
                                     "يجب تثبيت مكتبة pandas لاستيراد ملفات Excel. استخدم الأمر: pip install pandas openpyxl")
                return

            try:
                # قراءة الملف
                df = pd.read_excel(file_path)

                # التحقق من وجود الأعمدة المطلوبة
                required_columns = {"national_id", "name"}
                found_columns = set(df.columns)

                # دعم أسماء الأعمدة بالعربية
                arabic_columns = {
                    "رقم الهوية": "national_id",
                    "الاسم": "name",
                    "الرتبة": "rank",
                    "رقم الجوال": "phone"
                }

                # تغيير الأسماء العربية إلى إنجليزية إذا وجدت
                rename_dict = {}
                for arabic, english in arabic_columns.items():
                    if arabic in df.columns:
                        rename_dict[arabic] = english

                if rename_dict:
                    df = df.rename(columns=rename_dict)
                    found_columns = set(df.columns)

                missing_columns = required_columns - found_columns
                if missing_columns:
                    messagebox.showerror("خطأ",
                                         f"الأعمدة التالية مفقودة في الملف: {', '.join(missing_columns)}\n\nيجب أن يحتوي الملف على عمود 'national_id' (رقم الهوية) وعمود 'name' (الاسم) على الأقل.")
                    return

                # إنشاء نافذة تأكيد المعاينة
                preview_window = tk.Toplevel(add_window)
                preview_window.title("معاينة بيانات المتدربين")
                preview_window.geometry("800x600")
                preview_window.configure(bg=self.colors["light"])
                preview_window.transient(add_window)
                preview_window.grab_set()

                tk.Label(
                    preview_window,
                    text=f"معاينة المتدربين المراد إضافتهم إلى دورة: {course_name}" +
                         (f" - فصل: {section_name}" if section_name and section_name != "بدون فصل" else ""),
                    font=self.fonts["title"],
                    bg=self.colors["primary"],
                    fg="white",
                    padx=10, pady=10
                ).pack(fill=tk.X)

                # عرض إحصائيات حول البيانات
                total_records = len(df)
                tk.Label(
                    preview_window,
                    text=f"إجمالي المتدربين في الملف: {total_records}",
                    font=self.fonts["text_bold"],
                    bg=self.colors["light"],
                    pady=5
                ).pack()

                # إطار لعرض معاينة البيانات
                preview_frame = tk.Frame(preview_window, bg=self.colors["light"])
                preview_frame.pack(fill=tk.BOTH, expand=True, padx=20, pady=5)

                preview_scroll = tk.Scrollbar(preview_frame)
                preview_scroll.pack(side=tk.RIGHT, fill=tk.Y)

                # شجرة للمعاينة
                preview_tree = ttk.Treeview(
                    preview_frame,
                    columns=["id", "national_id", "name", "rank", "phone", "status"],
                    show="headings",
                    yscrollcommand=preview_scroll.set,
                    style="Bold.Treeview"
                )

                preview_tree.column("id", width=50, anchor=tk.CENTER)
                preview_tree.column("national_id", width=150, anchor=tk.CENTER)
                preview_tree.column("name", width=200, anchor=tk.CENTER)
                preview_tree.column("rank", width=100, anchor=tk.CENTER)
                preview_tree.column("phone", width=120, anchor=tk.CENTER)
                preview_tree.column("status", width=150, anchor=tk.CENTER)

                preview_tree.heading("id", text="#")
                preview_tree.heading("national_id", text="رقم الهوية")
                preview_tree.heading("name", text="الاسم")
                preview_tree.heading("rank", text="الرتبة")
                preview_tree.heading("phone", text="رقم الجوال")
                preview_tree.heading("status", text="الحالة")

                preview_tree.pack(fill=tk.BOTH, expand=True)
                preview_scroll.config(command=preview_tree.yview)

                # إضافة الإمكانية لتمييز المتدربين حسب حالتهم
                preview_tree.tag_configure("new", background="#e8f5e9")  # خلفية خضراء فاتحة للمتدربين الجدد
                preview_tree.tag_configure("existing_same",
                                           background="#ffebee")  # خلفية حمراء فاتحة للمتدربين الموجودين في نفس الدورة
                preview_tree.tag_configure("existing_other",
                                           background="#fff8e1")  # خلفية صفراء فاتحة للمتدربين في دورات أخرى

                # العداد للمتدربين حسب حالتهم
                new_students = 0
                existing_same_course = 0
                existing_other_course = 0

                # تحضير بيانات المتدربين للعرض
                student_rows = []

                for i, row in df.iterrows():
                    national_id = str(row["national_id"]).strip()
                    name = str(row["name"]).strip()

                    # إذا كان الصف يحتوي على بيانات فارغة أو غير صالحة، تخطيه
                    if not national_id or not name:
                        continue

                    rank = str(row.get("rank", "")).strip() if "rank" in row else ""
                    phone = str(row.get("phone", "")).strip() if "phone" in row else ""

                    # التحقق من وجود المتدرب في قاعدة البيانات
                    cursor.execute("SELECT course FROM trainees WHERE national_id=?", (national_id,))
                    existing = cursor.fetchone()

                    status = ""
                    tag = ""

                    if existing:
                        existing_course = existing[0]
                        if existing_course == course_name:
                            status = f"موجود بالفعل في دورة: {existing_course}"
                            tag = "existing_same"
                            existing_same_course += 1
                        else:
                            status = f"موجود في دورة أخرى: {existing_course}"
                            tag = "existing_other"
                            existing_other_course += 1
                    else:
                        status = "جديد"
                        tag = "new"
                        new_students += 1

                    student_rows.append((i + 1, national_id, name, rank, phone, status, tag))

                # إضافة الصفوف إلى الشجرة
                for row in student_rows:
                    item_id = preview_tree.insert("", tk.END, values=row[:-1])
                    preview_tree.item(item_id, tags=(row[-1],))

                # إضافة ملخص الإحصائيات
                stats_frame = tk.Frame(preview_window, bg=self.colors["light"], padx=20, pady=10)
                stats_frame.pack(fill=tk.X)

                tk.Label(
                    stats_frame,
                    text=f"متدربين جدد: {new_students}",
                    font=self.fonts["text"],
                    bg="#e8f5e9", fg="black",
                    padx=10, pady=5
                ).pack(side=tk.RIGHT, padx=5)

                tk.Label(
                    stats_frame,
                    text=f"متدربين موجودون في نفس الدورة: {existing_same_course}",
                    font=self.fonts["text"],
                    bg="#ffebee", fg="black",
                    padx=10, pady=5
                ).pack(side=tk.RIGHT, padx=5)

                tk.Label(
                    stats_frame,
                    text=f"متدربين موجودون في دورات أخرى: {existing_other_course}",
                    font=self.fonts["text"],
                    bg="#fff8e1", fg="black",
                    padx=10, pady=5
                ).pack(side=tk.RIGHT, padx=5)

                # أزرار التأكيد أو الإلغاء
                button_frame = tk.Frame(preview_window, bg=self.colors["light"], pady=10)
                button_frame.pack(fill=tk.X, padx=10)

                # متغيرات الخيارات
                import_new_var = tk.IntVar(value=1)
                import_other_courses_var = tk.IntVar(value=0)
                import_same_course_var = tk.IntVar(value=0)

                # مربعات الاختيار
                tk.Checkbutton(
                    button_frame,
                    text="إضافة المتدربين الجدد",
                    variable=import_new_var,
                    font=self.fonts["text"],
                    bg=self.colors["light"]
                ).pack(anchor=tk.W)

                tk.Checkbutton(
                    button_frame,
                    text="إضافة المتدربين الموجودين في دورات أخرى (سيتم نقلهم)",
                    variable=import_other_courses_var,
                    font=self.fonts["text"],
                    bg=self.colors["light"]
                ).pack(anchor=tk.W)

                tk.Checkbutton(
                    button_frame,
                    text="تحديث بيانات المتدربين الموجودين في نفس الدورة",
                    variable=import_same_course_var,
                    font=self.fonts["text"],
                    bg=self.colors["light"]
                ).pack(anchor=tk.W)

                # إطار زر التنفيذ
                btn_frame = tk.Frame(preview_window, bg=self.colors["light"], pady=10)
                btn_frame.pack(fill=tk.X, padx=10)

                def execute_import():
                    # التحقق من تحديد خيار واحد على الأقل
                    if import_new_var.get() == 0 and import_other_courses_var.get() == 0 and import_same_course_var.get() == 0:
                        messagebox.showwarning("تنبيه", "الرجاء تحديد خيار واحد على الأقل للاستيراد")
                        return

                    # إنشاء نافذة تقدم العملية
                    progress_window = tk.Toplevel(preview_window)
                    progress_window.title("استيراد المتدربين")
                    progress_window.geometry("400x150")
                    progress_window.configure(bg=self.colors["light"])
                    progress_window.transient(preview_window)
                    progress_window.grab_set()

                    tk.Label(
                        progress_window,
                        text="جاري استيراد المتدربين...",
                        font=self.fonts["text_bold"],
                        bg=self.colors["light"],
                        pady=10
                    ).pack()

                    progress_var = tk.DoubleVar()
                    progress_bar = ttk.Progressbar(
                        progress_window,
                        variable=progress_var,
                        maximum=100,
                        length=350
                    )
                    progress_bar.pack(pady=10)

                    status_label = tk.Label(
                        progress_window,
                        text="جاري التحضير...",
                        font=self.fonts["text"],
                        bg=self.colors["light"]
                    )
                    status_label.pack(pady=5)

                    progress_window.update()

                    # حساب عدد العمليات المراد تنفيذها
                    operations_count = 0
                    if import_new_var.get() == 1:
                        operations_count += new_students
                    if import_other_courses_var.get() == 1:
                        operations_count += existing_other_course
                    if import_same_course_var.get() == 1:
                        operations_count += existing_same_course

                    current_date = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
                    operations_done = 0
                    imported_new = 0
                    imported_from_other = 0
                    updated_same = 0
                    errors = 0

                    try:
                        with self.conn:
                            # معالجة المتدربين حسب نوعهم
                            for row_data in student_rows:
                                _, national_id, name, rank, phone, _, tag = row_data

                                # تحديث شريط التقدم
                                progress_var.set((operations_done / max(1, operations_count)) * 100)
                                status_label.config(text=f"معالجة المتدرب: {name}")
                                progress_window.update()

                                # متدربين جدد
                                if tag == "new" and import_new_var.get() == 1:
                                    try:
                                        # إضافة المتدرب للدورة
                                        self.conn.execute("""
                                            INSERT INTO trainees (national_id, name, rank, course, phone)
                                            VALUES (?, ?, ?, ?, ?)
                                        """, (national_id, name, rank, course_name, phone))

                                        # إذا تم اختيار فصل (غير "بدون فصل")، أضف المتدرب للفصل
                                        if section_name and section_name != "بدون فصل":
                                            self.conn.execute("""
                                                INSERT INTO student_sections 
                                                (national_id, course_name, section_name, assigned_date)
                                                VALUES (?, ?, ?, ?)
                                            """, (national_id, course_name, section_name, current_date))

                                        imported_new += 1
                                    except Exception as e:
                                        print(f"خطأ في إضافة متدرب جديد: {str(e)}")
                                        errors += 1

                                # متدربين من دورات أخرى
                                elif tag == "existing_other" and import_other_courses_var.get() == 1:
                                    try:
                                        # حذف سجلات الحضور للمتدرب
                                        self.conn.execute("DELETE FROM attendance WHERE national_id=?", (national_id,))
                                        # حذف توزيع الفصول السابق
                                        self.conn.execute("DELETE FROM student_sections WHERE national_id=?",
                                                          (national_id,))
                                        # تحديث معلومات المتدرب مع الدورة الجديدة
                                        self.conn.execute("""
                                            UPDATE trainees
                                            SET name=?, rank=?, course=?, phone=?
                                            WHERE national_id=?
                                        """, (name, rank, course_name, phone, national_id))

                                        # إذا تم اختيار فصل (غير "بدون فصل")، أضف المتدرب للفصل
                                        if section_name and section_name != "بدون فصل":
                                            self.conn.execute("""
                                                INSERT INTO student_sections 
                                                (national_id, course_name, section_name, assigned_date)
                                                VALUES (?, ?, ?, ?)
                                            """, (national_id, course_name, section_name, current_date))

                                        imported_from_other += 1
                                    except Exception as e:
                                        print(f"خطأ في نقل متدرب من دورة أخرى: {str(e)}")
                                        errors += 1

                                # متدربين في نفس الدورة
                                elif tag == "existing_same" and import_same_course_var.get() == 1:
                                    try:
                                        # تحديث معلومات المتدرب
                                        self.conn.execute("""
                                            UPDATE trainees
                                            SET name=?, rank=?, phone=?
                                            WHERE national_id=?
                                        """, (name, rank, phone, national_id))

                                        # التحقق مما إذا كان المتدرب موجود في فصل حاليًا
                                        cursor.execute("""
                                            SELECT section_name FROM student_sections
                                            WHERE national_id=? AND course_name=?
                                        """, (national_id, course_name))
                                        current_section = cursor.fetchone()

                                        # إذا اختار المستخدم فصلًا غير "بدون فصل"
                                        if section_name and section_name != "بدون فصل":
                                            if current_section:
                                                # إذا كان المتدرب في فصل مختلف، حدّث الفصل
                                                if current_section[0] != section_name:
                                                    self.conn.execute("""
                                                        UPDATE student_sections
                                                        SET section_name=?, assigned_date=?
                                                        WHERE national_id=? AND course_name=?
                                                    """, (section_name, current_date, national_id, course_name))
                                            else:
                                                # إذا لم يكن المتدرب في أي فصل، أضفه إلى الفصل المحدد
                                                self.conn.execute("""
                                                    INSERT INTO student_sections 
                                                    (national_id, course_name, section_name, assigned_date)
                                                    VALUES (?, ?, ?, ?)
                                                """, (national_id, course_name, section_name, current_date))
                                        # إذا اختار المستخدم "بدون فصل" وكان المتدرب في فصل
                                        elif section_name == "بدون فصل" and current_section:
                                            # إزالة المتدرب من الفصل
                                            self.conn.execute("""
                                                DELETE FROM student_sections
                                                WHERE national_id=? AND course_name=?
                                            """, (national_id, course_name))

                                        updated_same += 1
                                    except Exception as e:
                                        print(f"خطأ في تحديث متدرب موجود: {str(e)}")
                                        errors += 1

                                operations_done += 1

                        # إظهار ملخص النتائج
                        progress_var.set(100)
                        status_label.config(text="تم استيراد المتدربين بنجاح!")
                        progress_window.update()

                        # إغلاق نافذة التقدم بعد ثانيتين
                        progress_window.after(2000, progress_window.destroy)

                        # عرض ملخص النتائج
                        summary = f"تم إكمال عملية الاستيراد بنجاح:\n\n"
                        if import_new_var.get() == 1:
                            summary += f"• تم إضافة {imported_new} متدرب جديد\n"
                        if import_other_courses_var.get() == 1:
                            summary += f"• تم نقل {imported_from_other} متدرب من دورات أخرى\n"
                        if import_same_course_var.get() == 1:
                            summary += f"• تم تحديث بيانات {updated_same} متدرب موجود\n"
                        if errors > 0:
                            summary += f"\nملاحظة: حدث {errors} أخطاء أثناء الاستيراد"

                        if section_name and section_name != "بدون فصل":
                            summary += f"\n\nتم توزيع المتدربين على فصل: {section_name}"

                        messagebox.showinfo("نتائج الاستيراد", summary)

                        # تحديث عرض المتدربين
                        self.update_students_tree()

                        # إغلاق نافذة المعاينة
                        preview_window.destroy()

                    except Exception as e:
                        # في حالة حدوث خطأ
                        try:
                            progress_window.destroy()
                        except:
                            pass
                        messagebox.showerror("خطأ", f"حدث خطأ أثناء الاستيراد: {str(e)}")

                confirm_btn = tk.Button(
                    btn_frame,
                    text="تنفيذ الاستيراد",
                    font=self.fonts["text_bold"],
                    bg=self.colors["success"],
                    fg="white",
                    padx=15, pady=5,
                    bd=0, relief=tk.FLAT,
                    cursor="hand2",
                    command=execute_import
                )
                confirm_btn.pack(side=tk.LEFT, padx=5)

                cancel_btn = tk.Button(
                    btn_frame,
                    text="إلغاء",
                    font=self.fonts["text_bold"],
                    bg=self.colors["danger"],
                    fg="white",
                    padx=15, pady=5,
                    bd=0, relief=tk.FLAT,
                    cursor="hand2",
                    command=preview_window.destroy
                )
                cancel_btn.pack(side=tk.RIGHT, padx=5)

            except Exception as e:
                messagebox.showerror("خطأ", f"حدث خطأ أثناء قراءة ملف Excel: {str(e)}")

        # تحديث قائمة الفصول عند فتح النافذة
        update_sections()

    def close_window(self):
        # إلغاء ربط الحدث قبل إغلاق النافذة
        try:
            self.name_search_entry.unbind("<KeyRelease>")
        except (tk.TclError, AttributeError):
            pass
        # إغلاق النافذة
        self.window.destroy()

    def is_widget_valid(self, widget):
        """التحقق من صلاحية عنصر واجهة المستخدم"""
        try:
            widget.winfo_exists()
            return True
        except (tk.TclError, AttributeError):
            return False

    def setup_students_tab(self):
        search_frame = tk.LabelFrame(self.students_tab, text="البحث عن متدرب", font=self.fonts["subtitle"],
                                     bg=self.colors["light"], fg=self.colors["dark"], padx=10, pady=10)
        search_frame.pack(fill=tk.X, padx=10, pady=5)

        search_inner_frame = tk.Frame(search_frame, bg=self.colors["light"])
        search_inner_frame.pack(fill=tk.X, padx=5, pady=5)

        # تعديل: تغيير القائمة المنسدلة لعرض جميع المتدربين والمتدربين المستبعدين
        tk.Label(search_inner_frame, text="عرض:", font=self.fonts["text_bold"],
                 bg=self.colors["light"]).pack(side=tk.RIGHT, padx=5)

        self.course_type_var = tk.StringVar(value="جميع المتدربين")
        course_type_combo = ttk.Combobox(
            search_inner_frame,
            textvariable=self.course_type_var,
            values=["جميع المتدربين", "المتدربين النشطين", "المتدربين المستبعدين"],  # تغيير الخيارات
            state="readonly",
            width=20,
            font=self.fonts["text"]
        )
        course_type_combo.pack(side=tk.RIGHT, padx=5)
        course_type_combo.bind("<<ComboboxSelected>>", lambda e: self.update_students_tree())

        tk.Label(search_inner_frame, text="بحث (الاسم أو الهوية):", font=self.fonts["text_bold"],
                 bg=self.colors["light"]).pack(side=tk.RIGHT, padx=5)
        self.search_entry = tk.Entry(search_inner_frame, font=self.fonts["text"], width=30, bd=2, relief=tk.GROOVE)
        self.search_entry.pack(side=tk.RIGHT, padx=5)
        self.search_entry.bind("<Return>", lambda e: self.search_student())

        search_button = tk.Button(
            search_inner_frame, text="بحث", font=self.fonts["text_bold"], bg=self.colors["primary"], fg="white",
            padx=10, pady=3, bd=0, relief=tk.FLAT, command=self.search_student, cursor="hand2"
        )
        search_button.pack(side=tk.RIGHT, padx=5)

        show_all_button = tk.Button(
            search_inner_frame, text="عرض الكل", font=self.fonts["text_bold"], bg=self.colors["secondary"], fg="white",
            padx=10, pady=3, bd=0, relief=tk.FLAT, command=self.update_students_tree, cursor="hand2"
        )
        show_all_button.pack(side=tk.LEFT, padx=5)

        button_frame = tk.Frame(search_frame, bg=self.colors["light"])
        button_frame.pack(fill=tk.X, padx=5, pady=5)

        add_to_course_btn = tk.Button(
            button_frame,
            text="إضافة متدربين جدد لدورة موجودة",
            font=self.fonts["text_bold"],
            bg="#3949AB",  # لون مميز
            fg="white",
            padx=10, pady=3,
            bd=0, relief=tk.FLAT,
            cursor="hand2",
            command=self.add_students_to_existing_course
        )
        add_to_course_btn.pack(side=tk.LEFT, padx=5)

        if self.current_user["permissions"]["can_add_students"]:
            add_button = tk.Button(
                button_frame, text="إضافة متدرب جديد", font=self.fonts["text_bold"], bg=self.colors["success"],
                fg="white",
                padx=10, pady=3, bd=0, relief=tk.FLAT, command=self.add_new_student, cursor="hand2"
            )
            add_button.pack(side=tk.RIGHT, padx=5)

        if self.current_user["permissions"]["can_edit_students"]:
            edit_button = tk.Button(
                button_frame, text="تعديل المتدرب المحدد", font=self.fonts["text_bold"], bg=self.colors["late"],
                fg="white",
                padx=10, pady=3, bd=0, relief=tk.FLAT, command=lambda: self.edit_student(from_selection=True),
                cursor="hand2"
            )
            edit_button.pack(side=tk.RIGHT, padx=5)

        if self.current_user["permissions"]["can_delete_students"]:
            delete_button = tk.Button(
                button_frame, text="حذف المتدرب المحدد", font=self.fonts["text_bold"], bg=self.colors["danger"],
                fg="white",
                padx=10, pady=3, bd=0, relief=tk.FLAT, command=self.delete_selected_student, cursor="hand2"
            )
            delete_button.pack(side=tk.RIGHT, padx=5)

            multi_courses_button = tk.Button(
                button_frame, text="إدارة الفصول وتصدير الكشوفات", font=self.fonts["text_bold"], bg="#4285f4",
                # لون أزرق أكثر بروزًا
                fg="white",
                padx=10, pady=3, bd=0, relief=tk.FLAT, cursor="hand2", command=self.manage_multi_section_courses
            )
            multi_courses_button.pack(side=tk.LEFT, padx=5)

        if self.current_user["permissions"]["can_import_data"]:
            import_course_button = tk.Button(
                button_frame, text="استيراد دورة جديدة", font=self.fonts["text_bold"], bg=self.colors["secondary"],
                fg="white",
                padx=10, pady=3, bd=0, relief=tk.FLAT, command=self.import_new_course, cursor="hand2"
            )
            import_course_button.pack(side=tk.LEFT, padx=5)

        view_profile_button = tk.Button(
            button_frame, text="عرض ملف المتدرب", font=self.fonts["text_bold"], bg=self.colors["secondary"], fg="white",
            padx=10, pady=3, bd=0, relief=tk.FLAT, command=self.view_student_profile, cursor="hand2"
        )
        view_profile_button.pack(side=tk.RIGHT, padx=5)

        students_display_frame = tk.LabelFrame(self.students_tab, text="قائمة المتدربين", font=self.fonts["subtitle"],
                                               bg=self.colors["light"], fg=self.colors["dark"], padx=10, pady=10)
        students_display_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=5)

        self.students_tree_scroll = tk.Scrollbar(students_display_frame)
        self.students_tree_scroll.pack(side=tk.RIGHT, fill=tk.Y)

        # إضافة عمود لإظهار حالة الاستبعاد
        self.students_tree = ttk.Treeview(
            students_display_frame,
            columns=("id", "name", "rank", "course", "phone", "status"),
            show="headings",
            yscrollcommand=self.students_tree_scroll.set,
            style="Bold.Treeview"
        )
        self.students_tree.column("id", width=120, anchor=tk.CENTER)
        self.students_tree.column("name", width=150, anchor=tk.CENTER)
        self.students_tree.column("rank", width=80, anchor=tk.CENTER)
        self.students_tree.column("course", width=150, anchor=tk.CENTER)
        self.students_tree.column("phone", width=120, anchor=tk.CENTER)
        self.students_tree.column("status", width=80, anchor=tk.CENTER)

        self.students_tree.heading("id", text="رقم الهوية")
        self.students_tree.heading("name", text="الاسم")
        self.students_tree.heading("rank", text="الرتبة")
        self.students_tree.heading("course", text="اسم الدورة")
        self.students_tree.heading("phone", text="رقم الجوال")
        self.students_tree.heading("status", text="الحالة")

        self.students_tree.pack(fill=tk.BOTH, expand=True)
        self.students_tree_scroll.config(command=self.students_tree.yview)

        # إضافة نمط للمتدربين المستبعدين
        self.students_tree.tag_configure("excluded", background="#f8bbd0")

        self.students_tree.bind("<Double-1>", self.on_student_double_click)
