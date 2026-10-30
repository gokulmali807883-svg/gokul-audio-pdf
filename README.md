<!DOCTYPE html>
<html lang="mr">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Gokul Audio PDF Reader</title>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js"></script>
  <style>
    body {
      background-color: #121212;
      color: #FFFFFF;
      font-family: system-ui, -apple-system, sans-serif;
      margin: 0;
      padding: 16px;
      display: flex;
      flex-direction: column;
      align-items: center;
    }
    h1 {
      font-size: 22px;
      margin-bottom: 16px;
      text-align: center;
    }
    .status-card {
      background: #1e1e1e;
      border: 2px solid #3b82f6;
      border-radius: 12px;
      padding: 16px;
      width: 100%;
      max-width: 480px;
      box-sizing: border-box;
      margin-bottom: 20px;
      font-size: 18px;
      text-align: center;
      line-height: 1.5;
    }
    .controls-container {
      display: flex;
      flex-direction: column;
      gap: 16px;
      width: 100%;
      max-width: 480px;
    }
    .file-input-wrapper {
      background: #1e293b;
      padding: 14px;
      border-radius: 12px;
      border: 2px dashed #64748b;
    }
    input[type="file"] {
      display: block;
      width: 100%;
      font-size: 16px;
      color: #ffffff;
    }
    button {
      background-color: #2563eb;
      color: #ffffff;
      border: none;
      border-radius: 12px;
      padding: 18px;
      font-size: 18px;
      font-weight: bold;
      text-align: center;
      cursor: pointer;
      box-shadow: 0 4px 6px rgba(0,0,0,0.4);
      min-height: 56px;
    }
    button:active {
      background-color: #1d4ed8;
    }
    .btn-secondary {
      background-color: #334155;
    }
    .btn-secondary:active {
      background-color: #1e293b;
    }
    .grid-2 {
      display: grid;
      grid-template-columns: 1fr 1fr;
      gap: 12px;
    }
  </style>
