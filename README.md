{
  "name": "mon-budget",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "lucide-react": "^0.383.0",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "recharts": "^2.12.7"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.3.1",
    "autoprefixer": "^10.4.19",
    "postcss": "^8.4.39",
    "tailwindcss": "^3.4.4",
    "vite": "^5.3.1"
  }
}
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
});
<!doctype html>
<html lang="fr">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, viewport-fit=cover" />
    <meta name="theme-color" content="#C1604A" />
    <meta name="apple-mobile-web-app-capable" content="yes" />
    <meta name="apple-mobile-web-app-status-bar-style" content="default" />
    <meta name="apple-mobile-web-app-title" content="Mon Budget" />
    <title>Mon Budget</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
export default {
  content: ["./index.html", "./src/**/*.{js,jsx}"],
  theme: { extend: {} },
  plugins: [],
};
export default {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
};
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App.jsx";
import "./index.css";

ReactDOM.createRoot(document.getElementById("root")).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
@tailwind base;
@tailwind components;
@tailwind utilities;

@import url('https://fonts.googleapis.com/css2?family=Fraunces:opsz,wght@9..144,500;9..144,600;9..144,700&family=Inter:wght@400;500;600&family=JetBrains+Mono:wght@400;500&display=swap');

html, body, #root { height: 100%; }
body { font-family: 'Inter', sans-serif; margin: 0; }
.font-display { font-family: 'Fraunces', serif; }
input[type=number]::-webkit-inner-spin-button,
input[type=number]::-webkit-outer-spin-button { -webkit-appearance: none; margin: 0; }
input[type=number] { -moz-appearance: textfield; }
import React, { useState, useEffect, useRef, useCallback } from "react";
import {
  BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer, Cell,
} from "recharts";
import {
  Plus, Trash2, Check, X, Plane, PiggyBank, Wallet, Home, Calendar, RotateCcw, Loader2,
} from "lucide-react";

/* ---------------------------------------------------------------------- */
/*  TOKENS                                                                 */
/* ---------------------------------------------------------------------- */
const T = {
  paper: "#FBF8F2",
  paperDeep: "#F3EEE2",
  ink: "#26241F",
  inkSoft: "#6B6557",
  line: "#E3DBC8",
  terracotta: "#C1604A",
  terracottaSoft: "#EFD9CF",
  sage: "#74875F",
  sageSoft: "#DEE6D2",
  dusty: "#5C7C92",
  dustySoft: "#D9E3E8",
  mustard: "#C9932F",
  mustardSoft: "#F1E1BC",
  plum: "#7C5B73",
  plumSoft: "#E7DAE3",
  danger: "#B23A33",
};

const GOAL_COLORS = [T.terracotta, T.dusty, T.plum, T.sage, T.mustard];

const STORAGE_KEY = "budget-suivi:data";

const MOIS_FR = [
  "JANVIER","FÉVRIER","MARS","AVRIL","MAI","JUIN",
  "JUILLET","AOÛT","SEPTEMBRE","OCTOBRE","NOVEMBRE","DÉCEMBRE",
];

/* ---------------------------------------------------------------------- */
/*  PERSISTENCE (localStorage — fully offline, data stays on this device)  */
/* ---------------------------------------------------------------------- */
function loadFromStorage() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    return raw ? JSON.parse(raw) : null;
  } catch (e) {
    return null;
  }
}

function saveToStorage(data) {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(data));
    return true;
  } catch (e) {
    return false;
  }
}

/* ---------------------------------------------------------------------- */
/*  HELPERS                                                                */
/* ---------------------------------------------------------------------- */
let _id = 0;
function uid(prefix = "id") {
  _id += 1;
  return `${prefix}-${Date.now().toString(36)}-${_id}`;
}

function euro(n) {
  const v = Math.round((Number(n) || 0) * 100) / 100;
  const isInt = Number.isInteger(v);
  return (
    v.toLocaleString("fr-FR", {
      minimumFractionDigits: isInt ? 0 : 2,
      maximumFractionDigits: 2,
    }) + " €"
  );
}

function sum(arr, key) {
  return arr.reduce((acc, r) => acc + (Number(r[key]) || 0), 0);
}

