
import { CSSProperties, FormEvent, useEffect, useMemo, useState } from "react";

type Expense = { id: string; amount: number; category: string; merchant: string; payment: string; date: string; note: string };
type Vault = { name: string; budget: number; expenses: Expense[] };

const categories = [
  { name: "Comida", emoji: "🍔", color: "mint", hex: "#55d6a0" },
  { name: "Transporte", emoji: "🚌", color: "blue", hex: "#5b7cff" },
  { name: "Hogar", emoji: "🛋️", color: "coral", hex: "#ff7465" },
  { name: "Ocio", emoji: "🎮", color: "purple", hex: "#8b63f6" },
  { name: "Salud", emoji: "🩺", color: "pink", hex: "#f67cad" },
  { name: "Compras", emoji: "🛍️", color: "yellow", hex: "#f6c945" },
  { name: "Cuentas", emoji: "🧾", color: "cyan", hex: "#56c7df" },
  { name: "Otros", emoji: "✨", color: "gray", hex: "#aeb4c5" },
];

const money = (n: number) => new Intl.NumberFormat("es-CL", { style: "currency", currency: "CLP", maximumFractionDigits: 0 }).format(n);
const today = () => new Intl.DateTimeFormat("en-CA", { timeZone: "America/Santiago", year: "numeric", month: "2-digit", day: "2-digit" }).format(new Date());
const monthKey = (date: string) => date.slice(0, 7);

async function deriveKey(pin: string, salt: Uint8Array) {
  const material = await crypto.subtle.importKey("raw", new TextEncoder().encode(pin), "PBKDF2", false, ["deriveKey"]);
  return crypto.subtle.deriveKey({ name: "PBKDF2", salt: salt.buffer as ArrayBuffer, iterations: 180000, hash: "SHA-256" }, material, { name: "AES-GCM", length: 256 }, false, ["encrypt", "decrypt"]);
}
const bytesToB64 = (bytes: Uint8Array) => btoa(String.fromCharCode(...bytes));
const b64ToBytes = (value: string) => Uint8Array.from(atob(value), c => c.charCodeAt(0));

async function encryptVault(vault: Vault, pin: string) {
  if (!crypto.subtle) return JSON.stringify({ v: 0, dev: btoa(unescape(encodeURIComponent(JSON.stringify(vault)))), pin });
  const salt = crypto.getRandomValues(new Uint8Array(16));
  const iv = crypto.getRandomValues(new Uint8Array(12));
  const key = await deriveKey(pin, salt);
  const encrypted = await crypto.subtle.encrypt({ name: "AES-GCM", iv: iv.buffer as ArrayBuffer }, key, new TextEncoder().encode(JSON.stringify(vault)));
  return JSON.stringify({ v: 1, salt: bytesToB64(salt), iv: bytesToB64(iv), data: bytesToB64(new Uint8Array(encrypted)) });
}

async function decryptVault(payload: string, pin: string): Promise<Vault> {
  const parsed = JSON.parse(payload);
  if (parsed.v === 0) {
    if (parsed.pin !== pin) throw new Error("invalid pin");
    return JSON.parse(decodeURIComponent(escape(atob(parsed.dev))));
  }
  const key = await deriveKey(pin, b64ToBytes(parsed.salt));
  const decrypted = await crypto.subtle.decrypt({ name: "AES-GCM", iv: b64ToBytes(parsed.iv).buffer as ArrayBuffer }, key, b64ToBytes(parsed.data).buffer as ArrayBuffer);
  return JSON.parse(new TextDecoder().decode(decrypted));
}