</head>
<body>

  <h1>Gokul Audio PDF Reader</h1>

  <div class="status-card" id="statusBox" aria-live="polite" aria-atomic="true">
    स्वागत आहे! कृपया खाली दिलेल्या 'Choose File' वर टॅप करून पीडीएफ निवडा.
  </div>

  <div class="controls-container">
    <div class="file-input-wrapper">
      <input type="file" id="pdfFileInput" accept="application/pdf" aria-label="पीडीएफ फाईल निवडा">
    </div>

    <button id="playPauseBtn" aria-label="वाचा किंवा थांबा">
      ▶️ वाचा (Play)
    </button>

    <div class="grid-2">
      <button id="prevBtn" class="btn-secondary" aria-label="मागील पान">
        ⏮️ मागील पान
      </button>
      <button id="nextBtn" class="btn-secondary" aria-label="पुढील पान">
        ⏭️ पुढील पान
      </button>
    </div>

    <button id="stopBtn" class="btn-secondary" aria-label="वाचन पूर्ण थांबवा">
      ⏹️ वाचन थांबवा (Stop)
    </button>
  </div>

  <script>
    pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js';

    let pdfDoc = null;
    let currentPage = 1;
    let totalPages = 0;
    let isSpeaking = false;
    let pageText = "";

    const statusBox = document.getElementById('statusBox');
    const fileInput = document.getElementById('pdfFileInput');
    const playPauseBtn = document.getElementById('playPauseBtn');
    const prevBtn = document.getElementById('prevBtn');
    const nextBtn = document.getElementById('nextBtn');
    const stopBtn = document.getElementById('stopBtn');

    function updateStatus(message) {
      statusBox.innerText = message;
    }

    fileInput.addEventListener('change', async function(e) {
      const file = e.target.files[0];
      if (!file) return;

      updateStatus("पीडीएफ लोड होत आहे, कृपया थांबा...");
      window.speechSynthesis.cancel();
      isSpeaking = false;
      playPauseBtn.innerText = "▶️ वाचा (Play)";

      try {
        const arrayBuffer = await file.arrayBuffer();
        pdfDoc = await pdfjsLib.getDocument({ data: arrayBuffer }).promise;
        totalPages = pdfDoc.numPages;
        currentPage = 1;
        updateStatus(`पीडीएफ यशस्वीरित्या लोड झाली. एकूण पाने: ${totalPages}. पान १ उघडले आहे.`);
        await extractAndPreparePage(currentPage, false);
      } catch (err) {
        updateStatus("पीडीएफ फाईल वाचताना त्रुटी आली. कृपया दुसरी फाईल निवडा.");
      }
    });

    async function extractAndPreparePage(pageNum, autoPlay = false) {
      if (!pdfDoc) return;
      window.speechSynthesis.cancel();
      isSpeaking = false;
      playPauseBtn.innerText = "▶️ वाचा (Play)";

      updateStatus(`पान ${pageNum} पैकी ${totalPages} लोड होत आहे...`);

      try {
        const page = await pdfDoc.getPage(pageNum);
        const textContent = await page.getTextContent();
        pageText = textContent.items.map(item => item.str).join(' ').trim();

        if (pageText.length === 0) {
          updateStatus(`पान ${pageNum} पैकी ${totalPages}. या पानावर कोणताही टेक्स्ट सापडला नाही.`);
          return;
        }

        updateStatus(`पान ${pageNum} पैकी ${totalPages} तयार आहे. वाचन सुरू करण्यासाठी प्ले दाबा.`);

        if (autoPlay) {
          startSpeaking(pageText);
        }
      } catch (e) {
        updateStatus("पानावरील मजकूर वाचताना त्रुटी आली.");
      }
    }

    function startSpeaking(text) {
      window.speechSynthesis.cancel();

      if (!text || text.trim().length === 0) {
        updateStatus("वाचण्यासाठी मजकूर उपलब्ध नाही.");
        return;
      }

      const utterance = new SpeechSynthesisUtterance(text);
      utterance.rate = 1.0;

      utterance.onstart = function() {
        isSpeaking = true;
        playPauseBtn.innerText = "⏸️ थांबा (Pause)";
        updateStatus(`पान ${currentPage} चे वाचन सुरू आहे...`);
      };

      utterance.onend = function() {
        isSpeaking = false;
        playPauseBtn.innerText = "▶️ वाचा (Play)";
        if (currentPage < totalPages) {
          currentPage++;
          extractAndPreparePage(currentPage, true);
        } else {
          updateStatus("पीडीएफचे संपूर्ण वाचन पूर्ण झाले आहे.");
        }
      };

      utterance.onerror = function() {
        isSpeaking = false;
        playPauseBtn.innerText = "▶️ वाचा (Play)";
        updateStatus("वाचनादरम्यान त्रुटी आली.");
      };

      window.speechSynthesis.speak(utterance);
    }

    playPauseBtn.addEventListener('click', function() {
      if (!pdfDoc) {
        updateStatus("कृपया आधी 'Choose File' वरून पीडीएफ फाईल निवडा.");
        return;
      }

      if (window.speechSynthesis.speaking) {
        if (window.speechSynthesis.paused) {
          window.speechSynthesis.resume();
          isSpeaking = true;
          playPauseBtn.innerText = "⏸️ थांबा (Pause)";
          updateStatus(`पान ${currentPage} चे वाचन पुन्हा सुरू झाले.`);
        } else {
          window.speechSynthesis.pause();
          isSpeaking = false;
          playPauseBtn.innerText = "▶️ पुन्हा सुरू करा (Resume)";
          updateStatus(`वाचन थांबवले आहे.`);
        }
      } else {
        startSpeaking(pageText);
      }
    });

    prevBtn.addEventListener('click', function() {
      if (pdfDoc && currentPage > 1) {
        currentPage--;
        extractAndPreparePage(currentPage, isSpeaking);
      } else if (!pdfDoc) {
        updateStatus("कृपया आधी पीडीएफ फाईल निवडा.");
      } else {
        updateStatus("हे पहिलेच पान आहे, मागे जाता येत नाही.");
      }
    });

    nextBtn.addEventListener('click', function() {
      if (pdfDoc && currentPage < totalPages) {
        currentPage++;
        extractAndPreparePage(currentPage, isSpeaking);
      } else if (!pdfDoc) {
        updateStatus("कृपया आधी पीडीएफ फाईल निवडा.");
      } else {
        updateStatus("हे शेवटचे पान आहे.");
      }
    });

    stopBtn.addEventListener('click', function() {
      if (window.speechSynthesis.speaking || isSpeaking) {
        window.speechSynthesis.cancel();
        isSpeaking = false;
        playPauseBtn.innerText = "▶️ वाचा (Play)";
        updateStatus(`वाचन पूर्ण थांबवले आहे. (पान ${currentPage})`);
      }
    });
  </script>
</body>
</html>