function defaultMonth(label, dateDebut, dateFin, restePrecedent = 0) {
  return {
    id: uid("mois"),
    label,
    dateDebut,
    dateFin,
    salaire: 0,
    primeActivite: 0,
    restePrecedent,
    credit: { prevu: 0, reel: 0 },
    depensesFixes: [
      { id: uid(), name: "Loyer", prevu: 575, reel: 0 },
      { id: uid(), name: "On Air", prevu: 32, reel: 0 },
      { id: uid(), name: "Forfait mobile", prevu: 45, reel: 0 },
      { id: uid(), name: "Spotify", prevu: 9, reel: 0 },
      { id: uid(), name: "Assurance Voiture", prevu: 50, reel: 0 },
      { id: uid(), name: "Coaching", prevu: 60, reel: 0 },
      { id: uid(), name: "Apple iCloud", prevu: 3, reel: 0 },
      { id: uid(), name: "Banque Coti Jeune", prevu: 5, reel: 0 },
      { id: uid(), name: "Disney+", prevu: 4, reel: 0 },
    ],
    depensesVariables: [
      { id: uid(), name: "Courses", budget: 20, reel: 0 },
      { id: uid(), name: "Essence", budget: 130, reel: 0 },
      { id: uid(), name: "Animaux", budget: 40, reel: 0 },
      { id: uid(), name: "Restaurant", budget: 20, reel: 0 },
      { id: uid(), name: "Cinéma", budget: 8, reel: 0 },
      { id: uid(), name: "Médecin", budget: 0, reel: 0 },
      { id: uid(), name: "Assurance Habitation", budget: 10, reel: 0 },
      { id: uid(), name: "Électricité", budget: 30, reel: 0 },
    ],
    appartement: [
      { id: uid(), name: "Frigo", mehdi: 275, djamila: 275, prix: 550 },
      { id: uid(), name: "Plaque induction", mehdi: 70, djamila: 70, prix: 140 },
    ],
    epargneComptes: [
      { id: uid(), name: "Livret A / LEP", prevu: 0, reel: 0 },
      { id: uid(), name: "Compte Titre", prevu: 0, reel: 0 },
      { id: uid(), name: "Crypto", prevu: 0, reel: 0 },
    ],
    goals: [
      {
        id: uid("goal"),
        name: "Épargne Japon",
        dureeLabel: "1 mois",
        items: [
          { id: uid(), name: "Billets d'avion", epargne: 0, aEpargner: 1500 },
          { id: uid(), name: "Budget sur place", epargne: 0, aEpargner: 7000 },
        ],
      },
    ],
  };
}

function computeMonth(m) {
  const revenu = Number(m.salaire) || 0;
  const prime = Number(m.primeActivite) || 0;
  const creditP = Number(m.credit.prevu) || 0;
  const creditR = Number(m.credit.reel) || 0;
  const dfP = sum(m.depensesFixes, "prevu");
  const dfR = sum(m.depensesFixes, "reel");
  const dvP = sum(m.depensesVariables, "budget");
  const dvR = sum(m.depensesVariables, "reel");
  const epP = sum(m.epargneComptes, "prevu");
  const epR = sum(m.epargneComptes, "reel");
  const apPrix = sum(m.appartement, "prix");
  const apReel = m.appartement.reduce(
    (a, r) => a + (Number(r.mehdi) || 0) + (Number(r.djamila) || 0),
    0
  );
  const restePrec = Number(m.restePrecedent) || 0;
  const revenuTotal = revenu + prime + restePrec;

  const restePrevu =
    revenu + prime + restePrec - creditP - dfP - dvP - epP - apPrix;
  const resteReel =
    revenu + prime + restePrec - creditR - dfR - dvR - epR - apReel;

  const goalsComputed = m.goals.map((g) => {
    const epargne = sum(g.items, "epargne");
    const aEpargner = sum(g.items, "aEpargner");
    const objectif = epargne + aEpargner;
    const pct = objectif > 0 ? Math.min(100, (epargne / objectif) * 100) : 0;
    return { ...g, epargneTotal: epargne, aEpargnerTotal: aEpargner, objectif, pct };
  });

  return {
    revenu, prime, restePrec, revenuTotal,
    creditP, creditR, dfP, dfR, dvP, dvR, epP, epR,
    apPrix, apReel, restePrevu, resteReel, goalsComputed,
  };
}

function nextLabel(prevLabel) {
  const idx = MOIS_FR.indexOf((prevLabel || "").toUpperCase());
  if (idx === -1) return "NOUVEAU MOIS";
  return MOIS_FR[(idx + 1) % 12];
}

/* ---------------------------------------------------------------------- */
/*  SMALL UI PRIMITIVES                                                    */
/* ---------------------------------------------------------------------- */
function NumberField({ value, onChange, align = "right", placeholder = "0" }) {
  return (
    <input
      type="number"
      step="0.01"
      value={value === 0 ? "" : value}
      placeholder={placeholder}
      onChange={(e) => onChange(e.target.value === "" ? 0 : parseFloat(e.target.value))}
      className={`w-full bg-transparent outline-none text-${align} px-1 py-1 text-sm`}
      style={{ fontFamily: "'JetBrains Mono', monospace", color: T.ink }}
    />
  );
}

function TextField({ value, onChange, placeholder = "" }) {
  return (
    <input
      type="text"
      value={value}
      placeholder={placeholder}
      onChange={(e) => onChange(e.target.value)}
      className="w-full bg-transparent outline-none px-1 py-1 text-sm"
      style={{ color: T.ink }}
    />
  );
}

