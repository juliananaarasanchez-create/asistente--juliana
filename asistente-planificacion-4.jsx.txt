import { useState, useRef, useEffect } from "react";

const CONSIDERACIONES = `
METODOLOGÍA Y CONSIDERACIONES IMPORTANTES (de Juliana Sánchez - @julisancheztrail):

RESPIRACIÓN: Es un componente clave del entrenamiento. Hay videos explicativos en el canal de YouTube.

PERCEPCIÓN DE ESFUERZO (RPE): Escala del 1 al 5. 1 es mínimo, 5 máximo. No se habla de pesos sino de percepción de esfuerzo.

RIR (Repetitions In Reserve): Repeticiones que dejás de hacer para no llegar al fallo muscular.
- Ejemplo: Zercher Squat 10 reps RIR 2 → elegís un peso con el que podrías hacer 12 reps máximas, pero hacés solo 10.
- Ejemplo: LM Push Press 6 reps RIR 4 → peso para 10 reps máximas, hacés solo 6.

PERCEPCIÓN DE TERMOSTATO SUBJETIVO: Antes de entrenar la atleta debe considerar:
1. ¿Qué tan fatigada estás hoy? (fatiga muscular)
2. ¿Cómo está tu ánimo y energía para el entrenamiento de fuerza?
3. ¿Cómo fue tu calidad de sueño?
4. ¿Tenés dolor muscular o agujetas?
El estado de ánimo influye a nivel mitocondrial y determina el tipo de entrenamiento.

SERIES, REPETICIONES Y TIPOS DE TRABAJO:
- Circuito: más de 3 ejercicios
- Biserie: 2 ejercicios
- Triserie: 3 ejercicios
- Las vueltas se indican: x3, x2, x4 (según corresponda)
- Cluster: ej 4x(2-2-2) = cuatro veces [2 reps + 2 reps + 2 reps], puede haber descanso o no entre repeticiones según lo indique la planificación.
`;

const SYSTEM_PROMPT = `Eres el asistente de entrenamiento de Juliana Sánchez (@julisancheztrail), entrenadora especializada en trail running y fuerza.

Tu rol es responder dudas de las atletas sobre su planificación de entrenamiento de forma clara, cercana y motivadora, como si fueras la propia Juliana respondiendo.

CONSIDERACIONES DE LA METODOLOGÍA DE JULIANA:
${CONSIDERACIONES}

LA PLANIFICACIÓN DE ESTA ATLETA ES:
---
{PLAN}
---

SOBRE LOS VIDEOS DE YOUTUBE:
Juliana tiene más de 400 videos en su canal @julisancheztrail donde explica ejercicios con títulos descriptivos. Cuando una atleta pregunte cómo ejecutar un ejercicio, SIEMPRE incluí al final de tu respuesta un link de búsqueda de YouTube así:
https://www.youtube.com/@julisancheztrail/search?query=NOMBRE+DEL+EJERCICIO

Por ejemplo, si pregunta por "squat unipodal", el link sería:
https://www.youtube.com/@julisancheztrail/search?query=squat+unipodal

Presentá el link así: "🎥 Ver video de Juliana: [nombre del ejercicio](link)"

REGLAS:
- Respondé siempre en español, tono cercano y motivador
- Usá la metodología de Juliana para explicar RPE, RIR, series, etc.
- Si la pregunta no está en la planificación, decilo amablemente
- Cuando expliques un ejercicio, siempre incluí el link al canal
`;

