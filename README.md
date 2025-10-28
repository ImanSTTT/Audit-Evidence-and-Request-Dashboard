import React, { useEffect, useMemo, useState } from "react";
import { motion } from "framer-motion";
import {
  PieChart,
  Pie,
  Cell,
  Tooltip as ReTooltip,
  ResponsiveContainer,
  BarChart,
  Bar,
  XAxis,
  YAxis,
  CartesianGrid,
  Legend,
} from "recharts";
import {
  Plus,
  Edit3,
  Trash2,
  CheckCircle2,
  FileDown,
  FileUp,
  Link as LinkIcon,
  Search,
  Layers,
  ClipboardList,
  LayoutDashboard,
  X,
  Clock3,
  AlertTriangle,
  Settings as SettingsIcon,
  Smartphone,
} from "lucide-react";
import JSZip from "jszip";

// --- PWA (Installable App) Helpers ----------------------------------------
// Lightweight typing for the install prompt event
// eslint-disable-next-line @typescript-eslint/no-explicit-any
type BeforeInstallPromptEvent = Event & {
  prompt: () => Promise<void>;
  userChoice: Promise<{ outcome: "accepted" | "dismissed" }>;
};

function usePWA() {
  const [installPrompt, setInstallPrompt] = useState<BeforeInstallPromptEvent | null>(null);
  const [installed, setInstalled] = useState(false);

  useEffect(() => {
    const onBIP = (e: Event) => {
      (e as any).preventDefault?.();
      setInstallPrompt(e as BeforeInstallPromptEvent);
    };
    const onInstalled = () => { setInstalled(true); setInstallPrompt(null); };

    window.addEventListener("beforeinstallprompt", onBIP);
    window.addEventListener("appinstalled", onInstalled);

    // Inject manifest at runtime so we don't need a separate file in this preview
    const iconSvg = encodeURIComponent(`<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 256 256'><rect width='256' height='256' rx='56' fill='%234f46e5'/><g fill='white'><circle cx='72' cy='96' r='22'/><circle cx='184' cy='96' r='22'/><rect x='60' y='132' width='136' height='24' rx='12'/></g></svg>`);
    const manifest = {
      name: "Audit Evidence",
      short_name: "Audit App",
      start_url: ".",
      scope: "./",
      display: "standalone",
      theme_color: "#4f46e5",
      background_color: "#ffffff",
      icons: [
        { src: `data:image/svg+xml,${iconSvg}`, sizes: "192x192", type: "image/svg+xml", purpose: "any" },
        { src: `data:image/svg+xml,${iconSvg}`, sizes: "512x512", type: "image/svg+xml", purpose: "any" },
      ],
    };
    const link = document.createElement("link");
    link.rel = "manifest";
    const mblob = new Blob([JSON.stringify(manifest)], { type: "application/json" });
    link.href = URL.createObjectURL(mblob);
    document.head.appendChild(link);

    // Register a tiny service worker for offline/basic caching
    if ("serviceWorker" in navigator) {
      const swCode = `
        const CACHE = 'audit-evidence-v1';
        self.addEventListener('install', (e) => {
          e.waitUntil((async () => {
            const c = await caches.open(CACHE);
            try { await c.addAll([]); } catch (e) {}
            self.skipWaiting();
          })());
        });
        self.addEventListener('activate', (e) => {
          e.waitUntil((async () => {
            const keys = await caches.keys();
            await Promise.all(keys.filter(k => k !== CACHE).map(k => caches.delete(k)));
            self.clients.claim();
          })());
        });
        self.addEventListener('fetch', (e) => {
          const req = e.request;
          if (req.method !== 'GET') return;
          e.respondWith((async () => {
            const cached = await caches.match(req);
            if (cached) return cached;
            try {
              const resp = await fetch(req);
              try { const c = await caches.open(CACHE); c.put(req, resp.clone()); } catch (e) {}
              return resp;
            } catch (err) {
              return cached || Response.error();
            }
          })());
        });
      `;
      const swBlob = new Blob([swCode], { type: 'text/javascript' });
      const swUrl = URL.createObjectURL(swBlob);
      navigator.serviceWorker.register(swUrl).catch(() => {});
    }

    // initial installed state (PWA standalone)
    const standalone = window.matchMedia?.('(display-mode: standalone)')?.matches || (navigator as any).standalone;
    if (standalone) setInstalled(true);

    return () => {
      window.removeEventListener("beforeinstallprompt", onBIP);
      window.removeEventListener("appinstalled", onInstalled);
      try { document.head.removeChild(link); } catch {}
    };
  }, []);

  return { installPrompt, installed };
}

// NOTE: No `file-saver` dependency — use a tiny helper that works in preview
function saveAsBlob(blob: Blob, filename: string) {
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = filename;
  document.body.appendChild(a);
  a.click();
  setTimeout(() => {
    URL.revokeObjectURL(url);
    a.remove();
  }, 100);
}

// --- Helpers ---------------------------------------------------------------
const cls = (...xs: (string | false | null | undefined)[]) => xs.filter(Boolean).join(" ");
const STORAGE_KEY = "audit-evidence-app:v3"; // bump to v3: add `waktu` field