function RowDeleteButton({ onConfirm }) {
  const [confirming, setConfirming] = useState(false);
  if (confirming) {
    return (
      <div className="flex items-center gap-1 justify-center">
        <button onClick={() => { onConfirm(); setConfirming(false); }} className="p-1 rounded" style={{ color: T.sage }} aria-label="Confirmer la suppression">
          <Check size={14} />
        </button>
        <button onClick={() => setConfirming(false)} className="p-1 rounded" style={{ color: T.inkSoft }} aria-label="Annuler">
          <X size={14} />
        </button>
      </div>
    );
  }
  return (
    <button onClick={() => setConfirming(true)} className="p-1 rounded opacity-50 hover:opacity-100 transition-opacity" style={{ color: T.danger }} aria-label="Supprimer la ligne">
      <Trash2 size={14} />
    </button>
  );
}

function SectionCard({ title, icon, accentSoft, children, footer }) {
  return (
    <div className="rounded-xl overflow-hidden border" style={{ borderColor: T.line, background: T.paper }}>
      <div className="px-4 py-3 flex items-center gap-2" style={{ background: accentSoft, color: T.ink }}>
        {icon}
        <h3 className="font-display text-sm tracking-wide uppercase font-semibold" style={{ letterSpacing: "0.04em" }}>
          {title}
        </h3>
      </div>
      <div className="p-3">{children}</div>
      {footer && (
        <div className="px-4 py-2 flex items-center justify-between text-sm font-semibold" style={{ borderTop: `1px solid ${T.line}`, color: T.ink }}>
          {footer}
        </div>
      )}
    </div>
  );
}

/* ---------------------------------------------------------------------- */
/*  TABLE: Dépenses fixes / Dépenses variables (prevu|budget + reel)       */
/* ---------------------------------------------------------------------- */
function TwoColTable({ rows, leftLabel, onUpdate, onAdd, onRemove, leftKey }) {
  return (
    <div>
      <div className="grid grid-cols-12 gap-1 text-xs font-semibold pb-1" style={{ color: T.inkSoft }}>
        <div className="col-span-5">Poste</div>
        <div className="col-span-3 text-right">{leftLabel}</div>
        <div className="col-span-3 text-right">Réel</div>
        <div className="col-span-1" />
      </div>
      <div className="divide-y" style={{ borderColor: T.line }}>
        {rows.map((r) => (
          <div key={r.id} className="grid grid-cols-12 gap-1 items-center" style={{ borderColor: T.line }}>
            <div className="col-span-5">
              <TextField value={r.name} onChange={(v) => onUpdate(r.id, "name", v)} placeholder="Nom du poste" />
            </div>
            <div className="col-span-3">
              <NumberField value={r[leftKey]} onChange={(v) => onUpdate(r.id, leftKey, v)} />
            </div>
            <div className="col-span-3">
              <NumberField value={r.reel} onChange={(v) => onUpdate(r.id, "reel", v)} />
            </div>
            <div className="col-span-1">
              <RowDeleteButton onConfirm={() => onRemove(r.id)} />
            </div>
          </div>
        ))}
      </div>
      <button onClick={onAdd} className="mt-2 flex items-center gap-1 text-xs font-medium px-2 py-1 rounded-md hover:opacity-80" style={{ color: T.ink, background: T.paperDeep }}>
        <Plus size={12} /> Ajouter une ligne
      </button>
    </div>
  );
}

/* ---------------------------------------------------------------------- */
/*  APPARTEMENT TABLE                                                      */
/* ---------------------------------------------------------------------- */
function AppartementTable({ rows, onUpdate, onAdd, onRemove }) {
  return (
    <div>
      <div className="grid grid-cols-12 gap-1 text-xs font-semibold pb-1" style={{ color: T.inkSoft }}>
        <div className="col-span-4">Article</div>
        <div className="col-span-2 text-right">Mehdi</div>
        <div className="col-span-2 text-right">Djamila</div>
        <div className="col-span-3 text-right">Prix total</div>
        <div className="col-span-1" />
      </div>
      <div className="divide-y" style={{ borderColor: T.line }}>
        {rows.map((r) => (
          <div key={r.id} className="grid grid-cols-12 gap-1 items-center">
            <div className="col-span-4">
              <TextField value={r.name} onChange={(v) => onUpdate(r.id, "name", v)} placeholder="Article" />
            </div>
            <div className="col-span-2">
              <NumberField value={r.mehdi} onChange={(v) => onUpdate(r.id, "mehdi", v)} />
            </div>
            <div className="col-span-2">
              <NumberField value={r.djamila} onChange={(v) => onUpdate(r.id, "djamila", v)} />
            </div>
            <div className="col-span-3">
              <NumberField value={r.prix} onChange={(v) => onUpdate(r.id, "prix", v)} />
            </div>
            <div className="col-span-1">
              <RowDeleteButton onConfirm={() => onRemove(r.id)} />
            </div>
          </div>
        ))}
      </div>
      <button onClick={onAdd} className="mt-2 flex items-center gap-1 text-xs font-medium px-2 py-1 rounded-md hover:opacity-80" style={{ color: T.ink, background: T.paperDeep }}>
        <Plus size={12} /> Ajouter un article
      </button>
    </div>
  );
}

