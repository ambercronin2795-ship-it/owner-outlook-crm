import React, { useEffect, useMemo, useRef, useState } from "react";
import { v4 as uuidv4 } from "uuid";
import { set as idbSet, get as idbGet } from "idb-keyval";
import { FileDown, FileText, Plus, Printer, Save, Search, Trash2, Upload, Users, Wrench } from "lucide-react";
import html2canvas from "html2canvas";
import jsPDF from "jspdf";

/**
 * Owner Outlook CRM – Single‑file React app
 * -------------------------------------------------
 * What you get in this single component:
 * 1) A ready‑to‑use local CRM for your "Owner Outlook" sheet
 * 2) Add/Edit customers with auto‑calculated fields (math happens for you)
 * 3) Offline‑first storage in IndexedDB (persists on the device/browser)
 * 4) Export a customer sheet to PDF (or use the browser's Print ➜ Save as PDF)
 * 5) Backup/Restore all data (JSON) + customizable field labels
 *
 * HOW TO RUN (quick start):
 *   - Requires Node.js 18+.
 *   - Create an app with Vite:  npm create vite@latest owner-outlook -- --template react
 *   - cd owner-outlook && npm i && npm i idb-keyval lucide-react html2canvas jspdf uuid
 *   - Replace src/App.jsx with this file's contents.
 *   - npm run dev  (open the local URL in your browser)
 *
 * OPTIONAL (make it installable like an app):
 *   - Convert to a PWA later (Vite plugin: vite-plugin-pwa) so it can be “installed” on your phone.
 *   - Or wrap with Capacitor to build a native Android/iOS shell.
 *
 * SAFETY NOTE: This app stores data only on your device (IndexedDB). Back it up using the Backup tab.
 */

// ------------------------
// UTILITIES
// ------------------------
const currency = (n) =>
  new Intl.NumberFormat(undefined, { style: "currency", currency: "USD" }).format(Number(n || 0));

const numberOrZero = (v) => {
  const n = typeof v === "number" ? v : parseFloat(String(v).replace(/,/g, ""));
  return isNaN(n) ? 0 : n;
};

/**
 * Very small expression evaluator for formulas like:
 *  "monthly_payment * 12 + maintenance_fee_annual"
 * Allowed tokens: letters, digits, underscore, math ops, dot, spaces, parentheses
 * Variables are field ids (e.g., monthly_payment). We map missing variables to 0.
 */
function evalFormula(expr, vars) {
  if (!expr) return "";
  // Allow only safe characters
  const safe = expr.replace(/[^\w\d_+\-*/().\s]/g, "");
  const keys = Object.keys(vars);
  const vals = keys.map((k) => numberOrZero(vars[k]));
  try {
    // eslint-disable-next-line no-new-func
    const fn = new Function(...keys, `return (${safe});`);
    const result = fn(...vals);
    return Number.isFinite(result) ? result : "";
  } catch (e) {
    return "";
  }
}

