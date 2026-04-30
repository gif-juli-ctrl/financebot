import { useState, useEffect, useRef } from "react";
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  import.meta.env.VITE_SUPABASE_URL,
  import.meta.env.VITE_SUPABASE_ANON_KEY
);

const USER_ID = "julian"; // identificador fijo para tu uso personal
const TEAL = "#0D7377";
const TEAL_DARK = "#085457";
const TEAL_MID = "#14A9AE";

const DEFAULT_CATEGORIES = {
  supermercado: ["dia", "super", "carrefour", "coto", "jumbo", "disco", "vea", "walmart"],
  transporte: ["taxi", "uber", "sube", "remis", "colectivo", "subte", "tren", "cabify"],
  delivery: ["pedidos", "rappi", "mcdonald", "burger", "mcdonalds", "pizza"],
  salud: ["farmacia", "medico", "doctor", "clinica", "hospital", "dentista", "oculista"],
  entretenimiento: ["cine", "teatro", "bar", "boliche", "concert", "netflix", "spotify", "disney"],
  servicios: ["luz", "gas", "agua", "internet", "wifi", "celular", "telefono"],
  ropa: ["ropa", "zapatillas", "zapatos", "zara"],
  varios: ["varios", "otro", "otros"]
};

const CAT_COLORS = {
  supermercado: "#1D9E75", transporte: "#378ADD", delivery: "#D85A30",
  salud: "#D4537E", entretenimiento: "#7F77DD", servicios: "#BA7517",
  ropa: "#639922", varios: "#888780"
};

const DEFAULT_STATE = {
  salary_usd: 0, exchange_rate: 0, salary_ars: 0,
  fixed_expenses: [], variable_expenses: [], shared_expenses: [],
  balance: 0, month: "", setup_done: false,
  categories: DEFAULT_CATEGORIES
};

function fmt(n) { return Number(n).toLocaleString("es-AR", { maximumFractionDigits: 0 }); }
function fmtUSD(n) { return Number(n).toLocaleString("es-AR", { minimumFractionDigits: 2, maximumFractionDigits: 2 }); }

function detectPayment(txt) {
  const t = txt.toLowerCase();
  if (t.indexOf("credito") >= 0 || t.indexOf("visa") >= 0 || t.indexOf("master") >= 0) return "credito";
  return "debito";
}

function resolveCategory(word, categories) {
  const w = word.toLowerCase().trim();
  if (categories[w]) return w;
  for (var cat in categories) {
    var aliases = categories[cat];
    for (var i = 0; i < aliases.length; i++) {
      if (aliases[i] === w || w.indexOf(aliases[i]) >= 0 || aliases[i].indexOf(w) >= 0) return cat;
    }
  }
  return null;
}

function parseDate(txt) {
  const now = new Date();
  if (txt.indexOf("ayer") >= 0) {
    const y = new Date(now); y.setDate(y.getDate() - 1);
    return y.toLocaleDateString("es-AR");
  }
  const m = txt.match(/\b(\d{1,2})\/(\d{1,2})\b/);
  if (m) {
    const parsed = new Date(now.getFullYear(), parseInt(m[2]) - 1, parseInt(m[1]));
    if (!isNaN(parsed.getTime())) return parsed.toLocaleDateString("es-AR");
  }
  return now.toLocaleDateString("es-AR");
}