/* ---------------------------------------------------------------------- */
/*  GOAL CARD (Épargne Japon, Budget Vacances, ...)                        */
/* ---------------------------------------------------------------------- */
function GoalCard({ goal, color, onUpdateMeta, onUpdateItem, onAddItem, onRemoveItem, onRemoveGoal }) {
  const epargneTotal = sum(goal.items, "epargne");
  const aEpargnerTotal = sum(goal.items, "aEpargner");
  const objectif = epargneTotal + aEpargnerTotal;
  const pct = objectif > 0 ? Math.min(100, (epargneTotal / objectif) * 100) : 0;

  return (
    <div className="rounded-xl overflow-hidden border" style={{ borderColor: T.line, background: T.paper }}>
      <div className="px-4 py-3 flex items-center justify-between" style={{ background: color, color: "#fff" }}>
        <div className="flex items-center gap-2 flex-1">
          <Plane size={16} />
          <input
            value={goal.name}
            onChange={(e) => onUpdateMeta("name", e.target.value)}
            className="bg-transparent outline-none font-display font-semibold text-sm uppercase tracking-wide w-full"
            style={{ letterSpacing: "0.04em" }}
          />
        </div>
        <button onClick={onRemoveGoal} className="opacity-70 hover:opacity-100 ml-2" aria-label="Supprimer cet objectif">
          <Trash2 size={14} />
        </button>
      </div>
      <div className="p-3">
        <div className="flex items-center gap-2 mb-2">
          <Calendar size={12} style={{ color: T.inkSoft }} />
          <input
            value={goal.dureeLabel}
            onChange={(e) => onUpdateMeta("dureeLabel", e.target.value)}
            placeholder="Durée / échéance"
            className="text-xs bg-transparent outline-none flex-1"
            style={{ color: T.inkSoft }}
          />
        </div>

        <div className="mb-3">
          <div className="h-2.5 rounded-full overflow-hidden" style={{ background: T.paperDeep }}>
            <div className="h-full rounded-full transition-all" style={{ width: `${pct}%`, background: color }} />
          </div>
          <div className="flex justify-between text-xs mt-1" style={{ color: T.inkSoft }}>
            <span>{euro(epargneTotal)} épargné</span>
            <span>{pct.toFixed(0)}% • objectif {euro(objectif)}</span>
          </div>
        </div>

        <div className="grid grid-cols-12 gap-1 text-xs font-semibold pb-1" style={{ color: T.inkSoft }}>
          <div className="col-span-5">Poste</div>
          <div className="col-span-3 text-right">Épargné</div>
          <div className="col-span-3 text-right">À épargner</div>
          <div className="col-span-1" />
        </div>
        <div className="divide-y" style={{ borderColor: T.line }}>
          {goal.items.map((it) => (
            <div key={it.id} className="grid grid-cols-12 gap-1 items-center">
              <div className="col-span-5">
                <TextField value={it.name} onChange={(v) => onUpdateItem(it.id, "name", v)} />
              </div>
              <div className="col-span-3">
                <NumberField value={it.epargne} onChange={(v) => onUpdateItem(it.id, "epargne", v)} />
              </div>
              <div className="col-span-3">
                <NumberField value={it.aEpargner} onChange={(v) => onUpdateItem(it.id, "aEpargner", v)} />
              </div>
              <div className="col-span-1">
                <RowDeleteButton onConfirm={() => onRemoveItem(it.id)} />
              </div>
            </div>
          ))}
        </div>
        <button onClick={onAddItem} className="mt-2 flex items-center gap-1 text-xs font-medium px-2 py-1 rounded-md hover:opacity-80" style={{ color: T.ink, background: T.paperDeep }}>
          <Plus size={12} /> Ajouter un poste
        </button>
      </div>
    </div>
  );
}

/* ---------------------------------------------------------------------- */
/*  STAT CHIP                                                              */
/* ---------------------------------------------------------------------- */
function StatChip({ label, value, color }) {
  return (
    <div className="rounded-lg px-3 py-2 flex-1 min-w-[120px]" style={{ background: color }}>
      <div className="text-[10px] uppercase tracking-wide font-semibold opacity-80" style={{ color: "#fff" }}>{label}</div>
      <div className="font-display text-lg font-bold" style={{ color: "#fff" }}>{euro(value)}</div>
    </div>
  );
}

