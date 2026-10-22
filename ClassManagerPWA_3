(function () {
  "use strict";

  var DB_KEY = "classManagerPwa.v1";
  var SETTINGS_KEY = "classManagerPwa.settings.v1";

  function emptyDB() {
    return { classes: [], students: [], attendance: [], grades: [], positives: [], negatives: [], homework: [], lessons: [] };
  }
  function defaultSettings() {
    return { theme: "light", font: "Vazirmatn", fontSize: "14", calendar: "jalali" };
  }

  var db = loadDB();
  var settings = loadSettings();
  var deferredInstall = null;
  var dialogSave = null;

  function $(id) { return document.getElementById(id); }
  function faNum(n) { return Number(n || 0).toLocaleString("fa-IR"); }
  function pad2(n) { return String(n).padStart(2, "0"); }
  function today() {
    var d = new Date();
    return d.getFullYear() + "-" + pad2(d.getMonth() + 1) + "-" + pad2(d.getDate());
  }
  function esc(s) {
    return String(s == null ? "" : s).replace(/[&<>"']/g, function (c) {
      return { "&": "&amp;", "<": "&lt;", ">": "&gt;", "\u0022": "&quot;", "\u0027": "&#39;" }[c];
    });
  }
  function normDig(s) {
    return String(s == null ? "" : s)
      .replace(/[\u06F0-\u06F9]/g, function (d) { return "\u06F0\u06F1\u06F2\u06F3\u06F4\u06F5\u06F6\u06F7\u06F8\u06F9".indexOf(d); })
      .replace(/[\u0660-\u0669]/g, function (d) { return "\u0660\u0661\u0662\u0663\u0664\u0665\u0666\u0667\u0668\u0669".indexOf(d); });
  }

  function g2j(gy, gm, gd) {
    var m = [0,31,59,90,120,151,181,212,243,273,304,334];
    var jy = (gy <= 1600) ? 0 : 979;
    gy -= (gy <= 1600) ? 621 : 1600;
    var gy2 = (gm > 2) ? (gy + 1) : gy;
    var days = 365 * gy + Math.floor((gy2 + 3) / 4) - Math.floor((gy2 + 99) / 100)
             + Math.floor((gy2 + 399) / 400) - 80 + gd + m[gm - 1];
    jy += 33 * Math.floor(days / 12053); days %= 12053;
    jy += 4 * Math.floor(days / 1461); days %= 1461;
    if (days > 365) { jy += Math.floor((days - 1) / 365); days = (days - 1) % 365; }
    var jm = (days < 186) ? 1 + Math.floor(days / 31) : 7 + Math.floor((days - 186) / 30);
    var jd = 1 + ((days < 186) ? (days % 31) : ((days - 186) % 30));
    return [jy, jm, jd];
  }
  function j2g(jy, jm, jd) {
    var gy = (jy <= 979) ? 621 : 1600;
    jy -= (jy <= 979) ? 0 : 979;
    var days = 365 * jy + Math.floor(jy / 33) * 8 + Math.floor(((jy % 33) + 3) / 4)
             + 78 + jd + ((jm < 7) ? (jm - 1) * 31 : ((jm - 7) * 30) + 186);
    gy += 400 * Math.floor(days / 146097); days %= 146097;
    if (days > 36524) { gy += 100 * Math.floor(--days / 36524); days %= 36524; if (days >= 365) days++; }
    gy += 4 * Math.floor(days / 1461); days %= 1461;
    if (days > 365) { gy += Math.floor((days - 1) / 365); days = (days - 1) % 365; }
    var gd = days + 1;
    var sa = [0,31,((gy % 4 === 0 && gy % 100 !== 0) || gy % 400 === 0) ? 29 : 28,31,30,31,30,31,31,30,31,30,31];
    var gm = 0;
    for (gm = 0; gm < 13; gm++) { var v = sa[gm]; if (gd <= v) break; gd -= v; }
    return [gy, gm, gd];
  }
  var J_M = ["فروردین","اردیبهشت","خرداد","تیر","مرداد","شهریور","مهر","آبان","آذر","دی","بهمن","اسفند"];
  var G_M = ["ژانویه","فوریه","مارس","آپریل","مه","ژوئن","جولای","اوت","سپتامبر","اکتبر","نوامبر","دسامبر"];
  var WD = ["یکشنبه","دوشنبه","سه‌شنبه","چهارشنبه","پنجشنبه","جمعه","شنبه"];

  function parts(iso) {
    if (!iso) return null;
    var p = String(iso).split("-");
    return p.length === 3 ? [parseInt(p[0],10), parseInt(p[1],10), parseInt(p[2],10)] : null;
  }
  function fmtDate(iso) {
    var p = parts(iso); if (!p) return iso || "";
    if (settings.calendar === "gregorian") return p[2] + " " + G_M[p[1] - 1] + " " + p[0];
    var j = g2j(p[0], p[1], p[2]);
    return j[2] + " " + J_M[j[1] - 1] + " " + j[0];
  }
  function todayFa() {
    var t = today(), p = parts(t), wd = WD[new Date(t).getDay()];
    if (settings.calendar === "gregorian") return wd + " " + p[2] + " " + G_M[p[1] - 1] + " " + p[0];
    var j = g2j(p[0], p[1], p[2]);
    return wd + " " + j[2] + " " + J_M[j[1] - 1] + " " + j[0];
  }
  function isoToInput(iso) {
    if (!iso) return "";
    var p = parts(iso); if (!p) return iso;
    if (settings.calendar === "gregorian") return iso;
    var j = g2j(p[0], p[1], p[2]);
    return j[0] + "/" + pad2(j[1]) + "/" + pad2(j[2]);
  }
  function parseDate(s) {
    if (!s) return null;
    var v = normDig(String(s).trim());
    var m = v.match(/^(\d{4})[\/\-\.](\d{1,2})[\/\-\.](\d{1,2})$/);
    if (!m) return null;
    var y = +m[1], mo = +m[2], d = +m[3];
    if (settings.calendar === "gregorian") {
      if (y < 1000 || y > 3000 || mo < 1 || mo > 12 || d < 1 || d > 31) return null;
      return y + "-" + pad2(mo) + "-" + pad2(d);
    }
    if (y < 1200 || y > 1600 || mo < 1 || mo > 12 || d < 1 || d > 31) return null;
    var g = j2g(y, mo, d);
    return g[0] + "-" + pad2(g[1]) + "-" + pad2(g[2]);
  }
  function datePH() { return settings.calendar === "gregorian" ? "YYYY-MM-DD" : "۱۴۰۳/۰۵/۱۵"; }

  function loadDB() {
    try {
      var raw = localStorage.getItem(DB_KEY);
      if (!raw) return emptyDB();
      var p = JSON.parse(raw), b = emptyDB();
      for (var k in p) if (p.hasOwnProperty(k)) b[k] = p[k];
      if (!Array.isArray(b.positives)) b.positives = [];
      if (!Array.isArray(b.negatives)) b.negatives = [];
      if (!Array.isArray(b.homework)) b.homework = [];
      return b;
    } catch (e) { return emptyDB(); }
  }
  function saveDB() {
    try { localStorage.setItem(DB_KEY, JSON.stringify(db)); renderAll(); }
    catch (e) { toast("ذخیره‌سازی انجام نشد"); }
  }
  function loadSettings() {
    var d = defaultSettings();
    try {
      var raw = localStorage.getItem(SETTINGS_KEY);
      if (!raw) return d;
      var p = JSON.parse(raw);
      for (var k in d) if (p[k] != null) d[k] = p[k];
      return d;
    } catch (e) { return d; }
  }
  function saveSettings() {
    try { localStorage.setItem(SETTINGS_KEY, JSON.stringify(settings)); } catch (e) {}
  }

  function applyDisplay() {
    var root = document.documentElement;
    var ap = settings.theme;
    if (settings.theme === "auto") {
      ap = (window.matchMedia && window.matchMedia("(prefers-color-scheme: dark)").matches) ? "dark" : "light";
    }
    root.setAttribute("data-theme", ap);
    var fm = {
      Vazirmatn: "'Vazirmatn',Tahoma,Arial,sans-serif",
      Sahel: "'Sahel',Tahoma,Arial,sans-serif",
      Samim: "'Samim',Tahoma,Arial,sans-serif",
      Shabnam: "'Shabnam',Tahoma,Arial,sans-serif",
      Estedad: "'Estedad',Tahoma,Arial,sans-serif",
      Nazanin: "'Nazanin','B Nazanin',Tahoma,serif",
      Tahoma: "Tahoma,Arial,sans-serif",
      Arial: "Arial,sans-serif",
      system: "system-ui,-apple-system,'Segoe UI',sans-serif"
    };
    root.style.setProperty("--font-family", fm[settings.font] || fm.Vazirmatn);
    var sm = { "13": "13px", "14": "14px", "16": "16px", "18": "18px" };
    root.style.setProperty("--base-font", sm[settings.fontSize] || "14px");
    if ($("setTheme")) $("setTheme").value = settings.theme;
    if ($("setFont")) $("setFont").value = settings.font;
    if ($("setFontSize")) $("setFontSize").value = settings.fontSize;
    if ($("setCalendar")) $("setCalendar").value = settings.calendar;
    syncAttDate();
  }

  function toast(m) {
    var e = $("toast"); e.textContent = m; e.classList.add("show");
    clearTimeout(toast.t);
    toast.t = setTimeout(function () { e.classList.remove("show"); }, 2600);
  }
  function empty(m, s) { return '<div class="empty"><strong>' + esc(m) + '</strong>' + esc(s || "") + '</div>'; }
  function className(id) { var c = db.classes.find(function (x) { return x.id === id; }); return c ? c.name : "بدون کلاس"; }
  function studentName(id) { var s = db.students.find(function (x) { return x.id === id; }); return s ? s.name : "حذف‌شده"; }
  function findClass(n) {
    n = String(n || "").trim(); if (!n) return null;
    return db.classes.find(function (c) { return c.name.trim().toLowerCase() === n.toLowerCase(); }) || null;
  }
  function ensureClass(n) {
    var c = findClass(n); if (c) return c.id;
    if (!n) return db.classes[0] ? db.classes[0].id : null;
    var i = { id: makeId(), name: n, subject: "", note: "" };
    db.classes.push(i); return i.id;
  }
  function makeId() {
    if (window.crypto && crypto.randomUUID) return crypto.randomUUID();
    return Date.now() + "-" + Math.random().toString(16).slice(2);
  }
  function sumPos(sid) {
    return db.positives.filter(function (p) { return p.studentId === sid; })
      .reduce(function (a, b) { return a + Number(b.points || 0); }, 0);
  }

  function nav(p) {
    var tabs = document.querySelectorAll(".tab");
    for (var i = 0; i < tabs.length; i++) {
      tabs[i].classList.toggle("active", tabs[i].getAttribute("data-page") === p);
    }
    var pages = document.querySelectorAll(".page");
    for (var j = 0; j < pages.length; j++) {
      pages[j].classList.toggle("active", pages[j].id === "page-" + p);
    }
    if ($("main")) $("main").focus({ preventScroll: true });
    if (p === "reports") renderReports();
    if (p === "positive") renderPos();
    if (p === "negative") renderNegatives();
    if (p === "homework") renderHomework();
  }

  function showDialog(title, fields, onSave) {
    $("dialogTitle").textContent = title;
    $("dialogFields").innerHTML = fields;
    dialogSave = onSave;
    $("itemDialog").showModal();
  }
  function closeDialog() { $("itemDialog").close(); dialogSave = null; }

  function classOpts(sel, inc) {
    sel = sel || "";
    var h = inc ? '<option value="">بدون کلاس</option>' : "";
    db.classes.forEach(function (c) {
      h += '<option value="' + esc(c.id) + '"' + (c.id === sel ? " selected" : "") + ">" + esc(c.name) + "</option>";
    });
    return h;
  }
  function studentOpts(sel) {
    sel = sel || "";
    var h = "";
    db.students.forEach(function (s) {
      h += '<option value="' + esc(s.id) + '"' + (s.id === sel ? " selected" : "") + ">" + esc(s.name) + " (" + esc(className(s.classId)) + ")</option>";
    });
    return h;
  }
  function renderSelects() {
    var of = $("studentClassFilter").value;
    var oa = $("attendanceClass").value;
    var op = $("positiveClassFilter").value;
    var sf = '<option value="">همه کلاس‌ها</option>';
    db.classes.forEach(function (c) { sf += '<option value="' + esc(c.id) + '">' + esc(c.name) + "</option>"; });
    $("studentClassFilter").innerHTML = sf;
    $("positiveClassFilter").innerHTML = sf;
    $("studentClassFilter").value = db.classes.some(function (c) { return c.id === of; }) ? of : "";
    $("positiveClassFilter").value = db.classes.some(function (c) { return c.id === op; }) ? op : "";
    if (db.classes.length) {
      $("attendanceClass").innerHTML = classOpts(oa, false);
      if (db.classes.some(function (c) { return c.id === oa; })) $("attendanceClass").value = oa;
    } else {
      $("attendanceClass").innerHTML = '<option value="">ابتدا کلاس بسازید</option>';
    }
  }

  function renderDash() {
    var todayRecords = db.attendance.filter(function (a) { return a.date === today(); });
    var pt = todayRecords.filter(function (a) { return a.status === "present"; }).length;
    var ab = todayRecords.filter(function (a) { return a.status === "absent"; }).length;
    var st = [
      ["\u25A6", db.classes.length, "کلاس"],
      ["\u2659", db.students.length, "دانش‌آموز"],
      ["\u2713", pt, "حاضرین امروز"],
      ["\u2715", ab, "غایبین امروز"]
    ];
    var h = "";
    st.forEach(function (s) {
      h += '<div class="stat"><div class="stat-icon">' + s[0] + "</div><strong>" + faNum(s[1]) + "</strong><span>" + s[2] + "</span></div>";
    });
    $("dashboardStats").innerHTML = h;
    $("todayInfo").textContent = todayFa();
  }

  function renderClasses() {
    if (!db.classes.length) {
      $("classesList").innerHTML = empty("هنوز کلاسی نساخته‌اید", "یک کلاس جدید اضافه کنید.");
      return;
    }
    var h = "";
    db.classes.forEach(function (c) {
      var n = db.students.filter(function (s) { return s.classId === c.id; }).length;
      h += '<article class="data-card"><div><h3>' + esc(c.name) + "</h3><p>" + esc(c.subject || "بدون درس")
        + " \u00B7 " + faNum(n) + " دانش‌آموز" + (c.note ? " \u00B7 " + esc(c.note) : "") + "</p></div>"
        + '<div class="card-actions"><button class="small-btn" data-edit-class="' + c.id + '">ویرایش</button>'
        + '<button class="small-btn delete" data-delete-class="' + c.id + '">حذف</button></div></article>';
    });
    $("classesList").innerHTML = h;
  }

  function renderStudents() {
    var q = $("studentSearch").value.trim().toLowerCase();
    var cf = $("studentClassFilter").value;
    var l = db.students.filter(function (s) {
      if (cf && s.classId !== cf) return false;
      if (!q) return true;
      return (s.name + " " + (s.code || "") + " " + (s.phone || "")).toLowerCase().indexOf(q) !== -1;
    });
    if (!l.length) {
      $("studentsList").innerHTML = empty("دانش‌آموزی پیدا نشد", "از «افزودن گروهی» استفاده کنید یا تکی اضافه کنید.");
      return;
    }
    var h = "";
    l.forEach(function (s) {
      var c = s.code ? "شماره: " + esc(s.code) : "بدون شماره";
      var ph = s.phone ? " \u00B7 " + esc(s.phone) : "";
      var p = sumPos(s.id);
      var pb = p ? ' <span class="positive-badge">+' + faNum(p) + ' مثبت</span>' : "";
      h += '<article class="data-card"><div><h3>' + esc(s.name) + pb + "</h3><p>" + esc(className(s.classId))
        + " \u00B7 " + c + ph + "</p></div>"
        + '<div class="card-actions"><button class="small-btn" data-edit-student="' + s.id + '">ویرایش</button>'
        + '<button class="small-btn delete" data-delete-student="' + s.id + '">حذف</button></div></article>';
    });
    $("studentsList").innerHTML = h;
  }

  function renderGrades() {
    var srt = db.grades.slice().sort(function (a, b) { return b.date.localeCompare(a.date); });
    if (!srt.length) {
      $("gradesList").innerHTML = empty("نمره‌ای ثبت نشده", "از دکمه «ثبت نمره» استفاده کنید.");
      return;
    }
    var h = "";
    srt.forEach(function (g) {
      var st = db.students.find(function (x) { return x.id === g.studentId; });
      var cl = st ? className(st.classId) : "\u2014";
      h += '<article class="data-card"><div><h3>' + esc(studentName(g.studentId)) + ' <span class="grade-badge">'
        + esc(g.score) + " از " + esc(g.maxScore) + "</span></h3><p>" + esc(g.title) + " \u00B7 "
        + esc(fmtDate(g.date)) + " \u00B7 " + esc(cl) + (g.note ? " \u00B7 " + esc(g.note) : "") + "</p></div>"
        + '<div class="card-actions"><button class="small-btn" data-edit-grade="' + g.id + '">ویرایش</button>'
        + '<button class="small-btn delete" data-delete-grade="' + g.id + '">حذف</button></div></article>';
    });
    $("gradesList").innerHTML = h;
  }

  function renderPos() {
    var tot = db.positives.reduce(function (a, b) { return a + Number(b.points || 0); }, 0);
    var ds = {}; db.positives.forEach(function (p) { ds[p.studentId] = true; });
    var top = null;
    db.students.forEach(function (s) {
      var sm = sumPos(s.id);
      if (sm > 0 && (!top || sm > top.sum)) top = { name: s.name, sum: sm };
    });
    $("positiveSummary").innerHTML =
      '<div class="report-box"><strong>' + faNum(tot) + '</strong><span>مجموع نمرات مثبت</span></div>'
      + '<div class="report-box"><strong>' + faNum(Object.keys(ds).length) + '</strong><span>دانش‌آموز تشویق‌شده</span></div>'
      + '<div class="report-box"><strong>' + (top ? esc(top.name) + " (" + faNum(top.sum) + ")" : "\u2014")
      + '</strong><span>برترین</span></div>';

    var q = ($("positiveSearch") ? $("positiveSearch").value : "").trim().toLowerCase();
    var cf = $("positiveClassFilter") ? $("positiveClassFilter").value : "";
    var l = db.positives.slice().sort(function (a, b) { return b.date.localeCompare(a.date); });
    l = l.filter(function (p) {
      var st = db.students.find(function (x) { return x.id === p.studentId; });
      if (cf && (!st || st.classId !== cf)) return false;
      if (!q) return true;
      return (p.title + " " + (p.reason || "") + " " + (st ? st.name : "")).toLowerCase().indexOf(q) !== -1;
    });
    if (!l.length) {
      $("positivesList").innerHTML = empty("نمره مثبتی ثبت نشده", "با دکمه «ثبت نمره مثبت» امتیاز وارد کنید.");
      return;
    }
    var h = "";
    l.forEach(function (p) {
      var st = db.students.find(function (x) { return x.id === p.studentId; });
      var cl = st ? className(st.classId) : "\u2014";
      h += '<article class="data-card"><div><h3>' + esc(studentName(p.studentId)) + ' <span class="positive-badge">+'
        + esc(p.points) + " امتیاز</span></h3><p>" + esc(p.title) + " \u00B7 " + esc(fmtDate(p.date))
        + " \u00B7 " + esc(cl) + (p.reason ? " \u00B7 " + esc(p.reason) : "") + "</p></div>"
        + '<div class="card-actions"><button class="small-btn" data-edit-positive="' + p.id + '">ویرایش</button>'
        + '<button class="small-btn delete" data-delete-positive="' + p.id + '">حذف</button></div></article>';
    });
    $("positivesList").innerHTML = h;
  }

  function renderLessons() {
    var days = ["شنبه","یکشنبه","دوشنبه","سه‌شنبه","چهارشنبه","پنجشنبه","جمعه"];
    var srt = db.lessons.slice().sort(function (a, b) {
      return days.indexOf(a.day) - days.indexOf(b.day) || a.time.localeCompare(b.time);
    });
    if (!srt.length) {
      $("timetableList").innerHTML = empty("برنامه‌ای ثبت نشده", "زنگ‌ها را اضافه کنید.");
      return;
    }
    var h = "";
    srt.forEach(function (l) {
      h += '<article class="data-card"><div><h3>' + esc(l.day) + " \u00B7 " + esc(l.time) + " \u2014 "
        + esc(l.subject) + "</h3><p>" + esc(className(l.classId))
        + (l.place ? " \u00B7 " + esc(l.place) : "") + (l.note ? " \u00B7 " + esc(l.note) : "") + "</p></div>"
        + '<div class="card-actions"><button class="small-btn" data-edit-lesson="' + l.id + '">ویرایش</button>'
        + '<button class="small-btn delete" data-delete-lesson="' + l.id + '">حذف</button></div></article>';
    });
    $("timetableList").innerHTML = h;
  }

  function reportFilteredStudents() {
    var cid = $("reportClassFilter") ? $("reportClassFilter").value : "";
    var sid = $("reportStudentFilter") ? $("reportStudentFilter").value : "";
    return db.students.filter(function(st) { return (!cid || st.classId===cid) && (!sid || st.id===sid); });
  }
  function reportDateValue(id) {
    var el = $(id); if (!el || !el.value.trim()) return "";
    var value=el.value.trim();
    return /^\d{4}-\d{2}-\d{2}$/.test(value) ? value : (parseDate(value) || "");
  }
  function refreshReportFilters() {
    var c=$("reportClassFilter"), st=$("reportStudentFilter"); if(!c||!st) return;
    var oldC=c.value, oldS=st.value;
    c.innerHTML='<option value="">همه کلاس‌ها</option>'+db.classes.map(function(x){return '<option value="'+esc(x.id)+'">'+esc(x.name)+'</option>';}).join("");
    c.value=db.classes.some(function(x){return x.id===oldC;})?oldC:"";
    var list=db.students.filter(function(x){return !c.value||x.classId===c.value;});
    st.innerHTML='<option value="">همه دانش‌آموزان</option>'+list.map(function(x){return '<option value="'+esc(x.id)+'">'+esc(x.name)+' ('+esc(className(x.classId))+')</option>';}).join("");
    st.value=list.some(function(x){return x.id===oldS;})?oldS:"";
  }
  function updateReportDateLabels() {
    [["reportDateFrom","reportDateFromFa"],["reportDateTo","reportDateToFa"]].forEach(function(pair){
      var input=$(pair[0]), label=$(pair[1]); if(!input||!label)return;
      var iso=/^\d{4}-\d{2}-\d{2}$/.test(input.value) ? input.value : parseDate(input.value); label.textContent=input.value ? (iso ? "معادل شمسی: "+fmtDate(iso) : "تاریخ معتبر نیست") : "معادل شمسی: انتخاب نشده";
    });
  }
  function renderReports() {
    var from=reportDateValue("reportDateFrom"), to=reportDateValue("reportDateTo");
    var students=reportFilteredStudents(), ids=students.map(function(st){return st.id;});
    var inRange=function(x){return (!from||!x.date||x.date>=from)&&(!to||!x.date||x.date<=to);};
    var att=db.attendance.filter(function(x){return ids.indexOf(x.studentId)!==-1&&inRange(x);});
    var grades=db.grades.filter(function(x){return ids.indexOf(x.studentId)!==-1&&inRange(x);});
    var positives=db.positives.filter(function(x){return ids.indexOf(x.studentId)!==-1&&inRange(x);});
    var negatives=(db.negatives||[]).filter(function(x){return ids.indexOf(x.studentId)!==-1&&inRange(x);});
    var homework=(db.homework||[]).filter(function(x){return ids.indexOf(x.studentId)!==-1&&inRange(x);});
    var present=att.filter(function(x){return x.status==="present";}).length;
    var absent=att.filter(function(x){return x.status==="absent";}).length;
    var excused=att.filter(function(x){return x.status==="excused";}).length;
    var late=att.filter(function(x){return x.status==="late";}).length;
    var pct=att.length?Math.round((present+late)/att.length*100):null;
    var avg=grades.length?grades.reduce(function(a,x){return a+Number(x.score)/Math.max(1,Number(x.maxScore))*20;},0)/grades.length:null;
    var posScore=positives.reduce(function(a,x){return a+Number(x.points||0);},0);
    var negScore=negatives.reduce(function(a,x){return a+Math.abs(Number(x.points!=null?x.points:x.score)||0);},0);
    var stt=function(x){return String(x.status||x.state||"").toLowerCase();};
    var hc=homework.filter(function(x){return ["complete","completed","done","کامل","کامل‌شده","کامل شده"].indexOf(stt(x))!==-1;}).length;
    var hi=homework.filter(function(x){return ["incomplete","partial","ناقص","ناتمام"].indexOf(stt(x))!==-1;}).length;
    var hm=homework.filter(function(x){return ["not_submitted","missing","none","تحویل نشده","تحویل‌نشده","نداده"].indexOf(stt(x))!==-1;}).length;
    var hs=homework.reduce(function(a,x){return a+Number(x.score!=null?x.score:(x.points||0));},0);
    var h='<div class="report-grid">'
      +'<div class="report-box"><strong>'+faNum(present)+'</strong><span>تعداد حضور</span></div>'
      +'<div class="report-box"><strong>'+faNum(absent)+'</strong><span>تعداد غیبت</span></div>'
      +'<div class="report-box"><strong>'+faNum(excused)+'</strong><span>تعداد موجه</span></div>'
      +'<div class="report-box"><strong>'+faNum(late)+'</strong><span>تعداد تأخیر</span></div>'
      +'<div class="report-box"><strong>'+(pct===null?"—":faNum(pct)+"٪")+'</strong><span>درصد کل حضور (با احتساب تأخیر)</span></div>'
      +'<div class="report-box"><strong>'+(avg===null?"—":faNum(Math.round(avg*100)/100))+'</strong><span>میانگین نمرات آزمون از ۲۰</span></div>'
      +'<div class="report-box"><strong>'+faNum(posScore)+'</strong><span>نمره مثبت کلاسی</span></div>'
      +'<div class="report-box"><strong>'+faNum(negScore)+'</strong><span>نمره منفی کلاسی</span></div>'
      +'<div class="report-box"><strong>'+faNum(hc)+'</strong><span>تعداد تکلیف کامل</span></div>'
      +'<div class="report-box"><strong>'+faNum(hi)+'</strong><span>تعداد تکلیف ناقص</span></div>'
      +'<div class="report-box"><strong>'+faNum(hm)+'</strong><span>تعداد تکلیف تحویل‌نشده</span></div>'
      +'<div class="report-box"><strong>'+faNum(hs)+'</strong><span>نمره کل تکلیف</span></div></div>';
    h+='<div class="panel"><h3>وضعیت حضور و غیاب دانش‌آموزان</h3><div style="overflow:auto"><table class="report-table"><thead><tr><th>دانش‌آموز</th><th>کلاس</th><th>حاضر</th><th>غایب</th><th>موجه</th><th>تأخیر</th><th>کل ثبت‌ها</th><th>درصد حضور</th></tr></thead><tbody>';
    if(!students.length)h+='<tr><td colspan="8">دانش‌آموزی برای این فیلتر پیدا نشد.</td></tr>';
    students.forEach(function(st){var a=att.filter(function(x){return x.studentId===st.id;}), pr=a.filter(function(x){return x.status==="present";}).length, ab=a.filter(function(x){return x.status==="absent";}).length, ex=a.filter(function(x){return x.status==="excused";}).length, lt=a.filter(function(x){return x.status==="late";}).length, pp=a.length?Math.round((pr+lt)/a.length*100):null;
      h+='<tr><td>'+esc(st.name)+'</td><td>'+esc(className(st.classId))+'</td><td>'+faNum(pr)+'</td><td>'+faNum(ab)+'</td><td>'+faNum(ex)+'</td><td>'+faNum(lt)+'</td><td>'+faNum(a.length)+'</td><td>'+(pp===null?"—":faNum(pp)+"٪")+'</td></tr>';});
    h+='</tbody></table></div></div><div class="panel"><h3>خلاصه عملکرد هر دانش‌آموز</h3><div style="overflow:auto"><table class="report-table"><thead><tr><th>دانش‌آموز</th><th>کلاس</th><th>میانگین آزمون</th><th>نمره مثبت</th><th>نمره منفی</th><th>تکلیف کامل</th><th>تکلیف ناقص</th><th>تحویل‌نشده</th><th>نمره تکلیف</th></tr></thead><tbody>';
    students.forEach(function(st){var gs=grades.filter(function(x){return x.studentId===st.id;}), ga=gs.length?gs.reduce(function(a,x){return a+Number(x.score)/Math.max(1,Number(x.maxScore))*20;},0)/gs.length:null, ps=positives.filter(function(x){return x.studentId===st.id;}).reduce(function(a,x){return a+Number(x.points||0);},0), ns=negatives.filter(function(x){return x.studentId===st.id;}).reduce(function(a,x){return a+Math.abs(Number(x.points!=null?x.points:x.score)||0);},0), sh=homework.filter(function(x){return x.studentId===st.id;}), c=sh.filter(function(x){return ["complete","completed","done","کامل","کامل‌شده","کامل شده"].indexOf(stt(x))!==-1;}).length, i=sh.filter(function(x){return ["incomplete","partial","ناقص","ناتمام"].indexOf(stt(x))!==-1;}).length, m=sh.filter(function(x){return ["not_submitted","missing","none","تحویل نشده","تحویل‌نشده","نداده"].indexOf(stt(x))!==-1;}).length, sc=sh.reduce(function(a,x){return a+Number(x.score!=null?x.score:(x.points||0));},0);
      h+='<tr><td>'+esc(st.name)+'</td><td>'+esc(className(st.classId))+'</td><td>'+(ga===null?"—":faNum(Math.round(ga*100)/100))+'</td><td>'+faNum(ps)+'</td><td>'+faNum(ns)+'</td><td>'+faNum(c)+'</td><td>'+faNum(i)+'</td><td>'+faNum(m)+'</td><td>'+faNum(sc)+'</td></tr>';});
    h+='</tbody></table></div></div>';
    $("reportContent").innerHTML=h; updateReportDateLabels();
  }
  function exportReportCSV() {
    var students=reportFilteredStudents(), from=reportDateValue("reportDateFrom"), to=reportDateValue("reportDateTo"), inRange=function(x){return (!from||!x.date||x.date>=from)&&(!to||!x.date||x.date<=to);};
    var rows=[["نام دانش‌آموز","کلاس","حضور","غیبت","موجه","تأخیر","کل ثبت‌ها","درصد کل حضور","میانگین آزمون از ۲۰","نمره مثبت کلاسی","نمره منفی کلاسی","تکلیف کامل","تکلیف ناقص","تکلیف تحویل‌نشده","نمره کل تکلیف"]];
    students.forEach(function(st){var a=db.attendance.filter(function(x){return x.studentId===st.id&&inRange(x);}), g=db.grades.filter(function(x){return x.studentId===st.id&&inRange(x);}), p=db.positives.filter(function(x){return x.studentId===st.id&&inRange(x);}), n=(db.negatives||[]).filter(function(x){return x.studentId===st.id&&inRange(x);}), hw=(db.homework||[]).filter(function(x){return x.studentId===st.id&&inRange(x);}), pr=a.filter(function(x){return x.status==="present";}).length, ab=a.filter(function(x){return x.status==="absent";}).length, ex=a.filter(function(x){return x.status==="excused";}).length, lt=a.filter(function(x){return x.status==="late";}).length, avg=g.length?g.reduce(function(s,x){return s+Number(x.score)/Math.max(1,Number(x.maxScore))*20;},0)/g.length:"", pos=p.reduce(function(s,x){return s+Number(x.points||0);},0), neg=n.reduce(function(s,x){return s+Math.abs(Number(x.points!=null?x.points:x.score)||0);},0), status=function(x){return String(x.status||x.state||"").toLowerCase();}, hc=hw.filter(function(x){return ["complete","completed","done","کامل","کامل‌شده","کامل شده"].indexOf(status(x))!==-1;}).length, hi=hw.filter(function(x){return ["incomplete","partial","ناقص","ناتمام"].indexOf(status(x))!==-1;}).length, hm=hw.filter(function(x){return ["not_submitted","missing","none","تحویل نشده","تحویل‌نشده","نداده"].indexOf(status(x))!==-1;}).length, hs=hw.reduce(function(s,x){return s+Number(x.score!=null?x.score:(x.points||0));},0);
      rows.push([st.name,className(st.classId),pr,ab,ex,lt,a.length,a.length?Math.round((pr+lt)/a.length*100)+"%":"",avg===""?"":Math.round(avg*100)/100,pos,neg,hc,hi,hm,hs]);});
    var csv="\uFEFF"+rows.map(function(r){return r.map(function(v){return '"'+String(v==null?"":v).replace(/"/g,'""')+'"';}).join(",");}).join("\r\n"), blob=new Blob([csv],{type:"text/csv;charset=utf-8;"}), url=URL.createObjectURL(blob), a=document.createElement("a"); a.href=url; a.download="class-report-"+today()+".csv"; a.click(); URL.revokeObjectURL(url); toast("فایل گزارش برای Excel آماده شد.");
  }

  function addNegative() {
    if (!db.students.length) { toast("ابتدا دانش‌آموز ثبت کنید."); nav("students"); return; }
    showDialog("ثبت نمره منفی کلاسی",
      selectField("دانش‌آموز", "studentId", studentOpts(""), true)
      + '<div class="field-row">' + field("تاریخ", "date", "date", today())
      + field("نمره منفی", "points", "number", 1, true, 'min="0.1" step="0.1"') + '</div>'
      + field("دلیل", "reason", "text", "", true),
      function(v) {
        var points=Number(v.points||0); if(points<=0){toast("مقدار نمره منفی باید بیشتر از صفر باشد.");return;}
        db.negatives.push({id:makeId(),studentId:v.studentId,date:v.date||today(),points:points,reason:v.reason||""});
        saveDB(); renderNegatives(); renderReports(); toast("نمره منفی ثبت شد.");
      });
  }
  function addHomework() {
    if (!db.students.length) { toast("ابتدا دانش‌آموز ثبت کنید."); nav("students"); return; }
    var opts='<option value="complete">کامل</option><option value="incomplete">ناقص</option><option value="not_submitted">تحویل نشده</option>';
    showDialog("ثبت وضعیت تکلیف",
      selectField("دانش‌آموز", "studentId", studentOpts(""), true)
      + field("عنوان تکلیف", "title", "text", "", true)
      + '<div class="field-row">' + field("تاریخ", "date", "date", today())
      + selectField("وضعیت", "status", opts, true) + '</div>'
      + field("نمره تکلیف", "score", "number", 0, true, 'min="0" step="0.25"'),
      function(v) {
        db.homework.push({id:makeId(),studentId:v.studentId,title:v.title||"تکلیف",date:v.date||today(),status:v.status,score:Number(v.score||0)});
        saveDB(); renderHomework(); renderReports(); toast("وضعیت تکلیف ثبت شد.");
      });
  }

  function renderNegatives() {
    var list = (db.negatives || []).slice().sort(function(a,b){return String(b.date||"").localeCompare(String(a.date||""));});
    var h = "";
    if (!list.length) { $("negativesList").innerHTML = empty("هنوز نمره منفی ثبت نشده", "از دکمه «ثبت نمره منفی» استفاده کنید."); return; }
    list.forEach(function(n){
      h += '<article class="data-card"><div><h3>' + esc(studentName(n.studentId)) + ' <span class="grade-badge" style="color:var(--danger)">−' + faNum(n.points || n.score || 0) + '</span></h3><p>' + esc(n.reason || "بدون توضیح") + ' · ' + esc(fmtDate(n.date || today())) + '</p><p>' + esc(className((db.students.find(function(st){return st.id===n.studentId;})||{}).classId)) + '</p></div><div class="card-actions"><button class="small-btn delete" data-delete-negative="' + n.id + '">حذف</button></div></article>';
    });
    $("negativesList").innerHTML = h;
  }
  function renderHomework() {
    var list = (db.homework || []).slice().sort(function(a,b){return String(b.date||"").localeCompare(String(a.date||""));});
    var h = "", labels = {complete:"کامل",completed:"کامل",done:"کامل",incomplete:"ناقص",partial:"ناقص",not_submitted:"تحویل نشده",missing:"تحویل نشده",none:"تحویل نشده"};
    if (!list.length) { $("homeworkList").innerHTML = empty("هنوز تکلیفی ثبت نشده", "از دکمه «ثبت وضعیت تکلیف» استفاده کنید."); return; }
    list.forEach(function(w){
      var status = String(w.status || w.state || "");
      var label = labels[status] || status || "ثبت نشده";
      h += '<article class="data-card"><div><h3>' + esc(studentName(w.studentId)) + ' <span class="grade-badge">' + esc(label) + '</span></h3><p>' + esc(w.title || "تکلیف") + ' · ' + esc(fmtDate(w.date || today())) + ' · نمره: ' + faNum(w.score || w.points || 0) + '</p><p>' + esc(className((db.students.find(function(st){return st.id===w.studentId;})||{}).classId)) + '</p></div><div class="card-actions"><button class="small-btn delete" data-delete-homework="' + w.id + '">حذف</button></div></article>';
    });
    $("homeworkList").innerHTML = h;
  }

  function renderAll() {
    renderSelects(); renderDash(); renderClasses(); renderStudents();
    renderGrades(); renderPos(); renderNegatives(); renderHomework(); renderLessons(); refreshReportFilters(); renderReports();
    $("storageInfo").textContent = faNum(db.classes.length) + " کلاس، " + faNum(db.students.length)
      + " دانش‌آموز، " + faNum(db.lessons.length) + " زنگ.";
  }

  function field(label, name, type, value, required, extra) {
    type = type || "text"; value = value == null ? "" : value; required = required !== false;
    return "<label>" + label + '<input name="' + name + '" type="' + type + '" value="' + esc(value) + '"'
      + (required ? " required" : "") + " " + (extra || "") + "></label>";
  }
  function dateField(label, name, iso, required) {
    required = required !== false;
    var d = iso || today();
    return "<label>" + label + '<input name="' + name + '" type="date" data-datepicker="1" value="' + esc(d) + '"'
      + (required ? " required" : "") + "></label>";
  }
  function selectField(label, name, opts, required) {
    required = required !== false;
    return "<label>" + label + '<select name="' + name + '"' + (required ? " required" : "") + ">" + opts + "</select></label>";
  }

  function addClass(id) {
    var old = id ? (db.classes.find(function (c) { return c.id === id; }) || {}) : {};
    showDialog(id ? "ویرایش کلاس" : "افزودن کلاس",
      field("نام کلاس", "name", "text", old.name || "")
      + field("درس", "subject", "text", old.subject || "", false)
      + field("یادداشت", "note", "text", old.note || "", false),
      function (v) {
        if (id) { for (var k in v) if (v.hasOwnProperty(k)) old[k] = v[k]; }
        else { var i = { id: makeId() }; for (var k2 in v) if (v.hasOwnProperty(k2)) i[k2] = v[k2]; db.classes.push(i); }
        saveDB(); toast("کلاس ذخیره شد.");
      });
  }
  function addStudent(id) {
    if (!db.classes.length) { toast("ابتدا کلاس بسازید."); nav("classes"); return; }
    var old = id ? (db.students.find(function (s) { return s.id === id; }) || {}) : {};
    showDialog(id ? "ویرایش دانش‌آموز" : "افزودن دانش‌آموز",
      field("نام و نام خانوادگی", "name", "text", old.name || "")
      + selectField("کلاس", "classId", classOpts(old.classId, false), true)
      + '<div class="field-row">'
      + field("شماره دانش‌آموزی", "code", "text", old.code || "", false)
      + field("تماس", "phone", "tel", old.phone || "", false) + "</div>"
      + field("یادداشت", "note", "text", old.note || "", false),
      function (v) {
        if (id) { for (var k in v) if (v.hasOwnProperty(k)) old[k] = v[k]; }
        else { var i = { id: makeId() }; for (var k2 in v) if (v.hasOwnProperty(k2)) i[k2] = v[k2]; db.students.push(i); }
        saveDB(); toast("ذخیره شد.");
      });
  }
  function addGrade(id) {
    if (!db.students.length) { toast("ابتدا دانش‌آموز ثبت کنید."); nav("students"); return; }
    var old = id ? (db.grades.find(function (g) { return g.id === id; }) || {}) : {};
    showDialog(id ? "ویرایش نمره" : "ثبت نمره",
      selectField("دانش‌آموز", "studentId", studentOpts(old.studentId), true)
      + '<div class="field-row">'
      + field("عنوان", "title", "text", old.title || "آزمون")
      + dateField("تاریخ", "date", old.date || today()) + "</div>"
      + '<div class="field-row">'
      + field("نمره", "score", "number", old.score == null ? "" : old.score, true, 'min="0" step="0.25"')
      + field("حداکثر", "maxScore", "number", old.maxScore || 20, true, 'min="0.01" step="0.25"') + "</div>"
      + field("یادداشت", "note", "text", old.note || "", false),
      function (v) {
        if (!v.date) { toast("تاریخ معتبر وارد کنید (" + datePH() + ")."); return; }
        if (Number(v.score) < 0 || Number(v.maxScore) <= 0 || Number(v.score) > Number(v.maxScore)) {
          toast("نمره نامعتبر است."); return;
        }
        if (id) { for (var k in v) if (v.hasOwnProperty(k)) old[k] = v[k]; }
        else { var i = { id: makeId() }; for (var k2 in v) if (v.hasOwnProperty(k2)) i[k2] = v[k2]; db.grades.push(i); }
        saveDB(); toast("نمره ذخیره شد.");
      });
  }
  function addPositive(id) {
    if (!db.students.length) { toast("ابتدا دانش‌آموز ثبت کنید."); nav("students"); return; }
    var old = id ? (db.positives.find(function (p) { return p.id === id; }) || {}) : {};
    var rs = ["پاسخ صحیح کلاسی","فعالیت در گروه","انجام تکلیف","نظم و انضباط","کمک به هم‌کلاسی","خلاقیت و ایده","پیشرفت تحصیلی","رفتار محترمانه"];
    var opts = ""; rs.forEach(function (r) { opts += "<option" + (r === old.reason ? " selected" : "") + ">" + r + "</option>"; });
    showDialog(id ? "ویرایش نمره مثبت" : "ثبت نمره مثبت",
      selectField("دانش‌آموز", "studentId", studentOpts(old.studentId), true)
      + '<div class="field-row">'
      + field("عنوان", "title", "text", old.title || "امتیاز تشویقی")
      + dateField("تاریخ", "date", old.date || today()) + "</div>"
      + '<div class="field-row">'
      + field("امتیاز", "points", "number", old.points == null ? 1 : old.points, true, 'min="0.5" step="0.5"')
      + selectField("دلیل", "reason", '<option value="">\u2014</option>' + opts, false) + "</div>"
      + field("توضیح", "note", "text", old.note || "", false),
      function (v) {
        if (!v.date) { toast("تاریخ معتبر وارد کنید (" + datePH() + ")."); return; }
        if (!v.points || Number(v.points) <= 0) { toast("امتیاز باید مثبت باشد."); return; }
        if (id) { for (var k in v) if (v.hasOwnProperty(k)) old[k] = v[k]; }
        else { var i = { id: makeId() }; for (var k2 in v) if (v.hasOwnProperty(k2)) i[k2] = v[k2]; db.positives.push(i); }
        saveDB(); toast("ذخیره شد.");
      });
  }
  function addLesson(id) {
    if (!db.classes.length) { toast("ابتدا کلاس بسازید."); nav("classes"); return; }
    var old = id ? (db.lessons.find(function (l) { return l.id === id; }) || {}) : {};
    var days = ["شنبه","یکشنبه","دوشنبه","سه‌شنبه","چهارشنبه","پنجشنبه","جمعه"];
    var do2 = ""; var cd = old.day || "شنبه";
    days.forEach(function (d) { do2 += "<option" + (d === cd ? " selected" : "") + ">" + d + "</option>"; });
    showDialog(id ? "ویرایش زنگ" : "افزودن زنگ",
      selectField("روز", "day", do2, true)
      + '<div class="field-row">'
      + field("ساعت", "time", "time", old.time || "08:00")
      + field("درس", "subject", "text", old.subject || "") + "</div>"
      + selectField("کلاس", "classId", classOpts(old.classId, false), true)
      + '<div class="field-row">'
      + field("مکان", "place", "text", old.place || "", false)
      + field("یادداشت", "note", "text", old.note || "", false) + "</div>",
      function (v) {
        if (id) { for (var k in v) if (v.hasOwnProperty(k)) old[k] = v[k]; }
        else { var i = { id: makeId() }; for (var k2 in v) if (v.hasOwnProperty(k2)) i[k2] = v[k2]; db.lessons.push(i); }
        saveDB(); toast("ذخیره شد.");
      });
  }

  function syncAttDate() {
    var inp = $("attendanceDate"); if (!inp) return;
    var iso = inp.dataset.iso || today();
    inp.dataset.iso = iso;
    inp.value = isoToInput(iso);
    inp.placeholder = datePH();
    $("attendanceDateFa").textContent = iso ? "معادل: " + fmtDate(iso) : "";
  }
  function loadAtt() {
    var cid = $("attendanceClass").value, date = $("attendanceDate").dataset.iso;
    if (!cid || !date) { $("attendanceArea").innerHTML = empty("کلاس و تاریخ را انتخاب کنید"); return; }
    var sts = db.students.filter(function (s) { return s.classId === cid; });
    if (!sts.length) { $("attendanceArea").innerHTML = empty("دانش‌آموزی در این کلاس نیست"); return; }
    var statuses = [["present","حاضر"],["absent","غایب"],["late","تأخیر"],["excused","موجه"]];
    var h = '<div class="panel-head"><h3>' + esc(className(cid)) + " \u00B7 " + esc(fmtDate(date))
      + '</h3><button class="btn" data-save-attendance>ذخیره</button></div>';
    sts.forEach(function (s) {
      var old = db.attendance.find(function (a) { return a.studentId === s.id && a.date === date; });
      var st = old ? old.status : "present";
      h += '<div class="attendance-row" data-att-student="' + s.id + '"><div><strong>' + esc(s.name)
        + "</strong><small>" + (s.code ? "شماره: " + esc(s.code) : "") + '</small></div><div class="status-options">';
      statuses.forEach(function (x) {
        h += '<button type="button" data-status="' + x[0] + '"' + (st === x[0] ? ' class="selected"' : "") + ">" + x[1] + "</button>";
      });
      h += "</div></div>";
    });
    $("attendanceArea").innerHTML = h;
  }

  function detectD(l) {
    if (l.indexOf("\t") >= 0) return "\t";
    if (l.indexOf(";") >= 0) return ";";
    if (l.indexOf("|") >= 0) return "|";
    return ",";
  }
  function splitCSV(line, d) {
    var c = [], cur = "", q = false;
    for (var i = 0; i < line.length; i++) {
      var ch = line[i];
      if (q) {
        if (ch === '"' && line[i + 1] === '"') { cur += '"'; i++; }
        else if (ch === '"') q = false;
        else cur += ch;
      } else {
        if (ch === '"') q = true;
        else if (ch === d) { c.push(cur); cur = ""; }
        else cur += ch;
      }
    }
    c.push(cur);
    return c.map(function (x) { return String(x).trim(); });
  }
  function parseBulk(text) {
    var lines = String(text || "").replace(/\uFEFF/g, "").split(/\r?\n/);
    lines = lines.filter(function (l) { return l.trim().length > 0; });
    if (!lines.length) return { rows: [], hasHeader: false };
    var d = detectD(lines[0]);
    var fc = splitCSV(lines[0], d);
    var hk = /^(نام|name|دانش.?آموز|کلاس|class|شماره|phone|tel)/i;
    var hh = fc.some(function (c) { return hk.test(c); });
    var st = hh ? 1 : 0;
    var rows = [];
    for (var i = st; i < lines.length; i++) {
      var cs = splitCSV(lines[i], d);
      if (!cs.length || !cs[0]) continue;
      rows.push(cs);
    }
    return { rows: rows, hasHeader: hh };
  }
  function doBulk(text) {
    var p = parseBulk(text);
    if (!p.rows.length) { toast("ردیف معتبری پیدا نشد."); $("bulkPreview").textContent = ""; return; }
    var nc = [];
    p.rows.forEach(function (r) {
      var cn = r[1] || "";
      if (cn && !findClass(cn) && nc.indexOf(cn) === -1) nc.push(cn);
    });
    var msg = "تعداد ردیف: " + faNum(p.rows.length)
      + (p.hasHeader ? " (سرصفحه نادیده)" : "")
      + "\nستون: [0]نام [1]کلاس [2]شماره [3]تلفن [4]یادداشت"
      + (nc.length ? "\nکلاس‌های جدید: " + nc.join("، ") : "")
      + "\nادامه؟";
    if (!confirm(msg)) return;
    var added = 0;
    var before = db.classes.length;
    p.rows.forEach(function (r) {
      var n = normDig(r[0] || "").trim();
      if (!n) return;
      var cn = (r[1] || "").trim();
      var cid;
      if (cn) cid = ensureClass(cn);
      else cid = db.classes[0] ? db.classes[0].id : ensureClass("کلاس پیش‌فرض");
      db.students.push({
        id: makeId(), name: n, classId: cid,
        code: (r[2] || "").trim(),
        phone: normDig(r[3] || "").trim(),
        note: (r[4] || "").trim()
      });
      added++;
    });
    var cc = db.classes.length - before;
    saveDB();
    $("bulkPreview").innerHTML = "<b>" + faNum(added) + "</b> دانش‌آموز افزوده شد"
      + (cc ? " و <b>" + faNum(cc) + "</b> کلاس جدید." : ".");
    toast(faNum(added) + " دانش‌آموز اضافه شد.");
  }

  function bindEvents() {
    document.querySelectorAll(".tab").forEach(function (b) {
      b.addEventListener("click", function () { nav(b.getAttribute("data-page")); });
    });
    document.querySelectorAll("[data-goto]").forEach(function (b) {
      b.addEventListener("click", function () { nav(b.getAttribute("data-goto")); });
    });

    $("closeDialog").onclick = closeDialog;
    $("cancelDialog").onclick = closeDialog;
    $("itemForm").addEventListener("submit", function (e) {
      e.preventDefault();
      if (!dialogSave) return;
      var v = {};
      new FormData(e.currentTarget).forEach(function (x, k) { v[k] = String(x).trim(); });
      e.currentTarget.querySelectorAll("[data-datepicker]").forEach(function (inp) {
        v[inp.name] = /^\d{4}-\d{2}-\d{2}$/.test(inp.value) ? inp.value : (parseDate(inp.value) || "");
      });
      var fn = dialogSave;
      closeDialog();
      fn(v);
    });

    $("addClassBtn").onclick = function () { addClass(null); };
    $("addStudentBtn").onclick = function () { addStudent(null); };
    $("addGradeBtn").onclick = function () { addGrade(null); };
    $("addPositiveBtn").onclick = function () { addPositive(null); };
    $("addLessonBtn").onclick = function () { addLesson(null); };

    $("studentSearch").addEventListener("input", renderStudents);
    $("studentClassFilter").addEventListener("change", renderStudents);
    $("positiveSearch").addEventListener("input", renderPos);
    $("positiveClassFilter").addEventListener("change", renderPos);
    ["reportClassFilter","reportStudentFilter","reportDateFrom","reportDateTo"].forEach(function(id){
      var el=$(id); if(el) { el.addEventListener("change",function(){if(id==="reportClassFilter")refreshReportFilters();renderReports();}); el.addEventListener("input",function(){if(id.indexOf("reportDate")===0)updateReportDateLabels();}); }
    });
    if ($("exportReportBtn")) $("exportReportBtn").onclick=exportReportCSV;
    if ($("addNegativeBtn")) $("addNegativeBtn").onclick=addNegative;
    if ($("addHomeworkBtn")) $("addHomeworkBtn").onclick=addHomework;
    if ($("addNegativePageBtn")) $("addNegativePageBtn").onclick=addNegative;
    if ($("addHomeworkPageBtn")) $("addHomeworkPageBtn").onclick=addHomework;
    if ($("quickPositiveBtn")) $("quickPositiveBtn").onclick=function(){nav("positive");addPositive(null);};

    document.addEventListener("click", function (e) {
      var b = e.target.closest ? e.target.closest("button") : null;
      if (!b) return;
      if (b.dataset.editClass) addClass(b.dataset.editClass);
      if (b.dataset.deleteClass && confirm("این کلاس حذف شود؟")) {
        db.classes = db.classes.filter(function (x) { return x.id !== b.dataset.deleteClass; });
        saveDB(); toast("کلاس حذف شد.");
      }
      if (b.dataset.editStudent) addStudent(b.dataset.editStudent);
      if (b.dataset.deleteStudent && confirm("این دانش‌آموز و سوابق حذف شوند؟")) {
        var ids = b.dataset.deleteStudent;
        db.students = db.students.filter(function (x) { return x.id !== ids; });
        db.grades = db.grades.filter(function (x) { return x.studentId !== ids; });
        db.attendance = db.attendance.filter(function (x) { return x.studentId !== ids; });
        db.positives = db.positives.filter(function (x) { return x.studentId !== ids; });
        saveDB(); toast("حذف شد.");
      }
      if (b.dataset.editGrade) addGrade(b.dataset.editGrade);
      if (b.dataset.deleteGrade && confirm("نمره حذف شود؟")) {
        db.grades = db.grades.filter(function (x) { return x.id !== b.dataset.deleteGrade; });
        saveDB(); toast("حذف شد.");
      }
      if (b.dataset.editPositive) addPositive(b.dataset.editPositive);
      if (b.dataset.deletePositive && confirm("نمره مثبت حذف شود؟")) {
        db.positives = db.positives.filter(function (x) { return x.id !== b.dataset.deletePositive; });
        saveDB(); toast("حذف شد.");
      }
      if (b.dataset.deleteNegative && confirm("این نمره منفی حذف شود؟")) {
        db.negatives = (db.negatives || []).filter(function(x){return x.id !== b.dataset.deleteNegative;});
        saveDB(); renderNegatives(); renderReports(); toast("نمره منفی حذف شد.");
      }
      if (b.dataset.deleteHomework && confirm("این تکلیف حذف شود؟")) {
        db.homework = (db.homework || []).filter(function(x){return x.id !== b.dataset.deleteHomework;});
        saveDB(); renderHomework(); renderReports(); toast("تکلیف حذف شد.");
      }
      if (b.dataset.editLesson) addLesson(b.dataset.editLesson);
      if (b.dataset.deleteLesson && confirm("زنگ حذف شود؟")) {
        db.lessons = db.lessons.filter(function (x) { return x.id !== b.dataset.deleteLesson; });
        saveDB(); toast("حذف شد.");
      }
    });

    $("attendanceDate").dataset.iso = today();
    syncAttDate();
    $("attendanceDate").addEventListener("change", function () {
      var iso = /^\d{4}-\d{2}-\d{2}$/.test(this.value) ? this.value : parseDate(this.value);
      if (!iso) { toast("تاریخ معتبر نیست: " + datePH()); syncAttDate(); return; }
      this.dataset.iso = iso;
      this.value = iso;
      $("attendanceDateFa").textContent = "معادل شمسی: " + fmtDate(iso);
    });
    $("loadAttendanceBtn").onclick = loadAtt;
    $("attendanceArea").addEventListener("click", function (e) {
      var sb = e.target.closest ? e.target.closest("[data-status]") : null;
      if (sb) {
        sb.parentElement.querySelectorAll("button").forEach(function (x) { x.classList.remove("selected"); });
        sb.classList.add("selected");
        return;
      }
      if (e.target.closest && e.target.closest("[data-save-attendance]")) {
        var cid = $("attendanceClass").value;
        var date = $("attendanceDate").dataset.iso;
        if (!date) { toast("تاریخ معتبر نیست."); return; }
        document.querySelectorAll("[data-att-student]").forEach(function (row) {
          var sid = row.dataset.attStudent;
          var sel = row.querySelector(".selected");
          var st = sel ? sel.dataset.status : "present";
          var old = db.attendance.find(function (a) { return a.studentId === sid && a.date === date; });
          if (old) old.status = st;
          else db.attendance.push({ id: makeId(), classId: cid, studentId: sid, date: date, status: st });
        });
        saveDB(); toast("ذخیره شد.");
      }
    });

    $("toggleBulkBtn").onclick = function () { $("bulkArea").classList.toggle("hidden"); };
    $("bulkClearBtn").onclick = function () { $("bulkText").value = ""; $("bulkPreview").textContent = ""; };
    $("bulkSampleBtn").onclick = function () {
      $("bulkText").value = "نام,کلاس,شماره دانش‌آموزی,شماره تماس,یادداشت\nعلی رضایی,دهم الف,101,09121234567,\nزهرا محمدی,دهم الف,102,,\n";
      $("bulkPreview").textContent = "نمونه قالب قرار گرفت.";
    };
    $("bulkFile").addEventListener("change", function (e) {
      var file = e.target.files[0]; if (!file) return;
      var r = new FileReader();
      r.onload = function (ev) {
        $("bulkText").value = String(ev.target.result || "");
        $("bulkPreview").textContent = "فایل بارگذاری شد.";
      };
      r.readAsText(file, "UTF-8");
      e.target.value = "";
    });
    $("bulkParseBtn").onclick = function () { doBulk($("bulkText").value); };

    $("setTheme").addEventListener("change", function () { settings.theme = this.value; saveSettings(); applyDisplay(); });
    $("setFont").addEventListener("change", function () { settings.font = this.value; saveSettings(); applyDisplay(); });
    $("setFontSize").addEventListener("change", function () { settings.fontSize = this.value; saveSettings(); applyDisplay(); });
    $("setCalendar").addEventListener("change", function () {
      settings.calendar = this.value;
      saveSettings(); applyDisplay(); renderAll();
      toast("تقویم تغییر کرد.");
    });
    $("resetDisplayBtn").onclick = function () {
      settings.theme = "light"; settings.font = "Vazirmatn";
      settings.fontSize = "14"; settings.calendar = "jalali";
      saveSettings(); applyDisplay(); renderAll();
      toast("بازگردانی شد.");
    };
    if (window.matchMedia) {
      try {
        window.matchMedia("(prefers-color-scheme: dark)").addEventListener("change", function () {
          if (settings.theme === "auto") applyDisplay();
        });
      } catch (e) {}
    }

    $("exportBtn").onclick = function () {
      var p = { format: "ClassManagerPWA", version: 3, exportedAt: new Date().toISOString(), settings: settings, data: db };
      var b = new Blob([JSON.stringify(p, null, 2)], { type: "application/json" });
      var u = URL.createObjectURL(b);
      var a = document.createElement("a");
      a.href = u; a.download = "class-manager-backup-" + today() + ".json"; a.click();
      URL.revokeObjectURL(u); toast("پشتیبان آماده شد.");
    };
    $("importFile").addEventListener("change", function (e) {
      var f = e.target.files[0]; if (!f) return;
      f.text().then(function (txt) {
        try {
          var p = JSON.parse(txt);
          var n = p.data || p;
          if (!n || !Array.isArray(n.classes) || !Array.isArray(n.students)
            || !Array.isArray(n.attendance) || !Array.isArray(n.grades) || !Array.isArray(n.lessons)) {
            throw new Error("bad");
          }
          if (!confirm("بازیابی، اطلاعات فعلی را جایگزین می‌کند. ادامه؟")) return;
          var fr = emptyDB();
          for (var k in n) if (n.hasOwnProperty(k)) fr[k] = n[k];
          if (!Array.isArray(fr.positives)) fr.positives = [];
          db = fr;
          if (p.settings && typeof p.settings === "object") {
            settings = p.settings; saveSettings(); applyDisplay();
          }
          saveDB(); toast("بازیابی شد.");
        } catch (err) { toast("فایل معتبر نیست."); }
        finally { e.target.value = ""; }
      });
    });
    $("clearDataBtn").onclick = function () {
      if (confirm("همه اطلاعات حذف شود؟ (تنظیمات نمایش حفظ می‌شود)")) {
        db = emptyDB(); saveDB(); toast("پاک شد.");
      }
    };
    $("printReportBtn").onclick = function () {
      nav("reports"); setTimeout(function () { window.print(); }, 100);
    };

    window.addEventListener("beforeinstallprompt", function (e) {
      e.preventDefault(); deferredInstall = e;
      $("installBtn").classList.remove("hidden");
    });
    $("installBtn").onclick = function () {
      if (!deferredInstall) { toast("از منوی مرورگر «افزودن به صفحه اصلی» را بزنید."); return; }
      deferredInstall.prompt();
      deferredInstall.userChoice.then(function () {
        deferredInstall = null; $("installBtn").classList.add("hidden");
      });
    };
    var ob = $("offlineBadge");
    function updOnline() {
      if (navigator.onLine) ob.classList.add("hidden");
      else ob.classList.remove("hidden");
    }
    window.addEventListener("online", updOnline);
    window.addEventListener("offline", updOnline);
    updOnline();

    if ("serviceWorker" in navigator && (location.protocol === "https:" || location.hostname === "localhost")) {
      window.addEventListener("load", function () {
        navigator.serviceWorker.register("./sw.js").catch(function () {});
      });
    }
  }

  bindEvents();
  applyDisplay();
  renderAll();

})();