function getBotResponse(input, state, setState) {
  const txt = input.trim().toLowerCase();
  const monthName = new Date().toLocaleString("es-AR", { month: "long", year: "numeric" });

  if (txt.indexOf("borrar todo") >= 0 || txt.indexOf("reset") >= 0 || txt.indexOf("empezar de cero") >= 0) {
    setState(Object.assign({}, DEFAULT_STATE, { categories: state.categories }));
    return "Borre todos los registros. Las categorias se mantuvieron.";
  }

  const salaryMatch = input.match(/cobr[eé]\s+([\d.,]+)[^0-9]*cambio\s*([\d.,]+)/i);
  if (salaryMatch) {
    const usd = parseFloat(salaryMatch[1].replace(",", "."));
    const rate = parseFloat(salaryMatch[2].replace(",", "."));
    const ars = Math.round(usd * rate);
    const totalFixed = state.fixed_expenses.reduce((a,e)=>a+e.amount,0);
    setState(Object.assign({}, state, { salary_usd: usd, exchange_rate: rate, salary_ars: ars, balance: ars - totalFixed, month: monthName, setup_done: true }));
    return "Salario registrado: USD " + fmtUSD(usd) + " x $" + fmt(rate) + " = $" + fmt(ars) + " ARS.\nBalance inicial: $" + fmt(ars - totalFixed) + ".";
  }

  const rateMatch = txt.match(/^cambio\s*([\d.,]+)/);
  if (rateMatch && state.salary_usd > 0) {
    const rate = parseFloat(rateMatch[1].replace(",", "."));
    const ars = Math.round(state.salary_usd * rate);
    const diff = ars - state.salary_ars;
    setState(Object.assign({}, state, { exchange_rate: rate, salary_ars: ars, balance: state.balance + diff }));
    return "Cambio actualizado a $" + fmt(rate) + ". Balance: $" + fmt(state.balance + diff) + ".";
  }

  if (txt.indexOf("borrar ultimo") >= 0 || txt.indexOf("deshacer") >= 0) {
    if (state.variable_expenses.length === 0) return "No hay gastos para borrar.";
    const last = state.variable_expenses[state.variable_expenses.length - 1];
    const restore = last.payment === "credito" ? 0 : last.payment === "reintegro" ? -last.amount : last.amount;
    setState(Object.assign({}, state, { variable_expenses: state.variable_expenses.slice(0,-1), balance: state.balance + restore }));
    return "Borre: $" + fmt(last.amount) + " en " + last.category + " (" + last.date + ").";
  }

  const deleteMatch = input.match(/borrar[:\s]+(.+)/i);
  if (deleteMatch) {
    const term = deleteMatch[1].trim().toLowerCase();
    const found = state.variable_expenses.map((e,i)=>({e,i})).reverse().find(x=>x.e.category.indexOf(term)>=0||x.e.description.indexOf(term)>=0);
    if (!found) return "No encontre ningun gasto con \"" + term + "\".";
    const restore = found.e.payment === "credito" ? 0 : found.e.payment === "reintegro" ? -found.e.amount : found.e.amount;
    setState(Object.assign({}, state, { variable_expenses: state.variable_expenses.filter((_,i)=>i!==found.i), balance: state.balance + restore }));
    return "Borre: $" + fmt(found.e.amount) + " en " + found.e.category + " del " + found.e.date + ".";
  }

  const reintegroMatch = input.match(/reintegro[:\s]+([a-z\u00e1\u00e9\u00ed\u00f3\u00fa\u00f1\s]*?)\s*([\d.,]+)/i);
  if (reintegroMatch) {
    const desc = reintegroMatch[1].trim() || "reintegro";
    const amount = parseFloat(reintegroMatch[2].replace(",", "."));
    const exp = { id: Date.now(), category: "reintegro", description: desc, amount, payment: "reintegro", date: parseDate(txt) };
    setState(Object.assign({}, state, { variable_expenses: state.variable_expenses.concat([exp]), balance: state.balance + amount }));
    return "Reintegro: +$" + fmt(amount) + " (" + desc + "). Balance: $" + fmt(state.balance + amount) + ".";
  }

  const newCatMatch = input.match(/categor[ií]a[:\s]+([a-z\u00e1\u00e9\u00ed\u00f3\u00fa\u00f1\s]+?)\s*=\s*(.+)/i);
  if (newCatMatch) {
    const catName = newCatMatch[1].trim().toLowerCase();
    const aliases = newCatMatch[2].split(",").map(a=>a.trim().toLowerCase()).filter(a=>a.length>0);
    const newCats = Object.assign({}, state.categories);
    newCats[catName] = (newCats[catName]||[]).concat(aliases.filter(a=>(newCats[catName]||[]).indexOf(a)<0));
    setState(Object.assign({}, state, { categories: newCats }));
    return "Categoria \"" + catName + "\" con aliases: " + aliases.join(", ") + ".";
  }

  const aliasMatch = input.match(/alias[:\s]+([a-z\u00e1\u00e9\u00ed\u00f3\u00fa\u00f1]+)\s*=\s*(.+)/i);
  if (aliasMatch) {
    const catName = aliasMatch[1].trim().toLowerCase();
    if (!state.categories[catName]) return "No existe la categoria \"" + catName + "\".";
    const newAliases = aliasMatch[2].split(",").map(a=>a.trim().toLowerCase()).filter(a=>a.length>0);
    const newCats = Object.assign({}, state.categories);
    newCats[catName] = newCats[catName].concat(newAliases.filter(a=>newCats[catName].indexOf(a)<0));
    setState(Object.assign({}, state, { categories: newCats }));
    return "Agregue aliases a \"" + catName + "\": " + newAliases.join(", ") + ".";
  }

  if (txt.indexOf("categorias") >= 0) {
    return "Tus categorias:\n\n" + Object.entries(state.categories).map(x=>"- "+x[0]+": "+x[1].join(", ")).join("\n");
  }

  const fixedMatch = input.match(/fijo[:\s]+([a-z\u00e1\u00e9\u00ed\u00f3\u00fa\u00f1\s]+?)\s+([\d.,]+)/i);
  if (fixedMatch) {
    const name = fixedMatch[1].trim();
    const amount = parseFloat(fixedMatch[2].replace(",", "."));
    const payment = detectPayment(input);
    const newFixed = state.fixed_expenses.filter(e=>e.name.toLowerCase()!==name.toLowerCase());
    newFixed.push({ name, amount, payment });
    const totalFixed = newFixed.reduce((a,e)=>a+e.amount,0);
    const totalVar = state.variable_expenses.filter(e=>e.payment!=="credito").reduce((a,e)=>a+e.amount,0);
    setState(Object.assign({}, state, { fixed_expenses: newFixed, balance: state.salary_ars - totalFixed - totalVar - state.shared_expenses.reduce((a,e)=>a+e.total/2,0) }));
    return "Fijo \"" + name + "\": $" + fmt(amount) + ". Total fijos: $" + fmt(totalFixed) + ".";
  }

  const sharedMatch = input.match(/compartido[:\s]+([a-z\u00e1\u00e9\u00ed\u00f3\u00fa\u00f1\s]+?)\s+([\d.,]+)/i);
  if (sharedMatch) {
    const desc = sharedMatch[1].trim();
    const total = parseFloat(sharedMatch[2].replace(",", "."));
    const myHalf = total / 2;
    const paidByMe = txt.indexOf("pago ella") < 0 && txt.indexOf("pago novia") < 0;
    const exp = { id: Date.now(), desc, total, paid_by: paidByMe ? "yo" : "ella", payment: detectPayment(input), date: parseDate(txt) };
    setState(Object.assign({}, state, { shared_expenses: state.shared_expenses.concat([exp]), balance: state.balance - myHalf }));
    return "Compartido \"" + desc + "\": $" + fmt(total) + " total.\n" + (paidByMe ? "Ella te debe $" : "Le deb\u00e9s $") + fmt(myHalf) + ". Balance: $" + fmt(state.balance - myHalf) + ".";
  }

  if (txt.indexOf("cuadramos") >= 0 || txt.indexOf("saldamos") >= 0) {
    const net = state.shared_expenses.reduce((a,e)=>a+(e.paid_by==="yo"?e.total/2:-(e.total/2)),0);
    setState(Object.assign({}, state, { shared_expenses: [] }));
    return "Saldado. " + (net>0?"Ella te habia pagado $"+fmt(net):net<0?"Vos le habian pagado $"+fmt(Math.abs(net)):"Estaban a mano.") + "\nHistorial reiniciado.";
  }

  const cardMatch = input.match(/tarjeta\s+([\d.,]+)/i) || input.match(/pagu[eé] tarjeta\s+([\d.,]+)/i);
  if (cardMatch) {
    const amount = parseFloat(cardMatch[1].replace(",", "."));
    const exp = { id: Date.now(), category: "tarjeta", description: "Pago resumen tarjeta", amount, payment: "resumen", date: parseDate(txt) };
    setState(Object.assign({}, state, { variable_expenses: state.variable_expenses.concat([exp]), balance: state.balance - amount }));
    return "Pago resumen tarjeta: $" + fmt(amount) + ". Balance: $" + fmt(state.balance - amount) + ".";
  }

  const spendMatch = input.match(/gast[eé]\s+([\d.,]+)\s+en\s+([a-z\u00e1\u00e9\u00ed\u00f3\u00fa\u00f1\s,]+)/i);
  if (spendMatch) {
    const amount = parseFloat(spendMatch[1].replace(",", "."));
    const rawWord = spendMatch[2].trim().split(/[,\-]/)[0].trim().toLowerCase();
    const payment = detectPayment(input);
    const date = parseDate(txt);
    const resolved = resolveCategory(rawWord, state.categories);
    if (!resolved) return "No reconoci \"" + rawWord + "\".\n\nPodes agregar:\n\"Alias: supermercado = " + rawWord + "\"\n\nCategorias: " + Object.keys(state.categories).join(", ");
    const exp = { id: Date.now(), category: resolved, description: rawWord, amount, payment, date };
    const newExps = state.variable_expenses.concat([exp]);
    const newBalance = payment === "credito" ? state.balance : state.balance - amount;
    const catTotal = newExps.filter(e=>e.category===resolved).reduce((a,e)=>a+e.amount,0);
    const catPct = state.salary_ars > 0 ? (catTotal / state.salary_ars) * 100 : 0;
    var alertMsg = catPct > 20 ? "\n\nAlerta: llevas $"+fmt(catTotal)+" en "+resolved+" ("+catPct.toFixed(0)+"% del salario)." : catPct > 12 ? "\n\nAtencion: ya llevas $"+fmt(catTotal)+" en "+resolved+"." : "";
    setState(Object.assign({}, state, { variable_expenses: newExps, balance: newBalance }));
    return "Registre $"+fmt(amount)+" en "+rawWord+(rawWord!==resolved?" ("+resolved+")":"")+" ("+payment+", "+date+"). Balance: $"+fmt(newBalance)+"."+(payment==="credito"?"\n(Credito: se paga mes vencido.)":"")+alertMsg;
  }

  if (txt.indexOf("resumen") >= 0 || txt.indexOf("como voy") >= 0) {
    if (!state.setup_done) return "Todavia no registraste tu salario.";
    const tF = state.fixed_expenses.reduce((a,e)=>a+e.amount,0);
    const tV = state.variable_expenses.filter(e=>e.payment!=="credito").reduce((a,e)=>a+e.amount,0);
    const tC = state.variable_expenses.filter(e=>e.payment==="credito").reduce((a,e)=>a+e.amount,0);
    const tS = state.shared_expenses.reduce((a,e)=>a+e.total,0);
    const net = state.shared_expenses.reduce((a,e)=>a+(e.paid_by==="yo"?e.total/2:-(e.total/2)),0);
    var byCat = {}; state.variable_expenses.forEach(e=>{byCat[e.category]=(byCat[e.category]||0)+e.amount;});
    const catLines = Object.entries(byCat).sort((a,b)=>b[1]-a[1]).map(x=>"- "+x[0]+": $"+fmt(x[1])).join("\n");
    return "Resumen de "+state.month+":\n\nSalario: $"+fmt(state.salary_ars)+"\nFijos: $"+fmt(tF)+"\nVariables: $"+fmt(tV)+"\nCredito acumulado: $"+fmt(tC)+"\nCompartidos: $"+fmt(tS)+" (tu mitad: $"+fmt(tS/2)+")\nSaldo novia: "+(net>0?"ella te debe $"+fmt(net):net<0?"le deb\u00e9s $"+fmt(Math.abs(net)):"a mano")+"\nBalance: $"+fmt(state.balance)+"\n\nPor categoria:\n"+(catLines||"Sin gastos")+"\n\nEscribi \"exportar\" para el reporte.";
  }

  if (txt.indexOf("exportar") >= 0 || txt.indexOf("reporte") >= 0) {
    const rows = [];
    state.fixed_expenses.forEach(e=>rows.push({fecha:"Fijo",tipo:"Fijo",categoria:e.name,descripcion:e.name,monto:e.amount,mi_parte:e.amount,pago:e.payment||"debito",quien:"-"}));
    state.variable_expenses.forEach(e=>rows.push({fecha:e.date,tipo:"Variable",categoria:e.category,descripcion:e.description,monto:e.amount,mi_parte:e.amount,pago:e.payment,quien:"-"}));
    state.shared_expenses.forEach(e=>rows.push({fecha:e.date,tipo:"Compartido",categoria:"compartido",descripcion:e.desc,monto:e.total,mi_parte:e.total/2,pago:e.payment,quien:e.paid_by}));
    const header = ["Fecha","Tipo","Categoria","Descripcion","Monto","Mi parte","Medio pago","Quien pago"];
    const csvText = [header.join(",")].concat(rows.map(r=>[r.fecha,r.tipo,r.categoria,r.descripcion,r.monto,r.mi_parte,r.pago,r.quien].join(","))).join("\n");
    const totalMiParte = rows.reduce((a,r)=>r.pago==="reintegro"?a-r.mi_parte:a+r.mi_parte,0);
    const tableRows = rows.map(r=>{
      const col = r.pago==="credito"?"#D85A30":r.pago==="reintegro"?"#1D9E75":TEAL;
      return "<tr style='border-bottom:0.5px solid #e0e0da'><td style='padding:6px 8px;font-size:12px;color:#666'>"+r.fecha+"</td><td style='padding:6px 8px;font-size:12px'>"+r.categoria+"</td><td style='padding:6px 8px;font-size:12px;color:#666'>"+r.descripcion+"</td><td style='padding:6px 8px;font-size:12px;text-align:right;font-weight:500'>$"+fmt(r.monto)+"</td><td style='padding:6px 8px;font-size:11px;text-align:center;color:"+col+"'>"+r.pago+"</td></tr>";
    }).join("");
    const html = "<div style='background:#fff;border:0.5px solid #e0e0da;border-radius:12px;overflow:hidden;margin:4px 0'><div style='padding:10px 12px;display:flex;justify-content:space-between;align-items:center;border-bottom:0.5px solid #e0e0da'><span style='font-size:13px;font-weight:500'>Reporte "+(state.month||"del mes")+"</span><button onclick=\"navigator.clipboard.writeText(document.getElementById('csv-fb').value).then(function(){var b=document.getElementById('copy-fb');b.textContent='Copiado!';setTimeout(function(){b.textContent='Copiar CSV';},2000)})\" id='copy-fb' style='font-size:12px;padding:5px 12px;border-radius:20px;border:0.5px solid "+TEAL+";color:"+TEAL+";background:transparent;cursor:pointer'>Copiar CSV</button></div><div style='overflow-x:auto'><table style='width:100%;border-collapse:collapse'><thead><tr style='background:#f5f5f0'><th style='padding:6px 8px;font-size:11px;text-align:left;color:#666'>Fecha</th><th style='padding:6px 8px;font-size:11px;text-align:left;color:#666'>Categoria</th><th style='padding:6px 8px;font-size:11px;text-align:left;color:#666'>Descripcion</th><th style='padding:6px 8px;font-size:11px;text-align:right;color:#666'>Monto</th><th style='padding:6px 8px;font-size:11px;text-align:center;color:#666'>Pago</th></tr></thead><tbody>"+tableRows+"</tbody><tfoot><tr style='background:#f5f5f0'><td colspan='3' style='padding:8px;font-size:13px;font-weight:500'>Total</td><td style='padding:8px;font-size:13px;font-weight:500;text-align:right'>$"+fmt(totalMiParte)+"</td><td></td></tr></tfoot></table></div><textarea id='csv-fb' style='display:none'>"+csvText+"</textarea></div>";
    return "TABLE:" + html;
  }

  if (txt.indexOf("ayuda") >= 0) {
    return "Comandos:\n\n SALARIO\n- \"Cobre 1200 dolares, cambio 1250\"\n- \"Cambio 1300\"\n\n GASTOS\n- \"Gaste 5000 en dia\"\n- \"Gaste 5000 en taxi, credito\"\n- \"Gaste 5000 en taxi, ayer\" / \"15/4\"\n- \"Fijo: alquiler 150000\"\n- \"Pague tarjeta 80000\"\n- \"Reintegro banco 3000\"\n\n BORRAR\n- \"Borrar ultimo\"\n- \"Borrar supermercado\"\n- \"Borrar todo\"\n\n COMPARTIDOS\n- \"Compartido: super 20000, pague yo\"\n- \"Compartido: resto 15000, pago ella\"\n- \"Cuadramos\"\n\n CATEGORIAS\n- \"Categorias\"\n- \"Categoria: restaurante = sushi, parrilla\"\n- \"Alias: salud = farmacity\"\n\n REPORTES\n- \"Resumen\" / \"Exportar\"";
  }

  return "No entendi. Proba con:\n- \"Gaste 5000 en dia\"\n- \"Ayuda\" para todos los comandos.";
}

