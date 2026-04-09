import { useState, useCallback, useRef, useEffect } from "react";

const PDFJS_CDN = "https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js";
const PDFJS_WORKER = "https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js";

function loadPdfJs() {
  return new Promise((resolve, reject) => {
    if (window.pdfjsLib) return resolve(window.pdfjsLib);
    const s = document.createElement("script");
    s.src = PDFJS_CDN;
    s.onload = () => {
      window.pdfjsLib.GlobalWorkerOptions.workerSrc = PDFJS_WORKER;
      resolve(window.pdfjsLib);
    };
    s.onerror = reject;
    document.head.appendChild(s);
  });
}

function analyzeItems(items) {
  const lines = [];
  let currentLine = [];
  let lastY = null;

  for (const item of items) {
    if (item.str.trim() === "" && currentLine.length === 0) continue;
    const y = Math.round(item.transform[5]);
    if (lastY !== null && Math.abs(y - lastY) > 5 && currentLine.length > 0) {
      lines.push({ items: [...currentLine], y: lastY });
      currentLine = [];
    }
    currentLine.push(item);
    lastY = y;
  }
  if (currentLine.length > 0) lines.push({ items: [...currentLine], y: lastY });
  return lines;
}

function getFontSize(item) {
  const m = item.transform;
  return Math.round(Math.sqrt(m[0] * m[0] + m[1] * m[1]) * 10) / 10;
}

function detectHeadingLevel(fontSize, avgSize) {
  if (fontSize >= avgSize * 1.8) return 1;
  if (fontSize >= avgSize * 1.5) return 2;
  if (fontSize >= avgSize * 1.25) return 3;
  return 0;
}

function lineToText(line) {
  return line.items.map(i => i.str).join("").trim();
}

function isBold(item) {
  const name = (item.fontName || "").toLowerCase();
  return name.includes("bold") || name.includes("black") || name.includes("heavy");
}

function isItalic(item) {
  const name = (item.fontName || "").toLowerCase();
  return name.includes("italic") || name.includes("oblique");
}

function formatLineText(line) {
  let parts = [];
  for (const item of line.items) {
    let t = item.str;
    if (!t) continue;
    if (isBold(item) && isItalic(item)) t = `***${t}***`;
    else if (isBold(item)) t = `**${t}**`;
    else if (isItalic(item)) t = `*${t}*`;
    parts.push(t);
  }
  let text = parts.join("").trim();
  text = text.replace(/\*{2,3}\s*\*{2,3}/g, " ");
  return text;
}

function isBulletLine(text) {
  return /^[\u2022\u2023\u25E6\u2043\u25AA•●○■◦–—-]\s/.test(text) || /^[a-z]\)\s/i.test(text);
}

function isNumberedLine(text) {
  return /^\d+[\.\)]\s/.test(text);
}

