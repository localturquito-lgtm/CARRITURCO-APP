<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Carriturco - Control de Stock y Ventas</title>
    <style>
        body { font-family: Arial, sans-serif; background-color: #f4f4f9; margin: 0; padding: 15px; color: #333; }
        h1, h2, h3 { text-align: center; color: #d9534f; }
        .card { background: white; padding: 15px; margin-bottom: 15px; border-radius: 8px; box-shadow: 0 2px 5px rgba(0,0,0,0.1); }
        label { font-weight: bold; display: block; margin-top: 10px; }
        input, select, button { width: 100%; padding: 10px; margin-top: 5px; box-sizing: border-box; border: 1px solid #ccc; border-radius: 5px; font-size: 16px; }
        button { background-color: #28a745; color: white; border: none; font-weight: bold; margin-top: 15px; cursor: pointer; }
        button:hover { background-color: #218838; }
        .btn-cierre { background-color: #dc3545; }
        .btn-cierre:hover { background-color: #c82333; }
        .btn-eliminar { background-color: #ff4d4d; color: white; padding: 5px 10px; width: auto; font-size: 12px; margin-top: 0; }
        table { width: 100%; border-collapse: collapse; margin-top: 10px; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: center; }
        th { background-color: #f2f2f2; }
        .stock-item { display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px; gap: 5px; }
        .stock-item input, .stock-item select { width: 45%; }
        .alerta-stock { background-color: #ffcccc; color: #990000; padding: 10px; border-radius: 5px; margin-bottom: 10px; font-weight: bold; display: none; }
        .sync-box { background-color: #e3f2fd; padding: 10px; border-radius: 5px; margin-bottom: 15px; font-size: 13px; }
    </style>
</head>
<body>

    <h1>🛒 Carriturco</h1>

    <div class="sync-box">
        <strong>🌐 Sincronización en la nube:</strong><br>
        Introduce la clave del servidor para compartir los datos entre varios teléfonos:
        <input type="text" id="binKey" placeholder="Ingresa tu ID de Base de Datos" onchange="guardarConfigSync()">
    </div>

    <!-- ALERTAS DE STOCK BANJO -->
    <div id="alertaStock" class="alerta-stock"></div>

    <!-- SELECCIÓN DE FECHA Y VENTA DEL DÍA -->
    <div class="card">
        <h2>📅 Registro Diario</h2>
        <label for="fecha">Fecha:</label>
        <input type="date" id="fecha">

        <label for="ventaDia">Venta total del día ($):</label>
        <input type="number" id="ventaDia" placeholder="0.00">
        <button onclick="guardarVentaDia()">Guardar Venta del Día</button>
    </div>

    <!-- CONTROL DE STOCK -->
    <div class="card">
        <h2>📦 Inventario / Stock</h2>
        <div id="listaStock"></div>

        <h3>➕ Agregar nuevo producto / insumo</h3>
        <input type="text" id="nuevoProdNombre" placeholder="Nombre del producto">
        <select id="nuevoProdTipo">
            <option value="UN">Medida: UN (Unidad)</option>
            <option value="KG">Medida: KG (Kilos)</option>
            <option value="ESTADO">Estado (SI / NO / COMPRAR)</option>
        </select>
        <button onclick="agregarNuevoProducto()">Agregar a la lista</button>

        <button onclick="guardarStock()" style="margin-top: 20px;">Guardar Cambios de Stock</button>
    </div>

    <!-- REGISTRO DE GASTOS -->
    <div class="card">
        <h2>💸 Gastos Diarios</h2>
        <label for="gastoDetalle">Detalle del Gasto:</label>
        <input type="text" id="gastoDetalle" placeholder="Ej: Pan, Mercadería, Gas">
        
        <label for="gastoMonto">Monto ($):</label>
        <input type="number" id="gastoMonto" placeholder="0.00">
        
        <button onclick="agregarGasto()">Agregar Gasto</button>

        <h3>Lista de Gastos Registrados</h3>
        <table>
            <thead>
                <tr>
                    <th>Fecha</th>
                    <th>Detalle</th>
                    <th>Monto ($)</th>
                </tr>
            </thead>
            <tbody id="tablaGastos"></tbody>
        </table>
    </div>

    <!-- CIERRE DE SEMANA Y RESUMEN -->
    <div class="card">
        <h2>📊 Cierre de Semana</h2>
        <p><strong>Total Ventas Semana:</strong> $<span id="totalVentasSemana">0</span></p>
        <p><strong>Total Gastos Semana:</strong> $<span id="totalGastosSemana">0</span></p>
        <p><strong>Balance / Ganancia Neta:</strong> $<span id="balanceSemana">0</span></p>
        <button class="btn-cierre" onclick="cierreDeSemana()">Realizar Cierre de Semana</button>
    </div>

    <!-- HISTORIAL DE CIERRES -->
    <div class="card">
        <h2>📜 Historial de Cierres Semanales</h2>
        <table>
            <thead>
                <tr>
                    <th>Fecha Cierre</th>
                    <th>Ventas Total</th>
                    <th>Gastos Total</th>
                    <th>Ganancia</th>
                </tr>
            </thead>
            <tbody id="tablaHistorial"></tbody>
        </table>
    </div>

    <script>
        document.getElementById('fecha').valueAsDate = new Date();

        const productosIniciales = [
            { id: 'pan_pancho', nombre: 'Pan p/pancho', tipo: 'UN', min: 10 },
            { id: 'pan_hambur', nombre: 'Pan p/hambur', tipo: 'UN', min: 10 },
            { id: 'paty', nombre: 'Paty', tipo: 'UN', min: 10 },
            { id: 'salchichas', nombre: 'Salchichas', tipo: 'UN', min: 10 },
            { id: 'cheddar_fetas', nombre: 'Cheddar (fetas)', tipo: 'UN', min: 5 },
            { id: 'papas_baston', nombre: 'Papas Bastón', tipo: 'KG', min: 2 },
            { id: 'papas_pay', nombre: 'Papas Pay', tipo: 'UN', min: 2 },
            { id: 'mayonesa', nombre: 'Mayonesa', tipo: 'ESTADO' },
            { id: 'ketchup', nombre: 'Ketchup', tipo: 'ESTADO' },
            { id: 'savora', nombre: 'Savora', tipo: 'ESTADO' },
            { id: 'cheddar', nombre: 'Cheddar', tipo: 'ESTADO' }
        ];

        function obtenerDatos() {
            return JSON.parse(localStorage.getItem('carriturco_db')) || { 
                ventas: [], 
                gastos: [], 
                stock: {}, 
                productos: productosIniciales,
                historial: [] 
            };
        }

        function guardarDatos(data) {
            localStorage.setItem('carriturco_db', JSON.stringify(data));
            actualizarResumen();
            verificarAlertas();
            sincronizarConNube(data);
        }

        function renderStockUI() {
            let db = obtenerDatos();
            let container = document.getElementById('listaStock');
            container.innerHTML = '';

            db.productos.forEach(p => {
                let val = db.stock[p.id] || (p.tipo === 'ESTADO' ? 'SI' : 0);
                let html = `<div class="stock-item">
                    <label>${p.nombre} (${p.tipo}):</label>`;

                if(p.tipo === 'ESTADO') {
                    html += `<select id="stk_${p.id}">
                        <option value="SI" ${val==='SI'?'selected':''}>SI</option>
                        <option value="NO" ${val==='NO'?'selected':''}>NO</option>
                        <option value="COMPRAR" ${val==='COMPRAR'?'selected':''}>COMPRAR</option>
                    </select>`;
                } else {
                    html += `<input type="number" step="0.1" id="stk_${p.id}" value="${val}">`;
                }

                html += `<button class="btn-eliminar" onclick="eliminarProducto('${p.id}')">✕</button></div>`;
                container.innerHTML += html;
            });
        }

        function agregarNuevoProducto() {
            let nombre = document.getElementById('nuevoProdNombre').value;
            let tipo = document.getElementById('nuevoProdTipo').value;
            if(!nombre) return alert('Ingresa un nombre');

            let db = obtenerDatos();
            let id = nombre.toLowerCase().replace(/ /g, '_');
            db.productos.push({ id, nombre, tipo, min: 5 });
            guardarDatos(db);
            renderStockUI();
            document.getElementById('nuevoProdNombre').value = '';
        }

        function eliminarProducto(id) {
            let db = obtenerDatos();
            db.productos = db.productos.filter(p => p.id !== id);
            delete db.stock[id];
            guardarDatos(db);
            renderStockUI();
        }

        function guardarStock() {
            let db = obtenerDatos();
            db.productos.forEach(p => {
                let el = document.getElementById(`stk_${p.id}`);
                if(el) db.stock[p.id] = el.value;
            });
            guardarDatos(db);
            alert('Stock actualizado');
        }

        function guardarVentaDia() {
            const fecha = document.getElementById('fecha').value;
            const monto = parseFloat(document.getElementById('ventaDia').value) || 0;
            if(!fecha) return alert('Selecciona una fecha');

            let db = obtenerDatos();
            db.ventas.push({ fecha, monto });
            guardarDatos(db);
            alert('Venta guardada');
            document.getElementById('ventaDia').value = '';
        }

        function agregarGasto() {
            const fecha = document.getElementById('fecha').value;
            const detalle = document.getElementById('gastoDetalle').value;
            const monto = parseFloat(document.getElementById('gastoMonto').value) || 0;
            if(!fecha || !detalle || !monto) return alert('Completa todos los campos');

            let db = obtenerDatos();
            db.gastos.push({ fecha, detalle, monto });
            guardarDatos(db);

            document.getElementById('gastoDetalle').value = '';
            document.getElementById('gastoMonto').value = '';
            renderGastos();
        }

        function renderGastos() {
            let db = obtenerDatos();
            let tbody = document.getElementById('tablaGastos');
            tbody.innerHTML = '';
            db.gastos.forEach(g => {
                tbody.innerHTML += `<tr><td>${g.fecha}</td><td>${g.detalle}</td><td>$${g.monto}</td></tr>`;
            });
        }

        function renderHistorial() {
            let db = obtenerDatos();
            let tbody = document.getElementById('tablaHistorial');
            tbody.innerHTML = '';
            (db.historial || []).forEach(h => {
                tbody.innerHTML += `<tr><td>${h.fecha}</td><td>$${h.ventas}</td><td>$${h.gastos}</td><td>$${h.ganancia}</td></tr>`;
            });
        }

        function actualizarResumen() {
            let db = obtenerDatos();
            let totalVentas = db.ventas.reduce((acc, curr) => acc + curr.monto, 0);
            let totalGastos = db.gastos.reduce((acc, curr) => acc + curr.monto, 0);

            document.getElementById('totalVentasSemana').innerText = totalVentas;
            document.getElementById('totalGastosSemana').innerText = totalGastos;
            document.getElementById('balanceSemana').innerText = totalVentas - totalGastos;
        }

        function verificarAlertas() {
            let db = obtenerDatos();
            let alertas = [];

            db.productos.forEach(p => {
                let val = db.stock[p.id];
                if(p.tipo === 'ESTADO' && val === 'COMPRAR') {
                    alertas.push(`⚠️ Falta comprar: ${p.nombre}`);
                } else if(p.tipo !== 'ESTADO' && val !== undefined && parseFloat(val) <= (p.min || 5)) {
                    alertas.push(`⚠️ Stock bajo de ${p.nombre}: Quedan ${val} ${p.tipo}`);
                }
            });

            let box = document.getElementById('alertaStock');
            if(alertas.length > 0) {
                box.style.display = 'block';
                box.innerHTML = alertas.join('<br>');
            } else {
                box.style.display = 'none';
            }
        }

        function cierreDeSemana() {
            if(confirm('¿Realizar cierre de semana? Los totales actuales se guardarán en el Historial.')) {
                let db = obtenerDatos();
                let totalVentas = db.ventas.reduce((acc, curr) => acc + curr.monto, 0);
                let totalGastos = db.gastos.reduce((acc, curr) => acc + curr.monto, 0);

                if(!db.historial) db.historial = [];
                db.historial.push({
                    fecha: new Date().toLocaleDateString(),
                    ventas: totalVentas,
                    gastos: totalGastos,
                    ganancia: totalVentas - totalGastos
                });

                db.ventas = [];
                db.gastos = [];
                guardarDatos(db);
                renderGastos();
                renderHistorial();
                alert('Semana cerrada correctamente.');
            }
        }

        // SINCRONIZACIÓN NUBE CON JSONBIN
        function guardarConfigSync() {
            let key = document.getElementById('binKey').value;
            localStorage.setItem('carriturco_bin_id', key);
            cargarDesdeNube();
        }

        async function sincronizarConNube(data) {
            let binId = localStorage.getItem('carriturco_bin_id');
            if(!binId) return;
            try {
                await fetch(`https://api.jsonbin.io/v3/b/${binId}`, {
                    method: 'PUT',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(data)
                });
            } catch(e) { console.error(e); }
        }

        async function cargarDesdeNube() {
            let binId = localStorage.getItem('carriturco_bin_id');
            if(!binId) return;
            try {
                let res = await fetch(`https://api.jsonbin.io/v3/b/${binId}/latest`);
                let json = await res.json();
                if(json.record) {
                    localStorage.setItem('carriturco_db', JSON.stringify(json.record));
                    renderStockUI();
                    renderGastos();
                    renderHistorial();
                    actualizarResumen();
                    verificarAlertas();
                }
            } catch(e) { console.error(e); }
        }

        window.onload = function() {
            let binId = localStorage.getItem('carriturco_bin_id');
            if(binId) document.getElementById('binKey').value = binId;
            renderStockUI();
            renderGastos();
            renderHistorial();
            actualizarResumen();
            verificarAlertas();
            cargarDesdeNube();
        };
    </script>
</body>
</html>