// ------------------------
// SCHEMA – Starter template derived from a typical “Owner Outlook” sheet.
// You can rename labels or add fields in the Template tab later.
// ------------------------
const DEFAULT_SCHEMA = {
  title: "Owner Outlook",
  sections: [
    {
      id: "customer_info",
      title: "Customer Information",
      columns: 2,
      fields: [
        { id: "customer_name", label: "Customer Name", type: "text", required: true },
        { id: "phone", label: "Phone", type: "text" },
        { id: "email", label: "Email", type: "email" },
        { id: "address", label: "Address", type: "textarea" },
      ],
    },
    {
      id: "ownership",
      title: "Ownership Details",
      columns: 2,
      fields: [
        { id: "resort_name", label: "Resort / Program", type: "text" },
        { id: "owner_id", label: "Owner # / Member #", type: "text" },
        { id: "anniversary", label: "Anniversary Date", type: "date" },
        { id: "nights_owned", label: "Nights Owned (annual)", type: "number" },
        { id: "nights_used_ytd", label: "Nights Used (YTD)", type: "number" },
        {
          id: "remaining_nights",
          label: "Remaining Nights",
          type: "calculated",
          formula: "nights_owned - nights_used_ytd",
        },
      ],
    },
    {
      id: "financials",
      title: "Financials",
      columns: 3,
      fields: [
        { id: "purchase_price", label: "Purchase Price", type: "number" },
        { id: "down_payment", label: "Down Payment", type: "number" },
        { id: "loan_apr", label: "APR (%)", type: "number" },
        { id: "loan_term_months", label: "Loan Term (months)", type: "number" },
        { id: "monthly_payment", label: "Monthly Payment", type: "number" },
        { id: "maintenance_fee_annual", label: "Maintenance Fee (annual)", type: "number" },
        {
          id: "annual_loan_cost",
          label: "Annual Loan Cost",
          type: "calculated",
          formula: "monthly_payment * 12",
        },
        {
          id: "annual_cost_total",
          label: "Total Annual Cost",
          type: "calculated",
          formula: "annual_loan_cost + maintenance_fee_annual",
        },
        {
          id: "cost_per_night",
          label: "Cost per Night",
          type: "calculated",
          formula: "annual_cost_total / nights_owned",
        },
      ],
    },
    {
      id: "value",
      title: "Value / Savings",
      columns: 3,
      fields: [
        { id: "rack_rate_per_night", label: "Rack Rate per Night", type: "number" },
        { id: "est_rack_value_year", label: "Est. Rack Value (year)", type: "calculated", formula: "rack_rate_per_night * nights_owned" },
        { id: "est_rack_value_used", label: "Est. Value Used", type: "calculated", formula: "rack_rate_per_night * nights_used_ytd" },
        { id: "est_value_remaining", label: "Est. Value Remaining", type: "calculated", formula: "rack_rate_per_night * remaining_nights" },
        { id: "est_savings_vs_rack", label: "Est. Savings vs Rack", type: "calculated", formula: "est_rack_value_year - annual_cost_total" },
      ],
    },
    {
      id: "notes",
      title: "Notes",
      columns: 1,
      fields: [
        { id: "notes", label: "Internal Notes", type: "textarea" },
      ],
    },
  ],
};

// ------------------------
// UI SUB‑COMPONENTS (shadcn‑like minimal replacements to keep this file standalone)
// ------------------------
const Label = ({ children }) => (
  <label className="block text-sm font-medium text-gray-700 mb-1">{children}</label>
);

const Input = (props) => (
  <input
    {...props}
    className={
      "w-full rounded-2xl border border-gray-300 px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500 shadow-sm " +
      (props.className || "")
    }
  />
);

const Textarea = (props) => (
  <textarea
    {...props}
    className={
      "w-full rounded-2xl border border-gray-300 px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500 shadow-sm min-h-[90px] " +
      (props.className || "")
    }
  />
);

const Button = ({ children, variant = "primary", ...rest }) => {
  const base = "inline-flex items-center gap-2 rounded-2xl px-4 py-2 text-sm font-semibold shadow-sm transition";
  const kind =
    variant === "primary"
      ? "bg-indigo-600 text-white hover:bg-indigo-700"
      : variant === "ghost"
      ? "bg-transparent hover:bg-gray-100"
      : variant === "danger"
      ? "bg-rose-600 text-white hover:bg-rose-700"
      : "bg-gray-200 text-gray-900 hover:bg-gray-300";
  return (
    <button {...rest} className={`${base} ${kind} ${rest.className || ""}`} />
  );
};

const Card = ({ children }) => (
  <div className="rounded-2xl border border-gray-200 bg-white shadow-sm">{children}</div>
);
const CardHeader = ({ children }) => (
  <div className="border-b border-gray-100 px-5 py-4 flex items-center gap-3">{children}</div>
);
const CardTitle = ({ children }) => (
  <h3 className="text-lg font-semibold text-gray-900">{children}</h3>
);
const CardContent = ({ children }) => <div className="p-5">{children}</div>;