export default function FinanceBot() {
  const [messages, setMessages] = useState([]);
  const [input, setInput] = useState("");
  const [state, setStateRaw] = useState(DEFAULT_STATE);
  const [loading, setLoading] = useState(true);
  const [saving, setSaving] = useState(false);
  const endRef = useRef(null);
  const initialized = useRef(false);

  async function loadData() {
    try {
      const { data } = await supabase.from("financebot").select("data").eq("id", USER_ID).single();
      if (data && data.data) {
        if (data.data.state) setStateRaw(data.data.state);
        if (data.data.messages && data.data.messages.length > 0) {
          setMessages(data.data.messages);
        } else {
          setMessages([{ from: "bot", text: "Bienvenido de vuelta! Tus datos fueron restaurados.\n\nEscribi \"resumen\" para ver como estas." }]);
        }
      } else {
        setMessages([{ from: "bot", text: "Hola! Soy tu asistente financiero.\n\nPara empezar:\n\"Cobre 1200 dolares, cambio 1250\"\n\nO \"ayuda\" para ver los comandos." }]);
      }
    } catch (_) {
      setMessages([{ from: "bot", text: "Hola! Soy tu asistente financiero.\n\nPara empezar:\n\"Cobre 1200 dolares, cambio 1250\"\n\nO \"ayuda\" para ver los comandos." }]);
    }
    setLoading(false);
  }

  useEffect(() => {
    if (initialized.current) return;
    initialized.current = true;
    loadData();
  }, []);

  function setState(s) { setStateRaw(s); }

  useEffect(() => {
    if (loading || messages.length === 0) return;
    setSaving(true);
    supabase.from("financebot").upsert({ id: USER_ID, data: { state, messages: messages.slice(-50) }, updated_at: new Date().toISOString() })
      .then(() => setSaving(false))
      .catch(() => setSaving(false));
  }, [state, messages]);

  useEffect(() => { endRef.current?.scrollIntoView({ behavior: "smooth" }); }, [messages]);

  function send() {
    const txt = input.trim();
    if (!txt || loading) return;
    const botText = getBotResponse(txt, state, setState);
    setMessages(prev => prev.concat([{ from: "user", text: txt }, { from: "bot", text: botText }]));
    setInput("");
  }

  const totalFixed = state.fixed_expenses.reduce((a,e)=>a+e.amount,0);
  const totalVar = state.variable_expenses.filter(e=>e.payment!=="credito").reduce((a,e)=>a+e.amount,0);
  const totalCred = state.variable_expenses.filter(e=>e.payment==="credito").reduce((a,e)=>a+e.amount,0);
  const totalSpent = totalFixed + totalVar;
  const pctUsed = state.salary_ars > 0 ? Math.min((totalSpent / state.salary_ars) * 100, 100) : 0;
  const credPct = (totalSpent + totalCred) > 0 ? Math.min((totalCred / (totalSpent + totalCred)) * 100, 100) : 0;
  const sharedTotal = state.shared_expenses.reduce((a,e)=>a+e.total,0);
  const netShared = state.shared_expenses.reduce((a,e)=>a+(e.paid_by==="yo"?e.total/2:-(e.total/2)),0);
  const sharedCount = state.shared_expenses.length;
  var byCat = {}; state.variable_expenses.forEach(e=>{byCat[e.category]=(byCat[e.category]||0)+e.amount;});
  const topCats = Object.entries(byCat).sort((a,b)=>b[1]-a[1]).slice(0,4);
  const balanceColor = state.balance < 0 ? "#A32D2D" : state.balance < state.salary_ars * 0.15 ? "#BA7517" : TEAL;
  const progressColor = pctUsed > 85 ? "#E24B4A" : pctUsed > 60 ? "#BA7517" : TEAL_MID;

  if (loading) return (
    <div style={{display:"flex",flexDirection:"column",height:"100vh",alignItems:"center",justifyContent:"center",background:"#f5f5f0"}}>
      <div style={{width:40,height:40,borderRadius:"50%",background:TEAL,display:"flex",alignItems:"center",justifyContent:"center",fontSize:20,color:"#fff",marginBottom:12}}>$</div>
      <div style={{fontSize:14,color:"#666"}}>Cargando tus datos...</div>
    </div>
  );

  return (
    <div style={{fontFamily:"system-ui,sans-serif",display:"flex",flexDirection:"column",height:"100vh",background:"#f5f5f0"}}>
      <div style={{background:TEAL_DARK,color:"#fff",padding:"12px 16px",display:"flex",alignItems:"center",gap:10,flexShrink:0,paddingTop:"env(safe-area-inset-top, 12px)"}}>
        <div style={{width:36,height:36,borderRadius:"50%",background:TEAL,display:"flex",alignItems:"center",justifyContent:"center",fontSize:16,fontWeight:500}}>$</div>
        <div style={{flex:1}}>
          <div style={{fontWeight:500,fontSize:15}}>FinanceBot</div>
          <div style={{fontSize:11,opacity:0.8}}>{state.month||"Sin datos aun"}</div>
        </div>
        {saving && <div style={{fontSize:10,opacity:0.6}}>guardando...</div>}
      </div>

      <div style={{flex:1,overflowY:"auto",padding:"12px 14px",display:"flex",flexDirection:"column",gap:8,minHeight:0}}>
        {messages.map((m,i) => {
          const isUser = m.from==="user";
          if (!isUser && m.text.indexOf("TABLE:") === 0) return <div key={i} style={{maxWidth:"100%",alignSelf:"flex-start",width:"100%"}} dangerouslySetInnerHTML={{__html:m.text.slice(6)}} />;
          return <div key={i} style={{maxWidth:"82%",padding:"9px 13px",borderRadius:isUser?"16px 16px 4px 16px":"16px 16px 16px 4px",background:isUser?TEAL:"#fff",color:isUser?"#fff":"#1a1a1a",alignSelf:isUser?"flex-end":"flex-start",fontSize:14,lineHeight:1.5,whiteSpace:"pre-wrap",border:isUser?"none":"0.5px solid #e0e0da"}}>{m.text}</div>;
        })}
        <div ref={endRef}/>
      </div>

      <div style={{display:"flex",gap:8,padding:"10px 14px",background:"#fff",borderTop:"0.5px solid #e0e0da",flexShrink:0}}>
        <input style={{flex:1,padding:"9px 13px",borderRadius:20,border:"0.5px solid #d0d0ca",fontSize:14,background:"#f5f5f0",color:"#1a1a1a",outline:"none"}} value={input} onChange={e=>setInput(e.target.value)} onKeyDown={e=>e.key==="Enter"&&send()} placeholder="Gaste 5000 en dia, ayer..."/>
        <button onClick={send} style={{width:38,height:38,borderRadius:"50%",background:TEAL,border:"none",color:"#fff",fontSize:20,cursor:"pointer",display:"flex",alignItems:"center",justifyContent:"center"}}>›</button>
      </div>

      <div style={{background:"#fff",borderTop:"0.5px solid #e0e0da",padding:"12px 14px",flexShrink:0}}>
        <div style={{display:"flex",justifyContent:"space-between",alignItems:"baseline",marginBottom:4}}>
          <span style={{fontSize:12,color:"#666"}}>Balance disponible</span>
          {state.salary_usd>0&&<span style={{fontSize:11,color:"#999"}}>USD {fmtUSD(state.salary_usd)} x ${fmt(state.exchange_rate)}</span>}
        </div>
        <div style={{fontSize:22,fontWeight:500,color:balanceColor,marginBottom:4}}>${fmt(state.balance)} <span style={{fontSize:13,fontWeight:400,color:"#666"}}>ARS</span></div>
        {totalCred>0&&<div style={{fontSize:11,color:"#D85A30",marginBottom:6}}>+ ${fmt(totalCred)} en credito pendiente</div>}
        <div style={{height:6,borderRadius:4,background:"#ebebeb",marginBottom:10,overflow:"hidden"}}>
          <div style={{height:"100%",borderRadius:4,width:pctUsed+"%",background:progressColor,transition:"width 0.4s"}}/>
        </div>
        <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:8,marginBottom:8}}>
          <div style={{background:"#f5f5f0",borderRadius:10,padding:"8px 10px"}}>
            <div style={{fontSize:11,color:"#666",marginBottom:2}}>Gastos fijos</div>
            <div style={{fontSize:15,fontWeight:500}}>${fmt(totalFixed)}</div>
          </div>
          <div style={{background:"#f5f5f0",borderRadius:10,padding:"8px 10px"}}>
            <div style={{fontSize:11,color:"#666",marginBottom:2}}>Variables (debito)</div>
            <div style={{fontSize:15,fontWeight:500}}>${fmt(totalVar)}</div>
          </div>
        </div>
        <div style={{background:"#f5f5f0",borderRadius:10,padding:"10px 12px",marginBottom:8}}>
          <div style={{fontSize:11,color:"#666",marginBottom:8}}>Medio de pago</div>
          <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:8,marginBottom:6}}>
            <div><div style={{fontSize:11,color:"#999",marginBottom:2}}>Debito / Efectivo</div><div style={{fontSize:15,fontWeight:500,color:TEAL}}>${fmt(totalVar+totalFixed)}</div></div>
            <div><div style={{fontSize:11,color:"#999",marginBottom:2}}>Credito (mes vencido)</div><div style={{fontSize:15,fontWeight:500,color:"#D85A30"}}>${fmt(totalCred)}</div></div>
          </div>
          <div style={{height:5,borderRadius:4,background:"#e0e0da",overflow:"hidden"}}>
            <div style={{height:"100%",borderRadius:4,width:credPct+"%",background:"#D85A30",transition:"width 0.4s"}}/>
          </div>
          <div style={{fontSize:10,color:"#999",marginTop:4,textAlign:"right"}}>{credPct.toFixed(0)}% en credito</div>
        </div>
        <div style={{background:"#f5f5f0",borderRadius:10,padding:"10px 12px",marginBottom:8}}>
          <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:8}}>
            <div style={{fontSize:11,color:"#666"}}>Gastos compartidos</div>
            {sharedCount>0&&<div style={{fontSize:10,color:"#999"}}>{sharedCount} registro{sharedCount!==1?"s":""}</div>}
          </div>
          {sharedCount===0
            ? <div style={{fontSize:13,color:"#999"}}>Sin gastos compartidos aun</div>
            : <div>
                <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:8,marginBottom:8}}>
                  <div><div style={{fontSize:11,color:"#999",marginBottom:2}}>Total juntos</div><div style={{fontSize:15,fontWeight:500}}>${fmt(sharedTotal)}</div></div>
                  <div><div style={{fontSize:11,color:"#999",marginBottom:2}}>Tu mitad</div><div style={{fontSize:15,fontWeight:500}}>${fmt(sharedTotal/2)}</div></div>
                </div>
                <div style={{borderTop:"0.5px solid #e0e0da",paddingTop:8}}>
                  <div style={{fontSize:11,color:"#999",marginBottom:3}}>Saldo neto</div>
                  <div style={{fontSize:15,fontWeight:500,color:netShared>0?TEAL:netShared<0?"#D85A30":"#666"}}>{netShared>0?"Ella te debe $"+fmt(netShared):netShared<0?"Le deb\u00e9s $"+fmt(Math.abs(netShared)):"A mano"}</div>
                  <div style={{fontSize:10,color:"#999",marginTop:4}}>Escribe "cuadramos" para saldar</div>
                </div>
              </div>
          }
        </div>
        {topCats.length>0&&(
          <div style={{display:"flex",gap:6,flexWrap:"wrap"}}>
            {topCats.map(x=>{const col=CAT_COLORS[x[0]]||"#888780";return <div key={x[0]} style={{fontSize:11,padding:"3px 9px",borderRadius:20,background:col+"22",color:col,border:"0.5px solid "+col+"44"}}>{x[0]} ${fmt(x[1])}</div>;})}
          </div>
        )}
      </div>
    </div>
  );
}