/* ---------------------------------------------------------------------- */
/*  MAIN APP                                                               */
/* ---------------------------------------------------------------------- */
export default function App() {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [saveState, setSaveState] = useState("idle");
  const [showNewMonth, setShowNewMonth] = useState(false);
  const [newLabel, setNewLabel] = useState("");
  const [newDebut, setNewDebut] = useState("");
  const [newFin, setNewFin] = useState("");
  const [confirmReset, setConfirmReset] = useState(false);
  const saveTimer = useRef(null);

  useEffect(() => {
    const stored = loadFromStorage();
    if (stored) {
      setData(stored);
    } else {
      const m = defaultMonth("JUILLET", "2026-06-01", "2026-06-30");
      setData({ months: { [m.id]: m }, order: [m.id], currentId: m.id });
    }
    setLoading(false);
  }, []);

  useEffect(() => {
    if (loading || !data) return;
    setSaveState("saving");
    if (saveTimer.current) clearTimeout(saveTimer.current);
    saveTimer.current = setTimeout(() => {
      const ok = saveToStorage(data);
      setSaveState(ok ? "saved" : "idle");
    }, 400);
    return () => clearTimeout(saveTimer.current);
  }, [data, loading]);

  const updateMonth = useCallback((monthId, fn) => {
    setData((prev) => {
      const month = prev.months[monthId];
      const updated = fn(JSON.parse(JSON.stringify(month)));
      return { ...prev, months: { ...prev.months, [monthId]: updated } };
    });
  }, []);

  if (loading || !data) {
    return (
      <div className="min-h-screen flex items-center justify-center" style={{ background: T.paper }}>
        <div className="flex items-center gap-2" style={{ color: T.inkSoft }}>
          <Loader2 className="animate-spin" size={18} />
          <span>Chargement de votre budget…</span>
        </div>
      </div>
    );
  }

  const month = data.months[data.currentId];
  const comp = computeMonth(month);

  const chartData = month.depensesFixes
    .map((r) => ({ name: r.name, value: Number(r.reel) > 0 ? Number(r.reel) : Number(r.prevu) || 0 }))
    .filter((d) => d.value > 0)
    .sort((a, b) => b.value - a.value);

  function patchMonth(fn) {
    updateMonth(month.id, fn);
  }

  function handleCreateMonth() {
    const label = newLabel.trim() || nextLabel(month.label);
    const m = defaultMonth(label, newDebut || "", newFin || "", Math.round(comp.resteReel * 100) / 100);
    m.depensesFixes = month.depensesFixes.map((r) => ({ id: uid(), name: r.name, prevu: r.prevu, reel: 0 }));
    m.depensesVariables = month.depensesVariables.map((r) => ({ id: uid(), name: r.name, budget: r.budget, reel: 0 }));
    m.appartement = month.appartement.map((r) => ({ id: uid(), name: r.name, mehdi: 0, djamila: 0, prix: r.prix }));
    m.epargneComptes = month.epargneComptes.map((r) => ({ id: uid(), name: r.name, prevu: r.prevu, reel: 0 }));
    m.goals = month.goals.map((g) => ({
      id: uid("goal"),
      name: g.name,
      dureeLabel: g.dureeLabel,
      items: g.items.map((it) => ({ id: uid(), name: it.name, epargne: it.epargne, aEpargner: Math.max(0, it.aEpargner - it.epargne) })),
    }));
    setData((prev) => ({
      months: { ...prev.months, [m.id]: m },
      order: [...prev.order, m.id],
      currentId: m.id,
    }));
    setShowNewMonth(false);
    setNewLabel("");
    setNewDebut("");
    setNewFin("");
  }

  function handleDeleteMonth(id) {
    setData((prev) => {
      if (prev.order.length <= 1) return prev;
      const order = prev.order.filter((x) => x !== id);
      const months = { ...prev.months };
      delete months[id];
      const currentId = prev.currentId === id ? order[order.length - 1] : prev.currentId;
      return { months, order, currentId };
    });
  }

  function handleResetAll() {
    const m = defaultMonth("JUILLET", "2026-06-01", "2026-06-30");
    const fresh = { months: { [m.id]: m }, order: [m.id], currentId: m.id };
    setData(fresh);
    saveToStorage(fresh);
    setConfirmReset(false);
  }

  return (
    <div className="min-h-screen pb-16" style={{ background: T.paper, color: T.ink }}>
      <div className="sticky top-0 z-10 backdrop-blur border-b" style={{ background: "rgba(251,248,242,0.92)", borderColor: T.line }}>
        <div className="max-w-6xl mx-auto px-4 py-3">
          <div className="flex items-center justify-between gap-3 flex-wrap">
            <div className="flex items-center gap-2">
              <PiggyBank size={20} style={{ color: T.terracotta }} />
              <h1 className="font-display text-xl font-bold">Mon Budget</h1>
            </div>
            <div className="flex items-center gap-2 text-xs" style={{ color: T.inkSoft }}>
              {saveState === "saving" && <span className="flex items-center gap-1"><Loader2 className="animate-spin" size={12} /> Sauvegarde…</span>}
              {saveState === "saved" && <span className="flex items-center gap-1"><Check size={12} /> Sauvegardé</span>}
              <button onClick={() => setConfirmReset((c) => !c)} className="ml-2 flex items-center gap-1 px-2 py-1 rounded-md hover:opacity-80" style={{ background: T.paperDeep }}>
                <RotateCcw size={12} /> Réinitialiser
              </button>
            </div>
          </div>

          {confirmReset && (
            <div className="mt-2 p-2 rounded-md flex items-center justify-between text-xs" style={{ background: T.terracottaSoft }}>
              <span>Effacer toutes les données et repartir de zéro ?</span>
              <div className="flex gap-2">
                <button onClick={handleResetAll} className="px-2 py-1 rounded font-semibold" style={{ background: T.terracotta, color: "#fff" }}>Confirmer</button>
                <button onClick={() => setConfirmReset(false)} className="px-2 py-1 rounded" style={{ background: "#fff" }}>Annuler</button>
              </div>
            </div>
          )}

          <div className="flex items-center gap-2 mt-3 overflow-x-auto pb-1">
            {data.order.map((id) => {
              const m = data.months[id];
              const active = id === data.currentId;
              return (
                <button key={id} onClick={() => setData((p) => ({ ...p, currentId: id }))} className="px-3 py-1.5 rounded-full text-xs font-semibold whitespace-nowrap transition-colors" style={{ background: active ? T.ink : T.paperDeep, color: active ? "#fff" : T.inkSoft }}>
                  {m.label}
                </button>
              );
            })}
            <button onClick={() => { setNewLabel(nextLabel(month.label)); setShowNewMonth((s) => !s); }} className="px-3 py-1.5 rounded-full text-xs font-semibold flex items-center gap-1 whitespace-nowrap" style={{ background: T.terracottaSoft, color: T.terracotta }}>
              <Plus size={12} /> Nouveau mois
            </button>
          </div>

          {showNewMonth && (
            <div className="mt-2 p-3 rounded-lg grid grid-cols-1 sm:grid-cols-4 gap-2 items-end" style={{ background: T.paperDeep }}>
              <div>
                <label className="text-[10px] uppercase font-semibold" style={{ color: T.inkSoft }}>Nom du mois</label>
                <input value={newLabel} onChange={(e) => setNewLabel(e.target.value)} className="w-full rounded px-2 py-1 text-sm" style={{ border: `1px solid ${T.line}` }} />
              </div>
              <div>
                <label className="text-[10px] uppercase font-semibold" style={{ color: T.inkSoft }}>Date début</label>
                <input type="date" value={newDebut} onChange={(e) => setNewDebut(e.target.value)} className="w-full rounded px-2 py-1 text-sm" style={{ border: `1px solid ${T.line}` }} />
              </div>
              <div>
                <label className="text-[10px] uppercase font-semibold" style={{ color: T.inkSoft }}>Date fin</label>
                <input type="date" value={newFin} onChange={(e) => setNewFin(e.target.value)} className="w-full rounded px-2 py-1 text-sm" style={{ border: `1px solid ${T.line}` }} />
              </div>
              <button onClick={handleCreateMonth} className="rounded px-3 py-1.5 text-sm font-semibold" style={{ background: T.terracotta, color: "#fff" }}>Créer</button>
            </div>
          )}
        </div>
      </div>

      <div className="max-w-6xl mx-auto px-4 mt-4 space-y-4">
        <div className="grid grid-cols-1 lg:grid-cols-3 gap-4">
          <div className="rounded-xl border p-4 space-y-2" style={{ borderColor: T.line, background: T.paper }}>
            <input value={month.label} onChange={(e) => patchMonth((m) => { m.label = e.target.value.toUpperCase(); return m; })} className="font-display text-lg font-bold bg-transparent outline-none w-full" />
            <div className="flex items-center gap-2 text-sm">
              <Calendar size={14} style={{ color: T.inkSoft }} />
              <input type="date" value={month.dateDebut} onChange={(e) => patchMonth((m) => { m.dateDebut = e.target.value; return m; })} className="bg-transparent outline-none" />
              <span style={{ color: T.inkSoft }}>au</span>
              <input type="date" value={month.dateFin} onChange={(e) => patchMonth((m) => { m.dateFin = e.target.value; return m; })} className="bg-transparent outline-none" />
            </div>
            <div className="grid grid-cols-2 gap-2 pt-2 text-sm">
              <div>
                <label className="text-[10px] uppercase font-semibold block" style={{ color: T.inkSoft }}>Salaire</label>
                <NumberField value={month.salaire} onChange={(v) => patchMonth((m) => { m.salaire = v; return m; })} align="left" />
              </div>
              <div>
                <label className="text-[10px] uppercase font-semibold block" style={{ color: T.inkSoft }}>Prime d'activité</label>
                <NumberField value={month.primeActivite} onChange={(v) => patchMonth((m) => { m.primeActivite = v; return m; })} align="left" />
              </div>
              <div className="col-span-2">
                <label className="text-[10px] uppercase font-semibold block" style={{ color: T.inkSoft }}>Restes du mois précédent</label>
                <NumberField value={month.restePrecedent} onChange={(v) => patchMonth((m) => { m.restePrecedent = v; return m; })} align="left" />
              </div>
            </div>
          </div>

          <div className="lg:col-span-2 flex flex-wrap gap-2">
            <StatChip label="Revenu total" value={comp.revenuTotal} color={T.plum} />
            <StatChip label="Dépenses fixes" value={comp.dfR || comp.dfP} color={T.dusty} />
            <StatChip label="Dépenses" value={comp.dvR || comp.dvP} color={T.mustard} />
            {comp.goalsComputed.map((g, i) => (
              <StatChip key={g.id} label={g.name} value={g.epargneTotal} color={GOAL_COLORS[i % GOAL_COLORS.length]} />
            ))}
            <StatChip label="Épargne" value={comp.epR || comp.epP} color={T.sage} />
          </div>
        </div>

        <SectionCard title="Répartition des dépenses fixes" icon={<Home size={14} />} accentSoft={T.dustySoft}>
          {chartData.length === 0 ? (
            <div className="text-sm py-6 text-center" style={{ color: T.inkSoft }}>
              Renseignez le « Réel » de vos dépenses fixes pour voir apparaître le graphique.
            </div>
          ) : (
            <div style={{ width: "100%", height: Math.max(160, chartData.length * 32) }}>
              <ResponsiveContainer>
                <BarChart data={chartData} layout="vertical" margin={{ left: 10, right: 20 }}>
                  <CartesianGrid strokeDasharray="3 3" stroke={T.line} horizontal={false} />
                  <XAxis type="number" tickFormatter={(v) => `${v}€`} stroke={T.inkSoft} fontSize={11} />
                  <YAxis type="category" dataKey="name" width={110} stroke={T.inkSoft} fontSize={11} />
                  <Tooltip formatter={(v) => euro(v)} contentStyle={{ borderRadius: 8, border: `1px solid ${T.line}` }} />
                  <Bar dataKey="value" radius={[0, 4, 4, 0]}>
                    {chartData.map((_, i) => (<Cell key={i} fill={T.dusty} fillOpacity={1 - i * 0.06} />))}
                  </Bar>
                </BarChart>
              </ResponsiveContainer>
            </div>
          )}
        </SectionCard>

        <SectionCard
          title="Récapitulatif"
          icon={<Wallet size={14} />}
          accentSoft={T.terracottaSoft}
          footer={
            <>
              <span>Reste</span>
              <span className="flex gap-4">
                <span style={{ color: T.inkSoft, fontWeight: 500 }}>Prévu {euro(comp.restePrevu)}</span>
                <span style={{ color: comp.resteReel < 0 ? T.danger : T.sage }}>Réel {euro(comp.resteReel)}</span>
              </span>
            </>
          }
        >
          <div className="grid grid-cols-12 gap-1 text-xs font-semibold pb-1" style={{ color: T.inkSoft }}>
            <div className="col-span-6">Catégorie</div>
            <div className="col-span-3 text-right">Prévu</div>
            <div className="col-span-3 text-right">Réel</div>
          </div>
          <div className="divide-y text-sm" style={{ borderColor: T.line }}>
            <RecapRow label="Revenu" prevu={comp.revenu} reel={comp.revenu} />
            <RecapRow label="Prime d'activité" prevu={comp.prime} reel={comp.prime} />
            <RecapRow
              label="Crédit"
              editable
              prevu={month.credit.prevu}
              reel={month.credit.reel}
              onPrevu={(v) => patchMonth((m) => { m.credit.prevu = v; return m; })}
              onReel={(v) => patchMonth((m) => { m.credit.reel = v; return m; })}
            />
            <RecapRow label="Dépenses fixes" prevu={comp.dfP} reel={comp.dfR} />
            <RecapRow label="Dépenses variables" prevu={comp.dvP} reel={comp.dvR} />
            <RecapRow label="Épargne" prevu={comp.epP} reel={comp.epR} />
            <RecapRow label="Appartement" prevu={comp.apPrix} reel={comp.apReel} />
          </div>
        </SectionCard>

        <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
          {month.goals.map((g, i) => (
            <GoalCard
              key={g.id}
              goal={g}
              color={GOAL_COLORS[i % GOAL_COLORS.length]}
              onUpdateMeta={(field, value) => patchMonth((m) => { const goal = m.goals.find((x) => x.id === g.id); goal[field] = value; return m; })}
              onUpdateItem={(itemId, field, value) => patchMonth((m) => { const goal = m.goals.find((x) => x.id === g.id); const it = goal.items.find((x) => x.id === itemId); it[field] = value; return m; })}
              onAddItem={() => patchMonth((m) => { const goal = m.goals.find((x) => x.id === g.id); goal.items.push({ id: uid(), name: "Nouveau poste", epargne: 0, aEpargner: 0 }); return m; })}
              onRemoveItem={(itemId) => patchMonth((m) => { const goal = m.goals.find((x) => x.id === g.id); goal.items = goal.items.filter((x) => x.id !== itemId); return m; })}
              onRemoveGoal={() => patchMonth((m) => { m.goals = m.goals.filter((x) => x.id !== g.id); return m; })}
            />
          ))}
          <button
            onClick={() => patchMonth((m) => { m.goals.push({ id: uid("goal"), name: "Budget Vacances", dureeLabel: "", items: [{ id: uid(), name: "Poste 1", epargne: 0, aEpargner: 0 }] }); return m; })}
            className="rounded-xl border-2 border-dashed flex items-center justify-center gap-2 text-sm font-semibold py-6"
            style={{ borderColor: T.line, color: T.inkSoft }}
          >
            <Plus size={16} /> Ajouter un objectif d'épargne
          </button>
        </div>

        <div className="grid grid-cols-1 lg:grid-cols-3 gap-4">
          <SectionCard title="Dépenses fixes" icon={<Home size={14} />} accentSoft={T.dustySoft} footer={<><span>Reste</span><span>{euro(comp.dfP - comp.dfR)}</span></>}>
            <TwoColTable
              rows={month.depensesFixes}
              leftLabel="Prévu"
              leftKey="prevu"
              onUpdate={(id, field, v) => patchMonth((m) => { const r = m.depensesFixes.find((x) => x.id === id); r[field] = v; return m; })}
              onAdd={() => patchMonth((m) => { m.depensesFixes.push({ id: uid(), name: "Nouveau poste", prevu: 0, reel: 0 }); return m; })}
              onRemove={(id) => patchMonth((m) => { m.depensesFixes = m.depensesFixes.filter((x) => x.id !== id); return m; })}
            />
          </SectionCard>

          <SectionCard title="Dépenses" icon={<Wallet size={14} />} accentSoft={T.mustardSoft} footer={<><span>Reste</span><span>{euro(comp.dvP - comp.dvR)}</span></>}>
            <TwoColTable
              rows={month.depensesVariables}
              leftLabel="Budget"
              leftKey="budget"
              onUpdate={(id, field, v) => patchMonth((m) => { const r = m.depensesVariables.find((x) => x.id === id); r[field] = v; return m; })}
              onAdd={() => patchMonth((m) => { m.depensesVariables.push({ id: uid(), name: "Nouveau poste", budget: 0, reel: 0 }); return m; })}
              onRemove={(id) => patchMonth((m) => { m.depensesVariables = m.depensesVariables.filter((x) => x.id !== id); return m; })}
            />
          </SectionCard>

          <SectionCard title="Épargne" icon={<PiggyBank size={14} />} accentSoft={T.sageSoft} footer={<><span>Reste</span><span>{euro(comp.epR)}</span></>}>
            <TwoColTable
              rows={month.epargneComptes}
              leftLabel="Prévu"
              leftKey="prevu"
              onUpdate={(id, field, v) => patchMonth((m) => { const r = m.epargneComptes.find((x) => x.id === id); r[field] = v; return m; })}
              onAdd={() => patchMonth((m) => { m.epargneComptes.push({ id: uid(), name: "Nouveau compte", prevu: 0, reel: 0 }); return m; })}
              onRemove={(id) => patchMonth((m) => { m.epargneComptes = m.epargneComptes.filter((x) => x.id !== id); return m; })}
            />
          </SectionCard>
        </div>

        <SectionCard
          title="Appartement"
          icon={<Home size={14} />}
          accentSoft={T.dustySoft}
          footer={<><span>Reste</span><span className="flex gap-4"><span style={{ color: T.inkSoft, fontWeight: 500 }}>Prévu {euro(comp.apPrix)}</span><span>Réel {euro(comp.apReel)}</span></span></>}
        >
          <AppartementTable
            rows={month.appartement}
            onUpdate={(id, field, v) => patchMonth((m) => { const r = m.appartement.find((x) => x.id === id); r[field] = v; return m; })}
            onAdd={() => patchMonth((m) => { m.appartement.push({ id: uid(), name: "Nouvel article", mehdi: 0, djamila: 0, prix: 0 }); return m; })}
            onRemove={(id) => patchMonth((m) => { m.appartement = m.appartement.filter((x) => x.id !== id); return m; })}
          />
        </SectionCard>

        {data.order.length > 1 && (
          <div className="flex justify-end">
            <button onClick={() => handleDeleteMonth(month.id)} className="text-xs flex items-center gap-1 px-3 py-1.5 rounded-md" style={{ color: T.danger, background: T.terracottaSoft }}>
              <Trash2 size={12} /> Supprimer le mois « {month.label} »
            </button>
          </div>
        )}
      </div>
    </div>
  );
}

function RecapRow({ label, prevu, reel, editable, onPrevu, onReel }) {
  return (
    <div className="grid grid-cols-12 gap-1 items-center py-1">
      <div className="col-span-6">{label}</div>
      <div className="col-span-3 text-right">
        {editable ? <NumberField value={prevu} onChange={onPrevu} /> : <span style={{ fontFamily: "'JetBrains Mono', monospace" }}>{euro(prevu)}</span>}
      </div>
      <div className="col-span-3 text-right">
        {editable ? <NumberField value={reel} onChange={onReel} /> : <span style={{ fontFamily: "'JetBrains Mono', monospace" }}>{euro(reel)}</span>}
      </div>
    </div>
  );
}