// ------------------------
// MAIN APP
// ------------------------
export default function OwnerOutlookCRM() {
  const [schema, setSchema] = useState(DEFAULT_SCHEMA);
  const [customers, setCustomers] = useState([]);
  const [activeTab, setActiveTab] = useState("customers");
  const [query, setQuery] = useState("");

  const [draft, setDraft] = useState(() => emptyCustomer(DEFAULT_SCHEMA));
  const [editingId, setEditingId] = useState(null);

  const printRef = useRef(null);

  // Load from IndexedDB
  useEffect(() => {
    (async () => {
      const saved = (await idbGet("owner_outlook_customers")) || [];
      const savedSchema = (await idbGet("owner_outlook_schema")) || DEFAULT_SCHEMA;
      setCustomers(saved);
      setSchema(savedSchema);
      if (savedSchema !== DEFAULT_SCHEMA) {
        // Re-seed draft with the restored schema
        setDraft(emptyCustomer(savedSchema));
      }
    })();
  }, []);

  // Persist to IndexedDB
  useEffect(() => {
    idbSet("owner_outlook_customers", customers);
  }, [customers]);

  useEffect(() => {
    idbSet("owner_outlook_schema", schema);
  }, [schema]);

  const filtered = useMemo(() => {
    const q = query.trim().toLowerCase();
    if (!q) return customers;
    return customers.filter((c) => {
      return (
        (c.customer_name || "").toLowerCase().includes(q) ||
        (c.phone || "").toLowerCase().includes(q) ||
        (c.resort_name || "").toLowerCase().includes(q) ||
        (c.email || "").toLowerCase().includes(q)
      );
    });
  }, [customers, query]);

  // Compute calculated fields in the draft whenever inputs change
  const computedDraft = useMemo(() => applyCalculated(schema, draft), [schema, draft]);

  function handleFieldChange(id, value) {
    setDraft((d) => ({ ...d, [id]: value }));
  }

  function handleSave() {
    const record = { ...computedDraft, id: editingId || uuidv4(), createdAt: editingId ? computedDraft.createdAt : new Date().toISOString(), updatedAt: new Date().toISOString() };
    setCustomers((list) => {
      const exists = list.some((c) => c.id === record.id);
      const next = exists ? list.map((c) => (c.id === record.id ? record : c)) : [record, ...list];
      return next;
    });
    // Reset form
    setDraft(emptyCustomer(schema));
    setEditingId(null);
    setActiveTab("customers");
  }

  function handleEdit(c) {
    setDraft(c);
    setEditingId(c.id);
    setActiveTab("add");
  }

  function handleDelete(id) {
    if (!confirm("Delete this customer? This cannot be undone.")) return;
    setCustomers((list) => list.filter((c) => c.id !== id));
  }

  function handleNew() {
    setDraft(emptyCustomer(schema));
    setEditingId(null);
    setActiveTab("add");
  }

  async function exportOneAsPDF(cust) {
    // Render a hidden printable sheet for this customer and convert to PDF
    const target = document.getElementById(`printable-${cust.id}`);
    if (!target) return;

    const canvas = await html2canvas(target, { scale: 2, backgroundColor: "#ffffff" });
    const imgData = canvas.toDataURL("image/png");

    const pdf = new jsPDF({ orientation: "p", unit: "pt", format: "a4" });
    const pageWidth = pdf.internal.pageSize.getWidth();
    const pageHeight = pdf.internal.pageSize.getHeight();

    const imgWidth = pageWidth - 64; // margin
    const imgHeight = (canvas.height * imgWidth) / canvas.width;
    const x = 32, y = 32;

    pdf.addImage(imgData, "PNG", x, y, imgWidth, imgHeight);
    pdf.save(`${(cust.customer_name || "customer").replace(/[^a-z0-9]/gi, "_")}_OwnerOutlook.pdf`);
  }

  function exportAllJSON() {
    const blob = new Blob([JSON.stringify(customers, null, 2)], { type: "application/json" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = "owner_outlook_backup.json";
    a.click();
    URL.revokeObjectURL(url);
  }

  function importJSON(e) {
    const file = e.target.files?.[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = () => {
      try {
        const data = JSON.parse(reader.result);
        if (Array.isArray(data)) setCustomers(data);
        else alert("Invalid file.");
      } catch (err) {
        alert("Could not parse file.");
      }
    };
    reader.readAsText(file);
  }

  function downloadTemplate() {
    const blob = new Blob([JSON.stringify(schema, null, 2)], { type: "application/json" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = "owner_outlook_template.json";
    a.click();
    URL.revokeObjectURL(url);
  }

  function uploadTemplate(e) {
    const file = e.target.files?.[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = () => {
      try {
        const data = JSON.parse(reader.result);
        if (!data || !Array.isArray(data.sections)) throw new Error("Bad template");
        setSchema(data);
        setDraft(emptyCustomer(data));
      } catch (err) {
        alert("Invalid template file.");
      }
    };
    reader.readAsText(file);
  }

  return (
    <div className="mx-auto max-w-6xl p-6 text-gray-900">
      <header className="mb-6 flex items-center justify-between">
        <h1 className="text-2xl font-bold">Owner Outlook CRM</h1>
        <div className="flex gap-2">
          <Button onClick={handleNew}><Plus size={16} /> New Customer</Button>
          <Button variant="ghost" onClick={() => setActiveTab("backup")}><FileDown size={16} /> Backup</Button>
          <Button variant="ghost" onClick={() => setActiveTab("template")}><Wrench size={16} /> Template</Button>
        </div>
      </header>

      <nav className="mb-5">
        <div className="inline-flex rounded-2xl border border-gray-200 bg-gray-50 p-1">
          {[
            { id: "customers", label: "Customers", Icon: Users },
            { id: "add", label: editingId ? "Edit Customer" : "Add Customer", Icon: Plus },
            { id: "template", label: "Template", Icon: Wrench },
            { id: "backup", label: "Backup", Icon: FileDown },
          ].map((t) => (
            <button
              key={t.id}
              onClick={() => setActiveTab(t.id)}
              className={`flex items-center gap-2 rounded-xl px-4 py-2 text-sm font-medium transition ${
                activeTab === t.id ? "bg-white shadow" : "text-gray-600 hover:bg-white"
              }`}
            >
              <t.Icon size={16} /> {t.label}
            </button>
          ))}
        </div>
      </nav>

      {activeTab === "customers" && (
        <Card>
          <CardHeader>
            <CardTitle>Saved Customers</CardTitle>
            <div className="ml-auto flex items-center gap-2">
              <div className="relative">
                <Search className="absolute left-3 top-2.5 text-gray-400" size={16} />
                <Input placeholder="Search by name, phone, resort, email" value={query} onChange={(e) => setQuery(e.target.value)} className="pl-8 w-80" />
              </div>
            </div>
          </CardHeader>
          <CardContent>
            {filtered.length === 0 ? (
              <p className="text-gray-500">No customers yet. Click <strong>New Customer</strong> to add one.</p>
            ) : (
              <div className="overflow-x-auto">
                <table className="min-w-full divide-y divide-gray-200">
                  <thead>
                    <tr className="text-left text-sm text-gray-600">
                      <th className="px-3 py-2">Name</th>
                      <th className="px-3 py-2">Phone</th>
                      <th className="px-3 py-2">Email</th>
                      <th className="px-3 py-2">Resort</th>
                      <th className="px-3 py-2">Updated</th>
                      <th className="px-3 py-2">Actions</th>
                    </tr>
                  </thead>
                  <tbody className="divide-y divide-gray-100">
                    {filtered.map((c) => (
                      <tr key={c.id}>
                        <td className="px-3 py-2">{c.customer_name || "—"}</td>
                        <td className="px-3 py-2">{c.phone || "—"}</td>
                        <td className="px-3 py-2">{c.email || "—"}</td>
                        <td className="px-3 py-2">{c.resort_name || "—"}</td>
                        <td className="px-3 py-2">{new Date(c.updatedAt || c.createdAt).toLocaleString()}</td>
                        <td className="px-3 py-2">
                          <div className="flex items-center gap-2">
                            <Button variant="ghost" className="!px-2" onClick={() => handleEdit(c)}><FileText size={16} /> View/Edit</Button>
                            <Button variant="ghost" className="!px-2" onClick={() => exportOneAsPDF(c)}><Printer size={16} /> PDF</Button>
                            <Button variant="danger" className="!px-2" onClick={() => handleDelete(c.id)}><Trash2 size={16} /> Delete</Button>
                          </div>
                          {/* Hidden printable content */}
                          <PrintableSheet schema={schema} data={c} id={`printable-${c.id}`} />
                        </td>
                      </tr>
                    ))}
                  </tbody>
                </table>
              </div>
            )}
          </CardContent>
        </Card>
      )}

      {activeTab === "add" && (
        <EditorCard
          title={editingId ? "Edit Customer" : "Add New Customer"}
          schema={schema}
          data={computedDraft}
          onFieldChange={handleFieldChange}
          onSave={handleSave}
        />
      )}

      {activeTab === "template" && (
        <TemplateEditor schema={schema} setSchema={setSchema} downloadTemplate={downloadTemplate} uploadTemplate={uploadTemplate} />
      )}

      {activeTab === "backup" && (
        <BackupPanel customers={customers} exportAllJSON={exportAllJSON} importJSON={importJSON} />)
      }
    </div>
  );
}

// ------------------------
// HELPERS & PANELS
// ------------------------
function emptyCustomer(schema) {
  const obj = { id: null, createdAt: null, updatedAt: null };
  for (const sec of schema.sections) {
    for (const f of sec.fields) obj[f.id] = "";
  }
  return obj;
}

function applyCalculated(schema, data) {
  const next = { ...data };
  const scope = { ...data };
  // ensure scope has calculated values iteratively in case formulas depend on each other
  let safety = 0;
  while (safety++ < 10) {
    let changed = false;
    for (const sec of schema.sections) {
      for (const f of sec.fields) {
        if (f.type === "calculated" && f.formula) {
          const val = evalFormula(f.formula, scope);
          if (Number.isFinite(val) && scope[f.id] !== val) {
            scope[f.id] = val;
            next[f.id] = val;
            changed = true;
          }
        }
      }
    }
    if (!changed) break;
  }
  return next;
}

function EditorCard({ title, schema, data, onFieldChange, onSave }) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>{title}</CardTitle>
        <div className="ml-auto flex items-center gap-2">
          <Button onClick={onSave}><Save size={16} /> Save</Button>
          <Button variant="ghost" onClick={() => window.print()}><Printer size={16} /> Print</Button>
        </div>
      </CardHeader>
      <CardContent>
        {schema.sections.map((sec) => (
          <section key={sec.id} className="mb-6">
            <h4 className="mb-3 text-base font-semibold text-gray-800">{sec.title}</h4>
            <div className={`grid gap-4 md:grid-cols-${Math.max(1, Math.min(sec.columns || 2, 3))}`}>
              {sec.fields.map((f) => (
                <FieldInput key={f.id} field={f} value={data[f.id]} onChange={(v) => onFieldChange(f.id, v)} />
              ))}
            </div>
          </section>
        ))}
      </CardContent>
    </Card>
  );
}

function FieldInput({ field, value, onChange }) {
  const commonProps = {
    id: field.id,
    value: value ?? "",
    onChange: (e) => onChange(e.target.value),
    placeholder: field.placeholder || "",
  };

  const label = (
    <div className="flex items-center justify-between">
      <Label>{field.label}</Label>
      {field.type === "calculated" && field.formula ? (
        <span className="text-xs text-gray-400">= {field.formula}</span>
      ) : null}
    </div>
  );

  if (field.type === "textarea") {
    return (
      <div>
        {label}
        <Textarea {...commonProps} />
      </div>
    );
  }

  if (field.type === "calculated") {
    return (
      <div>
        {label}
        <Input value={formatMaybeCurrency(field, value)} readOnly />
      </div>
    );
  }

  if (field.type === "number") {
    return (
      <div>
        {label}
        <Input type="number" step="any" {...commonProps} />
      </div>
    );
  }

  return (
    <div>
      {label}
      <Input type={field.type || "text"} {...commonProps} />
    </div>
  );
}

function formatMaybeCurrency(field, value) {
  // Heuristic: show currency formatting for certain ids
  const moneyish = [
    "purchase_price",
    "down_payment",
    "monthly_payment",
    "maintenance_fee_annual",
    "annual_loan_cost",
    "annual_cost_total",
    "rack_rate_per_night",
    "est_rack_value_year",
    "est_rack_value_used",
    "est_value_remaining",
    "est_savings_vs_rack",
  ];
  if (moneyish.includes(field.id)) return currency(value);
  return Number.isFinite(value) ? String(value) : value || "";
}

function TemplateEditor({ schema, setSchema, downloadTemplate, uploadTemplate }) {
  const [local, setLocal] = useState(JSON.stringify(schema, null, 2));
  const [error, setError] = useState("");

  function tryApply() {
    try {
      const data = JSON.parse(local);
      if (!data || !Array.isArray(data.sections)) throw new Error("Template must have a sections array.");
      setSchema(data);
      setError("");
      alert("Template updated.");
    } catch (e) {
      setError(e.message);
    }
  }

  return (
    <Card>
      <CardHeader>
        <CardTitle>Template</CardTitle>
        <div className="ml-auto flex items-center gap-2">
          <Button variant="ghost" onClick={downloadTemplate}><FileDown size={16} /> Download</Button>
          <label className="inline-flex items-center gap-2 cursor-pointer">
            <Upload size={16} />
            <input type="file" accept="application/json" className="hidden" onChange={uploadTemplate} />
            <span className="text-sm font-medium">Upload</span>
          </label>
        </div>
      </CardHeader>
      <CardContent>
        <p className="mb-3 text-sm text-gray-600">
          You can fully customize the form by editing the JSON below. Each field supports:
          <code className="mx-1 rounded bg-gray-100 px-1">id</code>,
          <code className="mx-1 rounded bg-gray-100 px-1">label</code>,
          <code className="mx-1 rounded bg-gray-100 px-1">type</code> (text, email, number, date, textarea, calculated), and an optional
          <code className="mx-1 rounded bg-gray-100 px-1">formula</code> for calculated fields.
        </p>
        <Textarea className="min-h-[360px] font-mono" value={local} onChange={(e) => setLocal(e.target.value)} />
        {error && <p className="mt-2 text-sm text-rose-600">{error}</p>}
        <div className="mt-3 flex items-center gap-2">
          <Button onClick={tryApply}><Save size={16} /> Apply Template</Button>
        </div>
      </CardContent>
    </Card>
  );
}

function BackupPanel({ customers, exportAllJSON, importJSON }) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>Backup & Restore</CardTitle>
      </CardHeader>
      <CardContent>
        <div className="grid gap-4 md:grid-cols-2">
          <div className="rounded-2xl border border-gray-200 p-4">
            <h4 className="mb-2 font-semibold">Export All Data</h4>
            <p className="mb-3 text-sm text-gray-600">Download a JSON backup of all customers saved on this device.</p>
            <Button onClick={exportAllJSON}><FileDown size={16} /> Download JSON</Button>
          </div>
          <div className="rounded-2xl border border-gray-200 p-4">
            <h4 className="mb-2 font-semibold">Import Data</h4>
            <p className="mb-3 text-sm text-gray-600">Restore a JSON backup exported from this app.</p>
            <label className="inline-flex items-center gap-2 cursor-pointer">
              <Upload size={16} />
              <input type="file" accept="application/json" className="hidden" onChange={importJSON} />
              <span className="text-sm font-medium">Choose file…</span>
            </label>
          </div>
        </div>
      </CardContent>
    </Card>
  );
}

function PrintableSheet({ schema, data, id }) {
  return (
    <div id={id} className="hidden print:block">
      <div className="mx-auto w-[794px] p-6">{/* ~A4 width at 96dpi */}
        <h2 className="mb-4 text-center text-xl font-bold">{schema.title} – Customer Sheet</h2>
        {schema.sections.map((sec) => (
          <div key={sec.id} className="mb-4">
            <h3 className="mb-2 text-base font-semibold">{sec.title}</h3>
            <table className="w-full text-sm">
              <tbody>
                {sec.fields.map((f) => (
                  <tr key={f.id} className="align-top">
                    <td className="w-1/3 px-2 py-1 font-medium">{f.label}</td>
                    <td className="w-2/3 px-2 py-1 border-b border-gray-200">
                      {formatMaybeCurrency(f, data[f.id]) || String(data[f.id] ?? "")} 
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        ))}
      </div>
    </div>
  );
}

// ------------------------
// PRINT STYLES
// ------------------------
const style = document.createElement("style");
style.innerHTML = `
@media print {
  body { background: white !important; }
}
`;
if (typeof document !== "undefined") document.head.appendChild(style);
  