function convertToMarkdown(pages) {
  let allLines = [];
  let fontSizes = [];

  for (const page of pages) {
    const lines = analyzeItems(page);
    for (const line of lines) {
      for (const item of line.items) {
        if (item.str.trim()) fontSizes.push(getFontSize(item));
      }
      allLines.push(line);
    }
    allLines.push(null);
  }

  const avgSize = fontSizes.length > 0
    ? fontSizes.sort((a, b) => a - b)[Math.floor(fontSizes.length / 2)]
    : 12;

  let md = [];
  let inTable = false;
  let tableRows = [];

  const flushTable = () => {
    if (tableRows.length === 0) return;
    let colCount = Math.max(...tableRows.map(r => r.length));
    for (let row of tableRows) {
      while (row.length < colCount) row.push("");
      md.push("| " + row.join(" | ") + " |");
      if (tableRows.indexOf(row) === 0) {
        md.push("| " + row.map(() => "---").join(" | ") + " |");
      }
    }
    tableRows = [];
    inTable = false;
    md.push("");
  };

  for (const line of allLines) {
    if (line === null) {
      flushTable();
      md.push("\n---\n");
      continue;
    }

    const text = lineToText(line);
    if (!text) {
      flushTable();
      md.push("");
      continue;
    }

    const tabCount = line.items.filter(i => i.str.includes("\t")).length;
    const hasMultipleSpaces = line.items.some(i => /\s{4,}/.test(i.str));

    if (tabCount > 0 || hasMultipleSpaces) {
      const cells = text.split(/\t|\s{4,}/).map(c => c.trim()).filter(Boolean);
      if (cells.length >= 2) {
        tableRows.push(cells);
        inTable = true;
        continue;
      }
    }

    flushTable();

    const mainFontSize = line.items.length > 0 ? getFontSize(line.items[0]) : avgSize;
    const headingLevel = detectHeadingLevel(mainFontSize, avgSize);

    if (headingLevel > 0) {
      const prefix = "#".repeat(headingLevel);
      md.push(`${prefix} ${text}`);
      md.push("");
    } else if (isBulletLine(text)) {
      md.push(`- ${text.replace(/^[\u2022\u2023\u25E6\u2043\u25AA•●○■◦–—-]\s*/, "").replace(/^[a-z]\)\s*/i, "")}`);
    } else if (isNumberedLine(text)) {
      md.push(text);
    } else {
      md.push(formatLineText(line));
    }
  }

  flushTable();

  let result = md.join("\n");
  result = result.replace(/\n{4,}/g, "\n\n\n");
  result = result.replace(/(\n---\n)+$/g, "");
  return result.trim();
}

