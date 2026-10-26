<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>StockSense - Smart Expiry & Stock Manager</title>
    <!-- Tesseract for OCR Expiry Reading -->
    <script src="https://unpkg.com/tesseract.js@v4.0.2/dist/tesseract.min.js"></script>
    <!-- HTML5-QRCode for Real Barcode Scanning -->
    <script src="https://unpkg.com/html5-qrcode" type="text/javascript"></script>
    <style>
        :root {
            --primary: #4f46e5; --primary-hover: #4338ca; --bg-color: #0f172a;
            --card-bg: #1e293b; --text-main: #f8fafc; --text-muted: #94a3b8;
            --danger: #ef4444; --warning: #f59e0b; --success: #10b981; --border: #334155;
        }
        * { box-sizing: border-box; margin: 0; padding: 0; font-family: system-ui, -apple-system, sans-serif; }
        body { background-color: var(--bg-color); color: var(--text-main); display: flex; height: 100vh; overflow: hidden; }

        aside { width: 260px; background-color: var(--card-bg); border-right: 1px solid var(--border); display: flex; flex-direction: column; }
        .brand { padding: 24px; font-size: 1.25rem; font-weight: 700; color: var(--primary); border-bottom: 1px solid var(--border); display: flex; align-items: center; gap: 10px; }
        .nav-links { list-style: none; padding: 20px 12px; display: flex; flex-direction: column; gap: 8px; }
        .nav-links li button { width: 100%; padding: 12px 16px; background: transparent; border: none; color: var(--text-muted); text-align: left; font-size: 0.95rem; font-weight: 500; border-radius: 8px; cursor: pointer; transition: all 0.2s; }
        .nav-links li button.active, .nav-links li button:hover { background-color: var(--primary); color: white; }

        main { flex: 1; display: flex; flex-direction: column; overflow-y: auto; }
        header { padding: 20px 32px; background-color: var(--card-bg); border-bottom: 1px solid var(--border); display: flex; justify-content: space-between; align-items: center; }
        .content-body { padding: 32px; flex: 1; overflow-y: auto; }
        .view-section { display: none; }
        .view-section.active { display: block; }

        .alert-banner { background: rgba(239, 68, 68, 0.15); border: 1px solid var(--danger); padding: 16px 20px; border-radius: 10px; margin-bottom: 24px; display: flex; align-items: center; justify-content: space-between; }
        .alert-banner h4 { color: var(--danger); margin-bottom: 4px; }
        .alert-banner p { font-size: 0.9rem; color: var(--text-muted); }

        .metrics-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(220px, 1fr)); gap: 20px; margin-bottom: 32px; }
        .metric-card { background-color: var(--card-bg); border: 1px solid var(--border); padding: 20px; border-radius: 12px; display: flex; flex-direction: column; gap: 8px; }
        .metric-title { color: var(--text-muted); font-size: 0.9rem; }
        .metric-value { font-size: 2rem; font-weight: 700; }

        .workbench-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 24px; }
        @media(max-width: 900px) { .workbench-grid { grid-template-columns: 1fr; } body { flex-direction: column; height: auto; overflow: auto; } aside { width: 100%; } }

        .card { background-color: var(--card-bg); border: 1px solid var(--border); border-radius: 12px; padding: 24px; margin-bottom: 24px; }
        .card h3 { margin-bottom: 16px; font-size: 1.1rem; }

        .viewport-box { width: 100%; height: 260px; background-color: #000; border-radius: 8px; position: relative; overflow: hidden; display: flex; align-items: center; justify-content: center; border: 2px dashed var(--border); margin-bottom: 16px; }
        .viewport-box canvas { width: 100%; height: 100%; object-fit: contain; position: absolute; }
        .placeholder-text { color: var(--text-muted); z-index: 2; font-size: 0.9rem; text-align: center; padding: 0 20px; }

        #reader-box { width: 100%; height: 100%; display: none; }
        #reader-box video { object-fit: cover !important; }

        .btn { background-color: var(--primary); color: white; border: none; padding: 12px 20px; border-radius: 8px; font-weight: 600; cursor: pointer; transition: background 0.2s; width: 100%; margin-top: 8px; font-size: 1rem;}
        .btn:hover { background-color: var(--primary-hover); }
        .btn-secondary { background-color: transparent; border: 1px solid var(--border); color: var(--text-main); }
        .btn-secondary:hover { background-color: rgba(255,255,255,0.05); }

        .form-group { margin-bottom: 16px; }
        .form-group label { display: block; font-size: 0.85rem; color: var(--text-muted); margin-bottom: 6px; }
        .form-control { width: 100%; padding: 12px 14px; background-color: var(--bg-color); border: 1px solid var(--border); border-radius: 8px; color: var(--text-main); font-size: 0.95rem; }
        .form-control:focus { outline: none; border-color: var(--primary); }

        .inventory-table { width: 100%; border-collapse: collapse; background-color: var(--card-bg); border-radius: 12px; overflow: hidden; border: 1px solid var(--border); }
        .inventory-table th, .inventory-table td { padding: 14px 20px; text-align: left; border-bottom: 1px solid var(--border); font-size: 0.9rem; }
        .inventory-table th { background-color: rgba(255,255,255,0.02); color: var(--text-muted); font-weight: 600; }
        .badge { padding: 4px 10px; border-radius: 6px; font-size: 0.75rem; font-weight: 600; display: inline-block; }
        .badge-safe { background: rgba(16, 185, 129, 0.2); color: var(--success); }
        .badge-warning { background: rgba(245, 158, 11, 0.2); color: var(--warning); }
        .badge-danger { background: rgba(239, 68, 68, 0.2); color: var(--danger); }
        
        #ocr-status { font-size: 0.85rem; color: var(--warning); margin-top: 8px; text-align: center; font-weight: 500; }
    </style>
</head>
<body>

    <aside>
        <div class="brand">📦 StockSense</div>
        <ul class="nav-links">
            <li><button class="nav-btn active" onclick="switchView('dashboard')">Dashboard & Alerts</button></li>
            <li><button class="nav-btn" onclick="switchView('scanner')">Scan & Expiry</button></li>
            <li><button class="nav-btn" onclick="switchView('inventory')">Stock Ledger</button></li>
        </ul>
    </aside>

    <main>
        <header>
            <h1 id="view-title">Dashboard & Expiry Alerts</h1>
        </header>

        <div class="content-body">
            <!-- DASHBOARD -->
            <div id="view-dashboard" <div class="card" style="margin-top: 24px;">
    <h3>Database Management</h3>
    <p style="font-size: 0.85rem; color: var(--text-muted); margin-bottom: 12px;">Import a CSV file from your distributor to pre-load product recognition. Format: Barcode, Brand, Product Name.</p>
    <input type="file" id="csv-upload" accept=".csv" style="display:none;" onchange="importDistributorCSV(event)">
    <button class="btn btn-secondary" onclick="document.getElementById('csv-upload').click()">Import Master CSV Catalog</button>
    <div id="csv-status" style="font-size: 0.85rem; color: var(--success); margin-top: 8px;"></div>
</div>
 class="view-section active">
                <div id="urgent-alert-container"></div>
                <div class="metrics-grid">
                    <div class="metric-card"><span class="metric-title">Total Products</span><span class="metric-value" id="stat-total">0</span></div>
                    <div class="metric-card"><span class="metric-title">Safe Stock</span><span class="metric-value" style="color: var(--success);" id="stat-safe">0</span></div>
                    <div class="metric-card"><span class="metric-title">Expiring 1-2 Days</span><span class="metric-value" style="color: var(--warning);" id="stat-warning">0</span></div>
                    <div class="metric-card"><span class="metric-title">Expired Removals</span><span class="metric-value" style="color: var(--danger);" id="stat-danger">0</span></div>
                </div>
            </div>

            <!-- SCANNER VIEW -->
            <div id="view-scanner" class="view-section">
                <div class="workbench-grid">
                    
                    <!-- Real Barcode Scanner -->
                    <div class="card">
                        <h3>1. Live Barcode Scanner</h3>
                        <div class="viewport-box">
                            <div class="placeholder-text" id="barcode-placeholder">Camera off.<br>Click below to scan.</div>
                            <div id="reader-box"></div>
                        </div>
                        <button class="btn" id="barcode-btn" onclick="toggleBarcodeScanner()">Start Barcode Scanner</button>
                        
                        <div class="form-group" style="margin-top: 16px;">
                            <label>Product Barcode / SKU</label>
                            <input type="text" id="input-barcode" class="form-control" placeholder="Scanning..." onchange="lookupBarcodeAPI(this.value)">
                        </div>
                        <div class="form-group">
                            <label>Brand Name</label>
                            <input type="text" id="input-brand" class="form-control" placeholder="Brand fetches automatically">
                        </div>
                        <div class="form-group">
                            <label>Product Name</label>
                            <input type="text" id="input-name" class="form-control" placeholder="Product name">
                        </div>
                    </div>

                    <!-- Native Camera Expiry OCR -->
                    <div class="card">
                        <h3>2. Expiry OCR (High-Res)</h3>
                        <div class="viewport-box" id="expiry-viewport">
                            <div class="placeholder-text" id="expiry-placeholder">Take a clear photo of the expiry date using your phone's camera.</div>
                            <canvas id="snapshot-canvas" style="display:none;"></canvas>
                        </div>
                        
                        <input type="file" id="expiry-upload" accept="image/*" capture="environment" style="display: none;" onchange="processExpiryImage(event)">
                        <button class="btn btn-secondary" onclick="document.getElementById('expiry-upload').click()">Open Camera for Expiry Photo</button>
                        <div id="ocr-status"></div>

                        <div class="form-group" style="margin-top: 12px;">
                            <label>Verified Expiry Date</label>
                            <input type="date" id="input-expiry" class="form-control">
                        </div>

                        <div class="form-group">
                            <label>Stock Quantity</label>
                            <input type="number" id="input-qty" class="form-control" value="1" min="1">
                        </div>

                        <button class="btn" onclick="saveProductStock()">Save to Inventory</button>
                    </div>
                </div>
            </div>

            <!-- INVENTORY -->
            <div id="view-inventory" class="view-section">
                <div class="card">
                    <table class="inventory-table">
                        <thead>
                            <tr>
                                <th>Brand & Product</th>
                                <th>Barcode</th>
                                <th>Qty</th>
                                <th>Expiry Date</th>
                                <th>Status</th>
                                <th>Action</th>
                            </tr>
                        </thead>
                        <tbody id="inventory-table-body"></tbody>
                    </table>
                </div>
            </div>

        </div>
    </main>

    <script>
        let inventory = JSON.parse(localStorage.getItem('stocksense_inventory_v2')) || [];
        
        let html5QrCode = null;
        let barcodeScanning = false;

        window.addEventListener('DOMContentLoaded', refreshAllViews);

        function switchView(viewName) {
            document.querySelectorAll('.view-section').forEach(el => el.classList.remove('active'));
            document.querySelectorAll('.nav-btn').forEach(el => el.classList.remove('active'));
            document.getElementById(`view-${viewName}`).classList.add('active');
            event.target.classList.add('active');
            if(viewName !== 'scanner') stopBarcodeScanner();
        }

        // REAL BARCODE SCANNER INTEGRATION
        function toggleBarcodeScanner() {
            const readerBox = document.getElementById('reader-box');
            const placeholder = document.getElementById('barcode-placeholder');
            const btn = document.getElementById('barcode-btn');

            if (!barcodeScanning) {
                placeholder.style.display = 'none';
                readerBox.style.display = 'block';

                if (!html5QrCode) html5QrCode = new Html5Qrcode("reader-box");

                html5QrCode.start(
                    { facingMode: "environment" },
                    { fps: 10, qrbox: { width: 250, height: 150 } },
                    (decodedText) => {
                        // On successful scan
                        document.getElementById('input-barcode').value = decodedText;
                        lookupBarcodeAPI(decodedText);
                        stopBarcodeScanner(); // Stop camera after finding code
                    },
                    (err) => { /* ignore background scan errors */ }
                ).then(() => {
                    barcodeScanning = true;
                    btn.innerText = "Stop Scanner";
                    btn.style.backgroundColor = "var(--danger)";
                }).catch(err => {
                    alert("Camera error: " + err);
                    stopBarcodeScanner();
                });
            } else {
                stopBarcodeScanner();
            }
        }

        function stopBarcodeScanner() {
            if (html5QrCode && barcodeScanning) {
                html5QrCode.stop().then(() => {
                    barcodeScanning = false;
                    document.getElementById('reader-box').style.display = 'none';
                    document.getElementById('barcode-placeholder').style.display = 'block';
                    const btn = document.getElementById('barcode-btn');
                    btn.innerText = "Start Barcode Scanner";
                    btn.style.backgroundColor = "var(--primary)";
                });
            }
        }

        // AUTO BRAND LOOKUP
        async function lookupBarcodeAPI(barcode) {
            if(!barcode) return;
            try {
                const response = await fetch(`https://world.openfoodfacts.org/api/v0/product/${barcode}.json`);
                const data = await response.json();
                if (data.status === 1) {
                    document.getElementById('input-brand').value = data.product.brands || "Unknown Brand";
                    document.getElementById('input-name').value = data.product.product_name || "Unknown Product";
                } else {
                    document.getElementById('input-brand').value = "Custom Store Item";
                    document.getElementById('input-name').value = "Item SKU: " + barcode;
                }
            } catch (e) { console.error(e); }
        }

        // HIGH-RES OCR PROCESSING
        function processExpiryImage(event) {
            const file = event.target.files[0];
            if (!file) return;

            const statusEl = document.getElementById('ocr-status');
            const canvas = document.getElementById('snapshot-canvas');
            const ctx = canvas.getContext('2d');
            
            document.getElementById('expiry-placeholder').style.display = 'none';
            canvas.style.display = 'block';
            statusEl.innerText = "Loading high-res photo...";

            const img = new Image();
            img.onload = async () => {
                // Resize for canvas preview
                const ratio = Math.min(canvas.parentElement.clientWidth / img.width, canvas.parentElement.clientHeight / img.height);
                canvas.width = img.width * ratio;
                canvas.height = img.height * ratio;
                ctx.drawImage(img, 0, 0, canvas.width, canvas.height);

                statusEl.innerText = "Analyzing text (OCR)...";
                
                try {
                    const result = await Tesseract.recognize(img, 'eng', {
                        logger: m => { if(m.status === 'recognizing text') statusEl.innerText = `OCR Progress: ${Math.round(m.progress * 100)}%`; }
                    });
                    
                    const matchDate = result.data.text.match(/(\d{4}[-/.]\d{1,2}[-/.]\d{1,2})|(\d{1,2}[-/.]\d{1,2}[-/.]\d{2,4})/);
                    if(matchDate) {
                        try {
                            let parsed = new Date(matchDate[0]);
                            if(!isNaN(parsed)) {
                                document.getElementById('input-expiry').value = parsed.toISOString().split('T')[0];
                                statusEl.innerText = "Date successfully extracted!";
                                return;
                            }
                        } catch(e) {}
                    }
                    statusEl.innerText = "No clear date found. Please select manually.";
                } catch (err) {
                    statusEl.innerText = "OCR failed. Please select manually.";
                }
            };
            img.src = URL.createObjectURL(file);
        }

        // SAVE & RENDER LOGIC
        function saveProductStock() {
            const barcode = document.getElementById('input-barcode').value.trim();
            const brand = document.getElementById('input-brand').value.trim();
            const name = document.getElementById('input-name').value.trim();
            const expiry = document.getElementById('input-expiry').value;
            const qty = parseInt(document.getElementById('input-qty').value) || 1;

            if (!barcode || !name || !expiry) { alert('Missing Barcode, Name, or Expiry Date.'); return; }

            inventory.push({ id: barcode, brand: brand, name: name, qty: qty, expiry: expiry });
            localStorage.setItem('stocksense_inventory_v2', JSON.stringify(inventory));

            document.getElementById('input-barcode').value = '';
            document.getElementById('input-brand').value = '';
            document.getElementById('input-name').value = '';
            document.getElementById('input-expiry').value = '';
            document.getElementById('ocr-status').innerText = '';
            document.getElementById('snapshot-canvas').style.display = 'none';
            document.getElementById('expiry-placeholder').style.display = 'block';

            alert('Saved successfully!');
            refreshAllViews();
        }

        function deleteItem(index) {
            if(confirm("Remove this item?")) {
                inventory.splice(index, 1);
                localStorage.setItem('stocksense_inventory_v2', JSON.stringify(inventory));
                refreshAllViews();
            }
        }

        function calculateStatus(expiryDateStr) {
            const today = new Date(); today.setHours(0,0,0,0);
            const expDate = new Date(expiryDateStr); expDate.setHours(0,0,0,0);
            const diffDays = Math.round((expDate - today) / (1000 * 60 * 60 * 24));

            if (diffDays < 0) return { label: "Expired!", class: "badge-danger", sort: 1, urgent: true };
            if (diffDays <= 2) return { label: `Expiring in ${diffDays} day(s)`, class: "badge-danger", sort: 2, urgent: true };
            if (diffDays <= 7) return { label: `${diffDays} days left`, class: "badge-warning", sort: 3, urgent: false };
            return { label: `Safe (${diffDays}d)`, class: "badge-safe", sort: 4, urgent: false };
        }

        function refreshAllViews() {
            let safe = 0, warning = 0, expired = 0, urgentList = [];
            document.getElementById('inventory-table-body').innerHTML = '';

            inventory.forEach((item, index) => {
                const status = calculateStatus(item.expiry);
                if (status.sort === 4) safe++;
                else if (status.sort === 3) warning++;
                else { expired++; if(status.urgent) urgentList.push(`• ${item.brand} ${item.name} (Qty: ${item.qty})`); }

                document.getElementById('inventory-table-body').innerHTML += `
                    <tr>
                        <td><strong>${item.brand} ${item.name}</strong></td>
                        <td>${item.id}</td><td>${item.qty}</td><td>${item.expiry}</td>
                        <td><span class="badge ${status.class}">${status.label}</span></td>
                        <td><button class="btn btn-secondary" style="padding:4px; font-size:0.8rem;" onclick="deleteItem(${index})">Remove</button></td>
                    </tr>`;
            });

            document.getElementById('stat-total').innerText = inventory.length;
            document.getElementById('stat-safe').innerText = safe;
            document.getElementById('stat-warning').innerText = warning;
            document.getElementById('stat-danger').innerText = expired;

            const alertContainer = document.getElementById('urgent-alert-container');
            if (urgentList.length > 0) {
                alertContainer.innerHTML = `<div class="alert-banner"><div><h4>🚨 Immediate Action: Remove These</h4><p>${urgentList.join('<br>')}</p></div></div>`;
            } else { alertContainer.innerHTML = ''; }
        }
   // CSV Ingestion Engine
function importDistributorCSV(event) {
    const file = event.target.files[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = function(e) {
        const text = e.target.result;
        const rows = text.split('\n');
        let importCount = 0;

        // Assumes CSV format: Barcode, Brand, Product Name
        rows.forEach(row => {
            const cols = row.split(',');
            if (cols.length >= 3) {
                const barcode = cols[0].trim();
                const brand = cols[1].trim();
                const name = cols[2].trim();

                // Ignore headers or empty rows
                if (barcode && !isNaN(barcode)) {
                    customProductDB[barcode] = { brand: brand, name: name };
                    importCount++;
                }
            }
        });

        // Save the massive new catalog to local storage
        localStorage.setItem('stocksense_custom_db', JSON.stringify(customProductDB));
        document.getElementById('csv-status').innerText = `Successfully imported ${importCount} products into local memory.`;
    };
    
    reader.readAsText(file);
}

    
    </script>
</body>
</html>