function loadState() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) return null;
    return JSON.parse(raw);
  } catch (e) {
    console.warn("Failed to load state", e);
    return null;
  }
}
function saveState(state: any) {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(state));
  } catch (e) {
    console.warn("Failed to save state", e);
  }
}
function nextId(prefix: string, items: { id: string }[]) {
  const maxN = items
    .map((x) => parseInt(String(x.id).replace(/[^0-9]/g, ""), 10))
    .filter((n) => !Number.isNaN(n))
    .reduce((acc, n) => Math.max(acc, n), 0);
  const n = String(maxN + 1).padStart(3, "0");
  return `${prefix}-${n}`;
}
function csv(val: any) {
  const s = String(val ?? "");
  return '"' + s.replace(/"/g, '""') + '"';
}
function slugify(s: any) {
  return String(s || "")
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, "-")
    .replace(/(^-|-$)/g, "")
    .slice(0, 60);
}

const MS_DAY = 24 * 60 * 60 * 1000;
function dateOnly(d: Date) { return new Date(d.getFullYear(), d.getMonth(), d.getDate()); }
function parseDateOnly(s: string | null | undefined) {
  if (!s) return null;
  const d = new Date(s);
  if (Number.isNaN(d.getTime())) return null;
  return dateOnly(d);
}
function daysUntil(dateStr: string | null | undefined): number | null {
  const d = parseDateOnly(dateStr);
  if (!d) return null;
  const today = dateOnly(new Date());
  return Math.ceil((d.getTime() - today.getTime()) / MS_DAY);
}
function daysLabel(n: number | null) {
  if (n === null) return "-";
  if (n < 0) return `Terlambat ${Math.abs(n)} hari`;
  if (n === 0) return "Hari ini";
  if (n === 1) return "Besok";
  return `${n} hari lagi`;
}

// DD-MM-YY helpers ---------------------------------------------------------
function normalizeDDMMYYInput(v: string) {
  const digits = (v || "").replace(/[^0-9]/g, "");
  const dd = digits.slice(0, 2);
  const mm = digits.slice(2, 4);
  const yy = digits.slice(4, 6);
  let out = dd;
  if (mm) out += "-" + mm; else if (dd.length === 2) out += "-";
  if (yy) out += "-" + yy; else if (mm.length === 2) out += "-";
  return out;
}
function parseDDMMYY(s: string | null | undefined): Date | null {
  if (!s) return null;
  const m = String(s).trim().match(/^(\d{1,2})[-\/](\d{1,2})[-\/]?(\d{2})$/);
  if (!m) return null;
  const d = parseInt(m[1], 10);
  const mo = parseInt(m[2], 10);
  const yy = parseInt(m[3], 10);
  const y = 2000 + yy; // interpret 00-99 as 2000-2099
  if (mo < 1 || mo > 12 || d < 1 || d > 31) return null;
  const dt = new Date(y, mo - 1, d);
  if (dt.getMonth() !== mo - 1 || dt.getDate() !== d) return null; // invalid day/month combo
  return dateOnly(dt);
}
function daysUntilDDMMYY(s: string | null | undefined): number | null {
  const d = parseDDMMYY(s);
  if (!d) return null;
  const today = dateOnly(new Date());
  return Math.ceil((d.getTime() - today.getTime()) / MS_DAY);
}

const VALIDITAS = ["Valid", "Perlu Perbaikan"] as const;
const STATUS = ["Terpenuhi", "Belum"] as const;

type Bukti = {
  id: string; kategori: string; deskripsi: string; link: string; unit: string; pic: string; tanggalDiterima: string; validitas: typeof VALIDITAS[number]; catatan: string; idPermintaanTerkait: string;
};

type Permintaan = {
  id: string; tanggal: string; unit: string; deskripsi: string; tenggat: string; waktu: string; pic: string; idBuktiTerkait: string[]; status: typeof STATUS[number]; tanggalPemenuhan: string;
};

// --- Seed Data -------------------------------------------------------------
const seedBukti: Bukti[] = [
  { id: "BKT-001", kategori: "Kebijakan", deskripsi: "Kebijakan Keamanan Informasi", link: "https://example.com/kebijakan.pdf", unit: "TI", pic: "Andi", tanggalDiterima: "2025-10-01", validitas: "Valid", catatan: "", idPermintaanTerkait: "PRM-001" },
  { id: "BKT-002", kategori: "Prosedur", deskripsi: "SOP Backup Rutin", link: "https://example.com/sop-backup.docx", unit: "TI", pic: "Sari", tanggalDiterima: "2025-10-03", validitas: "Perlu Perbaikan", catatan: "Perlu tanda tangan terbaru", idPermintaanTerkait: "PRM-002" },
  { id: "BKT-003", kategori: "Catatan", deskripsi: "Log Backup Sept 2025", link: "https://example.com/log-backup.csv", unit: "TI", pic: "Budi", tanggalDiterima: "2025-10-05", validitas: "Valid", catatan: "", idPermintaanTerkait: "PRM-002" },
];

const seedReq: Permintaan[] = [
  { id: "PRM-001", tanggal: "2025-10-02", unit: "Kepatuhan", deskripsi: "Minta kebijakan keamanan informasi", tenggat: "2025-10-04", waktu: "", pic: "Rina", idBuktiTerkait: ["BKT-001"], status: "Terpenuhi", tanggalPemenuhan: "2025-10-02" },
  { id: "PRM-002", tanggal: "2025-10-03", unit: "TI", deskripsi: "Bukti SOP dan log backup", tenggat: "2025-10-06", waktu: "", pic: "Andi", idBuktiTerkait: ["BKT-002", "BKT-003"], status: "Terpenuhi", tanggalPemenuhan: "2025-10-05" },
  { id: "PRM-003", tanggal: "2025-10-07", unit: "Operasional", deskripsi: "Checklist harian DC", tenggat: "2025-10-10", waktu: "15-10-25", pic: "Budi", idBuktiTerkait: [], status: "Belum", tanggalPemenuhan: "" },
];