export default function PDFToMarkdown() {
  const [state, setState] = useState("idle");
  const [markdown, setMarkdown] = useState("");
  const [fileName, setFileName] = useState("");
  const [progress, setProgress] = useState(0);
  const [dragOver, setDragOver] = useState(false);
  const [copied, setCopied] = useState(false);
  const [error, setError] = useState("");
  const fileRef = useRef(null);

  const processFile = useCallback(async (file) => {
    if (!file || file.type !== "application/pdf") {
      setError("Veuillez sélectionner un fichier PDF valide.");
      return;
    }
    setError("");
    setState("loading");
    setFileName(file.name);
    setProgress(5);

    try {
      const pdfjsLib = await loadPdfJs();
      setProgress(10);

      const buf = await file.arrayBuffer();
      const pdf = await pdfjsLib.getDocument({ data: buf }).promise;
      const totalPages = pdf.numPages;
      let pages = [];

      for (let i = 1; i <= totalPages; i++) {
        const page = await pdf.getPage(i);
        const content = await page.getTextContent();
        pages.push(content.items);
        setProgress(10 + Math.round((i / totalPages) * 70));
      }

      setProgress(85);
      const md = convertToMarkdown(pages);
      setProgress(100);
      setMarkdown(md);
      setState("done");
    } catch (e) {
      console.error(e);
      setError("Erreur lors de la conversion : " + e.message);
      setState("idle");
    }
  }, []);

  const onDrop = useCallback((e) => {
    e.preventDefault();
    setDragOver(false);
    const file = e.dataTransfer.files[0];
    processFile(file);
  }, [processFile]);

  const download = () => {
    try {
      const blob = new Blob([markdown], { type: "text/plain;charset=utf-8" });
      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.href = url;
      a.download = fileName.replace(/\.pdf$/i, "") + ".md";
      a.style.display = "none";
      document.body.appendChild(a);
      a.click();
      setTimeout(() => {
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
      }, 100);
    } catch (e) {
      const dataUri = "data:text/plain;charset=utf-8," + encodeURIComponent(markdown);
      window.open(dataUri, "_blank");
    }
  };

  const reset = () => {
    setState("idle");
    setMarkdown("");
    setFileName("");
    setProgress(0);
    setError("");
  };

  return (
    <div style={{
      minHeight: "100vh",
      background: "linear-gradient(135deg, #0f0c29, #302b63, #24243e)",
      padding: "40px 20px",
      fontFamily: "'Segoe UI', system-ui, -apple-system, sans-serif",
      color: "#e0e0e0"
    }}>
      <div style={{ maxWidth: 720, margin: "0 auto" }}>
        <div style={{ textAlign: "center", marginBottom: 40 }}>
          <h1 style={{
            fontSize: 32,
            fontWeight: 700,
            background: "linear-gradient(90deg, #a78bfa, #818cf8, #6366f1)",
            WebkitBackgroundClip: "text",
            WebkitTextFillColor: "transparent",
            margin: 0
          }}>PDF → Markdown</h1>
          <p style={{ color: "#9ca3af", marginTop: 8, fontSize: 14 }}>
            Convertissez vos PDF en Markdown fidèle, directement dans votre navigateur.
          </p>
        </div>

        {state === "idle" && (
          <>
            <div
              onDragOver={(e) => { e.preventDefault(); setDragOver(true); }}
              onDragLeave={() => setDragOver(false)}
              onDrop={onDrop}
              onClick={() => fileRef.current?.click()}
              style={{
                border: `2px dashed ${dragOver ? "#818cf8" : "#4b5563"}`,
                borderRadius: 16,
                padding: "60px 30px",
                textAlign: "center",
                cursor: "pointer",
                background: dragOver
                  ? "rgba(129,140,248,0.08)"
                  : "rgba(255,255,255,0.03)",
                backdropFilter: "blur(10px)",
                transition: "all 0.3s ease"
              }}
            >
              <div style={{ fontSize: 48, marginBottom: 16 }}>📄</div>
              <p style={{ fontSize: 18, fontWeight: 600, color: "#c4b5fd" }}>
                Glissez-déposez votre PDF ici
              </p>
              <p style={{ fontSize: 14, color: "#6b7280", marginTop: 8 }}>
                ou cliquez pour sélectionner un fichier
              </p>
              <input
                ref={fileRef}
                type="file"
                accept=".pdf,application/pdf"
                style={{ display: "none" }}
                onChange={(e) => processFile(e.target.files[0])}
              />
            </div>
            {error && (
              <p style={{
                color: "#f87171",
                textAlign: "center",
                marginTop: 16,
                fontSize: 14,
                background: "rgba(248,113,113,0.1)",
                padding: "10px 16px",
                borderRadius: 8
              }}>{error}</p>
            )}
          </>
        )}

        {state === "loading" && (
          <div style={{
            background: "rgba(255,255,255,0.05)",
            backdropFilter: "blur(10px)",
            borderRadius: 16,
            padding: "40px 30px",
            textAlign: "center"
          }}>
            <p style={{ fontSize: 16, color: "#c4b5fd", marginBottom: 20 }}>
              Conversion de <strong>{fileName}</strong>…
            </p>
            <div style={{
              width: "100%",
              height: 8,
              background: "rgba(255,255,255,0.1)",
              borderRadius: 4,
              overflow: "hidden"
            }}>
              <div style={{
                width: `${progress}%`,
                height: "100%",
                background: "linear-gradient(90deg, #a78bfa, #6366f1)",
                borderRadius: 4,
                transition: "width 0.3s ease"
              }} />
            </div>
            <p style={{ fontSize: 13, color: "#6b7280", marginTop: 12 }}>{progress}%</p>
          </div>
        )}

        {state === "done" && (
          <div>
            <div style={{
              display: "flex",
              gap: 12,
              marginBottom: 20,
              flexWrap: "wrap"
            }}>
              <button onClick={download} style={{
                flex: 1,
                minWidth: 140,
                padding: "12px 24px",
                background: "linear-gradient(135deg, #7c3aed, #6366f1)",
                color: "#fff",
                border: "none",
                borderRadius: 10,
                fontSize: 15,
                fontWeight: 600,
                cursor: "pointer",
                transition: "transform 0.2s, box-shadow 0.2s",
                boxShadow: "0 4px 15px rgba(99,102,241,0.3)"
              }}
                onMouseEnter={e => { e.target.style.transform = "translateY(-2px)"; e.target.style.boxShadow = "0 6px 20px rgba(99,102,241,0.4)"; }}
                onMouseLeave={e => { e.target.style.transform = "translateY(0)"; e.target.style.boxShadow = "0 4px 15px rgba(99,102,241,0.3)"; }}
              >
                ⬇️ Télécharger .md
              </button>
              <button onClick={() => {
                navigator.clipboard.writeText(markdown).then(() => {
                  setCopied(true);
                  setTimeout(() => setCopied(false), 2000);
                }).catch(() => {
                  const ta = document.createElement("textarea");
                  ta.value = markdown;
                  document.body.appendChild(ta);
                  ta.select();
                  document.execCommand("copy");
                  document.body.removeChild(ta);
                  setCopied(true);
                  setTimeout(() => setCopied(false), 2000);
                });
              }} style={{
                flex: 1,
                minWidth: 140,
                padding: "12px 24px",
                background: copied
                  ? "linear-gradient(135deg, #059669, #10b981)"
                  : "linear-gradient(135deg, #4f46e5, #7c3aed)",
                color: "#fff",
                border: "none",
                borderRadius: 10,
                fontSize: 15,
                fontWeight: 600,
                cursor: "pointer",
                transition: "all 0.3s ease",
                boxShadow: copied
                  ? "0 4px 15px rgba(16,185,129,0.3)"
                  : "0 4px 15px rgba(79,70,229,0.3)"
              }}
                onMouseEnter={e => { if (!copied) { e.target.style.transform = "translateY(-2px)"; }}}
                onMouseLeave={e => { e.target.style.transform = "translateY(0)"; }}
              >
                {copied ? "✅ Copié !" : "📋 Copier le MD"}
              </button>
              <button onClick={reset} style={{
                flex: 1,
                minWidth: 140,
                padding: "12px 24px",
                background: "rgba(255,255,255,0.06)",
                color: "#c4b5fd",
                border: "1px solid rgba(139,92,246,0.3)",
                borderRadius: 10,
                fontSize: 15,
                fontWeight: 600,
                cursor: "pointer",
                transition: "background 0.2s"
              }}
                onMouseEnter={e => e.target.style.background = "rgba(255,255,255,0.1)"}
                onMouseLeave={e => e.target.style.background = "rgba(255,255,255,0.06)"}
              >
                🔄 Nouveau fichier
              </button>
            </div>

            <div style={{
              background: "rgba(0,0,0,0.3)",
              backdropFilter: "blur(10px)",
              borderRadius: 12,
              border: "1px solid rgba(139,92,246,0.2)",
              overflow: "hidden"
            }}>
              <div style={{
                padding: "10px 16px",
                background: "rgba(139,92,246,0.1)",
                borderBottom: "1px solid rgba(139,92,246,0.2)",
                display: "flex",
                justifyContent: "space-between",
                alignItems: "center"
              }}>
                <span style={{ fontSize: 13, color: "#a78bfa", fontWeight: 600 }}>
                  Aperçu Markdown
                </span>
                <span style={{ fontSize: 12, color: "#6b7280" }}>
                  {markdown.length} caractères
                </span>
              </div>
              <pre style={{
                padding: 20,
                margin: 0,
                fontSize: 13,
                lineHeight: 1.7,
                color: "#d1d5db",
                overflowX: "auto",
                maxHeight: 500,
                whiteSpace: "pre-wrap",
                wordBreak: "break-word",
                fontFamily: "'Fira Code', 'Cascadia Code', 'Consolas', monospace"
              }}>
                {markdown}
              </pre>
            </div>
          </div>
        )}
      </div>
    </div>
  );
}