export default function AsistentePlanificacion() {
  const [planTexto, setPlanTexto] = useState("");
  const [planActivo, setPlanActivo] = useState(false);
  const [nombreArchivo, setNombreArchivo] = useState("");
  const [mensajes, setMensajes] = useState([]);
  const [input, setInput] = useState("");
  const [cargando, setCargando] = useState(false);
  const [procesandoPDF, setProcesandoPDF] = useState(false);
  const [vista, setVista] = useState("inicio");
  const [dragOver, setDragOver] = useState(false);
  const chatRef = useRef(null);
  const fileRef = useRef(null);

  useEffect(() => {
    if (chatRef.current) {
      chatRef.current.scrollTop = chatRef.current.scrollHeight;
    }
  }, [mensajes, cargando]);

  const procesarPDF = async (file) => {
    if (!file || file.type !== "application/pdf") {
      alert("Por favor sube un archivo PDF.");
      return;
    }
    setProcesandoPDF(true);
    setNombreArchivo(file.name);
    try {
      const base64 = await new Promise((resolve, reject) => {
        const reader = new FileReader();
        reader.onload = () => resolve(reader.result.split(",")[1]);
        reader.onerror = reject;
        reader.readAsDataURL(file);
      });

      const response = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          model: "claude-sonnet-4-6",
          max_tokens: 1000,
          messages: [{
            role: "user",
            content: [
              { type: "document", source: { type: "base64", media_type: "application/pdf", data: base64 } },
              { type: "text", text: "Extrae todo el texto de este documento de planificación de entrenamiento, conservando la estructura completa. Devuelve solo el texto, sin comentarios." }
            ]
          }]
        })
      });

      const data = await response.json();
      const texto = data.content?.find(b => b.type === "text")?.text || "";
      if (!texto) throw new Error("Sin texto");

      setPlanTexto(texto);
      setPlanActivo(true);
      setVista("chat");
      setMensajes([{
        rol: "asistente",
        texto: `¡Hola! 💪 Ya leí tu planificación. Estoy lista para ayudarte con cualquier duda sobre los ejercicios, cómo ejecutarlos, RPE, series, tiempos... ¡o lo que necesites! También puedo enviarte los videos de Juliana cuando lo necesites. ¿Qué querés consultar?`
      }]);
    } catch {
      alert("Hubo un error al leer el PDF. Intenta de nuevo.");
      setNombreArchivo("");
    } finally {
      setProcesandoPDF(false);
    }
  };

  const handleFileInput = (e) => { const f = e.target.files[0]; if (f) procesarPDF(f); };
  const handleDrop = (e) => { e.preventDefault(); setDragOver(false); const f = e.dataTransfer.files[0]; if (f) procesarPDF(f); };

  const renderTexto = (texto) => {
    const urlRegex = /(https?:\/\/[^\s\)]+)/g;
    const mdLinkRegex = /\[([^\]]+)\]\((https?:\/\/[^\)]+)\)/g;
    
    const parts = [];
    let last = 0;
    let match;
    const fullText = texto;
    
    // Replace markdown links first
    const processedParts = [];
    let remaining = fullText;
    let mdMatch;
    const mdRegex = /\[([^\]]+)\]\((https?:\/\/[^\)]+)\)/g;
    let lastIdx = 0;
    
    while ((mdMatch = mdRegex.exec(fullText)) !== null) {
      if (mdMatch.index > lastIdx) {
        processedParts.push({ type: "text", content: fullText.slice(lastIdx, mdMatch.index) });
      }
      processedParts.push({ type: "link", label: mdMatch[1], url: mdMatch[2] });
      lastIdx = mdMatch.index + mdMatch[0].length;
    }
    if (lastIdx < fullText.length) {
      processedParts.push({ type: "text", content: fullText.slice(lastIdx) });
    }

    return processedParts.map((p, i) => {
      if (p.type === "link") {
        return (
          <a key={i} href={p.url} target="_blank" rel="noopener noreferrer" style={s.link}>
            {p.label}
          </a>
        );
      }
      // Handle plain URLs in text
      const segments = [];
      const plainUrlRegex = /(https?:\/\/[^\s]+)/g;
      let pLast = 0;
      let pMatch;
      while ((pMatch = plainUrlRegex.exec(p.content)) !== null) {
        if (pMatch.index > pLast) segments.push(p.content.slice(pLast, pMatch.index));
        segments.push(<a key={pMatch.index} href={pMatch[0]} target="_blank" rel="noopener noreferrer" style={s.link}>{pMatch[0]}</a>);
        pLast = pMatch.index + pMatch[0].length;
      }
      if (pLast < p.content.length) segments.push(p.content.slice(pLast));
      return <span key={i}>{segments}</span>;
    });
  };

  const enviarMensaje = async () => {
    if (!input.trim() || cargando) return;
    const nuevoMensaje = { rol: "usuario", texto: input };
    const historial = [...mensajes, nuevoMensaje];
    setMensajes(historial);
    setInput("");
    setCargando(true);
    try {
      const system = SYSTEM_PROMPT.replace("{PLAN}", planTexto);
      const msgs = historial.map(m => ({ role: m.rol === "usuario" ? "user" : "assistant", content: m.texto }));
      const res = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ model: "claude-sonnet-4-6", max_tokens: 1000, system, messages: msgs })
      });
      const data = await res.json();
      const resp = data.content?.find(b => b.type === "text")?.text || "No pude generar una respuesta.";
      setMensajes(prev => [...prev, { rol: "asistente", texto: resp }]);
    } catch {
      setMensajes(prev => [...prev, { rol: "asistente", texto: "Ocurrió un error. Intenta de nuevo." }]);
    } finally {
      setCargando(false);
    }
  };

  const reiniciar = () => {
    setPlanTexto(""); setPlanActivo(false); setNombreArchivo("");
    setMensajes([]); setInput(""); setVista("inicio");
    if (fileRef.current) fileRef.current.value = "";
  };

  return (
    <div style={s.app}>
      <header style={s.header}>
        <div style={s.headerInner}>
          <div style={s.logoMark}>JS</div>
          <div style={{ flex: 1 }}>
            <div style={s.headerTitle}>Asistente de Entrenamiento</div>
            <div style={s.headerSub}>by Juliana Sánchez · @julisancheztrail</div>
          </div>
          {planActivo && <button onClick={reiniciar} style={s.btnReset}>Cambiar plan</button>}
        </div>
      </header>

      {vista === "inicio" && (
        <div style={s.centrado}>
          <div style={s.uploadCard}>
            <p style={s.eyebrow}>@julisancheztrail</p>
            <h1 style={s.headline}>Subí tu planificación</h1>
            <p style={s.subtext}>
              Cargá el PDF de tu plan y consultá cualquier duda sobre ejercicios, series, RPE, tiempos y técnica. También te envío los videos de Juliana directamente.
            </p>
            <div
              style={{ ...s.dropZone, ...(dragOver ? s.dropZoneActive : {}), ...(procesandoPDF ? s.dropZoneLoading : {}) }}
              onDragOver={(e) => { e.preventDefault(); setDragOver(true); }}
              onDragLeave={() => setDragOver(false)}
              onDrop={handleDrop}
              onClick={() => !procesandoPDF && fileRef.current?.click()}
            >
              <input ref={fileRef} type="file" accept="application/pdf" style={{ display: "none" }} onChange={handleFileInput} />
              {procesandoPDF ? (
                <div style={s.loadingState}>
                  <div style={s.spinner} />
                  <p style={s.loadingText}>Leyendo tu planificación...</p>
                </div>
              ) : (
                <div style={s.dropContent}>
                  <div style={s.pdfIcon}>
                    <svg width="32" height="32" viewBox="0 0 24 24" fill="none" stroke="#6B7A3E" strokeWidth="1.5">
                      <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/>
                      <polyline points="14 2 14 8 20 8"/>
                      <line x1="12" y1="18" x2="12" y2="12"/>
                      <line x1="9" y1="15" x2="15" y2="15"/>
                    </svg>
                  </div>
                  <p style={s.dropMain}>Arrastrá tu PDF aquí</p>
                  <p style={s.dropSub}>o tocá para seleccionar el archivo</p>
                </div>
              )}
            </div>
            <div style={s.tagRow}>
              <span style={s.tag}>📹 Videos incluidos</span>
              <span style={s.tag}>Sin registro</span>
              <span style={s.tag}>Sin descargas</span>
            </div>
          </div>
        </div>
      )}

      {vista === "chat" && (
        <div style={s.chatWrapper}>
          <div style={s.mensajes} ref={chatRef}>
            {mensajes.map((m, i) => (
              <div key={i} style={{ ...s.row, ...(m.rol === "usuario" ? s.rowUser : s.rowBot) }}>
                {m.rol === "asistente" && (
                  <div style={s.avatar}>JS</div>
                )}
                <div style={{ ...s.bubble, ...(m.rol === "usuario" ? s.bubbleUser : s.bubbleBot) }}>
                  {m.rol === "asistente" ? renderTexto(m.texto) : m.texto}
                </div>
              </div>
            ))}
            {cargando && (
              <div style={{ ...s.row, ...s.rowBot }}>
                <div style={s.avatar}>JS</div>
                <div style={{ ...s.bubble, ...s.bubbleBot, display: "flex", gap: 5, alignItems: "center", padding: "14px 18px" }}>
                  {[0, 0.2, 0.4].map((d, i) => (
                    <span key={i} style={{ width: 7, height: 7, borderRadius: "50%", background: "#C4A882", display: "inline-block", animation: `pulse 1.2s infinite ${d}s` }} />
                  ))}
                </div>
              </div>
            )}
          </div>
          <div style={s.inputBar}>
            <input
              style={s.inputField}
              placeholder="Preguntá lo que quieras sobre tu plan..."
              value={input}
              onChange={e => setInput(e.target.value)}
              onKeyDown={e => e.key === "Enter" && enviarMensaje()}
              disabled={cargando}
            />
            <button
              style={{ ...s.sendBtn, opacity: input.trim() && !cargando ? 1 : 0.35 }}
              onClick={enviarMensaje}
              disabled={!input.trim() || cargando}
            >
              <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="white" strokeWidth="2.5">
                <line x1="22" y1="2" x2="11" y2="13"/>
                <polygon points="22 2 15 22 11 13 2 9 22 2"/>
              </svg>
            </button>
          </div>
        </div>
      )}

      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;600;700&display=swap');
        * { box-sizing: border-box; }
        body { margin: 0; }
        @keyframes spin { to { transform: rotate(360deg); } }
        @keyframes pulse { 0%,80%,100%{transform:scale(0.6);opacity:0.4} 40%{transform:scale(1);opacity:1} }
      `}</style>
    </div>
  );
}

const OLIVE = "#6B7A3E";
const CREAM = "#F5F2EB";
const DARK = "#1C1E14";
const MID = "#5A5C4F";
const ACCENT = "#C4A882";

const s = {
  app: { minHeight:"100vh", background:CREAM, fontFamily:"'DM Sans',sans-serif", display:"flex", flexDirection:"column", color:DARK },
  header: { background:OLIVE, padding:"0 20px", height:58, display:"flex", alignItems:"center", flexShrink:0 },
  headerInner: { display:"flex", alignItems:"center", gap:12, width:"100%", maxWidth:780, margin:"0 auto" },
  logoMark: { background:"rgba(255,255,255,0.2)", borderRadius:10, width:38, height:38, display:"flex", alignItems:"center", justifyContent:"center", color:"white", fontWeight:700, fontSize:13, flexShrink:0, letterSpacing:"-0.5px" },
  headerTitle: { color:"white", fontWeight:700, fontSize:15, lineHeight:1.2 },
  headerSub: { color:"rgba(255,255,255,0.6)", fontSize:11, marginTop:2 },
  btnReset: { marginLeft:"auto", background:"rgba(255,255,255,0.15)", color:"white", border:"1px solid rgba(255,255,255,0.25)", borderRadius:8, padding:"5px 12px", fontSize:12, cursor:"pointer", fontWeight:500, fontFamily:"inherit", flexShrink:0 },
  centrado: { flex:1, display:"flex", alignItems:"center", justifyContent:"center", padding:24 },
  uploadCard: { maxWidth:480, width:"100%", textAlign:"center" },
  eyebrow: { fontSize:12, fontWeight:600, letterSpacing:"0.12em", textTransform:"uppercase", color:OLIVE, margin:"0 0 10px" },
  headline: { fontSize:32, fontWeight:700, color:DARK, margin:"0 0 12px", letterSpacing:"-0.8px", lineHeight:1.1 },
  subtext: { fontSize:15, color:MID, lineHeight:1.65, margin:"0 0 28px" },
  dropZone: { border:`2px dashed ${ACCENT}`, borderRadius:16, padding:"40px 24px", cursor:"pointer", background:"white", transition:"all 0.2s", marginBottom:20 },
  dropZoneActive: { borderColor:OLIVE, background:"#F0F3E8" },
  dropZoneLoading: { cursor:"default", opacity:0.8 },
  dropContent: { display:"flex", flexDirection:"column", alignItems:"center", gap:8 },
  pdfIcon: { background:"#FBF6EE", borderRadius:12, width:64, height:64, display:"flex", alignItems:"center", justifyContent:"center", marginBottom:4 },
  dropMain: { fontSize:16, fontWeight:600, color:DARK, margin:0 },
  dropSub: { fontSize:13, color:MID, margin:0 },
  loadingState: { display:"flex", flexDirection:"column", alignItems:"center", gap:14 },
  spinner: { width:36, height:36, border:`3px solid #E8E4DA`, borderTop:`3px solid ${OLIVE}`, borderRadius:"50%", animation:"spin 0.8s linear infinite" },
  loadingText: { fontSize:14, color:MID, margin:0, fontWeight:500 },
  tagRow: { display:"flex", justifyContent:"center", gap:8, flexWrap:"wrap" },
  tag: { background:"white", border:"1px solid #E0DDD5", borderRadius:20, padding:"4px 12px", fontSize:12, color:MID, fontWeight:500 },
  chatWrapper: { flex:1, display:"flex", flexDirection:"column", maxWidth:780, width:"100%", margin:"0 auto", padding:"0 16px" },
  mensajes: { flex:1, overflowY:"auto", display:"flex", flexDirection:"column", gap:14, padding:"16px 0 8px" },
  row: { display:"flex", alignItems:"flex-end", gap:8 },
  rowUser: { flexDirection:"row-reverse" },
  rowBot: { flexDirection:"row" },
  avatar: { background:OLIVE, color:"white", fontWeight:700, fontSize:11, borderRadius:20, width:32, height:32, display:"flex", alignItems:"center", justifyContent:"center", flexShrink:0, letterSpacing:"-0.3px" },
  bubble: { maxWidth:"78%", padding:"11px 15px", borderRadius:16, fontSize:14, lineHeight:1.65, whiteSpace:"pre-wrap" },
  bubbleUser: { background:OLIVE, color:"white", borderBottomRightRadius:4 },
  bubbleBot: { background:"white", color:DARK, boxShadow:"0 1px 4px rgba(0,0,0,0.06)", borderBottomLeftRadius:4 },
  inputBar: { display:"flex", gap:10, padding:"10px 0 16px", background:CREAM, position:"sticky", bottom:0 },
  inputField: { flex:1, border:"1.5px solid #DDD9CE", borderRadius:12, padding:"12px 16px", fontSize:14, fontFamily:"inherit", color:DARK, outline:"none", background:"white" },
  sendBtn: { background:OLIVE, border:"none", borderRadius:12, width:46, height:46, display:"flex", alignItems:"center", justifyContent:"center", cursor:"pointer", flexShrink:0, transition:"opacity 0.2s" },
  link: { color:OLIVE, fontWeight:600, textDecoration:"underline", wordBreak:"break-all" },
};
