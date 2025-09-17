<script>
  import Papa from 'papaparse';
  import Plotly from 'plotly.js-dist-min';
  import { onMount } from 'svelte';

  let dataFile=  {
    parsedRows: [],
    columns:  []
  };

  let plotSelections = [{},{},{},{}];

  function normalizeRowKeys(row) {
    const normalized = {};
    for (const key in row) {
      if (!Object.prototype.hasOwnProperty.call(row, key)) continue;
      const nk = key.trim().toLowerCase().replace('.', '_').replace(' ', '_');
      normalized[nk] = row[key];
    }
    normalized.timestamp = Date.parse(normalized.timestamp);
    return normalized;
  }

  function handleFileChange(event) {
    const file = event.target.files && event.target.files[0];
    if (!file) return;
    Papa.parse(file, {
      header: true,
      skipEmptyLines: true,
      dynamicTyping: true,
      complete: (results) => {
        dataFile.parsedRows = results.data.map(normalizeRowKeys);
        dataFile.columns = Object.keys(dataFile.parsedRows[0]).filter(k => !["timestamp", "index", "gpu"].includes(k))
        console.log(dataFile)
      },
      error: (err) => {
        alert('CSV parse error: ' + err);
      }
    });
  }

  function plotData(plot_ind, y_key) {
    if (!plotSelections[plot_ind].selectedCol) return;

    if (!dataFile.columns.includes(y_key)){
      console.log("key "+y_key+" not found")
      return;
    }

    const groups = {};

    let yMax = 0;
    dataFile.parsedRows.forEach((row) => {
      if (row.index === undefined || row[y_key] === undefined || !row.timestamp) return;

      const gpuId = String(row.index).trim();
      const yVal = Number(row[y_key]);
      const xVal = row.timestamp;

      if (Number.isNaN(yVal)) {
        console.error("nan timestamp at row "+row+" and y_key "+y_key);
        return;
      }
      if (!groups[gpuId]) groups[gpuId] = [];

      groups[gpuId].push({ x: xVal, y: yVal });

      if (yVal > yMax) {
        yMax = yVal;
      }
    });

    let traces = [];
    Object.keys(groups).forEach((k) => {
      traces.push({
        x: groups[k].map(p => p.x),
        y: groups[k].map(p => p.y),
        mode: 'lines+markers',
        name: 'GPU ' + k,
        type: 'scatter'
      });
    });

    let yPad = 0.1*yMax
    const layout = {
      title: 'GPU Metrics',
      xaxis: { title: 'Timestamp', type: 'date' },
      yaxis: { title: y_key, range: [-yPad, yMax+yPad] }
    };

    Plotly.newPlot(plotSelections[plot_ind].plot, traces, layout, { responsive: true });
  }
</script>

<style>
  :global(body) {
    font-family: Arial, sans-serif;
    margin: 16px;
  }
  .controls {
    margin-bottom: 12px;
  }
  .small {
    font-size: 0.9em;
    color: #555;
  }
  .plot {
    width: 100%;
    aspect-ratio: 16 / 9;
    height: auto;
  }
.plots-grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 24px;
    margin-top: 24px;
  }
  .plot-panel {
    background: #fafbfc;
    border: 1px solid #e0e0e0;
    border-radius: 8px;
    padding: 12px;
    box-shadow: 0 1px 2px rgba(0,0,0,0.03);
  }
  .plot-controls {
    margin-bottom: 8px;
  }

</style>

<h2>GPU Vision</h2>
<div class="controls">
  <input type="file" accept=".csv,text/csv" on:change={handleFileChange} />
</div>

<div class="plots-grid">
  {#each [0,1,2,3] as i}
    <div class="plot-panel">
      <div class="plot-controls">
        <label class="small">Metric:
          <span class="small"> Choose a CSV with columns: <code>gpu</code>, and a metric column</span>
          <select
            bind:value={plotSelections[i].selectedCol}
            on:change={() => plotData(i, plotSelections[i].selectedCol)}
          >
            {#each dataFile.columns as col}
              <option value={col}>{col}</option>
            {/each}
          </select>
        </label>
      </div>
      <div bind:this={plotSelections[i].plot} class="plot"></div>
    </div>
  {/each}
</div>