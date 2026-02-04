<!doctype html>
<html lang="es">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>AI Interviewer</title>
    <style>
      /* RESET / BOX-SIZING */
      *, *::before, *::after { box-sizing: border-box; }

      body {
        font-family: system-ui, -apple-system, "Segoe UI", Roboto, Arial, sans-serif;
        margin: 0;
        background: #0b1220;
        color: #e8eefc;
        -webkit-font-smoothing: antialiased;
        -moz-osx-font-smoothing: grayscale;
      }
      .wrap {
        max-width: 900px;
        margin: 0 auto;
        padding: 40px 20px;
      }
      .card {
        background: rgba(255, 255, 255, 0.06);
        border: 1px solid rgba(255, 255, 255, 0.12);
        border-radius: 16px;
        padding: 22px;
      }
      h1 { margin: 0 0 8px; font-size: 28px; }
      p { margin: 0 0 14px; color: rgba(232, 238, 252, 0.85); line-height: 1.4; }
      .steps { margin-top: 14px; padding-left: 18px; color: rgba(232, 238, 252, 0.85); }
      .badge {
        display: inline-block;
        font-size: 12px;
        padding: 6px 10px;
        border-radius: 999px;
        background: rgba(99, 102, 241, 0.18);
        border: 1px solid rgba(99, 102, 241, 0.35);
        margin-bottom: 12px;
      }
      .footer { margin-top: 18px; font-size: 12px; color: rgba(232, 238, 252, 0.6); }
    </style>

    <!-- jsPDF (CDN): considera agregar SRI crossorigin en producción -->
    <script src="https://cdn.jsdelivr.net/npm/jspdf@2.5.1/dist/jspdf.umd.min.js"></script>
  </head>

  <body>
    <div class="wrap">
      <div class="badge" aria-hidden="true">Gratis · Sin registro</div>

      <main class="card" role="main" aria-labelledby="main-heading">
        <h1 id="main-heading">Simulador de entrevista</h1>
        <p>
          Practica entrevistas con un entrevistador IA. Responde con naturalidad, como si fuera una entrevista real.
        </p>

        <p><strong>Cómo usarlo:</strong></p>
        <ol class="steps">
          <li>Abre el chat (abajo a la derecha).</li>
          <li>Escribe lo que te pide el bot (Nombre, Empresa, Puesto, Seniority).</li>
          <li>Contesta las preguntas.</li>
          <li>Al final podrás descargar tu informe en PDF.</li>
        </ol>

        <div class="footer">
          Tip: responde con ejemplos concretos (situación → acción → resultado).
        </div>
      </main>
    </div>

    <script>
      // Generar y descargar PDF (con manejo de errores y normalización de nombre)
      function sanitizeFileName(name = "candidate") {
        // Normaliza y elimina diacríticos, luego deja letras, números, guiones/guiones_bajos
        try {
          const normalized = name.normalize ? name.normalize('NFKD') : name;
          return normalized
            .replace(/[\u0300-\u036f]/g, '') // eliminar diacríticos
            .replace(/[^\w\-]+/g, '_')
            .replace(/^_+|_+$/g, '')
            .slice(0, 200) || 'candidate';
        } catch (e) {
          return 'candidate';
        }
      }

      function downloadPdf(payload) {
        try {
          const { jsPDF } = window.jspdf;
          const doc = new jsPDF({ unit: "pt", format: "a4" });

          const report_text = String(payload?.report_text ?? "").trim();
          const candidate_name = String(payload?.candidate_name ?? "");
          const company = String(payload?.company ?? "");
          const role = String(payload?.role ?? "");
          const seniority = String(payload?.seniority ?? "");

          const title = "HR Evaluation Report";
          const meta = [
            candidate_name ? `Candidate: ${candidate_name}` : "",
            company ? `Company: ${company}` : "",
            role ? `Role: ${role}` : "",
            seniority ? `Seniority: ${seniority}` : "",
          ].filter(Boolean).join(" | ");

          const marginX = 40;
          let y = 56;

          doc.setFont("helvetica", "bold");
          doc.setFontSize(16);
          doc.text(title, marginX, y);
          y += 18;

          doc.setFont("helvetica", "normal");
          doc.setFontSize(10);
          const metaLines = doc.splitTextToSize(meta || "", 515);
          if (metaLines.length) {
            doc.text(metaLines, marginX, y);
            y += metaLines.length * 12 + 14;
          } else {
            y += 8;
          }

          doc.setFontSize(11);

          const text = report_text || "No report text received.";
          const lines = doc.splitTextToSize(text, 515);

          const pageHeight = doc.internal.pageSize.getHeight();
          const bottom = pageHeight - 40;

          for (const line of lines) {
            if (y > bottom) {
              doc.addPage();
              y = 56;
            }
            doc.text(line, marginX, y);
            y += 14;
          }

          const safeName = sanitizeFileName(candidate_name);
          doc.save(`HR_Report_${safeName}.pdf`);
        } catch (err) {
          console.error("Error generating PDF:", err);
          alert("No se pudo generar el PDF. Revisa la consola para más detalles.");
        }
      }

      // Voiceflow effect extension (comprobaciones y logging)
      const downloadReportExtension = {
        name: "DOWNLOAD_REPORT_PDF",
        type: "effect",
        match: ({ trace }) => {
          // Ajusta esto según la estructura real del trace. Se recomienda loggear el trace en desarrollo.
          if (!trace) return false;
          // Ejemplos de comprobación: trace.type === 'DOWNLOAD_REPORT_PDF' || trace?.payload?.name === 'DOWNLOAD_REPORT_PDF'
          return trace?.type === "DOWNLOAD_REPORT_PDF" || trace?.payload?.name === "DOWNLOAD_REPORT_PDF";
        },
        effect: async ({ trace }) => {
          try {
            const payload = trace?.payload || {};
            console.log("DOWNLOAD_REPORT_PDF payload:", payload, "full trace:", trace);
            downloadPdf(payload);
          } catch (err) {
            console.error("Error in downloadReportExtension effect:", err);
          }
        },
      };

      // Cargar widget y registrar extensions
      (function (d, t) {
        const v = d.createElement(t);
        const s = d.getElementsByTagName(t)[0];

        v.onload = function () {
          try {
            if (!window.voiceflow || !window.voiceflow.chat || !window.voiceflow.chat.load) {
              console.warn("voiceflow.chat no está disponible tras la carga del script.");
              return;
            }

            window.voiceflow.chat.load({
              verify: { projectID: "6981de99392d1c3e0b9084bf" },
              url: "https://general-runtime.voiceflow.com",
              versionID: "production",
              voice: { url: "https://runtime-api.voiceflow.com" },

              assistant: {
                extensions: [downloadReportExtension],
              },
            });
          } catch (err) {
            console.error("Error inicializando Voiceflow widget:", err);
          }
        };

        v.src = "https://cdn.voiceflow.com/widget-next/bundle.mjs";
        v.type = "module"; // si el bundle es ESM. Si no, usar "text/javascript".
        v.defer = true;
        s.parentNode.insertBefore(v, s);
      })(document, "script");
    </script>
  </body>
</html>