export default function Home() {
  const [vault, setVault] = useState<Vault | null>(null);
  const [pin, setPin] = useState("");
  const [stored, setStored] = useState(false);
  const [ready, setReady] = useState(false);
  const [authError, setAuthError] = useState("");
  const [showAdd, setShowAdd] = useState(false);
  const [showSettings, setShowSettings] = useState(false);
  const [filter, setFilter] = useState("Todas");
  const [editing, setEditing] = useState<Expense | null>(null);
  const [toast, setToast] = useState("");

  useEffect(() => {
    queueMicrotask(() => {
      setStored(Boolean(localStorage.getItem("mi-plata-vault")));
      setReady(true);
    });
  }, []);

  useEffect(() => {
    if (!vault || pin.length < 4) return;
    const timer = setTimeout(async () => localStorage.setItem("mi-plata-vault", await encryptVault(vault, pin)), 250);
    return () => clearTimeout(timer);
  }, [vault, pin]);

  const currentMonth = today().slice(0, 7);
  const monthExpenses = useMemo(() => vault?.expenses.filter(e => monthKey(e.date) === currentMonth) ?? [], [vault, currentMonth]);
  const total = monthExpenses.reduce((sum, e) => sum + e.amount, 0);
  const categoryTotals = categories.map(c => ({ ...c, total: monthExpenses.filter(e => e.category === c.name).reduce((s, e) => s + e.amount, 0) })).sort((a, b) => b.total - a.total);
  const visible = (filter === "Todas" ? monthExpenses : monthExpenses.filter(e => e.category === filter)).sort((a, b) => b.date.localeCompare(a.date));
  const progress = vault ? Math.min(100, Math.round((total / Math.max(vault.budget, 1)) * 100)) : 0;
  const monthSeries = useMemo(() => {
    const [year, month] = currentMonth.split("-").map(Number);
    return Array.from({ length: 6 }, (_, index) => {
      const date = new Date(year, month - 1 - (5 - index), 1, 12);
      const key = `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, "0")}`;
      const value = vault?.expenses.filter(e => monthKey(e.date) === key).reduce((sum, e) => sum + e.amount, 0) ?? 0;
      return { key, value, label: date.toLocaleDateString("es-CL", { month: "short" }).replace(".", "") };
    });
  }, [vault, currentMonth]);
  const maxMonth = Math.max(...monthSeries.map(m => m.value), 1);
  const activeCategories = categoryTotals.filter(c => c.total > 0);
  let donutCursor = 0;
  const donut = activeCategories.length ? `conic-gradient(${activeCategories.map(c => { const start = donutCursor; donutCursor += c.total / total * 100; return `${c.hex} ${start}% ${donutCursor}%`; }).join(",")})` : "#eeecf6";
  const daysInMonth = new Date(Number(currentMonth.slice(0, 4)), Number(currentMonth.slice(5, 7)), 0).getDate();
  const elapsedDays = Number(today().slice(8, 10));
  const spendDays = new Set(monthExpenses.map(e => e.date)).size;
  const averagePerSpendDay = spendDays ? total / spendDays : 0;

  const notify = (message: string) => { setToast(message); setTimeout(() => setToast(""), 2400); };

  async function unlock(e: FormEvent<HTMLFormElement>) {
    e.preventDefault(); setAuthError("");
    const fd = new FormData(e.currentTarget); const entered = String(fd.get("pin"));
    try { const data = await decryptVault(localStorage.getItem("mi-plata-vault")!, entered); setPin(entered); setVault(data); }
    catch { setAuthError("La clave no coincide. Inténtalo nuevamente."); }
  }

  function createVault(e: FormEvent<HTMLFormElement>) {
    e.preventDefault(); const fd = new FormData(e.currentTarget); const newPin = String(fd.get("pin"));
    if (newPin.length < 4) return setAuthError("Usa una clave de al menos 4 números.");
    setPin(newPin); setVault({ name: String(fd.get("name") || "Walter"), budget: Number(fd.get("budget")) || 800000, expenses: [] }); setStored(true);
  }

  function saveExpense(e: FormEvent<HTMLFormElement>) {
    e.preventDefault(); if (!vault) return; const fd = new FormData(e.currentTarget);
    const expense: Expense = { id: editing?.id ?? crypto.randomUUID?.() ?? `${Date.now()}-${Math.random().toString(36).slice(2)}`, amount: Number(fd.get("amount")), category: String(fd.get("category")), merchant: String(fd.get("merchant")), payment: String(fd.get("payment")), date: String(fd.get("date")), note: String(fd.get("note")) };
    setVault({ ...vault, expenses: editing ? vault.expenses.map(x => x.id === editing.id ? expense : x) : [expense, ...vault.expenses] });
    setShowAdd(false); setEditing(null); notify(editing ? "Gasto actualizado ✓" : "¡Gasto guardado! Tu registro va al día ✨");
  }

  function removeExpense(id: string) {
    if (!vault || !confirm("¿Eliminar este gasto?")) return;
    setVault({ ...vault, expenses: vault.expenses.filter(e => e.id !== id) }); setEditing(null); setShowAdd(false); notify("Gasto eliminado");
  }

  async function exportBackup() {
    if (!vault) return; const blob = new Blob([await encryptVault(vault, pin)], { type: "application/json" });
    const link = document.createElement("a"); link.href = URL.createObjectURL(blob); link.download = `mi-plata-respaldo-${today()}.json`; link.click(); URL.revokeObjectURL(link.href); notify("Respaldo descargado 🔐");
  }

  if (!ready) return <main className="loading">Preparando tu billetera…</main>;
  if (!vault) return (
    <main className="auth-page">
      <section className="auth-card">
        <div className="brand-mark">M<span>●</span></div><p className="eyebrow">MI PLATA</p>
        <h1>{stored ? "Qué bueno verte de nuevo" : "Tu plata, por fin clara"}</h1>
        <p>{stored ? "Ingresa tu clave para abrir tus registros." : "Registra cada compra en segundos y descubre en qué se va tu dinero."}</p>
        <form onSubmit={stored ? unlock : createVault}>
          {!stored && <><label>¿Cómo te llamas?<input name="name" defaultValue="Walter" autoComplete="name" /></label><label>Presupuesto mensual<input name="budget" type="number" inputMode="numeric" defaultValue="800000" /></label></>}
          <label>Tu clave privada<input name="pin" type="password" inputMode="numeric" minLength={4} placeholder="••••" autoFocus={stored} /></label>
          {authError && <p className="form-error">{authError}</p>}
          <button className="primary wide" type="submit">{stored ? "Abrir Mi Plata" : "Crear mi espacio"} <span>→</span></button>
        </form>
        <p className="privacy-note">🔒 Tus datos se cifran y quedan guardados solo en este dispositivo.</p>
      </section>
      <div className="auth-art"><div className="floating sticker s1">🍔</div><div className="floating sticker s2">🚌</div><div className="floating sticker s3">🎮</div><div className="big-coin">$</div><h2>Pequeños registros.<br/>Grandes decisiones.</h2></div>
    </main>
  );

  const firstName = vault.name.split(" ")[0];
  return (
    <main className="app-shell">
      <aside className="sidebar">
        <div className="logo"><div className="brand-mark small">M<span>●</span></div><strong>Mi Plata</strong></div>
        <nav><button className="active">⌂ <span>Resumen</span></button><button onClick={() => document.getElementById("movimientos")?.scrollIntoView({ behavior: "smooth" })}>☷ <span>Movimientos</span></button><button onClick={() => document.getElementById("categorias")?.scrollIntoView({ behavior: "smooth" })}>◔ <span>Categorías</span></button><button onClick={() => document.getElementById("analisis")?.scrollIntoView({ behavior: "smooth" })}>⌁ <span>Análisis</span></button></nav>
        <div className="side-bottom"><button onClick={() => setShowSettings(true)}>⚙ <span>Configuración</span></button><div className="profile"><div>{firstName[0]}</div><span>{firstName}</span><button aria-label="Bloquear" onClick={() => { setVault(null); setPin(""); }}>⌁</button></div></div>
      </aside>

      <section className="content">
        <header><div><p className="mobile-brand">MI PLATA</p><h1>Buenas tardes, {firstName} <span className="wave">👋</span></h1></div><button className="round-button" aria-label="Configuración" onClick={() => setShowSettings(true)}>⚙</button></header>
        <section className="hero-card">
          <div><p>Gastado este mes</p><strong>{money(total)}</strong><small>{monthExpenses.length === 0 ? "Aún no registras gastos este mes" : monthExpenses.length === 1 ? "1 movimiento registrado" : `${monthExpenses.length} movimientos registrados`}</small></div>
          <img src="./wallet-hero.png" alt="Billetera morada con monedas" />
          <button className="primary add-button" onClick={() => { setEditing(null); setShowAdd(true); }}><b>＋</b> Agregar gasto</button>
        </section>

        <div className="dashboard-grid">
          <div>
            <section id="categorias"><div className="section-title"><h2>Categorías</h2><span>Este mes</span></div><div className="category-grid">{categoryTotals.slice(0, 4).map(c => <button key={c.name} className={`category-card ${c.color} ${filter === c.name ? "selected" : ""}`} onClick={() => setFilter(filter === c.name ? "Todas" : c.name)}><span className="category-name">{c.name}</span><span className="category-emoji">{c.emoji}</span><strong>{money(c.total)}</strong><i>→</i></button>)}</div></section>
            <section className="movements" id="movimientos"><div className="section-title"><h2>Últimos movimientos</h2><button onClick={() => setFilter("Todas")}>{filter === "Todas" ? "Este mes" : `Filtro: ${filter} ×`}</button></div>{visible.length ? <div className="movement-list">{visible.slice(0, 8).map(e => { const cat = categories.find(c => c.name === e.category)!; return <button className="movement-row" key={e.id} onClick={() => { setEditing(e); setShowAdd(true); }}><span className={`merchant-icon ${cat.color}`}>{cat.emoji}</span><span className="merchant"><b>{e.merchant}</b><small>{e.category} · {e.payment}</small></span><time>{new Date(`${e.date}T12:00:00`).toLocaleDateString("es-CL", { day: "numeric", month: "short" })}</time><strong>-{money(e.amount)}</strong></button>})}</div> : <div className="empty"><span>🧾</span><h3>Tu lista está esperando</h3><p>Registra tu primera compra para comenzar a entender tus gastos.</p><button onClick={() => setShowAdd(true)}>Agregar primer gasto</button></div>}</section>
          </div>
          <aside className="right-column">
            <section className="tip-card"><div><b>¡Buen dato!</b><p>{total === 0 ? "Anotar incluso los gastos pequeños te dará una imagen mucho más real de tu mes." : progress > 80 ? "Ya usaste gran parte de tu presupuesto. Revisa tus categorías antes de la próxima compra." : "Vas bien. Registrar cada compra hace que tus decisiones sean más conscientes."}</p></div><img src="./coin-mascot.png" alt="Moneda sonriente dando un consejo" /></section>
            <section className="budget-card"><div className="budget-head"><span>👛</span><div><p>Presupuesto mensual</p><strong>{money(vault.budget)}</strong></div></div><div className="progress"><i style={{ width: `${progress}%` }} /></div><strong className="percentage">{progress}%</strong><p>del presupuesto utilizado</p><small>{money(Math.max(0, vault.budget - total))} disponible</small></section>
            <section className="insight-card"><span>✨</span><div><b>Tu categoría principal</b><p>{categoryTotals[0].total ? `${categoryTotals[0].name} concentra ${Math.round(categoryTotals[0].total / total * 100)}% de tus gastos.` : "Aparecerá cuando registres tus compras."}</p></div></section>
          </aside>
        </div>

        <section className="analytics" id="analisis">
          <div className="analytics-heading"><div><span>UNA MIRADA TRANQUILA</span><h2>Así se mueve tu plata</h2><p>Sin juicios ni planillas eternas. Solo información que te ayuda.</p></div><div className="analytics-sticker">📊<i>¡Vas entendiendo!</i></div></div>
          <div className="analytics-grid">
            <article className="chart-card trend-card"><div className="chart-title"><div><span>Últimos 6 meses</span><h3>Tu evolución mensual</h3></div><b>{money(total)}<small>este mes</small></b></div><div className="bar-chart" aria-label="Gastos de los últimos seis meses">{monthSeries.map(m => <div className="bar-column" key={m.key} style={{ "--height": `${Math.max(m.value ? 12 : 3, m.value / maxMonth * 100)}%` } as CSSProperties}><span className="bar-value">{m.value ? money(m.value) : ""}</span><div className={`bar ${m.key === currentMonth ? "current" : ""}`} /><small>{m.label}</small></div>)}</div><p className="chart-foot">{monthSeries.filter(m => m.value).length < 2 ? "Cuando completes otro mes, aquí verás si tus gastos subieron o bajaron." : total <= monthSeries[4].value ? "Este mes vienes gastando menos que el anterior. Buen dato para mantener a la vista." : "Este mes va por encima del anterior. Puedes revisar las categorías sin apuro."}</p></article>
            <article className="chart-card donut-card"><div className="chart-title"><div><span>Este mes</span><h3>¿Dónde se fue?</h3></div><span className="tiny-sticker">🧩</span></div><div className="donut-wrap"><div className="donut" style={{ background: donut }}><div><strong>{activeCategories.length}</strong><small>categorías</small></div></div><div className="donut-legend">{activeCategories.length ? activeCategories.slice(0, 4).map(c => <div key={c.name}><i style={{ background: c.hex }} /><span>{c.emoji} {c.name}</span><b>{Math.round(c.total / total * 100)}%</b></div>) : <div className="soft-empty"><span>🌱</span><p>Tu gráfico crecerá con cada gasto que registres.</p></div>}</div></div></article>
            <article className="chart-card rhythm-card"><div className="chart-title"><div><span>Ritmo del mes</span><h3>Tu pulso cotidiano</h3></div><span className="tiny-sticker">🌤️</span></div><div className="rhythm-stats"><div><span>Promedio por día con gastos</span><strong>{money(averagePerSpendDay)}</strong></div><div><span>Días sin movimientos</span><strong>{Math.max(0, elapsedDays - spendDays)} <small>de {elapsedDays}</small></strong></div></div><div className="day-dots" aria-label={`${spendDays} días con movimientos de ${elapsedDays} transcurridos`}>{Array.from({ length: Math.min(elapsedDays, daysInMonth) }, (_, i) => <i key={i} className={monthExpenses.some(e => Number(e.date.slice(8, 10)) === i + 1) ? "spent" : "calm"} title={`Día ${i + 1}`} />)}</div><p className="chart-foot">{spendDays === 0 ? "Tu mes todavía está en blanco. No necesitas llenarlo de una vez." : spendDays < elapsedDays / 2 ? "Tienes varios días tranquilos. Eso también cuenta como una buena señal." : "Estás registrando con constancia; con eso ya estás construyendo claridad."}</p></article>
          </div>
        </section>
      </section>

      <button className="mobile-add" onClick={() => { setEditing(null); setShowAdd(true); }}>＋ Agregar gasto</button>

      {showAdd && <div className="modal-backdrop" onMouseDown={e => e.target === e.currentTarget && setShowAdd(false)}><section className="modal" role="dialog" aria-modal="true" aria-labelledby="expense-title"><div className="modal-head"><div><p>{editing ? "CORREGIR REGISTRO" : "NUEVO MOVIMIENTO"}</p><h2 id="expense-title">{editing ? "Editar gasto" : "¿En qué gastaste?"}</h2></div><button aria-label="Cerrar" onClick={() => { setShowAdd(false); setEditing(null); }}>×</button></div><form onSubmit={saveExpense}><label className="amount-label">Monto<input name="amount" type="number" inputMode="numeric" min="1" required autoFocus defaultValue={editing?.amount} placeholder="$ 0" /></label><fieldset><legend>Categoría</legend><div className="category-picker">{categories.map((c, i) => <label key={c.name}><input type="radio" name="category" value={c.name} defaultChecked={editing ? editing.category === c.name : i === 0} /><span>{c.emoji}<small>{c.name}</small></span></label>)}</div></fieldset><div className="form-grid"><label>Comercio o descripción<input name="merchant" required defaultValue={editing?.merchant} placeholder="Ej: supermercado" /></label><label>Medio de pago<select name="payment" defaultValue={editing?.payment ?? "Débito"}><option>Débito</option><option>Crédito</option><option>Efectivo</option><option>Transferencia</option></select></label><label>Fecha<input name="date" type="date" required defaultValue={editing?.date ?? today()} /></label><label>Nota opcional<input name="note" defaultValue={editing?.note} placeholder="Algo que recordar…" /></label></div><div className="modal-actions">{editing && <button className="danger" type="button" onClick={() => removeExpense(editing.id)}>Eliminar</button>}<button className="primary" type="submit">{editing ? "Guardar cambios" : "Guardar gasto"} <span>✓</span></button></div></form></section></div>}

      {showSettings && <div className="modal-backdrop" onMouseDown={e => e.target === e.currentTarget && setShowSettings(false)}><section className="modal settings-modal"><div className="modal-head"><div><p>TU ESPACIO</p><h2>Configuración</h2></div><button onClick={() => setShowSettings(false)}>×</button></div><label>Tu nombre<input value={vault.name} onChange={e => setVault({ ...vault, name: e.target.value })} /></label><label>Presupuesto mensual<input type="number" inputMode="numeric" value={vault.budget} onChange={e => setVault({ ...vault, budget: Number(e.target.value) })} /></label><div className="settings-actions"><button onClick={exportBackup}>↓ Descargar respaldo cifrado</button><label className="import-button">↑ Restaurar respaldo<input type="file" accept="application/json" onChange={async e => { const file = e.target.files?.[0]; if (!file) return; try { setVault(await decryptVault(await file.text(), pin)); notify("Respaldo restaurado ✓"); } catch { notify("El respaldo no corresponde a esta clave"); } }} /></label></div><div className="security-box"><span>🛡️</span><p><b>Privacidad local</b><br/>Tus movimientos no se publican ni se envían a GitHub. El respaldo también queda cifrado con tu clave.</p></div></section></div>}
      {toast && <div className="toast" role="status">{toast}</div>}
    </main>
  );
}
