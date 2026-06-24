let cfgData = null;
let datData = null;
let gGraph = null;
let visibleChannels = [];

const FREQUENCY = 50.0;
const CYCLE_MS = 1000 / FREQUENCY;

document.getElementById('file-selector').addEventListener('change', handleFileSelect, false);

function handleFileSelect(event) {
    const files = event.target.files;
    let cfgFile = null;
    let datFile = null;

    for (let file of files) {
        if (file.name.toUpperCase().endsWith('.CFG')) cfgFile = file;
        if (file.name.toUpperCase().endsWith('.DAT')) datFile = file;
    }

    if (cfgFile && datFile) {
        parseComtrade(cfgFile, datFile);
    } else {
        alert("Error: Debes cargar el archivo .cfg y .dat del mismo registro de manera conjunta.");
    }
}

async function parseComtrade(cfgFile, datFile) {
    const cfgText = await cfgFile.text();
    cfgData = parseCFG(cfgText);
    
    document.getElementById('station-info').innerText = `${cfgData.stationName} - ${cfgData.deviceName} (${cfgData.analogCount} AI / ${cfgData.digitalCount} DI)`;

    if (cfgText.toUpperCase().includes('BINARY')) {
        const buffer = await datFile.arrayBuffer();
        datData = parseBinaryDAT(buffer, cfgData);
    } else {
        const datText = await datFile.text();
        datData = parseAsciiDAT(datText, cfgData);
    }

    buildChannelSelector();
    renderChart();
}

function parseCFG(text) {
    const lines = text.split(/\\r?\\n/).map(line => line.trim());
    const stationData = lines[0].split(',');
    const channelCounts = lines[1].split(',');
    
    const analogCount = parseInt(channelCounts[0]);
    const digitalCount = parseInt(channelCounts[1].replace('D',''));

    let channels = [];
    for (let i = 2; i < 2 + analogCount; i++) {
        const chInfo = lines[i].split(',');
        channels.push({
            index: i - 2,
            id: chInfo[0],
            name: chInfo[1],
            ph: chInfo[2],
            uu: chInfo[4],
            a: parseFloat(chInfo[5]),
            b: parseFloat(chInfo[6])
        });
    }
    return { 
        stationName: stationData[0], 
        deviceName: stationData[1], 
        analogCount, 
        digitalCount, 
        channels 
    };
}

function parseBinaryDAT(buffer, cfg) {
    const view = new DataView(buffer);
    let offset = 0;
    let dataPoints = [];

    const bytesPerAnalog = 2; // Int16 estándar COMTRADE-99
    const digitalWords = Math.ceil(cfg.digitalCount / 16);
    const bytesPerDigital = digitalWords * 2;
    const bytesPerSample = 4 + 4 + (cfg.analogCount * bytesPerAnalog) + bytesPerDigital;

    const totalSamples = Math.floor(buffer.byteLength / bytesPerSample);

    for (let s = 0; s < totalSamples; s++) {
        const sampleNum = view.getUint32(offset, true);
        const timestamp = view.getUint32(offset + 4, true); // Microsegundos
        
        let row = [timestamp / 1000]; // Eje X en ms
        let currentOffset = offset + 8;

        for (let i = 0; i < cfg.analogCount; i++) {
            const rawValue = view.getInt16(currentOffset, true);
            const realValue = (rawValue * cfg.channels[i].a) + cfg.channels[i].b;
            row.push(realValue);
            currentOffset += 2;
        }
        dataPoints.push(row);
        offset += bytesPerSample;
    }
    return dataPoints;
}

function parseAsciiDAT(text, cfg) {
    const lines = text.split(/\\r?\\n/);
    let dataPoints = [];
    lines.forEach(line => {
        if (!line) return;
        const tokens = line.split(',');
        if(tokens.length < 2) return;
        const timestamp = parseFloat(tokens[1]);
        let row = [timestamp / 1000];
        for (let i = 0; i < cfg.analogCount; i++) {
            const rawValue = parseFloat(tokens[2 + i]);
            const realValue = (rawValue * cfg.channels[i].a) + cfg.channels[i].b;
            row.push(realValue);
        }
        dataPoints.push(row);
    });
    return dataPoints;
}

