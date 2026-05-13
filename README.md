# CMS-Grader
## Statements and Testcases Downloader
1. **Login to Admin and go to the Tasks list (./tasks)**
2. **Open your Browser Console (F12)**
3. **Allow Pasting (Execute ```allow pasting```)**
4. **Execute [JSZip Library Loader](#jszip-library-loader)**
5. **Execute [Main Script](#main-script)**
### JSZip Library Loader
```javascript
var script = document.createElement('script');
script.src = "https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js";
document.head.appendChild(script);
console.log("✅ JSZip Library Loaded!"); ;
```
### Main Script
```javascript
(async () => {
    const CONFIG = {
        downloadStatement: true,
        downloadTestcases: true,

        taskDelay: 2500,

        statementName: "{task}_Statement.pdf", 
        
        inputTemplate: "input.*", 
        outputTemplate: "output.*",

        selectors: {
            taskLinks: 'table.bordered tbody tr td:nth-child(2) a',
            pdfLink: 'a[href*="/file/"][href$=".pdf"]',
            fileLink: 'a[onclick*="utils.show_file"]'
        }
    };

    if (typeof JSZip === "undefined") return console.error("❌ JSZip not found. Run the Library Loader first.");

    const taskLinks = Array.from(document.querySelectorAll(CONFIG.selectors.taskLinks));
    console.log(`🚀 Script started. Target: ${taskLinks.length} tasks.`);

    for (let i = 0; i < taskLinks.length; i++) {
        const taskUrl = taskLinks[i].href;
        const rawName = taskLinks[i].innerText.trim();
        const taskSlug = rawName.replace(/[^a-z0-9]/gi, '_');
        const zip = new JSZip();

        console.log(`[${i + 1}/${taskLinks.length}] 📂 Processing: ${rawName}`);

        try {
            const response = await fetch(taskUrl);
            const html = await response.text();
            const doc = new DOMParser().parseFromString(html, "text/html");

            if (CONFIG.downloadStatement) {
                const pdf = doc.querySelector(CONFIG.selectors.pdfLink);
                if (pdf) {
                    const res = await fetch(new URL(pdf.getAttribute('href'), taskUrl).href);
                    const blob = await res.blob();
                    const a = document.createElement('a');
                    a.href = URL.createObjectURL(blob);
                    a.download = CONFIG.statementName.replace('{task}', taskSlug);
                    a.click();
                    console.log("   ✅ Statement saved");
                }
            }

            if (CONFIG.downloadTestcases) {
                const links = Array.from(doc.querySelectorAll(CONFIG.selectors.fileLink));
                if (links.length > 0) {
                    for (const link of links) {
                        const match = link.getAttribute('onclick').match(/'([^']+)'\s*,\s*'([^']+)'/);
                        if (match) {
                            const rawFileName = match[1]; // e.g., input_01
                            const fileUrl = new URL(match[2], taskUrl).href;
                            const num = rawFileName.split('_')[1]; // Extracts '01'

                            const fRes = await fetch(fileUrl);
                            const fBlob = await fRes.blob();

                            // Apply Template
                            let finalName = rawFileName.startsWith('input') ? CONFIG.inputTemplate : CONFIG.outputTemplate;
                            finalName = finalName.replace('*', num);
                            
                            zip.file(finalName, fBlob);
                        }
                    }
                    const content = await zip.generateAsync({type: "blob"});
                    const zLink = document.createElement('a');
                    zLink.href = URL.createObjectURL(content);
                    zLink.download = `${taskSlug}_testcases.zip`;
                    zLink.click();
                    console.log("   ✅ ZIP generated");
                }
            }

            await new Promise(r => setTimeout(r, CONFIG.taskDelay));
        } catch (err) {
            console.error(`   ❌ Error at ${rawName}:`, err);
        }
    }
    console.log("%cDone", "color: #1FFF2A; font-weight: bold; font-size: 16px;");
})();
```
