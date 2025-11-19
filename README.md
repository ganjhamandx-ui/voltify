// src/App.js
import React, { useEffect, useState, useRef } from "react";
import { motion, AnimatePresence } from "framer-motion";
import {
  LineChart,
  Line,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  ResponsiveContainer,
  PieChart,
  Pie,
  Cell,
  Legend,
} from "recharts";
import {
  CheckCircle,
  Info,
  XCircle,
  Download,
  Save,
  LogIn,
  Sun,
  Moon,
  Monitor,
} from "lucide-react";

/**
 * Voltify Pro – Panel didáctico para hogares chilenos
 * - Historial mensual en kWh con CSV
 * - Catálogo de artefactos + cálculo estimado
 * - Potencia contratada (por defecto 10 kW BT1)
 * - Advertencia cuando se supera la potencia
 * - Gauge semicircular de uso de potencia
 */

export default function VoltifyPro() {
  // =====================
  // Tema (system / light / dark)
  // =====================
  const [theme, setTheme] = useState("system");
  const prefersDark = usePrefersDarkMode();
  const isDark = theme === "dark" || (theme === "system" && prefersDark);

  const bgGrad = isDark
    ? "from-slate-900 via-slate-800 to-slate-900 text-slate-100"
    : "from-white via-slate-50 to-white text-slate-900";
  const card = isDark
    ? "bg-slate-800 border-slate-700"
    : "bg-white border-slate-200";
  const borderBase = isDark ? "border-slate-700" : "border-slate-200";
  const subtleText = isDark ? "text-slate-400" : "text-slate-500";

  // =====================
  // Datos de ejemplo (gráficos demo)
  // =====================
  const colors = ["#3b82f6", "#22c55e", "#f97316", "#ef4444", "#8b5cf6"];

  const demoData = [
    { name: "Enero", kWh: 180 },
    { name: "Febrero", kWh: 220 },
    { name: "Marzo", kWh: 150 },
    { name: "Abril", kWh: 300 },
    { name: "Mayo", kWh: 250 },
  ];

  const pieData = [
    { name: "Refrigeración", value: 35 },
    { name: "Iluminación", value: 25 },
    { name: "Cocina", value: 15 },
    { name: "Climatización", value: 10 },
    { name: "Otros", value: 15 },
  ];

  // =====================
  // Toasts apilados
  // =====================
  const [toasts, setToasts] = useState([]);
  const toastCounter = useRef(0);

  function showToast(message, type = "success", position = "bottom") {
    const id = ++toastCounter.current;
    setToasts((prev) => [...prev, { id, message, type, position }]);
    window.setTimeout(() => dismissToast(id), 3500);
  }

  function dismissToast(id) {
    setToasts((prev) => prev.filter((t) => t.id !== id));
  }

  const topToasts = toasts.filter((t) => t.position === "top");
  const bottomToasts = toasts.filter((t) => t.position === "bottom");

  // =====================
  // Historial real (localStorage) + acciones
  // =====================
  const LS_KEY = "voltify_hist_v1";
  const [hist, setHist] = useState([]);
  const [useHistInChart, setUseHistInChart] = useState(true);

  // =====================
  // Catálogo de artefactos (pre-cargados) + selección
  // =====================
  const CAT_DEFAULT = [
    {
      id: "tv",
      nombre: "Televisor LED 42 pulgadas",
      potenciaW: 80,
      categoria: "Entretenimiento",
    },
    {
      id: "refri",
      nombre: "Refrigerador inverter",
      potenciaW: 120,
      categoria: "Cocina",
    },
    {
      id: "micro",
      nombre: "Microondas",
      potenciaW: 1200,
      categoria: "Cocina",
    },
    {
      id: "lavadora",
      nombre: "Lavadora",
      potenciaW: 500,
      categoria: "Lavandería",
    },
    {
      id: "plancha",
      nombre: "Plancha",
      potenciaW: 1000,
      categoria: "Lavandería",
    },
    {
      id: "pc",
      nombre: "Computador de escritorio",
      potenciaW: 200,
      categoria: "Oficina",
    },
    {
      id: "notebook",
      nombre: "Notebook",
      potenciaW: 60,
      categoria: "Oficina",
    },
    {
      id: "foco",
      nombre: "Foco LED 9W",
      potenciaW: 9,
      categoria: "Iluminación",
    },
    {
      id: "estufa",
      nombre: "Calefactor eléctrico",
      potenciaW: 2000,
      categoria: "Climatización",
    },
  ];

  const CAT_KEY = "voltify_catalog_user_v1";
  const [cat, setCat] = useState(CAT_DEFAULT);

  useEffect(() => {
    try {
      const raw = localStorage.getItem(CAT_KEY);
      if (raw) {
        const extra = JSON.parse(raw);
        setCat([...CAT_DEFAULT, ...extra]);
      }
    } catch {
      // ignore
    }
  }, []);

  useEffect(() => {
    try {
      const extras = cat.filter(
        (c) => !CAT_DEFAULT.some((d) => d.id === c.id)
      );
      localStorage.setItem(CAT_KEY, JSON.stringify(extras));
    } catch {
      // ignore
    }
  }, [cat]);

  const ITEMS_KEY = "voltify_items_v1";
  const [items, setItems] = useState([]);
  const [filtro, setFiltro] = useState("");
  const [newNombre, setNewNombre] = useState("");
  const [newPotencia, setNewPotencia] = useState(100);
  const [newCategoria, setNewCategoria] = useState("Otros");

  useEffect(() => {
    try {
      const raw = localStorage.getItem(ITEMS_KEY);
      if (raw) setItems(JSON.parse(raw));
    } catch {
      // ignore
    }
  }, []);

  useEffect(() => {
    try {
      localStorage.setItem(ITEMS_KEY, JSON.stringify(items));
    } catch {
      // ignore
    }
  }, [items]);

  const catalogoFiltrado = cat.filter((c) =>
    (c.nombre + " " + c.categoria)
      .toLowerCase()
      .includes(filtro.toLowerCase())
  );

  const mapaCat = Object.fromEntries(cat.map((c) => [c.id, c]));

  function agregarItem(id) {
    setItems((prev) => {
      const ex = prev.find((p) => p.id === id);
      if (ex) {
        return prev.map((p) =>
          p.id === id ? { ...p, qty: p.qty + 1 } : p
        );
      }
      return [...prev, { id, qty: 1, horasDia: 2, diasMes: 30 }];
    });
    showToast("Artefacto agregado", "success", "top");
  }

  function slugId(base) {
    const root =
      "user-" +
      base
        .toLowerCase()
        .replace(/[^a-z0-9]+/g, "-")
        .replace(/(^-|-$)/g, "");
    let id = root;
    let i = 1;
    while (cat.some((c) => c.id === id)) {
      id = root + "-" + i++;
    }
    return id;
  }

  function addCustom(nombre, potenciaW, categoria) {
    const n = nombre.trim();
    const catg = categoria.trim() || "Otros";
    const p = Math.round(Math.max(1, potenciaW));
    if (!n || !isFinite(p) || p <= 0) {
      showToast("Completa nombre y potencia válida", "error", "top");
      return;
    }
    const id = slugId(n);
    setCat((prev) => [...prev, { id, nombre: n, potenciaW: p, categoria: catg }]);
    showToast("Artefacto personalizado agregado", "success", "top");
  }

  function removeFromCatalog(id) {
    if (!id.startsWith("user-")) {
      showToast("Sólo puedes borrar los personalizados", "error", "top");
      return;
    }
    setCat((prev) => prev.filter((c) => c.id !== id));
  }

  function quitarItem(id) {
    setItems((prev) => prev.filter((p) => p.id !== id));
  }

  function updItem(id, patch) {
    setItems((prev) =>
      prev.map((p) => (p.id === id ? { ...p, ...patch } : p))
    );
  }

  // kWh estimado desde artefactos
  const estimadoKWh = items.reduce((acc, it) => {
    const catEl = mapaCat[it.id];
    if (!catEl) return acc;
    return (
      acc +
      (catEl.potenciaW * it.horasDia * it.diasMes * it.qty) / 1000
    );
  }, 0);

  // Formulario Guardar Mes
  const now = new Date();
  const [anio, setAnio] = useState(now.getFullYear());
  const [mes, setMes] = useState(now.getMonth() + 1); // 1..12
  const [kwhMes, setKwhMes] = useState(0);
  const [nota, setNota] = useState("");

  // Comparación de meses
  const [cmpA, setCmpA] = useState("");
  const [cmpB, setCmpB] = useState("");

  // Parámetros eléctricos y ayudas
  const [potenciaContratada, setPotenciaContratada] = useState(10); // 10 kW típico BT1
  const PRICE_KEY = "voltify_price_v1";
  const [precioKWh, setPrecioKWh] = useState(120); // CLP por kWh

  function horasDelMes(y, m) {
    const dias = new Date(y, m, 0).getDate();
    return dias * 24;
  }

  function demandaMediaKW(kwh, y, m) {
    return kwh / Math.max(1, horasDelMes(y, m));
  }

  useEffect(() => {
    try {
      const raw = localStorage.getItem(PRICE_KEY);
      if (raw) setPrecioKWh(parseFloat(raw));
    } catch {
      // ignore
    }
  }, []);

  useEffect(() => {
    try {
      localStorage.setItem(PRICE_KEY, String(precioKWh));
    } catch {
      // ignore
    }
  }, [precioKWh]);

  // "Tiempo real" simulado (cálculo instantáneo)
  const sumaKWInstalados = items.reduce((acc, it) => {
    const c = mapaCat[it.id];
    return acc + (c ? (c.potenciaW * it.qty) / 1000 : 0);
  }, 0);

  const [factorUso, setFactorUso] = useState(50); // % de artefactos encendidos ahora
  const instantKW = sumaKWInstalados * (factorUso / 100);

  // KPI: mejora/empeora respecto a mes anterior usando estimación actual
  const sortedHist = [...hist].sort((a, b) =>
    a.mesISO.localeCompare(b.mesISO)
  );
  const ultimoMes =
    sortedHist.length > 0 ? sortedHist[sortedHist.length - 1] : null;

  let mejoraVsAnterior = null;
  if (ultimoMes) {
    const diff = Math.round(estimadoKWh) - ultimoMes.kWh;
    mejoraVsAnterior = { diff, mejor: diff < 0 };
  }

  function mesISO(y, m) {
    return `${y}-${String(m).padStart(2, "0")}`;
  }

  useEffect(() => {
    try {
      const raw = localStorage.getItem(LS_KEY);
      if (raw) setHist(JSON.parse(raw));
    } catch {
      // ignore
    }
  }, []);

  function persist(newHist) {
    setHist(newHist);
    try {
      localStorage.setItem(LS_KEY, JSON.stringify(newHist));
    } catch {
      // ignore
    }
  }

  // Helper CSV
  function buildCSV(rows) {
    return rows
      .map((r) =>
        r
          .map((c) => {
            const value = (c ?? "").toString();
            const escaped = value.replace(/"/g, '""');
            return `"${escaped}"`;
          })
          .join(",")
      )
      .join("\n");
  }

  // Datos para el gráfico de línea desde historial (si se activa)
  const dataFromHist =
    hist.length === 0
      ? demoData
      : hist.map((h) => ({ name: h.mesISO, kWh: h.kWh }));

  // =====================
  // Acciones
  // =====================
  function onGuardarMes() {
    const id = mesISO(anio, mes);
    if (!kwhMes || kwhMes <= 0) {
      showToast("Ingresa kWh válidos", "error", "top");
      return;
    }
    if (hist.some((h) => h.mesISO === id)) {
      showToast("Ese mes ya existe en el historial", "error", "top");
      return;
    }
    const nuevo = {
      mesISO: id,
      kWh: Math.round(kwhMes),
      nota: nota.trim() || undefined,
    };
    const ordenado = [...hist, nuevo].sort((a, b) =>
      a.mesISO.localeCompare(b.mesISO)
    );
    persist(ordenado);
    showToast("Mes guardado exitosamente", "success", "top");
    setNota("");
    setKwhMes(0);
    if (!cmpA) setCmpA(id);
    else if (!cmpB) setCmpB(id);
  }

  function onExportarCSV() {
    if (hist.length === 0) {
      showToast("No hay historial para exportar", "error", "bottom");
      return;
    }
    const rows = [
      ["mesISO", "kWh", "nota"],
      ...hist.map((h) => [h.mesISO, String(h.kWh), h.nota ?? ""]),
    ];
    const csv = buildCSV(rows);
    const blob = new Blob([csv], { type: "text/csv;charset=utf-8;" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = "voltify_historial.csv";
    a.click();
    URL.revokeObjectURL(url);
    showToast("CSV exportado", "info", "bottom");
  }

  function onLoginOK() {
    showToast("Inicio de sesión ok", "success", "top");
  }

  function onEliminarMes(id) {
    const nuevo = hist.filter((h) => h.mesISO !== id);
    persist(nuevo);
    showToast("Mes eliminado", "info", "bottom");
  }

  // =====================
  // Tests mínimos (no cambian la UI)
  // =====================
  useEffect(() => {
    console.assert(
      mesISO(2025, 1) === "2025-01",
      "mesISO(2025,1) debe ser 2025-01"
    );
    console.assert(
      mesISO(2025, 12) === "2025-12",
      "mesISO(2025,12) debe ser 2025-12"
    );

    const sample = [
      ["mesISO", "kWh", "nota"],
      ["2025-01", "100", 'a"b'],
      ["2025-02", "200", ""],
    ];
    const built = buildCSV(sample);
    console.assert(built.includes("\n"), "CSV debe contener \\n");
    console.assert(
      built.includes('"a""b"'),
      'CSV debe escapar comillas dobles como ""'
    );
  }, []);

  // =====================
  // Render
  // =====================
  return (
    <div className={`min-h-screen bg-gradient-to-br ${bgGrad} font-sans`}>
      {/* HEADER */}
      <header
        className={`sticky top-0 z-40 backdrop-blur border-b ${borderBase} ${
          isDark ? "bg-slate-900/80" : "bg-white/70"
        }`}
      >
        <div className="max-w-7xl mx-auto flex justify-between items-center px-6 py-4">
          <motion.h1
            className={`text-2xl font-bold tracking-tight ${
              isDark ? "text-sky-400" : "text-sky-600"
            }`}
          >
            ⚡ Voltify Pro
          </motion.h1>
          <div className="flex items-center gap-2 text-sm">
            <span className={subtleText}>Tema:</span>
            <ThemeButton
              active={theme === "system"}
              onClick={() => setTheme("system")}
              title="Sistema"
            >
              <Monitor className="w-4 h-4" />
            </ThemeButton>
            <ThemeButton
              active={theme === "light"}
              onClick={() => setTheme("light")}
              title="Claro"
            >
              <Sun className="w-4 h-4" />
            </ThemeButton>
            <ThemeButton
              active={theme === "dark"}
              onClick={() => setTheme("dark")}
              title="Oscuro"
            >
              <Moon className="w-4 h-4" />
            </ThemeButton>
          </div>
        </div>
      </header>

      {/* KPI & Tiempo real */}
      <section className="max-w-7xl mx-auto px-6 pt-6 grid md:grid-cols-5 gap-4">
        <div className={`rounded-2xl border ${borderBase} ${card} p-4`}>
          <div className="text-sm font-medium">Estimación del mes (kWh)</div>
          <div className="text-2xl font-bold mt-1">
            {Math.round(estimadoKWh)}
          </div>
          {ultimoMes && (
            <div
              className={`text-xs mt-1 ${
                mejoraVsAnterior && mejoraVsAnterior.mejor
                  ? "text-emerald-500"
                  : "text-red-500"
              }`}
            >
              {mejoraVsAnterior && mejoraVsAnterior.mejor
                ? "Mejor que"
                : "Peor que"}{" "}
              {ultimoMes.mesISO} por{" "}
              {mejoraVsAnterior
                ? Math.abs(mejoraVsAnterior.diff)
                : 0}{" "}
              kWh
            </div>
          )}
        </div>

        <div className={`rounded-2xl border ${borderBase} ${card} p-4`}>
          <div className="text-sm font-medium">Potencia contratada (kW)</div>
          <div className="text-2xl font-bold mt-1">
            {potenciaContratada.toFixed(2)}
          </div>
          <div className="text-xs mt-2 flex items-center gap-2">
            <span className={subtleText}>Editar (BT1 máx. 10 kW):</span>
            <input
              type="number"
              min={1}
              max={10}
              step={0.1}
              value={potenciaContratada}
              onChange={(e) =>
                setPotenciaContratada(parseFloat(e.target.value || "0"))
              }
              className={`w-24 px-2 py-1 rounded-md border ${borderBase} ${
                isDark ? "bg-slate-900" : "bg-white"
              }`}
            />
          </div>
        </div>

        <div className={`rounded-2xl border ${borderBase} ${card} p-4`}>
          <div className="text-sm font-medium">Consumo instantáneo (kW)</div>
          <div
            className={`text-2xl font-bold mt-1 ${
              instantKW > potenciaContratada ? "text-red-500" : ""
            }`}
          >
            {instantKW.toFixed(2)}
          </div>
          <div className="mt-2">
            <input
              type="range"
              min={0}
              max={100}
              value={factorUso}
              onChange={(e) => setFactorUso(parseInt(e.target.value))}
              className="w-full"
            />
            <div className="text-xs mt-1">
              Factor de uso en vivo: {factorUso}%
            </div>
          </div>
        </div>

        <div className={`rounded-2xl border ${borderBase} ${card} p-4`}>
          <div className="text-sm font-medium">Advertencia</div>
          {instantKW > potenciaContratada ? (
            <div className="mt-1 text-red-500 text-sm">
              ⚠️ Superas tu potencia contratada (BT1 ≈ 10 kW). Reduce uso
              simultáneo.
            </div>
          ) : (
            <div className={`mt-1 text-sm ${subtleText}`}>
              Dentro del límite de potencia contratada.
            </div>
          )}
          {ultimoMes && (
            <div className={`mt-2 text-sm ${subtleText}`}>
              Mes anterior: {ultimoMes.kWh} kWh
            </div>
          )}
        </div>

        {/* Gauge */}
        <div
          className={`rounded-2xl border ${borderBase} ${card} p-4 flex flex-col items-center justify-center`}
        >
          <div className="text-sm font-medium mb-1">Uso de potencia</div>
          <Gauge
            value={instantKW}
            max={Math.max(0.1, potenciaContratada)}
            dark={isDark}
          />
          <div className="text-xs mt-1">
            {(
              (instantKW / Math.max(0.1, potenciaContratada)) *
              100
            ).toFixed(0)}
            % del límite
          </div>
        </div>

        {/* Precio y boleta */}
        <div className={`rounded-2xl border ${borderBase} ${card} p-4`}>
          <div className="text-sm font-medium">Precio kWh y boleta estimada</div>
          <div className="text-xs mt-1 flex items-center gap-2">
            <span className={subtleText}>Precio kWh (CLP):</span>
            <input
              type="number"
              min={0}
              step={1}
              value={precioKWh}
              onChange={(e) => setPrecioKWh(parseFloat(e.target.value || "0"))}
              className={`w-28 px-2 py-1 rounded-md border ${borderBase} ${
                isDark ? "bg-slate-900" : "bg-white"
              }`}
            />
          </div>
          <div className="text-2xl font-bold mt-2">
            {(
              Math.round(estimadoKWh) * precioKWh
            ).toLocaleString("es-CL", {
              style: "currency",
              currency: "CLP",
            })}
          </div>
          <div className={`text-xs mt-1 ${subtleText}`}>
            Basado en la estimación actual de kWh.
          </div>
        </div>
      </section>

      {/* TOASTS */}
      <ToastStack
        toasts={topToasts}
        onDismiss={dismissToast}
        isDark={isDark}
        position="top"
      />
      <ToastStack
        toasts={bottomToasts}
        onDismiss={dismissToast}
        isDark={isDark}
        position="bottom"
      />

      {/* MAIN */}
      <main className="max-w-7xl mx-auto px-6 py-10 space-y-10">
        {/* Catálogo y selección de artefactos */}
        <section
          className={`rounded-2xl border ${borderBase} ${card} p-5 space-y-5`}
        >
          <div className="flex items-center gap-3">
            <h3
              className={`text-lg font-semibold ${
                isDark ? "text-sky-300" : "text-sky-700"
              }`}
            >
              Selecciona artefactos (base pre-cargada)
            </h3>
            <input
              placeholder="Buscar artefacto o categoría..."
              value={filtro}
              onChange={(e) => setFiltro(e.target.value)}
              className={`ml-auto w-full md:w-1/2 px-3 py-2 rounded-lg border ${
                borderBase
              } ${isDark ? "bg-slate-900" : "bg-white"}`}
            />
          </div>

          {/* Alta manual */}
          <div className={`rounded-xl border ${borderBase} ${card} p-4`}>
            <h4
              className={`font-semibold ${
                isDark ? "text-sky-300" : "text-sky-700"
              }`}
            >
              Agregar artefacto manualmente
            </h4>
            <div className="grid md:grid-cols-4 gap-2 mt-2 text-sm items-end">
              <label className="md:col-span-2">
                <span className={subtleText}>Nombre</span>
                <input
                  value={newNombre}
                  onChange={(e) => setNewNombre(e.target.value)}
                  placeholder="Ej: Horno eléctrico"
                  className={`mt-1 w-full px-3 py-2 rounded-lg border ${
                    borderBase
                  } ${isDark ? "bg-slate-900" : "bg-white"}`}
                />
              </label>
              <label>
                <span className={subtleText}>Potencia (W)</span>
                <input
                  type="number"
                  min={1}
                  value={newPotencia}
                  onChange={(e) =>
                    setNewPotencia(parseFloat(e.target.value || "0"))
                  }
                  className={`mt-1 w-full px-3 py-2 rounded-lg border ${
                    borderBase
                  } ${isDark ? "bg-slate-900" : "bg-white"}`}
                />
              </label>
              <label>
                <span className={subtleText}>Categoría</span>
                <input
                  value={newCategoria}
                  onChange={(e) => setNewCategoria(e.target.value)}
                  placeholder="Ej: Cocina"
                  className={`mt-1 w-full px-3 py-2 rounded-lg border ${
                    borderBase
                  } ${isDark ? "bg-slate-900" : "bg-white"}`}
                />
              </label>
            </div>
            <div className="mt-3">
              <button
                onClick={() => {
                  addCustom(newNombre, newPotencia, newCategoria);
                  setNewNombre("");
                  setNewPotencia(100);
                }}
                className="px-3 py-1.5 rounded-lg bg-emerald-600 hover:bg-emerald-700 text-white text-sm"
              >
                Agregar manual
              </button>
            </div>
          </div>

          {/* Cards catálogo */}
          <div className="grid sm:grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-4">
            {catalogoFiltrado.map((c) => (
              <div
                key={c.id}
                className={`rounded-xl border ${borderBase} ${card} p-4`}
              >
                <div className="flex items-start justify-between gap-2">
                  <div>
                    <div className="text-sm font-medium">{c.nombre}</div>
                    <div className={`${subtleText} text-xs`}>
                      {c.categoria} • {c.potenciaW} W
                    </div>
                  </div>
                  {c.id.startsWith("user-") && (
                    <button
                      title="Eliminar del catálogo"
                      onClick={() => removeFromCatalog(c.id)}
                      className="text-red-500 text-xs hover:underline"
                    >
                      Borrar
                    </button>
                  )}
                </div>
                <button
                  onClick={() => agregarItem(c.id)}
                  className="mt-3 text-sm px-3 py-1.5 rounded-lg bg-emerald-600 hover:bg-emerald-700 text-white"
                >
                  Agregar
                </button>
              </div>
            ))}
            {catalogoFiltrado.length === 0 && (
              <div className={`${subtleText} text-sm`}>
                Sin resultados para “{filtro}”.
              </div>
            )}
          </div>

          {/* Seleccionados */}
          <div className={`rounded-xl border ${borderBase} ${card} p-4`}>
            <h4
              className={`font-semibold mb-3 ${
                isDark ? "text-sky-300" : "text-sky-700"
              }`}
            >
              Seleccionados
            </h4>
            <div className="overflow-x-auto">
              <table
                className={`w-full text-sm border ${borderBase} rounded-xl overflow-hidden`}
              >
                <thead
                  className={`${isDark ? "bg-slate-900" : "bg-slate-100"}`}
                >
                  <tr>
                    <th className="text-left px-3 py-2">Artefacto</th>
                    <th className="text-left px-3 py-2">Potencia (W)</th>
                    <th className="text-left px-3 py-2">Unidades</th>
                    <th className="text-left px-3 py-2">Horas/día</th>
                    <th className="text-left px-3 py-2">Días/mes</th>
                    <th className="text-left px-3 py-2">kWh/mes</th>
                    <th className="px-3 py-2">&nbsp;</th>
                  </tr>
                </thead>
                <tbody>
                  {items.length === 0 ? (
                    <tr>
                      <td colSpan={7} className="px-3 py-4 text-center">
                        <span className="text-slate-500">
                          Aún no agregas artefactos.
                        </span>
                      </td>
                    </tr>
                  ) : (
                    items.map((it) => {
                      const c = mapaCat[it.id];
                      const kwh = c
                        ? (c.potenciaW *
                            it.horasDia *
                            it.diasMes *
                            it.qty) /
                          1000
                        : 0;
                      return (
                        <tr
                          key={it.id}
                          className={`${
                            isDark
                              ? "border-t border-slate-700"
                              : "border-t border-slate-200"
                          }`}
                        >
                          <td className="px-3 py-2">{c ? c.nombre : it.id}</td>
                          <td className="px-3 py-2">
                            {c ? c.potenciaW : "—"}
                          </td>
                          <td className="px-3 py-2">
                            <input
                              type="number"
                              min={1}
                              value={it.qty}
                              onChange={(e) =>
                                updItem(it.id, {
                                  qty: Math.max(
                                    1,
                                    parseInt(e.target.value || "1", 10)
                                  ),
                                })
                              }
                              className={`w-20 px-2 py-1 rounded-md border ${
                                borderBase
                              } ${isDark ? "bg-slate-900" : "bg-white"}`}
                            />
                          </td>
                          <td className="px-3 py-2">
                            <input
                              type="number"
                              min={0}
                              step={0.5}
                              value={it.horasDia}
                              onChange={(e) =>
                                updItem(it.id, {
                                  horasDia: parseFloat(
                                    e.target.value || "0"
                                  ),
                                })
                              }
                              className={`w-24 px-2 py-1 rounded-md border ${
                                borderBase
                              } ${isDark ? "bg-slate-900" : "bg-white"}`}
                            />
                          </td>
                          <td className="px-3 py-2">
                            <input
                              type="number"
                              min={0}
                              max={31}
                              value={it.diasMes}
                              onChange={(e) =>
                                updItem(it.id, {
                                  diasMes: Math.max(
                                    0,
                                    Math.min(
                                      31,
                                      parseInt(e.target.value || "0", 10)
                                    )
                                  ),
                                })
                              }
                              className={`w-24 px-2 py-1 rounded-md border ${
                                borderBase
                              } ${isDark ? "bg-slate-900" : "bg-white"}`}
                            />
                          </td>
                          <td className="px-3 py-2">{kwh.toFixed(2)}</td>
                          <td className="px-3 py-2 text-right">
                            <button
                              onClick={() => quitarItem(it.id)}
                              className="text-red-500 hover:underline"
                            >
                              Quitar
                            </button>
                          </td>
                        </tr>
                      );
                    })
                  )}
                </tbody>
                <tfoot>
                  <tr
                    className={`${
                      isDark ? "bg-slate-900" : "bg-slate-50"
                    }`}
                  >
                    <td className="px-3 py-2 font-medium" colSpan={5}>
                      Total estimado
                    </td>
                    <td className="px-3 py-2 font-semibold">
                      {estimadoKWh.toFixed(2)}
                    </td>
                    <td className="px-3 py-2" />
                  </tr>
                </tfoot>
              </table>
            </div>
            <div className="mt-4 flex gap-2">
              <button
                onClick={() => {
                  setKwhMes(Math.round(estimadoKWh));
                  showToast(
                    "Estimación aplicada al formulario del mes",
                    "info",
                    "top"
                  );
                }}
                className="px-4 py-2 rounded-lg bg-indigo-600 hover:bg-indigo-700 text-white text-sm"
              >
                Usar estimación para Guardar mes
              </button>
              <button
                onClick={() => setItems([])}
                className={`px-4 py-2 rounded-lg border text-sm ${
                  isDark
                    ? "border-slate-600 hover:bg-slate-800"
                    : "border-slate-300 hover:bg-slate-100"
                }`}
              >
                Limpiar selección
              </button>
            </div>
          </div>
        </section>

        {/* Acciones reales */}
        <section
          className={`rounded-2xl border ${borderBase} ${card} p-5 space-y-5`}
        >
          <div className="grid md:grid-cols-2 gap-5">
            {/* Formulario Guardar Mes */}
            <div>
              <h4
                className={`font-semibold mb-2 ${
                  isDark ? "text-sky-300" : "text-sky-700"
                }`}
              >
                Guardar mes en historial
              </h4>
              <div className="grid grid-cols-2 gap-3">
                <label className="text-sm">
                  <span className={subtleText}>Año</span>
                  <input
                    type="number"
                    value={anio}
                    onChange={(e) =>
                      setAnio(parseInt(e.target.value || "0", 10))
                    }
                    className={`mt-1 w-full px-3 py-2 rounded-lg border ${
                      borderBase
                    } ${isDark ? "bg-slate-900" : "bg-white"}`}
                  />
                </label>
                <label className="text-sm">
                  <span className={subtleText}>Mes (1-12)</span>
                  <input
                    type="number"
                    min={1}
                    max={12}
                    value={mes}
                    onChange={(e) =>
                      setMes(
                        Math.max(
                          1,
                          Math.min(
                            12,
                            parseInt(e.target.value || "1", 10)
                          )
                        )
                      )
                    }
                    className={`mt-1 w-full px-3 py-2 rounded-lg border ${
                      borderBase
                    } ${isDark ? "bg-slate-900" : "bg-white"}`}
                  />
                </label>
                <label className="col-span-2 text-sm">
                  <span className={subtleText}>Consumo del mes (kWh)</span>
                  <input
                    type="number"
                    min={1}
                    value={kwhMes}
                    onChange={(e) =>
                      setKwhMes(parseFloat(e.target.value || "0"))
                    }
                    className={`mt-1 w-full px-3 py-2 rounded-lg border ${
                      borderBase
                    } ${isDark ? "bg-slate-900" : "bg-white"}`}
                  />
                </label>
                <label className="col-span-2 text-sm">
                  <span className={subtleText}>Nota (opcional)</span>
                  <textarea
                    value={nota}
                    onChange={(e) => setNota(e.target.value)}
                    rows={2}
                    className={`mt-1 w-full px-3 py-2 rounded-lg border ${
                      borderBase
                    } ${isDark ? "bg-slate-900" : "bg-white"}`}
                  />
                </label>
                <label className="col-span-2 text-sm">
                  <span className={subtleText}>
                    Demanda media estimada vs. potencia
                  </span>
                  <div className="mt-1 text-xs">
                    {(() => {
                      const id = mesISO(anio, mes);
                      const parts = id.split("-");
                      const yy = parseInt(parts[0], 10);
                      const mm = parseInt(parts[1], 10);
                      const baseKwh = Math.max(
                        kwhMes,
                        Math.round(estimadoKWh) || 0
                      );
                      const avg = demandaMediaKW(baseKwh, yy, mm);
                      const pct =
                        potenciaContratada > 0
                          ? (avg / potenciaContratada) * 100
                          : 0;
                      return (
                        <span
                          className={
                            avg > potenciaContratada ? "text-red-500" : ""
                          }
                        >
                          {avg.toFixed(2)} kW ({pct.toFixed(0)}% del límite)
                        </span>
                      );
                    })()}
                  </div>
                </label>
              </div>
              <div className="flex items-center gap-2 mt-3">
                <ActionButton
                  onClick={onGuardarMes}
                  icon={<Save className="w-4 h-4" />}
                  label="Guardar mes"
                  tone="emerald"
                />
                <ActionButton
                  onClick={onExportarCSV}
                  icon={<Download className="w-4 h-4" />}
                  label="Exportar CSV"
                  tone="sky"
                />
                <ActionButton
                  onClick={onLoginOK}
                  icon={<LogIn className="w-4 h-4" />}
                  label="Simular login OK"
                  tone="indigo"
                />
                <label className="ml-auto inline-flex items-center gap-2 text-sm">
                  <input
                    type="checkbox"
                    checked={useHistInChart}
                    onChange={(e) => setUseHistInChart(e.target.checked)}
                  />
                  Usar historial en gráfico
                </label>
              </div>
            </div>

            {/* Comparar meses */}
            <div>
              <h4
                className={`font-semibold mb-2 ${
                  isDark ? "text-sky-300" : "text-sky-700"
                }`}
              >
                Comparar meses
              </h4>
              <div className="grid grid-cols-2 gap-3">
                <label className="text-sm">
                  <span className={subtleText}>Mes A</span>
                  <select
                    value={cmpA}
                    onChange={(e) => setCmpA(e.target.value)}
                    className={`mt-1 w-full px-3 py-2 rounded-lg border ${
                      borderBase
                    } ${isDark ? "bg-slate-900" : "bg-white"}`}
                  >
                    <option value="">—</option>
                    {hist.map((h) => (
                      <option key={h.mesISO} value={h.mesISO}>
                        {h.mesISO}
                      </option>
                    ))}
                  </select>
                </label>
                <label className="text-sm">
                  <span className={subtleText}>Mes B</span>
                  <select
                    value={cmpB}
                    onChange={(e) => setCmpB(e.target.value)}
                    className={`mt-1 w-full px-3 py-2 rounded-lg border ${
                      borderBase
                    } ${isDark ? "bg-slate-900" : "bg-white"}`}
                  >
                    <option value="">—</option>
                    {hist.map((h) => (
                      <option key={h.mesISO} value={h.mesISO}>
                        {h.mesISO}
                      </option>
                    ))}
                  </select>
                </label>
              </div>
              <div className="mt-3 text-sm">
                {cmpA && cmpB ? (
                  (() => {
                    const a =
                      hist.find((h) => h.mesISO === cmpA)?.kWh ?? 0;
                    const b =
                      hist.find((h) => h.mesISO === cmpB)?.kWh ?? 0;
                    const diff = a - b;

                    let msg;
                    if (a === b) {
                      msg = `Mismos kWh en ${cmpA} y ${cmpB} (${a} kWh).`;
                    } else if (diff > 0) {
                      msg = `${cmpA} consumió ${diff} kWh más que ${cmpB}.`;
                    } else {
                      msg = `${cmpB} consumió ${Math.abs(
                        diff
                      )} kWh más que ${cmpA}.`;
                    }

                    const cls =
                      diff === 0
                        ? ""
                        : diff > 0
                        ? "text-red-500"
                        : "text-emerald-500";

                    return <p className={cls}>{msg}</p>;
                  })()
                ) : (
                  <p className={subtleText}>
                    Selecciona dos meses para comparar.
                  </p>
                )}
              </div>
            </div>
          </div>

          {/* Tabla historial */}
          <div className="overflow-x-auto">
            <table
              className={`w-full text-sm border ${borderBase} rounded-xl overflow-hidden`}
            >
              <thead
                className={isDark ? "bg-slate-900" : "bg-slate-100"}
              >
                <tr>
                  <th className="text-left px-3 py-2">Mes</th>
                  <th className="text-left px-3 py-2">kWh</th>
                  <th className="text-left px-3 py-2">Nota</th>
                  <th className="px-3 py-2">Acciones</th>
                </tr>
              </thead>
              <tbody>
                {hist.length === 0 ? (
                  <tr>
                    <td colSpan={4} className="px-3 py-4 text-center">
                      <span className="text-slate-500">
                        Sin registros aún.
                      </span>
                    </td>
                  </tr>
                ) : (
                  hist.map((h) => (
                    <tr
                      key={h.mesISO}
                      className={`${
                        isDark
                          ? "border-t border-slate-700"
                          : "border-t border-slate-200"
                      }`}
                    >
                      <td className="px-3 py-2">{h.mesISO}</td>
                      <td className="px-3 py-2">{h.kWh}</td>
                      <td className="px-3 py-2">{h.nota ?? "—"}</td>
                      <td className="px-3 py-2 text-right">
                        <button
                          onClick={() => onEliminarMes(h.mesISO)}
                          className="text-red-500 hover:underline"
                        >
                          Eliminar
                        </button>
                      </td>
                    </tr>
                  ))
                )}
              </tbody>
            </table>
          </div>
        </section>

        {/* Charts */}
        <section className="grid md:grid-cols-2 gap-8">
          <div
            className={`p-6 rounded-2xl shadow-xl border ${borderBase} ${card}`}
          >
            <h3
              className={`text-lg font-semibold mb-4 ${
                isDark ? "text-sky-400" : "text-sky-700"
              }`}
            >
              📈 Historial de consumo mensual{" "}
              <span className={`text-xs ${subtleText}`}>
                {useHistInChart ? "(usando historial)" : "(demo)"}
              </span>
            </h3>
            <ResponsiveContainer width="100%" height={300}>
              <LineChart data={useHistInChart ? dataFromHist : demoData}>
                <CartesianGrid
                  strokeDasharray="3 3"
                  stroke={isDark ? "#475569" : "#cbd5e1"}
                />
                <XAxis
                  dataKey="name"
                  stroke={isDark ? "#e2e8f0" : "#0f172a"}
                />
                <YAxis stroke={isDark ? "#e2e8f0" : "#0f172a"} />
                <Tooltip
                  contentStyle={{
                    background: isDark ? "#0f172a" : "#ffffff",
                    border: `1px solid ${
                      isDark ? "#475569" : "#cbd5e1"
                    }`,
                  }}
                  labelStyle={{
                    color: isDark ? "#e2e8f0" : "#0f172a",
                  }}
                />
                <Legend />
                <Line
                  type="monotone"
                  dataKey="kWh"
                  stroke="#38bdf8"
                  strokeWidth={2}
                  dot={{ r: 4 }}
                />
              </LineChart>
            </ResponsiveContainer>
          </div>

          <div
            className={`p-6 rounded-2xl shadow-xl border ${borderBase} ${card}`}
          >
            <h3
              className={`text-lg font-semibold mb-4 ${
                isDark ? "text-sky-400" : "text-sky-700"
              }`}
            >
              📊 Consumo por tipo de artefacto
            </h3>
            <ResponsiveContainer width="100%" height={300}>
              <PieChart>
                <Pie
                  data={pieData}
                  dataKey="value"
                  nameKey="name"
                  cx="50%"
                  cy="50%"
                  outerRadius={100}
                  label
                >
                  {pieData.map((entry, index) => (
                    <Cell
                      key={`cell-${index}`}
                      fill={colors[index % colors.length]}
                    />
                  ))}
                </Pie>
                <Tooltip
                  contentStyle={{
                    background: isDark ? "#0f172a" : "#ffffff",
                    border: `1px solid ${
                      isDark ? "#475569" : "#cbd5e1"
                    }`,
                  }}
                  labelStyle={{
                    color: isDark ? "#e2e8f0" : "#0f172a",
                  }}
                />
                <Legend />
              </PieChart>
            </ResponsiveContainer>
          </div>
        </section>
      </main>

      <footer
        className={`mt-16 border-t ${borderBase} py-6 text-center ${subtleText} text-sm`}
      >
        <p>
          © {new Date().getFullYear()} Voltify Pro — Herramienta didáctica para
          hogares chilenos.
        </p>
      </footer>
    </div>
  );
}

// =====================
// Subcomponentes
// =====================
function ThemeButton({ active, onClick, title, children }) {
  return (
    <button
      onClick={onClick}
      title={title}
      className={`inline-flex items-center gap-1 px-2.5 py-1.5 rounded-lg border text-xs transition ${
        active
          ? "bg-sky-500 text-white border-sky-500"
          : "hover:bg-slate-100 border-slate-300 text-slate-700 dark:text-slate-200 dark:hover:bg-slate-800 dark:border-slate-600"
      }`}
    >
      {children}
      <span className="hidden sm:inline">{title}</span>
    </button>
  );
}

function ActionButton({ onClick, icon, label, tone }) {
  const toneClass = {
    emerald: "bg-emerald-600 hover:bg-emerald-700",
    sky: "bg-sky-600 hover:bg-sky-700",
    indigo: "bg-indigo-600 hover:bg-indigo-700",
  }[tone];

  return (
    <button
      onClick={onClick}
      className={`${toneClass} text-white px-4 py-2 rounded-lg inline-flex items-center gap-2 text-sm`}
    >
      {icon}
      <span>{label}</span>
    </button>
  );
}

function ToastStack({ toasts, onDismiss, isDark, position }) {
  return (
    <div
      className={`fixed ${
        position === "top" ? "top-6" : "bottom-6"
      } right-6 z-50 flex ${
        position === "top" ? "flex-col" : "flex-col-reverse"
      } gap-3 items-end`}
    >
      <AnimatePresence>
        {toasts.map((toast) => (
          <motion.div
            key={toast.id}
            initial={{
              opacity: 0,
              y: position === "top" ? -20 : 20,
              scale: 0.98,
            }}
            animate={{ opacity: 1, y: 0, scale: 1 }}
            exit={{
              opacity: 0,
              y: position === "top" ? -20 : 20,
              scale: 0.98,
            }}
            transition={{ type: "spring", stiffness: 250, damping: 20 }}
            role="status"
            aria-live="polite"
            drag="y"
            dragConstraints={{ top: 0, bottom: 0 }}
            dragElastic={0.2}
            onDragEnd={(_, info) => {
              if (Math.abs(info.offset.y) > 60) onDismiss(toast.id);
            }}
            onClick={() => onDismiss(toast.id)}
            className={`cursor-pointer active:scale-[0.98] hover:opacity-90 flex items-center gap-2 px-4 py-3 rounded-xl shadow-2xl text-sm font-medium ${
              toast.type === "success"
                ? "bg-emerald-600 text-white"
                : toast.type === "info"
                ? "bg-sky-500 text-white"
                : "bg-red-500 text-white"
            }`}
          >
            <ToastIcon type={toast.type} />
            <motion.span
              initial={{ x: -6, opacity: 0 }}
              animate={{ x: 0, opacity: 1 }}
              exit={{ x: 6, opacity: 0 }}
              className="pr-1"
            >
              {toast.message}
            </motion.span>
            <button
              aria-label="Cerrar"
              onClick={(e) => {
                e.stopPropagation();
                onDismiss(toast.id);
              }}
              className="ml-1 grid place-items-center rounded-md/50 w-6 h-6 hover:bg-black/10"
              title="Cerrar"
            >
              ✕
            </button>
          </motion.div>
        ))}
      </AnimatePresence>
    </div>
  );
}

function ToastIcon({ type }) {
  const iconVariants = {
    initial: { scale: 0, rotate: -90, opacity: 0 },
    animate: { scale: 1, rotate: 0, opacity: 1 },
    exit: { scale: 0, rotate: 90, opacity: 0 },
  };

  if (type === "success")
    return (
      <motion.div
        variants={iconVariants}
        initial="initial"
        animate="animate"
        exit="exit"
      >
        <CheckCircle className="w-5 h-5 mr-1" />
      </motion.div>
    );
  if (type === "error")
    return (
      <motion.div
        variants={iconVariants}
        initial="initial"
        animate="animate"
        exit="exit"
      >
        <XCircle className="w-5 h-5 mr-1" />
      </motion.div>
    );
  return (
    <motion.div
      variants={iconVariants}
      initial="initial"
      animate="animate"
      exit="exit"
    >
      <Info className="w-5 h-5 mr-1" />
    </motion.div>
  );
}

// Gauge semicircular SVG
function Gauge({ value, max, dark }) {
  const pct = Math.max(0, Math.min(1, max > 0 ? value / max : 0));
  const start = Math.PI; // 180°
  const end = 0; // 0°
  const R = 70;
  const CX = 100;
  const CY = 100;
  const SW = 12;

  const polar = (ang) => ({
    x: CX + R * Math.cos(ang),
    y: CY + R * Math.sin(ang),
  });

  const pStart = polar(start);
  const pEndBg = polar(end);
  const pVal = polar(start + (end - start) * pct);

  const arcBg = `M ${pStart.x} ${pStart.y} A ${R} ${R} 0 0 1 ${pEndBg.x} ${pEndBg.y}`;
  const arcVal =
    pct <= 0
      ? ""
      : `M ${pStart.x} ${pStart.y} A ${R} ${R} 0 0 1 ${pVal.x} ${pVal.y}`;

  const strokeVal = pct > 0.9 ? "#ef4444" : pct > 0.7 ? "#f97316" : "#22c55e";
  const txt = `${(pct * 100).toFixed(0)}%`;

  return (
    <svg
      width={200}
      height={120}
      viewBox="0 0 200 120"
      role="img"
      aria-label={`Gauge ${txt}`}
    >
      <path
        d={arcBg}
        stroke={dark ? "#334155" : "#e2e8f0"}
        strokeWidth={SW}
        fill="none"
        strokeLinecap="round"
      />
      {pct > 0 && (
        <path
          d={arcVal}
          stroke={strokeVal}
          strokeWidth={SW}
          fill="none"
          strokeLinecap="round"
        />
      )}
      <text
        x={CX}
        y={CY}
        textAnchor="middle"
        fontSize={14}
        fill={dark ? "#e2e8f0" : "#0f172a"}
      >
        {txt}
      </text>
    </svg>
  );
}

// Hook para detectar el esquema del SO
function usePrefersDarkMode() {
  const [isDark, setIsDark] = useState(false);

  useEffect(() => {
    if (typeof window === "undefined" || !window.matchMedia) return;

    const mq = window.matchMedia("(prefers-color-scheme: dark)");
    const listener = (e) => setIsDark(e.matches);

    setIsDark(mq.matches);
    mq.addEventListener("change", listener);

    return () => {
      mq.removeEventListener("change", listener);
    };
  }, []);

  return isDark;
}