function buildChannelSelector() {
    const container = document.getElementById('channel-selector-list');
    container.innerHTML = '';
    visibleChannels = [];

    cfgData.channels.forEach((ch, idx) => {
        // Por defecto activamos solo los primeros 6 canales para evitar saturación visual inicial
        const isChecked = idx < 6;
        visibleChannels.push(isChecked);

        const li = document.createElement('li');
        li.className = 'channel-item';
        li.innerHTML = `
            <label style="display:flex; align-items:center; gap:8px; width:100%; cursor:pointer;">
                <input type="checkbox" id="ch-chk-${idx}" ${isChecked ? 'checked' : ''} onchange="onChannelToggle(${idx})">
                <span title="${ch.name}">${ch.name} [${ch.uu}]</span>
            </label>
        `;
        container.appendChild(li);
    });
}

function onChannelToggle(idx) {
    const chk = document.getElementById(`ch-chk-${idx}`);
    visibleChannels[idx] = chk.checked;
    if (gGraph) {
        gGraph.setVisibility(idx, chk.checked);
    }
}

function toggleAllChannels(status) {
    cfgData.channels.forEach((ch, idx) => {
        const chk = document.getElementById(`ch-chk-${idx}`);
        if(chk) {
            chk.checked = status;
            visibleChannels[idx] = status;
        }
    });
    if (gGraph) {
        for(let i=0; i<cfgData.analogCount; i++) {
            gGraph.setVisibility(i, status);
        }
    }
}

function renderChart() {
    const container = document.getElementById('chart-div');
    const labels = ['Tiempo (ms)'].concat(cfgData.channels.map(ch => ch.name));
    
    gGraph = new Dygraph(container, datData, {
        labels: labels,
        legend: 'always',
        showRoller: false,
        ylabel: 'Magnitud',
        xlabel: 'Tiempo (ms)',
        animatedZooms: true,
        visibility: visibleChannels,
        highlightCallback: function(e, x, pts, row) {
            calculateMetricsForCursor(x, pts);
        }
    });
}

function calculateMetricsForCursor(currentX, points) {
    const body = document.getElementById('metrics-body');
    body.innerHTML = '';
    
    const startX = currentX - CYCLE_MS;
    // Filtrado de la ventana del ciclo de 20ms
    const cycleSamples = datData.filter(row => row[0] >= startX && row[0] <= currentX);
    const n = cycleSamples.length;

    points.forEach((pt) => {
        // Dygraphs nos entrega pt.name. Buscamos correspondencia en metadatos del CFG
        const chCfg = cfgData.channels.find(c => c.name === pt.name);
        if (!chCfg) return;

        const channelColIndex = chCfg.index + 1; 
        const instVal = pt.yval;

        let maxPeak = 0;
        let sumSquares = 0;

        if (n > 0) {
            for (let s = 0; s < n; s++) {
                const val = cycleSamples[s][channelColIndex];
                if (Math.abs(val) > maxPeak) maxPeak = Math.abs(val);
                sumSquares += val * val;
            }
        }

        const rmsVal = n > 0 ? Math.sqrt(sumSquares / n) : 0;
        const unit = chCfg.uu || '';

        const rowHTML = `
            <tr>
                <td><strong>${pt.name}</strong></td>
                <td class="numeric" style="color:#aaa;">${instVal.toFixed(2)} ${unit}</td>
                <td class="numeric" style="color:#ffb400;">${maxPeak.toFixed(2)} ${unit}</td>
                <td class="numeric" style="color:#00ff88;">${rmsVal.toFixed(2)} ${unit}</td>
            </tr>
        `;
        body.insertAdjacentHTML('beforeend', rowHTML);
    });
}