// --- UI atoms --------------------------------------------------------------
function TextInput({ label, value, onChange, type = "text", placeholder, required, className, min, max, inputMode, pattern }: any) {
  return (
    <label className="block text-sm mb-3">
      <div className="mb-1 text-neutral-600 dark:text-neutral-300 font-medium">{label}</div>
      <input
        className={cls("w-full rounded-xl border border-neutral-300 dark:border-neutral-700 bg-white dark:bg-neutral-900 px-3 py-2 outline-none focus:ring-2 focus:ring-indigo-500", className)}
        value={value ?? ""}
        onChange={(e) => onChange?.(e.target.value)}
        type={type}
        placeholder={placeholder}
        required={required}
        min={min}
        max={max}
        inputMode={inputMode}
        pattern={pattern}
      />
    </label>
  );
}
function Select({ label, value, onChange, options = [], className }: any) {
  return (
    <label className="block text-sm mb-3">
      <div className="mb-1 text-neutral-600 dark:text-neutral-300 font-medium">{label}</div>
      <select
        className={cls("w-full rounded-xl border border-neutral-300 dark:border-neutral-700 bg-white dark:bg-neutral-900 px-3 py-2 outline-none focus:ring-2 focus:ring-indigo-500", className)}
        value={value ?? ""}
        onChange={(e) => onChange?.(e.target.value)}
      >
        <option value="">-- Pilih --</option>
        {options.map((opt: string) => (
          <option key={opt} value={opt}>{opt}</option>
        ))}
      </select>
    </label>
  );
}
function TextArea({ label, value, onChange, rows = 3, placeholder, className }: any) {
  return (
    <label className="block text-sm mb-3">
      <div className="mb-1 text-neutral-600 dark:text-neutral-300 font-medium">{label}</div>
      <textarea
        className={cls("w-full rounded-xl border border-neutral-300 dark:border-neutral-700 bg-white dark:bg-neutral-900 px-3 py-2 outline-none focus:ring-2 focus:ring-indigo-500", className)}
        value={value ?? ""}
        onChange={(e) => onChange?.(e.target.value)}
        rows={rows}
        placeholder={placeholder}
      />
    </label>
  );
}
function Button({ children, onClick, variant = "default", className, type = "button", disabled = false }: any) {
  const base = "inline-flex items-center gap-2 rounded-2xl px-4 py-2 text-sm font-medium transition shadow-sm active:scale-[.98]";
  const styles: Record<string, string> = {
    default: "bg-indigo-600 text-white hover:bg-indigo-700",
    outline: "border border-neutral-300 dark:border-neutral-700 text-neutral-800 dark:text-neutral-200 hover:bg-neutral-50/40 dark:hover:bg-neutral-800/40",
    ghost: "text-neutral-700 dark:text-neutral-300 hover:bg-neutral-100/60 dark:hover:bg-neutral-800/60",
    danger: "bg-rose-600 text-white hover:bg-rose-700",
    success: "bg-emerald-600 text-white hover:bg-emerald-700",
  };
  return (
    <button type={type} disabled={disabled} className={cls(base, styles[variant], disabled && "opacity-60 cursor-not-allowed", className)} onClick={onClick}>
      {children}
    </button>
  );
}
function Card({ children, className }: any) { return <div className={cls("bg-white dark:bg-neutral-900 rounded-2xl shadow p-5", className)}>{children}</div>; }
function Stat({ icon: Icon, label, value, suffix, tone = "indigo" }: any) {
  const toneBg: Record<string, string> = {
    indigo: "bg-indigo-50 dark:bg-indigo-900/40 text-indigo-600 dark:text-indigo-300",
    amber: "bg-amber-50 dark:bg-amber-900/30 text-amber-600 dark:text-amber-300",
    rose: "bg-rose-50 dark:bg-rose-900/30 text-rose-600 dark:text-rose-300",
    emerald: "bg-emerald-50 dark:bg-emerald-900/30 text-emerald-600 dark:text-emerald-300",
  };
  return (
    <Card>
      <div className="flex items-center gap-3">
        <div className={cls("p-2 rounded-xl", toneBg[tone])}><Icon size={20} /></div>
        <div>
          <div className="text-sm text-neutral-500">{label}</div>
          <div className="text-2xl font-semibold tracking-tight">
            {value}{suffix && (<span className="text-base font-normal text-neutral-500 ml-1">{suffix}</span>)}
          </div>
        </div>
      </div>
    </Card>
  );
}
function Modal({ open, onClose, title, children, footer }: any) {
  if (!open) return null;
  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center">
      <div className="absolute inset-0 bg-black/40" onClick={onClose} />
      <motion.div initial={{ opacity: 0, scale: 0.95 }} animate={{ opacity: 1, scale: 1 }} exit={{ opacity: 0, scale: 0.95 }} className="relative w-full max-w-3xl bg-white dark:bg-neutral-900 rounded-2xl shadow-xl p-6">
        <div className="flex items-center justify-between mb-4">
          <h3 className="text-lg font-semibold">{title}</h3>
          <button onClick={onClose} className="p-2 rounded-lg hover:bg-neutral-100 dark:hover:bg-neutral-800"><X size={18} /></button>
        </div>
        <div className="max-h-[70vh] overflow-auto pr-1">{children}</div>
        {footer && <div className="mt-4 flex justify-end gap-2">{footer}</div>}
      </motion.div>
    </div>
  );
}
function EmptyState({ icon: Icon, title, subtitle, action }: any) {
  return (
    <Card className="flex flex-col items-center text-center py-16">
      <div className="p-4 rounded-2xl bg-neutral-100 dark:bg-neutral-800 text-neutral-600 dark:text-neutral-300 mb-4"><Icon size={28} /></div>
      <div className="text-xl font-semibold mb-1">{title}</div>
      <div className="text-neutral-500 mb-4 max-w-md">{subtitle}</div>
      {action}
    </Card>
  );
}

// --- Main App --------------------------------------------------------------
export default function AuditEvidenceApp() {
  const loaded = useMemo(loadState, []);
  const [bukti, setBukti] = useState<Bukti[]>(loaded?.bukti ?? seedBukti);
  // migrate: ensure tiap permintaan punya `waktu`
  const initialPermintaan: Permintaan[] = (loaded?.permintaan ?? seedReq).map((p: any) => ({ waktu: p.waktu ?? "", ...p }));
  const [permintaan, setPermintaan] = useState<Permintaan[]>(initialPermintaan);
  const [threshold, setThreshold] = useState<number>(typeof loaded?.threshold === "number" ? loaded.threshold : 7);
  const [activeTab, setActiveTab] = useState("dashboard");
  const [query, setQuery] = useState("");
  const { installPrompt, installed } = usePWA();
  const isStandalone = useMemo(() => {
    try { return window.matchMedia?.('(display-mode: standalone)')?.matches || (navigator as any).standalone || false; } catch { return false; }
  }, []);

  useEffect(() => { saveState({ bukti, permintaan, threshold }); }, [bukti, permintaan, threshold]);

  // --- Derived metrics -----------------------------------------------------
  const totalReq = permintaan.length;
  const terpenuhi = permintaan.filter((p) => p.status === "Terpenuhi").length;
  const persen = totalReq ? Math.round((terpenuhi / totalReq) * 100) : 0;
  const statusData = [ { name: "Terpenuhi", value: terpenuhi }, { name: "Belum", value: totalReq - terpenuhi } ];
  const byUnit = useMemo(() => {
    const map = new Map<string, number>();
    for (const p of permintaan) map.set(p.unit, (map.get(p.unit) || 0) + 1);
    return Array.from(map, ([unit, count]) => ({ unit, count }));
  }, [permintaan]);
  const buktiByUnit = useMemo(() => {
    const map = new Map<string, number>();
    for (const b of bukti) map.set(b.unit, (map.get(b.unit) || 0) + 1);
    return Array.from(map, ([unit, count]) => ({ unit, count }));
  }, [bukti]);

  // Deadline analytics (tenggat)
  const withDays = useMemo(() => permintaan.map((p) => ({ ...p, _days: daysUntil(p.tenggat) })), [permintaan]);
  const openReq = useMemo(() => withDays.filter((p) => p.status !== "Terpenuhi"), [withDays]);
  const overdueCount = useMemo(() => openReq.filter((p) => (p._days ?? 0) < 0).length, [openReq]);
  const approachingCount = useMemo(() => openReq.filter((p) => (p._days ?? 9999) >= 0 && (p._days as number) <= threshold).length, [openReq, threshold]);
  const nearest = useMemo(() => { const upcoming = openReq.filter((p) => (p._days ?? -1) >= 0); return upcoming.sort((a, b) => (a._days as number) - (b._days as number))[0] || null; }, [openReq]);

  // --- Import / Export -----------------------------------------------------
  function exportJSON() {
    const blob = new Blob([JSON.stringify({ bukti, permintaan, threshold }, null, 2)], { type: "application/json;charset=utf-8" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url; a.download = `audit-evidence-${new Date().toISOString().slice(0, 10)}.json`; a.click();
    setTimeout(() => URL.revokeObjectURL(url), 1000);
  }
  function onImportFile(e: any) {
    const file = e.target.files?.[0]; if (!file) return;
    const reader = new FileReader();
    reader.onload = () => {
      try {
        const data = JSON.parse(reader.result as string);
        const incomingPermintaanRaw = Array.isArray(data?.permintaan) ? data.permintaan : Array.isArray(data?.requests) ? data.requests : null;
        const incomingPermintaan: Permintaan[] | null = incomingPermintaanRaw ? incomingPermintaanRaw.map((p: any) => ({ waktu: p.waktu ?? "", ...p })) : null;
        if (Array.isArray(data?.bukti) && incomingPermintaan) {
          setBukti(data.bukti); setPermintaan(incomingPermintaan);
          if (typeof data.threshold === "number") setThreshold(Math.max(0, data.threshold));
        } else { alert("Format file tidak valid. Harus berisi { bukti: [], permintaan: [] }."); }
      } catch { alert("Gagal membaca file JSON."); }
    };
    reader.readAsText(file); e.target.value = "";
  }

  // --- CRUD helpers --------------------------------------------------------
  function upsertBukti(entry: Bukti) {
    setBukti((prev) => { const idx = prev.findIndex((b) => b.id === entry.id); if (idx === -1) return [...prev, entry]; const next = prev.slice(); next[idx] = entry; return next; });
  }
  function deleteBukti(id: string) {
    if (!confirm(`Hapus ${id}?`)) return;
    setBukti((prev) => prev.filter((b) => b.id !== id));
    setPermintaan((prev) => prev.map((p) => ({ ...p, idBuktiTerkait: p.idBuktiTerkait.filter((x) => x !== id) })));
  }
  function upsertPermintaan(entry: Permintaan) {
    setPermintaan((prev) => { const idx = prev.findIndex((p) => p.id === entry.id); if (idx === -1) return [...prev, entry]; const next = prev.slice(); next[idx] = entry; return next; });
  }
  function deletePermintaan(id: string) {
    if (!confirm(`Hapus ${id}?`)) return;
    setPermintaan((prev) => prev.filter((p) => p.id !== id));
    setBukti((prev) => prev.map((b) => ({ ...b, idPermintaanTerkait: b.idPermintaanTerkait === id ? "" : b.idPermintaanTerkait })));
  }

  // --- Filtered views ------------------------------------------------------
  const filteredBukti = useMemo(() => {
    const q = query.toLowerCase(); if (!q) return bukti;
    return bukti.filter((b) => [b.id, b.kategori, b.deskripsi, b.unit, b.pic, b.validitas, b.catatan, b.idPermintaanTerkait].filter(Boolean).some((v) => String(v).toLowerCase().includes(q)));
  }, [bukti, query]);
  const filteredPermintaan = useMemo(() => {
    const q = query.toLowerCase(); if (!q) return permintaan;
    return permintaan.filter((p) => [p.id, p.unit, p.deskripsi, p.pic, p.status, p.tanggal, p.tenggat, p.waktu].filter(Boolean).some((v) => String(v).toLowerCase().includes(q)));
  }, [permintaan, query]);

  // --- Download ZIP (per permintaan) --------------------------------------
  async function downloadBuktiPermintaan(p: Permintaan) {
    const zip = new JSZip(); const fails: string[] = []; const manifest: string[] = ["ID_Bukti,Deskripsi,Link,Unit,PIC"];
    for (const bid of p.idBuktiTerkait || []) {
      const b = bukti.find((x) => x.id === bid);
      if (!b) { fails.push(`${bid}: tidak ditemukan di bank bukti`); continue; }
      manifest.push([csv(b.id), csv(b.deskripsi), csv(b.link), csv(b.unit), csv(b.pic)].join(","));
      if (!b.link) { fails.push(`${bid}: link tidak tersedia`); continue; }
      try {
        const resp = await fetch(b.link, { mode: "cors" }); if (!resp.ok) throw new Error(`HTTP ${resp.status}`);
        const blob = await resp.blob();
        const urlObj = new URL(b.link, window.location.href); const lastSeg = urlObj.pathname.split("/").filter(Boolean).pop() || "file";
        let filename = lastSeg;
        if (!/\./.test(filename)) { const ext = (blob.type && blob.type.split("/")[1]) || "bin"; filename = `${slugify(b.deskripsi) || "bukti"}-${bid}.${ext}`; } else { filename = `${bid}-${filename}`; }
        const buf = await blob.arrayBuffer(); zip.file(filename, buf);
      } catch (err: any) { fails.push(`${bid}: ${err?.message || err}`); }
    }
    zip.file("MANIFEST.csv", manifest.join('\n'));
    if (fails.length) zip.file("GAGAL-UNDUH.txt", fails.join('\n'));
    const out = await zip.generateAsync({ type: "blob" }); saveAsBlob(out, `${p.id}-bukti.zip`);
  }

  // --- Download ZIP (SEMUA permintaan Terpenuhi) --------------------------
  async function downloadSemuaTerpenuhi() {
    const list = permintaan.filter((p) => p.status === "Terpenuhi" && (p.idBuktiTerkait?.length || 0) > 0);
    if (!list.length) { alert("Tidak ada permintaan berstatus 'Terpenuhi' dengan bukti."); return; }
    const zip = new JSZip(); const fails: string[] = []; const manifest: string[] = ["ID_Permintaan,ID_Bukti,Deskripsi,Link,Unit,PIC,Path"];
    for (const p of list) {
      const folder = zip.folder(p.id)!;
      for (const bid of p.idBuktiTerkait) {
        const b = bukti.find((x) => x.id === bid);
        if (!b) { fails.push(`${p.id}/${bid}: tidak ditemukan di bank bukti`); continue; }
        let filename = `${bid}-file`;
        try {
          if (!b.link) throw new Error("link tidak tersedia");
          const resp = await fetch(b.link, { mode: "cors" }); if (!resp.ok) throw new Error(`HTTP ${resp.status}`);
          const blob = await resp.blob();
          const urlObj = new URL(b.link, window.location.href); const lastSeg = urlObj.pathname.split("/").filter(Boolean).pop() || "file";
          if (!/\./.test(lastSeg)) { const ext = (blob.type && blob.type.split("/")[1]) || "bin"; filename = `${slugify(b.deskripsi) || "bukti"}-${bid}.${ext}`; } else { filename = `${bid}-${lastSeg}`; }
          const buf = await blob.arrayBuffer(); folder.file(filename, buf);
          manifest.push([csv(p.id), csv(b.id), csv(b.deskripsi), csv(b.link), csv(b.unit), csv(b.pic), csv(`${p.id}/${filename}`)].join(","));
        } catch (err: any) { fails.push(`${p.id}/${bid}: ${err?.message || err}`); }
      }
    }
    zip.file("MANIFEST.csv", manifest.join('\n'));
    if (fails.length) zip.file("GAGAL-UNDUH.txt", fails.join('\n'));
    const out = await zip.generateAsync({ type: "blob" });
    saveAsBlob(out, `bukti-terpenuhi-${new Date().toISOString().slice(0, 10)}.zip`);
  }

  return (
    <div className="min-h-screen bg-gradient-to-b from-neutral-50 to-white dark:from-neutral-950 dark:to-neutral-900 text-neutral-900 dark:text-neutral-100">
      <div className="max-w-7xl mx-auto px-5 py-8">
        {/* Header */}
        <div className="flex flex-col md:flex-row md:items-center md:justify-between gap-4 mb-8">
          <div className="flex items-center gap-3">
            <div className="p-3 rounded-2xl bg-indigo-600 text-white shadow-lg"><Layers /></div>
            <div>
              <h1 className="text-2xl md:text-3xl font-bold tracking-tight">Instrument Bank Bukti & Permintaan Audit</h1>
              <p className="text-neutral-500">Kelola bank bukti, permintaan bukti, dan pantau persentase pemenuhan.</p>
            </div>
          </div>
          <div className="flex flex-col md:flex-row md:items-center gap-2 md:gap-3">
            <label className="relative">
              <Search className="absolute left-3 top-1/2 -translate-y-1/2 text-neutral-400" size={18} />
              <input value={query} onChange={(e) => setQuery(e.target.value)} placeholder="Cari ID, deskripsi, unit, PIC..." className="pl-9 pr-3 py-2 rounded-2xl border border-neutral-300 dark:border-neutral-700 bg-white dark:bg-neutral-900 outline-none focus:ring-2 focus:ring-indigo-500" />
            </label>
            <div className="flex items-center gap-2 md:ml-2">
              <SettingsIcon size={16} className="text-neutral-400" />
              <span className="text-sm text-neutral-600 dark:text-neutral-300">Batas Peringatan</span>
              <input type="number" min={0} max={365} value={threshold} onChange={(e) => setThreshold(Math.max(0, parseInt(e.target.value || "0", 10) || 0))} className="w-20 rounded-xl border border-neutral-300 dark:border-neutral-700 bg-white dark:bg-neutral-900 px-2 py-1 outline-none focus:ring-2 focus:ring-indigo-500 text-sm" title="Jumlah hari sebelum deadline yang dianggap mendekati" />
              <span className="text-sm text-neutral-500">hari</span>
            </div>
            <div className="flex items-center gap-2">
              {installPrompt ? (
                <Button onClick={async () => { try { await installPrompt.prompt(); await installPrompt.userChoice; } catch {} }}><Smartphone size={16} /> Install App</Button>
              ) : (isStandalone || installed) ? (
                <span className="text-xs px-2 py-1 rounded-lg bg-emerald-100 text-emerald-700 dark:bg-emerald-900/30 dark:text-emerald-300">Terinstal</span>
              ) : null}
              <Button variant="outline" onClick={exportJSON}><FileDown size={16} /> Export</Button>
              <label className="cursor-pointer">
                <input type="file" accept="application/json" className="hidden" onChange={onImportFile} />
                <span className="inline-flex items-center gap-2 rounded-2xl border border-neutral-300 dark:border-neutral-700 px-4 py-2 text-sm font-medium text-neutral-800 dark:text-neutral-200 hover:bg-neutral-50/40 dark:hover:bg-neutral-800/40"><FileUp size={16} /> Import</span>
              </label>
            </div>
          </div>
        </div>

        {/* Tabs */}
        <div className="flex items-center gap-2 mb-6">
          <TabButton icon={LayoutDashboard} active={activeTab === "dashboard"} onClick={() => setActiveTab("dashboard")}>Dashboard</TabButton>
          <TabButton icon={ClipboardList} active={activeTab === "permintaan"} onClick={() => setActiveTab("permintaan")}>Permintaan</TabButton>
          <TabButton icon={LinkIcon} active={activeTab === "bukti"} onClick={() => setActiveTab("bukti")}>Bank Bukti</TabButton>
        </div>

        {activeTab === "dashboard" && (
          <DashboardTab
            totalReq={totalReq}
            terpenuhi={terpenuhi}
            persen={persen}
            statusData={statusData}
            byUnit={byUnit}
            buktiByUnit={buktiByUnit}
            overdueCount={overdueCount}
            approachingCount={approachingCount}
            nearest={nearest}
          />
        )}

        {activeTab === "permintaan" && (
          <PermintaanTab
            permintaan={filteredPermintaan}
            bukti={bukti}
            threshold={threshold}
            onUpsert={(entry: Permintaan) => setPermintaan((prev) => { const idx = prev.findIndex((p) => p.id === entry.id); if (idx === -1) return [...prev, entry]; const next = prev.slice(); next[idx] = entry; return next; })}
            onDelete={deletePermintaan}
            onMarkDone={(id: string) => { const p = permintaan.find((x) => x.id === id); if (!p) return; const newP = { ...p, status: "Terpenuhi", tanggalPemenuhan: new Date().toISOString().slice(0, 10) } as Permintaan; setPermintaan((prev) => prev.map((it) => (it.id === id ? newP : it))); }}
            nextId={(items: Permintaan[]) => nextId("PRM", items)}
            onDownload={downloadBuktiPermintaan}
            onDownloadAll={downloadSemuaTerpenuhi}
          />
        )}

        {activeTab === "bukti" && (
          <BuktiTab bukti={filteredBukti} permintaan={permintaan} onUpsert={upsertBukti} onDelete={deleteBukti} nextId={(items: Bukti[]) => nextId("BKT", items)} />
        )}
      </div>
    </div>
  );
}

function TabButton({ children, icon: Icon, active, onClick }: any) {
  return (
    <button onClick={onClick} className={cls("flex items-center gap-2 px-4 py-2 rounded-2xl text-sm font-medium border", active ? "bg-indigo-600 text-white border-indigo-600" : "border-neutral-300 dark:border-neutral-700 text-neutral-700 dark:text-neutral-300 hover:bg-neutral-100/60 dark:hover:bg-neutral-800/60")}> <Icon size={16} /> {children} </button>
  );
}

// --- Dashboard Tab ---------------------------------------------------------
function DashboardTab({ totalReq, terpenuhi, persen, statusData, byUnit, buktiByUnit, overdueCount, approachingCount, nearest }: any) {
  return (
    <div className="space-y-6">
      <div className="grid grid-cols-1 md:grid-cols-4 gap-4">
        <Stat icon={ClipboardList} label="Total Permintaan" value={totalReq} />
        <Stat icon={CheckCircle2} label="Terpenuhi" value={terpenuhi} tone="emerald" />
        <Stat icon={AlertTriangle} label="Mendekati Deadline" value={approachingCount} tone="amber" />
        <Stat icon={Clock3} label="Terlambat" value={overdueCount} tone="rose" />
      </div>
      <Card>
        <div className="flex items-center justify-between gap-4 flex-wrap">
          <div className="flex items-center gap-6">
            <div>
              <div className="text-sm text-neutral-500">Persentase Terpenuhi</div>
              <div className="text-2xl font-semibold">{persen}<span className="text-base font-normal text-neutral-500 ml-1">%</span></div>
              <div className="mt-3 h-3 w-full rounded-full bg-neutral-200 dark:bg-neutral-800 overflow-hidden">
                <div className="h-full bg-indigo-600" style={{ width: `${Math.min(100, Math.max(0, persen))}%` }} />
              </div>
            </div>
          </div>
          <div className="flex gap-6 text-sm">
            {nearest ? (
              <div className="text-right min-w-[220px]">
                <div className="text-neutral-500">Deadline Terdekat</div>
                <div className="font-semibold">{nearest.id} · {daysLabel(nearest._days)} ({nearest.tenggat})</div>
              </div>
            ) : (
              <div className="text-neutral-500 min-w-[220px]">Tidak ada deadline mendatang</div>
            )}
          </div>
        </div>
      </Card>
      <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
        <Card>
          <div className="flex items-center justify-between mb-3"><h3 className="font-semibold">Distribusi Status Permintaan</h3></div>
          <div className="h-72">
            <ResponsiveContainer width="100%" height="100%">
              <PieChart>
                <Pie data={statusData} dataKey="value" nameKey="name" outerRadius={110} label>
                  {statusData.map((_: any, i: number) => (<Cell key={i} fill={["#4f46e5", "#e11d48", "#10b981", "#f59e0b", "#06b6d4", "#8b5cf6"][i % 6]} />))}
                </Pie>
                <ReTooltip />
                <Legend />
              </PieChart>
            </ResponsiveContainer>
          </div>
        </Card>
        <Card>
          <h3 className="font-semibold mb-3">Jumlah Permintaan per Unit</h3>
          <div className="h-72">
            <ResponsiveContainer width="100%" height="100%">
              <BarChart data={byUnit}>
                <CartesianGrid strokeDasharray="3 3" />
                <XAxis dataKey="unit" />
                <YAxis allowDecimals={false} />
                <ReTooltip />
                <Bar dataKey="count" name="Permintaan" fill="#4f46e5" />
              </BarChart>
            </ResponsiveContainer>
          </div>
        </Card>
        <Card className="lg:col-span-2">
          <h3 className="font-semibold mb-3">Jumlah Bukti per Unit</h3>
          <div className="h-72">
            <ResponsiveContainer width="100%" height="100%">
              <BarChart data={buktiByUnit}>
                <CartesianGrid strokeDasharray="3 3" />
                <XAxis dataKey="unit" />
                <YAxis allowDecimals={false} />
                <ReTooltip />
                <Bar dataKey="count" name="Bukti" fill="#10b981" />
              </BarChart>
            </ResponsiveContainer>
          </div>
        </Card>
      </div>
    </div>
  );
}

// --- Permintaan Tab --------------------------------------------------------
function PermintaanTab({ permintaan, bukti, threshold, onUpsert, onDelete, onMarkDone, nextId, onDownload, onDownloadAll }:{ permintaan: Permintaan[]; bukti: Bukti[]; threshold: number; onUpsert: (entry: Permintaan) => void; onDelete: (id: string) => void; onMarkDone: (id: string) => void; nextId: (items: Permintaan[]) => string; onDownload: (p: Permintaan) => void; onDownloadAll: () => void; }) {
  const [open, setOpen] = useState(false);
  const [editing, setEditing] = useState<Permintaan | null>(null);

  const hasAnyTerpenuhi = useMemo(() => permintaan.some((p) => p.status === "Terpenuhi" && (p.idBuktiTerkait?.length || 0) > 0), [permintaan]);

  function startCreate() {
    setEditing({ id: nextId(permintaan), tanggal: new Date().toISOString().slice(0, 10), unit: "", deskripsi: "", tenggat: "", waktu: "", pic: "", idBuktiTerkait: [], status: "Belum", tanggalPemenuhan: "" });
    setOpen(true);
  }
  function startEdit(p: Permintaan) { setEditing({ ...p, idBuktiTerkait: [...(p.idBuktiTerkait || [])] }); setOpen(true); }
  function save() { if (!editing || !editing.id || !editing.deskripsi) { alert("ID dan Deskripsi wajib diisi"); return; } onUpsert(editing); setOpen(false); }
  function sisaHari(p: Permintaan) { return daysUntil(p.tenggat); }
  function sisaWaktu(p: Permintaan) { return daysUntilDDMMYY(p.waktu); }

  return (
    <div className="space-y-4">
      <div className="flex justify-between items-center">
        <h2 className="text-xl font-semibold">Daftar Permintaan</h2>
        <div className="flex items-center gap-2">
          <Button variant="outline" onClick={onDownloadAll} disabled={!hasAnyTerpenuhi} title="Unduh semua bukti dari permintaan berstatus Terpenuhi"><FileDown size={16} /> Download Terpenuhi</Button>
          <Button onClick={startCreate}><Plus size={16} /> Tambah Permintaan</Button>
        </div>
      </div>

      {permintaan.length === 0 ? (
        <EmptyState icon={ClipboardList} title="Belum ada permintaan" subtitle="Tambahkan permintaan baru untuk mulai melacak pemenuhan bukti." action={<Button onClick={startCreate}><Plus size={16} /> Buat Permintaan</Button>} />
      ) : (
        <Card>
          <div className="overflow-x-auto">
            <table className="min-w-full text-sm">
              <thead>
                <tr className="text-left border-b border-neutral-200 dark:border-neutral-800">
                  {["ID","Tanggal","Unit","Deskripsi","Tenggat","Sisa Hari","Waktu (DD-MM-YY)","Sisa Waktu","PIC","Bukti Terkait","Status","Pemenuhan","Aksi"].map((h) => (
                    <th key={h} className="py-2 px-2 font-semibold text-neutral-600 dark:text-neutral-300 whitespace-nowrap">{h}</th>
                  ))}
                </tr>
              </thead>
              <tbody>
                {permintaan.map((p) => {
                  const dh = sisaHari(p);
                  const dw = sisaWaktu(p);
                  const toneD = dh === null ? "" : dh < 0 ? "bg-rose-100 text-rose-700 dark:bg-rose-900/30 dark:text-rose-300" : dh <= threshold ? "bg-amber-100 text-amber-700 dark:bg-amber-900/30 dark:text-amber-300" : "bg-emerald-100 text-emerald-700 dark:bg-emerald-900/30 dark:text-emerald-300";
                  const toneW = dw === null ? "" : dw < 0 ? "bg-rose-100 text-rose-700 dark:bg-rose-900/30 dark:text-rose-300" : dw <= threshold ? "bg-amber-100 text-amber-700 dark:bg-amber-900/30 dark:text-amber-300" : "bg-emerald-100 text-emerald-700 dark:bg-emerald-900/30 dark:text-emerald-300";
                  return (
                    <tr key={p.id} className="border-b border-neutral-100 dark:border-neutral-800 hover:bg-neutral-50/60 dark:hover:bg-neutral-800/40">
                      <td className="py-2 px-2 font-mono whitespace-nowrap">{p.id}</td>
                      <td className="py-2 px-2 whitespace-nowrap">{p.tanggal}</td>
                      <td className="py-2 px-2 whitespace-nowrap">{p.unit}</td>
                      <td className="py-2 px-2 max-w-xl truncate" title={p.deskripsi}>{p.deskripsi}</td>
                      <td className="py-2 px-2 whitespace-nowrap">{p.tenggat || <span className="text-neutral-400">-</span>}</td>
                      <td className="py-2 px-2 whitespace-nowrap"><span className={cls("px-2 py-0.5 rounded-full text-xs font-medium", toneD)}>{daysLabel(dh)}</span></td>
                      <td className="py-2 px-2 whitespace-nowrap">{p.waktu || <span className="text-neutral-400">-</span>}</td>
                      <td className="py-2 px-2 whitespace-nowrap"><span className={cls("px-2 py-0.5 rounded-full text-xs font-medium", toneW)}>{daysLabel(dw)}</span></td>
                      <td className="py-2 px-2 whitespace-nowrap">{p.pic}</td>
                      <td className="py-2 px-2">
                        {p.idBuktiTerkait?.length ? (
                          <div className="flex flex-wrap gap-1">
                            {p.idBuktiTerkait.map((id) => (
                              <span key={id} className="text-xs font-mono bg-neutral-100 dark:bg-neutral-800 rounded px-2 py-0.5">{id}</span>
                            ))}
                          </div>
                        ) : (<span className="text-neutral-400">-</span>)}
                      </td>
                      <td className="py-2 px-2 whitespace-nowrap">
                        <span className={cls("px-2 py-0.5 rounded-full t
